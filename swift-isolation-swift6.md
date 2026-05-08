# Swift Isolation di Swift 6
### Panduan Lengkap: Teori, Konsep Dasar, Contoh, dan Real Use Case

> **Target pembaca:** iOS/macOS developer yang sudah familiar dengan Swift concurrency dasar (`async`/`await`) dan ingin memahami isolation model di Swift 6 secara mendalam — termasuk teori di balik setiap keputusan desain.

---

## Daftar Isi

1. [Pengantar: Mengapa Isolation?](#1-pengantar-mengapa-isolation)
2. [Actor dan Isolation Dasar](#2-actor-dan-isolation-dasar)
3. [nonisolated dan isolated Parameter](#3-nonisolated-dan-isolated-parameter)
4. [Sendable — Mengapa Ada dan Cara Kerjanya](#4-sendable--mengapa-ada-dan-cara-kerjanya)
5. [Region-Based Isolation (Swift 6 Baru)](#5-region-based-isolation-swift-6-baru)
6. [Global Actors](#6-global-actors)
7. [Isolation + Async/Await](#7-isolation--asyncawait)
8. [Real Use Cases](#8-real-use-cases)
9. [Migration Guide Swift 5 → Swift 6](#9-migration-guide-swift-5--swift-6)
10. [Troubleshooting & Common Mistakes](#10-troubleshooting--common-mistakes)

---

## 1. Pengantar: Mengapa Isolation?

### 1.1 Masalah Fundamental: Memori Bersama

Komputer modern menjalankan banyak hal bersamaan. CPU punya banyak core, dan setiap core bisa mengeksekusi instruksi secara paralel. Ini bagus untuk performa — tapi menimbulkan masalah serius ketika dua core mengakses **lokasi memori yang sama** secara bersamaan.

Bayangkan dua orang mencoba menulis di papan tulis yang sama pada saat yang sama. Hasilnya bukan gabungan tulisan keduanya — hasilnya adalah coretan yang tidak terbaca.

Di level hardware, ini terjadi karena instruksi seperti `value += 1` sebenarnya bukan satu instruksi atomik — ia adalah tiga langkah:
```
1. LOAD  value dari memori → register CPU
2. ADD   register + 1
3. STORE hasil kembali ke memori
```

Jika dua thread melakukan ini bersamaan:
```
Thread A: LOAD value (dapat: 5)
Thread B: LOAD value (dapat: 5)   ← keduanya baca nilai lama!
Thread A: ADD  5 + 1 = 6
Thread B: ADD  5 + 1 = 6
Thread A: STORE 6
Thread B: STORE 6                 ← increment B menimpa increment A!
```

Hasil akhir: `value = 6`, padahal seharusnya `7`. Ini adalah **data race** — dan ini adalah **undefined behavior** di Swift (dan C/C++/Rust). Artinya compiler bebas menghasilkan output apa saja, termasuk crash, nilai salah, atau kerusakan memori yang baru muncul jauh kemudian.

### 1.2 Solusi Tradisional dan Masalahnya

Sebelum Swift 6, developer mengandalkan beberapa pendekatan:

**DispatchQueue (Serial Queue)**
```swift
// "Semua akses ke data ini harus lewat queue ini"
let queue = DispatchQueue(label: "com.app.counter")
var value = 0

queue.async { value += 1 }  // thread-safe
queue.async { value += 1 }  // thread-safe
```
Masalah: Tidak ada yang mencegah developer secara tidak sengaja mengakses `value` langsung tanpa queue. Compiler tidak tahu tentang aturan ini — itu ada di kepala developer saja.

**NSLock / os_unfair_lock**
```swift
let lock = NSLock()
var value = 0

lock.lock()
value += 1
lock.unlock()
```
Masalah: Mudah lupa unlock (→ deadlock), tidak bisa dikomposisi (lock di dalam lock → deadlock), dan tidak ada pemeriksaan compiler.

**Keduanya adalah konvensi, bukan kontrak.** Compiler tidak bisa memverifikasi bahwa kamu mengikuti aturan. Satu kesalahan programmer cukup untuk merusak segalanya.

### 1.3 Filosofi Swift 6: "If It Compiles, It's Safe"

Swift 6 mengambil pendekatan berbeda: **encode aturan keamanan ke dalam type system**. Jika kodenya compile, maka secara definisi tidak ada data race — karena compiler yang memverifikasinya, bukan manusia.

```swift
// Swift 5 — tidak ada compile-time check, ini bisa crash di runtime
class Counter {
    var value = 0
    
    func increment() {
        value += 1  // DANGER: tidak thread-safe
    }
}

let counter = Counter()
Task { counter.increment() }
Task { counter.increment() }  // data race! — hanya ketahuan saat runtime crash
```

```swift
// Swift 6 — ini adalah COMPILE ERROR, bukan runtime crash
// error: sending 'counter' risks causing data races
```

Pergeseran ini fundamental: dari "runtime safety" ke **"compile-time safety"**.

### 1.4 Isolation = Kepemilikan Data yang Jelas

**Isolation** adalah kontrak: *"data ini hanya boleh diakses dari satu domain pada satu waktu."*

Analoginya: bayangkan sebuah kantor dengan ruangan-ruangan terpisah. Dokumen yang ada di ruangan A hanya bisa diakses oleh orang yang berada di ruangan A. Untuk mengakses dokumen dari ruangan B, kamu harus mengirim pesan ke ruangan A dan menunggu balasan — kamu tidak bisa langsung masuk.

Domain isolasi di Swift:
- **`actor` instance** — setiap actor adalah "ruangan" sendiri
- **`@MainActor`** — ruangan khusus: main thread
- **Custom global actor** — ruangan bersama yang diberi nama
- **Nonisolated** — di luar ruangan mana pun, hanya bisa akses data yang aman dibagi

| | Swift 5 | Swift 6 |
|---|---|---|
| Data race check | Runtime (crash tidak terduga) | Compile-time (error jelas) |
| Concurrency warning | Opt-in | Error by default |
| Isolation enforcement | Manual (konvensi) | Type system (compiler) |
| Keterbacaan aturan | Di komentar / dokumen | Di kode itu sendiri |

### 1.5 Kapan Isolation Diperlukan dan Kapan Tidak

**Isolation diperlukan ketika:**
- Kamu punya mutable state yang bisa diakses dari beberapa concurrent contexts
- Kamu menggunakan async/await dan melewatkan data antar tasks
- Kamu menggunakan background processing + UI update

**Isolation tidak diperlukan ketika:**
- Semua state adalah immutable (`let`) — tidak bisa di-mutate, tidak ada race
- Kamu menggunakan value types yang di-copy saat diteruskan — setiap Task punya salinannya sendiri
- Kamu sudah menggunakan mekanisme lain yang terbukti aman (misalnya atomic operations untuk tipe primitif sederhana)

```swift
// TIDAK perlu isolation: value type + immutable
let config = AppConfig(timeout: 30, retries: 3)  // struct, let
Task { process(config) }  // Task punya salinannya sendiri — aman
Task { process(config) }  // aman juga

// PERLU isolation: reference type + mutable
class AppState {
    var currentUser: User?  // shared reference, bisa di-mutate
}
```

---

## 2. Actor dan Isolation Dasar

### 2.1 Sejarah dan Teori: Actor Model

**Actor model** bukan konsep baru — ia dicetuskan oleh Carl Hewitt pada tahun 1973 sebagai model komputasi untuk sistem concurrent. Prinsipnya sederhana:

1. Setiap "actor" adalah entitas independen dengan state privatnya sendiri
2. Actors berkomunikasi **hanya melalui pesan** — tidak ada shared state
3. Setiap actor memproses satu pesan pada satu waktu (secara serial)

Bahasa seperti Erlang dan Elixir membangun seluruh sistem di atas actor model — itulah mengapa sistem telecom yang menggunakan Erlang bisa mencapai "nine nines" availability (99.9999999% uptime).

Swift mengadopsi versi yang lebih pragmatis dari actor model: Swift actor tetap bisa memanggil fungsi async eksternal (sehingga bisa menunggu I/O tanpa memblokir), tapi state internalnya tetap serial dan terlindungi.

### 2.2 Cara Kerja Actor di Bawah Hood

Setiap `actor` di Swift memiliki sebuah **executor** — bayangkan ini sebagai "message queue" yang sangat efisien. Ketika kamu memanggil method actor dari luar, Swift tidak langsung mengeksekusinya — ia mengirim "pesan" (function call) ke queue actor tersebut.

```
Thread A: await counter.increment()
          │
          └─── Swift runtime: tambahkan 'increment' ke antrian Counter
                               Thread A suspend (tidak block!)
          
Thread B: ... mengerjakan hal lain ...

Counter executor: proses 'increment' dari antrian → value += 1
                  kirim hasil kembali ke Thread A

Thread A: resume, lanjutkan eksekusi
```

Ini berbeda dari mutex/lock:
- **Lock**: Thread A memblokir (busy-wait atau OS-wait) sambil menunggu lock bebas
- **Actor**: Thread A di-*suspend* (kooperatif) — thread tersebut bebas mengerjakan Task lain selagi menunggu

Ini adalah cooperative concurrency, bukan preemptive locking. Jauh lebih efisien.

### 2.3 `actor` vs `class`: Kapan Memilih Mana?

```
Gunakan `actor` jika:                    Gunakan `class` jika:
┌──────────────────────────────────┐     ┌──────────────────────────────────┐
│ Punya mutable state              │     │ Selalu @MainActor (UI layer)      │
│ Diakses dari concurrent contexts │     │ Perlu inheritance (subclassing)   │
│ Butuh thread-safety otomatis     │     │ Dipanggil hanya dari satu thread  │
│ Contoh: cache, data store,       │     │ Contoh: ViewController,           │
│         network layer, counter   │     │         ViewModel (@MainActor)    │
└──────────────────────────────────┘     └──────────────────────────────────┘
```

**Kapan TIDAK menggunakan actor:**
- Ketika datamu adalah value type (struct/enum) — tidak perlu isolation karena copy semantics
- Ketika semua akses sudah terjamin di satu thread (misalnya @MainActor)
- Ketika performance sangat kritis dan overhead actor hop tidak dapat diterima (misalnya tight loop numerik)
- Ketika kamu butuh inheritance — actor tidak bisa di-subclass

### 2.4 Konsep

`actor` adalah reference type seperti `class`, tapi semua stored property-nya **diisolasi secara otomatis** — hanya bisa diakses dari dalam actor itu sendiri tanpa `await`.

```swift
actor Counter {
    private(set) var value = 0
    
    func increment() {
        value += 1  // OK: berada di dalam isolation domain actor ini
    }
    
    func reset() {
        value = 0   // OK: sama
    }
}
```

### 2.5 Mengakses dari Luar Actor

Akses dari luar harus menggunakan `await` karena melewati isolation boundary — ini adalah "hop" ke executor actor:

```swift
let counter = Counter()

Task {
    await counter.increment()       // hop ke Counter executor, lalu kembali
    let v = await counter.value     // hop lagi untuk baca property
    print(v)
}
```

Setiap `await` adalah titik suspension potensial. Thread kamu dibebaskan untuk mengerjakan hal lain selagi menunggu actor.

### 2.6 Actor Reentrancy — Konsep Penting yang Sering Disalahpahami

Actor bersifat **reentrant** — ini keputusan desain yang disengaja, bukan bug.

**Mengapa reentrant?** Bayangkan actor A menunggu actor B, dan actor B menunggu actor A — jika actor tidak reentrant, ini adalah deadlock permanen. Reentrancy mencegah deadlock dengan memperbolehkan actor menerima pesan lain selagi menunggu `await`.

**Trade-off:** State bisa berubah di titik suspension.

```swift
actor BankAccount {
    var balance: Double = 1000
    
    func withdraw(amount: Double) async {
        guard balance >= amount else { return }
        
        // ⚠️ TITIK SUSPENSION — actor bisa terima pesan lain di sini
        await processPayment(amount: amount)
        
        // balance MUNGKIN sudah berubah saat kita kembali!
        // Jika ada dua concurrent withdraw(500), keduanya bisa lolos guard di atas
        // karena keduanya membaca balance = 1000 sebelum suspension
        balance -= amount  // bisa jadi negatif!
    }
}
```

**Aturan:** Selalu periksa ulang asumsi setelah setiap `await`, atau lakukan mutasi state **sebelum** `await`:

```swift
actor BankAccount {
    var balance: Double = 1000
    
    func withdraw(amount: Double) async -> Bool {
        // Kurangi balance SEBELUM await — tidak ada window untuk race
        guard balance >= amount else { return false }
        balance -= amount  // atomic dalam konteks actor
        
        // Bahkan jika ada concurrent withdraw lain masuk sini,
        // balance sudah dikurangi — mereka akan gagal di guard di atas
        await processPayment(amount: amount)
        return true
    }
    
    private func processPayment(amount: Double) async {
        try? await Task.sleep(for: .seconds(1))
    }
}
```

---

## 3. nonisolated dan isolated Parameter

### 3.1 Teori: Biaya Isolation

Setiap akses ke isolated property atau method membutuhkan **actor hop** — sebuah context switch ke executor actor. Ini memang lebih efisien dari thread switch konvensional (karena cooperative), tapi tetap ada overhead: bookkeeping, queue operation, dan potential task suspension.

Untuk data yang **tidak memerlukan perlindungan** (misalnya immutable constants atau computed properties murni), overhead ini tidak diperlukan. `nonisolated` adalah cara untuk menyatakan: *"bagian ini aman diakses langsung, tanpa perlu melalui antrian actor."*

### 3.2 `nonisolated` — Opt-Out dari Isolation

`nonisolated` memungkinkan method/property diakses tanpa `await` dari mana saja, tapi sebagai konsekuensi **tidak boleh mengakses isolated state** actor:

```swift
actor UserProfile {
    var name: String          // isolated — bisa berubah, butuh perlindungan
    var email: String         // isolated — sama
    let id: UUID              // immutable — tidak bisa berubah, tidak butuh perlindungan
    
    init(name: String, email: String) {
        self.name = name
        self.email = email
        self.id = UUID()
    }
    
    // Bisa dipanggil tanpa await dari mana saja
    // Tidak ada overhead actor hop — langsung eksekusi
    nonisolated func displayID() -> String {
        return id.uuidString  // OK: id adalah let (immutable)
    }
    
    // ❌ ERROR: name adalah var — bisa berubah, tidak aman tanpa isolation
    // nonisolated func displayName() -> String { name }
}

// Penggunaan:
let profile = UserProfile(name: "Ari", email: "ari@example.com")
let idString = profile.displayID()  // synchronous! tidak perlu await
```

**Kapan gunakan `nonisolated`:**
- Method hanya mengakses `let` property (immutable)
- Method hanya mengakses property lain yang juga `nonisolated`
- Pure functions / computed properties tanpa mutable state
- Protocol conformances yang harus synchronous (misalnya `CustomStringConvertible`)

**Kapan TIDAK gunakan `nonisolated`:**
- Jika method mengakses `var` property — compiler akan error
- Jika method memanggil isolated method lain — butuh await
- Jika ada keraguan — biarkan actor melindunginya

```swift
actor Logger {
    private var logs: [String] = []
    private let prefix: String  // immutable
    
    // nonisolated OK: hanya akses let
    nonisolated var description: String {
        "Logger[\(prefix)]"  // CustomStringConvertible tanpa await
    }
    
    // Harus isolated: mutasi logs
    func log(_ message: String) {
        logs.append("[\(prefix)] \(message)")
    }
    
    // nonisolated OK karena hanya akses let + pure computation
    nonisolated func format(_ message: String) -> String {
        "[\(prefix)] \(message)"
    }
}
```

### 3.3 `isolated` Parameter — Utility Function dalam Domain Actor

`isolated` parameter memungkinkan fungsi bebas (bukan method actor) untuk berjalan **di dalam isolation domain** actor tertentu:

```swift
actor Database {
    var records: [String: String] = [:]
}

// Fungsi ini bukan method Database, tapi berjalan DALAM isolation domain 'db'
// Artinya: tidak butuh await untuk akses db.records
func insertRecord(key: String, value: String, into db: isolated Database) {
    db.records[key] = value  // OK: kita sedang dalam isolation domain db
}

// Penggunaan — butuh await karena harus hop ke Database executor
let db = Database()
Task {
    await insertRecord(key: "user_1", value: "Ari", into: db)
}
```

**Mengapa ini berguna?** Bayangkan kamu punya banyak utility functions yang logikanya terkait erat dengan sebuah actor, tapi tidak ingin membuat actor membengkak dengan banyak method. `isolated` parameter memisahkan concerns sambil tetap mendapat keamanan isolation.

**Perbandingan pendekatan:**
```swift
// Pendekatan 1: method di dalam actor (meningkatkan ukuran actor)
actor DataProcessor {
    var data: [Int] = []
    func filterEven() { data = data.filter { $0.isMultiple(of: 2) } }
    func filterOdd()  { data = data.filter { !$0.isMultiple(of: 2) } }
    func sort()       { data.sort() }
    // ... actor bisa jadi sangat besar
}

// Pendekatan 2: isolated parameter (actor tetap kecil, logic di luar)
actor DataProcessor {
    var data: [Int] = []
}

func filterEven(processor: isolated DataProcessor) {
    processor.data = processor.data.filter { $0.isMultiple(of: 2) }
}

func sort(processor: isolated DataProcessor) {
    processor.data.sort()
}
```

**Kapan gunakan `isolated` parameter:**
- Utility functions yang logikanya terkait satu actor tapi bukan bagian inti actor
- Extension functions yang perlu akses isolated state
- Testing helpers yang perlu inspect/modify actor state

**Kapan TIDAK gunakan `isolated` parameter:**
- Jika function butuh akses ke dua actor berbeda — tidak bisa punya dua `isolated` parameter dari actor berbeda sekaligus (deadlock potential)
- Jika logic memang bagian inti dari actor — jadikan method biasa saja

---

## 4. Sendable — Mengapa Ada dan Cara Kerjanya

### 4.1 Akar Masalah: Value Semantics vs Reference Semantics

Ini adalah inti dari seluruh konsep Sendable. Pahami ini, dan selebihnya akan masuk akal sendiri.

**Value Semantics (struct, enum, tuple)**

Ketika kamu meng-assign atau meneruskan value type, Swift membuat **salinan independen**:

```swift
var a = Point(x: 1, y: 2)
var b = a           // b adalah SALINAN — alamat memori berbeda
b.x = 10

print(a.x)  // masih 1 — a tidak terpengaruh
print(b.x)  // 10
```

Karena setiap Task/thread punya salinannya sendiri, tidak ada yang "berbagi" memori. Tidak ada yang bisa race condition — tidak ada shared state. Ini kenapa `struct` dan `enum` secara alami **thread-safe**.

**Reference Semantics (class)**

Ketika kamu meng-assign atau meneruskan reference type, kamu meneruskan **pointer ke objek yang sama**:

```swift
class Counter { var count = 0 }

let a = Counter()
let b = a           // b menunjuk ke OBJEK YANG SAMA — alamat memori sama!
b.count = 10

print(a.count)  // 10 — a berubah! karena a dan b adalah objek yang sama
```

Sekarang bayangkan ini terjadi lintas thread:

```swift
let counter = Counter()  // satu objek di heap

Task {
    counter.count += 1   // Thread 1 menulis
}
Task {
    counter.count += 1   // Thread 2 juga menulis ke lokasi yang SAMA
}
// → Data race!
```

**Inilah masalah mendasarnya:** Reference semantics membuat aliasing — beberapa variabel menunjuk ke satu lokasi memori — dan ketika ada mutation yang terjadi concurrent, terjadilah data race.

### 4.2 Mengapa Swift Membutuhkan `Sendable`

Swift butuh cara untuk menyatakan ke compiler: *"nilai ini aman untuk dikirim ke isolation domain lain."*

Dalam dunia actor, "mengirim nilai ke isolation domain lain" artinya:
- Melewatkan argumen ke method actor dari luar actor
- Mengembalikan nilai dari actor ke caller
- Memasukkan nilai ke dalam `Task {}`
- Meneruskan nilai sebagai `@escaping` closure

Jika nilai yang dikirim adalah sebuah `class` (reference type) yang memiliki mutable state, maka setelah pengiriman, **dua isolation domain berbeda memegang referensi ke objek yang sama** — dan keduanya bisa mutasi objek itu concurrently. Data race.

`Sendable` adalah **promise ke compiler**: "nilai bertipe ini tidak akan menyebabkan data race ketika dikirim lintas isolation boundary."

```
Tanpa Sendable:           Dengan Sendable:
                          
Actor A ──── class ────▶ Actor B    Actor A ──── struct ────▶ Actor B
              │                                   (copy)
              ▼                     
         [shared object]            Actor A: punya salinannya
         ↑           ↑             Actor B: punya salinannya
    Actor A      Actor B           Tidak ada aliasing → tidak ada race
    (kedua)
    bisa mutasi → RACE!
```

### 4.3 Apa yang Membuat Sebuah Tipe "Sendable"?

Sebuah tipe aman untuk di-send jika dan hanya jika kita bisa menjamin salah satu dari:

1. **Immutable** — tidak bisa di-mutate, tidak ada race (misalnya `let` properties saja)
2. **Copied** — value semantics, setiap domain punya salinannya sendiri
3. **Internally synchronized** — punya mekanisme internal untuk mencegah concurrent mutation (misalnya actor, lock)
4. **Never shared** — tidak mungkin diakses concurrent (misalnya value yang hanya hidup dalam satu Task)

Swift mengkodekan ini ke dalam beberapa kategori:

### 4.4 Kategori Sendable dan Kapan Digunakan

**Kategori 1: Automatic Sendable (tidak perlu deklarasi)**

```swift
// Semua ini otomatis Sendable karena Swift bisa membuktikannya:

// Primitif: immutable dan copied
let n: Int = 42           // Sendable
let s: String = "hello"   // Sendable
let b: Bool = true        // Sendable

// Struct: value semantics — setiap assignment adalah copy
struct Point { var x: Double; var y: Double }
// Sendable otomatis karena:
// - x adalah Double (Sendable)
// - y adalah Double (Sendable)
// - struct memberikan copy semantics
// → tidak ada aliasing yang mungkin

// Enum: sama seperti struct
enum Status { case active, inactive, pending }  // Sendable otomatis

// Collection dari Sendable juga Sendable
let numbers: [Int] = [1, 2, 3]       // Array<Int> adalah Sendable
let dict: [String: Int] = [:]         // Dictionary<String, Int> adalah Sendable
```

**Kategori 2: Explicit Sendable untuk Struct/Class yang Kompleks**

```swift
// Struct dengan semua Sendable properties → otomatis, tapi bisa eksplisit
struct Message: Sendable {
    let id: UUID        // Sendable
    let content: String // Sendable
    let timestamp: Date // Sendable
}

// Actor — selalu Sendable (protected by design)
actor DataStore: Sendable {  // 'Sendable' di sini redundant tapi OK
    var items: [String] = []
}

// Final class dengan hanya immutable properties
final class ImmutableConfig: Sendable {
    let timeout: Int      // let — tidak bisa diubah setelah init
    let baseURL: URL      // let — immutable
    
    init(timeout: Int, baseURL: URL) {
        self.timeout = timeout
        self.baseURL = baseURL
    }
}

// ❌ Ini TIDAK bisa Sendable:
// final class MutableConfig: Sendable {  // ERROR
//     var timeout: Int  // var + class = bisa concurrent mutation
// }
```

**Kategori 3: @unchecked Sendable — Escape Hatch yang Berbahaya**

```swift
// @unchecked Sendable: "saya tahu ini aman, tapi compiler tidak bisa membuktikannya"
// Gunakan HANYA ketika kamu punya mekanisme synchronization manual yang benar

final class ThreadSafeCache: @unchecked Sendable {
    private var store: [String: Any] = [:]
    private let lock = NSLock()            // manual synchronization
    
    func set(_ value: Any, for key: String) {
        lock.withLock { store[key] = value }  // setiap akses dilindungi lock
    }
    
    func get(for key: String) -> Any? {
        lock.withLock { store[key] }
    }
}
```

> **Peringatan keras:** `@unchecked Sendable` menonaktifkan SEMUA pemeriksaan compiler untuk tipe ini. Kesalahan kecil di dalam implementasi (lupa lock satu akses) bisa menyebabkan data race yang sulit dideteksi. Gunakan sebagai last resort, dan dokumentasikan dengan jelas mengapa aman.

**Kapan terpaksa `@unchecked Sendable`:**
- Wrapping library C/Objective-C yang thread-safe tapi tidak diketahui compiler
- Legacy code yang sudah terbukti aman secara audit tapi tidak bisa direfactor sekarang
- Tipe dengan invariant keamanan yang tidak bisa diekspresikan dalam type system

**Alternatif yang lebih baik sebelum `@unchecked Sendable`:**
```swift
// Opsi 1: jadikan actor — compiler membuktikan keamanannya
actor Cache {
    private var store: [String: Any] = [:]
    func set(_ value: Any, for key: String) { store[key] = value }
    func get(for key: String) -> Any? { store[key] }
}

// Opsi 2: jadikan struct (jika value semantics cukup)
struct Configuration: Sendable {
    var timeout: Int = 30
    // struct copy semantics → tidak ada shared mutable state
}
```

### 4.5 @Sendable Closure — Mengapa Closure Spesial

Closure memiliki kemampuan untuk **capture** variable dari lingkup sekitarnya. Ini yang membuatnya berbeda dari function biasa:

```swift
var counter = 0

let closure = {
    counter += 1  // closure ini "capture" counter
}
```

Jika closure ini dikirim ke thread lain, dan `counter` juga diakses dari thread asal, terjadilah data race pada `counter`.

`@Sendable` closure berjanji: closure ini tidak menyimpan reference ke mutable state yang bisa di-race:

```swift
// ❌ Tidak bisa @Sendable: meng-capture mutable var
var mutableState = 0
Task { @Sendable in
    mutableState += 1  // ERROR: capture of 'mutableState' with non-sendable type
}

// ✅ @Sendable OK: capture immutable state
let immutableState = 42
Task { @Sendable in
    print(immutableState)  // OK: immutable, tidak ada race
}

// ✅ @Sendable OK: capture Sendable reference (actor)
actor Store { var items: [String] = [] }
let store = Store()
Task { @Sendable in
    await store.items.append("new")  // OK: actor melindungi state-nya
}
```

**Kapan butuh @Sendable closure:**
- `Task { }` — selalu @Sendable secara implisit
- `Task.detached { }` — selalu @Sendable
- `@escaping` closure yang dikirim ke thread lain
- Argumen ke concurrent functions

### 4.6 Sendable di Protocol — Implikasi Penting

```swift
// Protocol Sendable: semua conforming type harus Sendable
protocol DataProcessor: Sendable {
    func process(_ data: Data) -> Result<String, Error>
}

// Ini berguna ketika protocol digunakan sebagai existential dalam actor
actor Pipeline {
    var processors: [any DataProcessor] = []  // OK: DataProcessor adalah Sendable
    
    func addProcessor(_ processor: any DataProcessor) {
        processors.append(processor)  // aman: processor adalah Sendable
    }
}
```

**Kapan protocol perlu Sendable:**
- Ketika existential (`any Protocol`) akan disimpan di actor sebagai property
- Ketika implementasi protocol akan dikirim lintas isolation boundary

---

## 5. Region-Based Isolation (Swift 6 Baru)

### 5.1 Masalah yang Dipecahkan

Sebelum Swift 6, aturannya binary: sebuah nilai **harus Sendable** untuk dikirim ke isolation domain lain. Ini terlalu ketat dalam banyak kasus.

Pertimbangkan:
```swift
func buildRequest() -> URLRequest {
    var request = URLRequest(url: someURL)
    request.httpMethod = "POST"
    request.httpBody = someData
    return request
}

// URLRequest bukan Sendable (di Swift 5.x)
// Tapi kita tahu bahwa setelah kita "kirim" ke network task,
// kita tidak akan mengakses request itu lagi!

// Swift 5.x: ERROR — URLRequest tidak Sendable
Task { try await URLSession.shared.data(for: buildRequest()) }
```

Aturan Sendable terlalu keras di sini karena kita tahu secara statik bahwa tidak ada concurrent access — kita tidak menyimpan referensi lain ke request itu.

### 5.2 Teori: Isolation Regions

Swift 6 memperkenalkan analisis **region-based isolation** — compiler melacak "region" atau "kelompok kepemilikan" setiap nilai, bukan hanya tipenya.

Sebuah **isolation region** adalah sekumpulan nilai yang saling terhubung melalui referensi. Jika dua nilai bisa saling menjangkau satu sama lain (misalnya object graph), mereka berada dalam region yang sama.

```swift
class Node {
    var next: Node?
    var value: Int
}

var head = Node(value: 1)         // head
head.next = Node(value: 2)        // head.next
head.next?.next = Node(value: 3)  // head.next.next

// Ketiga node ini berada dalam SATU region
// karena mereka terhubung melalui referensi 'next'
```

**Ide kunci:** Jika sebuah region **tidak lagi dapat diakses** dari konteks asal setelah "dikirim", maka aman untuk dikirim ke isolation domain lain — meskipun nilainya tidak Sendable.

### 5.3 `sending` Keyword

`sending` parameter menandai bahwa nilai **ditransfer** — kepemilikannya berpindah ke callee. Setelah transfer, caller tidak boleh mengakses nilai tersebut lagi:

```swift
struct LargeModel {   // bukan Sendable
    var items: [String] = []
    var metadata: [String: Any] = [:]
}

// 'sending' = "saya mengambil alih kepemilikan model ini"
func processInBackground(_ model: sending LargeModel) async {
    // model sekarang milik fungsi ini
    // caller tidak bisa akses model setelah memanggil ini
    var processed = model
    processed.items.sort()
    await persist(processed)
}

// Penggunaan:
var model = LargeModel()
model.items = ["c", "a", "b"]

await processInBackground(model)  // transfer terjadi di sini

// ❌ ERROR: compiler tahu 'model' sudah di-transfer
// print(model.items)  // 'model' digunakan setelah dikirim ke isolation lain
```

**Analogi:** Ini seperti `move` semantics di C++ atau `move` di Rust. Nilai "dipindahkan", bukan "disalin" atau "dibagi".

**Kapan gunakan `sending`:**
- Fungsi yang mengkonsumsi nilai dan tidak perlu dikembalikan
- Pipeline processing: data masuk, diproses, tidak perlu kembali
- Ketika kamu ingin izinkan tipe non-Sendable melewati isolation boundary secara satu arah

**Kapan TIDAK gunakan `sending`:**
- Ketika caller masih butuh akses ke nilai setelah call
- Ketika nilai perlu dibaca dari beberapa tempat — gunakan Sendable struct sebagai gantinya

### 5.4 Mengapa Ini Penting di Praktik

```swift
// Swift 5.9 → Swift 6: Lebih banyak kode yang compile tanpa perubahan

// Sebelumnya butuh workaround
func sendRequest(_ request: URLRequest) async throws -> Data {
    let (data, _) = try await URLSession.shared.data(for: request)
    return data
}

// Swift 6: 'sending' memungkinkan ini secara eksplisit
func sendRequest(_ request: sending URLRequest) async throws -> Data {
    let (data, _) = try await URLSession.shared.data(for: request)
    return data
}
```

Region-based isolation adalah tentang memberi compiler informasi yang lebih kaya tentang **kapan** nilai bisa diakses, bukan hanya **bagaimana** nilai itu direpresentasikan di memori.

---

## 6. Global Actors

### 6.1 Teori: Singleton Isolation Domain

Actor biasa (`actor Foo {}`) membuat isolation domain per-instance — setiap instance `Foo` adalah domain yang berbeda. Dua instance tidak berbagi domain:

```swift
actor Counter {
    var count = 0
}

let c1 = Counter()  // isolation domain #1
let c2 = Counter()  // isolation domain #2 — terpisah dari #1
```

Terkadang kita butuh **satu domain yang digunakan bersama** oleh banyak tipe. Misalnya: semua UIKit calls harus di satu thread (main thread). Kita tidak ingin setiap ViewController punya actor-nya sendiri — kita ingin semua berbagi satu domain: "main thread".

**Global Actor** adalah actor singleton yang bisa digunakan sebagai **atribut** (`@MainActor`, `@DatabaseActor`, dll) untuk menyatakan bahwa sebuah tipe atau method berbagi domain tersebut.

### 6.2 `@MainActor` — Global Actor Paling Penting

`@MainActor` mewakili main thread. Ini adalah global actor bawaan Swift karena kebutuhan ini sangat universal dalam pengembangan aplikasi.

**Mengapa semua UI harus di main thread?**
- UIKit dan AppKit tidak thread-safe — mereka tidak dirancang untuk concurrent access
- Main thread adalah satu-satunya thread yang berkomunikasi dengan window server (macOS/iOS)
- Animasi, layout, dan rendering semua dikoordinasikan melalui run loop di main thread

```swift
@MainActor
class ViewModel: ObservableObject {
    @Published var title: String = ""
    @Published var isLoading: Bool = false
    
    // Semua property dan method di sini dijamin di main thread
    func loadData() async {
        isLoading = true  // OK: @MainActor, pasti di main thread
        
        // fetchFromNetwork() tidak @MainActor → berjalan di background
        let result = await fetchFromNetwork()
        
        // Setelah await, kita kembali ke @MainActor context
        title = result   // OK: kembali di main thread
        isLoading = false
    }
    
    private func fetchFromNetwork() async -> String {
        // Tidak ada @MainActor annotation → background thread
        try? await Task.sleep(for: .seconds(1))
        return "Data loaded"
    }
}
```

**Kapan gunakan `@MainActor`:**
- `ViewController` dan subclassnya — selalu
- `ViewModel` yang mengelola `@Published` properties — selalu
- Delegate methods yang update UI
- Callback dari timer/notification yang perlu update UI

**Kapan TIDAK gunakan `@MainActor`:**
- Business logic yang tidak menyentuh UI — ini akan memblokir/mengganggu responsivitas UI
- Network layer, database layer, parsing logic — semua ini lebih baik di background actor
- Fungsi komputasi berat — bisa menyebabkan UI freeze

### 6.3 Custom Global Actor

```swift
// Definisi global actor — satu per domain yang diinginkan
@globalActor
actor NetworkActor {
    static let shared = NetworkActor()  // WAJIB: singleton
}

// Semua tipe yang dianotasi @NetworkActor berbagi isolation domain yang sama
@NetworkActor
class NetworkManager {
    var activeRequests: [UUID: URLSessionTask] = [:]
    
    func send(_ request: URLRequest) async throws -> Data {
        let task = URLSession.shared.dataTask(with: request)
        return Data()
    }
}

@NetworkActor
class RequestInterceptor {
    // Ini berbagi domain dengan NetworkManager
    // Akses ke NetworkManager dari sini tidak butuh await!
}

@NetworkActor
func cancelAllRequests(manager: NetworkManager) {
    // Berada di NetworkActor domain — bisa akses manager.activeRequests langsung
    manager.activeRequests.values.forEach { $0.cancel() }
    manager.activeRequests.removeAll()
}
```

### 6.4 Global Actor vs Regular Actor: Kapan Memilih Mana?

```
Gunakan Global Actor (@DatabaseActor dll) jika:
┌──────────────────────────────────────────────────────┐
│ Banyak class/struct berbagi isolation domain yang sama│
│ Kamu ingin anotasi atribut (@DatabaseActor)           │
│ Domain punya nama konseptual yang jelas               │
│ Contoh: semua database access, semua network calls    │
└──────────────────────────────────────────────────────┘

Gunakan Regular Actor jika:
┌──────────────────────────────────────────────────────┐
│ Satu instance dengan state yang perlu dilindungi      │
│ Tidak ada kebutuhan berbagi domain dengan tipe lain   │
│ Contoh: cache instance, counter, data store           │
└──────────────────────────────────────────────────────┘
```

**Bahaya Global Actor:** Jika terlalu banyak tipe diisolasi ke satu global actor, kamu menciptakan bottleneck — semua operasi yang diisolasi harus antri di satu executor. Untuk throughput tinggi, lebih baik gunakan banyak actor instance terpisah.

### 6.5 Global Actor pada Partial Class/Struct

```swift
@globalActor
actor DatabaseActor {
    static let shared = DatabaseActor()
}

// Hanya method tertentu yang diisolasi ke DatabaseActor
class DataRepository {
    @DatabaseActor
    func fetchUser(id: String) async -> User? {
        // berjalan di DatabaseActor
        return nil
    }
    
    @DatabaseActor
    var cachedUsers: [User] = []
    
    // Method ini berjalan di caller's context — tidak diisolasi
    func prepareQuery(id: String) -> String {
        return "SELECT * WHERE id = '\(id)'"
    }
}
```

---

## 7. Isolation + Async/Await

### 7.1 Teori: Executor dan Context Switching

Swift's async/await tidak menggunakan thread-per-task seperti Grand Central Dispatch. Sebaliknya, ia menggunakan **cooperative multitasking** dengan **executor**:

- Setiap isolation domain (actor) punya **executor** (mirip serial queue, tapi lebih efisien)
- Sebuah **Task** berjalan pada executor tertentu pada satu waktu
- Ketika sebuah Task melakukan `await`, ia **melepaskan** executor-nya (bukan memblokir thread)
- Thread yang dibebaskan bisa langsung mengerjakan Task lain
- Ketika `await` selesai, Task dijadwalkan ulang ke executor yang sesuai

```
Thread Pool (misalnya 4 core):
┌────────┬────────┬────────┬────────┐
│Thread 1│Thread 2│Thread 3│Thread 4│
└───┬────┴───┬────┴───┬────┴────────┘
    │        │        │
    │ Task A │ Task B │ Task C
    │ (@Main)│ (actor)│ (nonisolated)
    │        │        │
    ▼await   │        │
    [suspend]│        │         ← Thread 1 bebas!
             │        │
    Thread 1 mengerjakan Task D sementara Task A menunggu
```

Ini jauh lebih efisien dari "satu thread per request" karena satu thread bisa mengerjakan ratusan Tasks yang saling bergantian.

### 7.2 Isolation Propagation

Ketika sebuah `async` function dipanggil, ia **mewarisi isolation** dari caller-nya — kecuali function itu sendiri punya anotasi isolation berbeda:

```swift
@MainActor
func updateUI(with text: String) async {
    label.text = text           // di main thread (diwarisi dari @MainActor)
    
    let processed = await processText(text)  // hop ke nonisolated context
    
    label.text = processed      // kembali ke main thread (kembali ke @MainActor)
}

// Fungsi ini TIDAK diisolasi — berjalan di background cooperative pool
func processText(_ text: String) async -> String {
    // heavy processing di background
    return text.uppercased()
}
```

**Aturan propagasi:**
1. Jika caller `@MainActor` dan callee tidak punya anotasi → callee berjalan di background, caller `await` dan kembali ke main
2. Jika caller adalah `actor` dan callee adalah `actor` lain → hop terjadi, butuh `await`
3. Jika caller dan callee sama isolation domain → tidak ada hop, tidak ada overhead

### 7.3 Task dan Isolation

`Task` mewarisi isolation dari konteks pembuatannya:

```swift
@MainActor
class ViewController: UIViewController {
    override func viewDidLoad() {
        super.viewDidLoad()
        
        // Task ini MEWARISI @MainActor isolation dari ViewController
        Task {
            // Berada di @MainActor — aman untuk UI update
            let data = await fetchData()  // hop ke background sementara
            updateUI(with: data)          // kembali ke @MainActor — aman
        }
        
        // Task.detached TIDAK mewarisi isolation — berjalan di pool umum
        Task.detached {
            // BUKAN @MainActor — tidak aman untuk UI update langsung
            let data = await self.fetchData()
            await MainActor.run {
                self.updateUI(with: data)  // harus eksplisit
            }
        }
    }
}
```

**Gunakan `Task.detached` hanya ketika:**
- Kamu secara eksplisit tidak ingin mewarisi isolation parent
- Task benar-benar independent dan tidak boleh terpengaruh cancellation dari parent
- Kamu ingin prioritas eksekusi berbeda dari parent

**Default: gunakan `Task {}` biasa** — lebih aman, kode lebih bersih.

### 7.4 Melintasi Isolation Boundary

Setiap crossing isolation boundary memerlukan `await`:

```swift
actor DataStore {
    var items: [String] = []
    
    func addItem(_ item: String) {
        items.append(item)
    }
}

@MainActor
func addAndDisplay(item: String, to store: DataStore) async {
    // Crossing: @MainActor → DataStore actor
    await store.addItem(item)            // hop diperlukan
    
    let count = await store.items.count  // hop lagi
    label.text = "Total: \(count)"       // kembali ke @MainActor
}
```

**Visualisasi boundary crossing:**
```
@MainActor                    DataStore actor
─────────────────────         ──────────────────
addAndDisplay() starts
                   ─ await store.addItem() ─▶  addItem() runs
(suspended)                                     items.append()
                   ◀─────────────────────── done
resumes
label.text = ...
```

### 7.5 `withIsolation` dan `assumeIsolated`

```swift
// assumeIsolated: "saya yakin kita sudah di isolation domain ini"
// Berguna ketika kamu memanggil dari callback yang compiler tidak bisa trace
actor MyActor {
    var value = 0
    
    func doSomething() {
        // Jalankan closure secara synchronous dalam isolation domain ini
        self.assumeIsolated { isolatedSelf in
            isolatedSelf.value += 1  // OK karena kita sudah dalam domain
        }
    }
    
    // Contoh nyata: delegate callback dari library lama
    func setupCallback() {
        SomeLegacyLibrary.onEvent = { [weak self] in
            // Library ini memanggil di thread yang tidak diketahui
            // Kita yakin kita harus handle ini dalam actor
            self?.assumeIsolated { actor in
                actor.handleEvent()
            }
        }
    }
    
    private func handleEvent() {
        value += 1
    }
}
```

**Kapan `assumeIsolated`:**
- Bridging dari callback-based API ke actor
- Ketika kamu yakin (melalui dokumentasi atau code audit) bahwa kamu sudah dalam isolation domain yang benar
- JANGAN gunakan untuk menghindari compiler error jika kamu tidak yakin

---

## 8. Real Use Cases

### Use Case 1: NetworkLayer Actor

```swift
// Production-ready network layer dengan actor isolation

actor NetworkLayer {
    private let session: URLSession
    private var activeTasks: [UUID: Task<Data, Error>] = [:]
    
    init(configuration: URLSessionConfiguration = .default) {
        self.session = URLSession(configuration: configuration)
    }
    
    func fetch(_ request: URLRequest) async throws -> Data {
        let taskID = UUID()
        
        let task = Task<Data, Error> {
            let (data, response) = try await session.data(for: request)
            guard let http = response as? HTTPURLResponse,
                  (200...299).contains(http.statusCode) else {
                throw NetworkError.invalidResponse
            }
            return data
        }
        
        activeTasks[taskID] = task
        defer { activeTasks.removeValue(forKey: taskID) }
        
        return try await task.value
    }
    
    func cancelAll() {
        activeTasks.values.forEach { $0.cancel() }
        activeTasks.removeAll()
    }
    
    var activeRequestCount: Int {
        activeTasks.count
    }
}

enum NetworkError: Error {
    case invalidResponse
}

// Penggunaan
let network = NetworkLayer()

Task {
    let request = URLRequest(url: URL(string: "https://api.example.com/users")!)
    do {
        let data = try await network.fetch(request)
        let users = try JSONDecoder().decode([User].self, from: data)
        await MainActor.run { /* update UI */ }
    } catch {
        print("Error: \(error)")
    }
}
```

---

### Use Case 2: Thread-Safe In-Memory Cache

```swift
actor Cache<Key: Hashable, Value> {
    private var store: [Key: CacheEntry<Value>] = [:]
    private let maxSize: Int
    private let ttl: Duration
    
    struct CacheEntry<V> {
        let value: V
        let expiresAt: ContinuousClock.Instant
    }
    
    init(maxSize: Int = 100, ttl: Duration = .seconds(300)) {
        self.maxSize = maxSize
        self.ttl = ttl
    }
    
    func set(_ value: Value, for key: Key) {
        evictExpiredIfNeeded()
        
        if store.count >= maxSize {
            store.removeFirst()
        }
        
        store[key] = CacheEntry(
            value: value,
            expiresAt: .now + ttl
        )
    }
    
    func get(for key: Key) -> Value? {
        guard let entry = store[key] else { return nil }
        
        if entry.expiresAt < .now {
            store.removeValue(forKey: key)
            return nil
        }
        
        return entry.value
    }
    
    func invalidate(for key: Key) {
        store.removeValue(forKey: key)
    }
    
    func invalidateAll() {
        store.removeAll()
    }
    
    private func evictExpiredIfNeeded() {
        let now = ContinuousClock.now
        store = store.filter { $0.value.expiresAt >= now }
    }
}

// Penggunaan
let userCache = Cache<String, User>(maxSize: 200, ttl: .seconds(600))

Task {
    if let cached = await userCache.get(for: "user_123") {
        print("Cache hit: \(cached.name)")
    } else {
        let user = try await fetchUser(id: "user_123")
        await userCache.set(user, for: "user_123")
    }
}
```

---

### Use Case 3: UI State Management dengan @MainActor

```swift
@MainActor
final class ArticleViewModel: ObservableObject {
    @Published private(set) var articles: [Article] = []
    @Published private(set) var isLoading = false
    @Published private(set) var errorMessage: String?
    
    private let repository: ArticleRepository
    private var loadTask: Task<Void, Never>?
    
    init(repository: ArticleRepository) {
        self.repository = repository
    }
    
    func loadArticles() {
        loadTask?.cancel()
        
        loadTask = Task {
            isLoading = true
            errorMessage = nil
            
            do {
                let fetched = try await repository.fetchAll()
                
                guard !Task.isCancelled else { return }
                
                articles = fetched
            } catch is CancellationError {
                // task dibatalkan — tidak perlu update UI
            } catch {
                errorMessage = error.localizedDescription
            }
            
            isLoading = false
        }
    }
    
    func refresh() async {
        loadTask?.cancel()
        await loadTask?.value
        loadArticles()
    }
    
    deinit {
        loadTask?.cancel()
    }
}

actor ArticleRepository {
    private let networkLayer: NetworkLayer
    private let cache: Cache<String, [Article]>
    
    init() {
        self.networkLayer = NetworkLayer()
        self.cache = Cache(ttl: .seconds(60))
    }
    
    func fetchAll() async throws -> [Article] {
        if let cached = await cache.get(for: "articles") {
            return cached
        }
        
        let request = URLRequest(url: URL(string: "https://api.example.com/articles")!)
        let data = try await networkLayer.fetch(request)
        let articles = try JSONDecoder().decode([Article].self, from: data)
        
        await cache.set(articles, for: "articles")
        return articles
    }
}

struct Article: Codable, Sendable {
    let id: String
    let title: String
    let body: String
}

struct User: Codable, Sendable {
    let id: String
    let name: String
}
```

---

### Use Case 4: Database Access Actor (Core Data)

```swift
@globalActor
actor CoreDataActor {
    static let shared = CoreDataActor()
}

@CoreDataActor
final class CoreDataStack {
    static let shared = CoreDataStack()
    
    private let container: NSPersistentContainer
    
    private init() {
        container = NSPersistentContainer(name: "AppModel")
        container.loadPersistentStores { _, error in
            if let error { fatalError("Core Data failed: \(error)") }
        }
        container.viewContext.automaticallyMergesChangesFromParent = true
    }
    
    var context: NSManagedObjectContext {
        container.viewContext
    }
    
    func performBackgroundTask<T: Sendable>(
        _ block: @CoreDataActor (NSManagedObjectContext) throws -> T
    ) async throws -> T {
        let backgroundContext = container.newBackgroundContext()
        return try await withCheckedThrowingContinuation { continuation in
            backgroundContext.perform {
                do {
                    let result = try block(backgroundContext)
                    continuation.resume(returning: result)
                } catch {
                    continuation.resume(throwing: error)
                }
            }
        }
    }
    
    func save() throws {
        guard context.hasChanges else { return }
        try context.save()
    }
}

@CoreDataActor
class UserRepository {
    private let stack = CoreDataStack.shared
    
    func createUser(name: String, email: String) throws {
        let user = UserEntity(context: stack.context)
        user.id = UUID()
        user.name = name
        user.email = email
        try stack.save()
    }
    
    func fetchUsers() throws -> [UserEntity] {
        let request = UserEntity.fetchRequest()
        request.sortDescriptors = [NSSortDescriptor(key: "name", ascending: true)]
        return try stack.context.fetch(request)
    }
}

@MainActor
class UserListViewModel: ObservableObject {
    @Published var users: [UserData] = []
    
    private let repository = UserRepository()
    
    func loadUsers() async {
        do {
            let entities = try await repository.fetchUsers()
            users = entities.map { UserData(id: $0.id!, name: $0.name!, email: $0.email!) }
        } catch {
            print("Failed to load users: \(error)")
        }
    }
}

struct UserData: Sendable {
    let id: UUID
    let name: String
    let email: String
}

class UserEntity: NSManagedObject {
    @NSManaged var id: UUID?
    @NSManaged var name: String?
    @NSManaged var email: String?
    
    static func fetchRequest() -> NSFetchRequest<UserEntity> {
        NSFetchRequest<UserEntity>(entityName: "UserEntity")
    }
}
```

---

### Use Case 5: Event Bus dengan AsyncStream

```swift
actor EventBus<Event: Sendable> {
    private var continuations: [UUID: AsyncStream<Event>.Continuation] = [:]
    
    func subscribe() -> (id: UUID, stream: AsyncStream<Event>) {
        let id = UUID()
        let stream = AsyncStream<Event> { [weak self] continuation in
            Task {
                await self?.register(continuation: continuation, id: id)
            }
            continuation.onTermination = { [weak self, id] _ in
                Task { await self?.unsubscribe(id: id) }
            }
        }
        return (id, stream)
    }
    
    func publish(_ event: Event) {
        continuations.values.forEach { $0.yield(event) }
    }
    
    func unsubscribe(id: UUID) {
        continuations[id]?.finish()
        continuations.removeValue(forKey: id)
    }
    
    private func register(continuation: AsyncStream<Event>.Continuation, id: UUID) {
        continuations[id] = continuation
    }
}

enum AppEvent: Sendable {
    case userLoggedIn(userId: String)
    case userLoggedOut
    case dataRefreshed
    case errorOccurred(message: String)
}

let eventBus = EventBus<AppEvent>()

Task {
    let (_, stream) = await eventBus.subscribe()
    for await event in stream {
        switch event {
        case .userLoggedIn(let userId):
            print("[Analytics] User logged in: \(userId)")
        case .userLoggedOut:
            print("[Analytics] User logged out")
        default:
            break
        }
    }
}

Task { @MainActor in
    let (_, stream) = await eventBus.subscribe()
    for await event in stream {
        switch event {
        case .dataRefreshed:
            print("[UI] Refreshing...")
        case .errorOccurred(let message):
            print("[UI] Error: \(message)")
        default:
            break
        }
    }
}

Task {
    await eventBus.publish(.userLoggedIn(userId: "user_123"))
    try? await Task.sleep(for: .seconds(2))
    await eventBus.publish(.dataRefreshed)
}
```

---

## 9. Migration Guide Swift 5 → Swift 6

### Langkah 1: Aktifkan Warning Dulu

Di Xcode, set `SWIFT_STRICT_CONCURRENCY = targeted` terlebih dahulu untuk melihat warning sebelum Swift 6 mode penuh:

```swift
// Package.swift
.target(
    name: "MyTarget",
    swiftSettings: [
        .enableExperimentalFeature("StrictConcurrency")
    ]
)
```

### Langkah 2: Pola Umum dan Solusinya

**Pattern 1: Class yang diakses dari banyak thread**

```swift
// BEFORE (Swift 5)
class AppSettings {
    var isDarkMode: Bool = false
    var language: String = "en"
}

// AFTER (Swift 6) — opsi A: jadikan actor
actor AppSettings {
    var isDarkMode: Bool = false
    var language: String = "en"
}

// AFTER (Swift 6) — opsi B: isolasi ke @MainActor jika hanya untuk UI
@MainActor
class AppSettings {
    var isDarkMode: Bool = false
    var language: String = "en"
}
```

**Pattern 2: Completion handler ke async/await**

```swift
// BEFORE
func fetchUser(id: String, completion: @escaping (User?) -> Void) {
    URLSession.shared.dataTask(with: request) { data, _, _ in
        completion(parse(data))
    }.resume()
}

// AFTER (Swift 6) — async + Sendable
func fetchUser(id: String) async -> User? {
    let (data, _) = try? await URLSession.shared.data(for: request)
    return parse(data)
}
```

**Pattern 3: Notification observer**

```swift
// BEFORE
NotificationCenter.default.addObserver(
    forName: .someNotification,
    object: nil,
    queue: .main
) { notification in
    self.handleNotification(notification)
}

// AFTER (Swift 6)
Task { @MainActor in
    for await notification in NotificationCenter.default.notifications(named: .someNotification) {
        handleNotification(notification)
    }
}
```

**Pattern 4: DispatchQueue → Actor**

```swift
// BEFORE
let queue = DispatchQueue(label: "com.app.data", attributes: .concurrent)
var dataStore: [String: Any] = [:]

func set(_ value: Any, for key: String) {
    queue.async(flags: .barrier) { self.dataStore[key] = value }
}

// AFTER (Swift 6)
actor DataStore {
    private var store: [String: Any] = [:]
    
    func set(_ value: Any, for key: String) {
        store[key] = value
    }
}
```

### Langkah 3: Urutan Prioritas Migrasi

1. **Model/Data layer** — jadikan actor atau Sendable struct
2. **Service layer** — jadikan actor atau global actor
3. **ViewModel** — anotasi `@MainActor`
4. **View/ViewController** — anotasi `@MainActor`
5. **Utility/Helper** — nonisolated jika memungkinkan, atau sesuaikan

---

## 10. Troubleshooting & Common Mistakes

### Kesalahan 1: Sendable Violation

```swift
// ERROR: 'NonSendableType' cannot be sent to actor-isolated method
class Config {  // class biasa, tidak Sendable
    var timeout: Int = 30
}

actor Service {
    func configure(with config: Config) { }  // ERROR di Swift 6
}

// Solusi A: jadikan struct (paling direkomendasikan)
struct Config: Sendable {
    var timeout: Int = 30
}

// Solusi B: jadikan final + immutable + Sendable
final class Config: Sendable {
    let timeout: Int
    init(timeout: Int) { self.timeout = timeout }
}

// Solusi C: pindahkan logika ke dalam actor
actor Service {
    var timeout: Int = 30
    
    func setTimeout(_ value: Int) {
        timeout = value
    }
}
```

### Kesalahan 2: Actor Reentrancy Bug

```swift
actor Inventory {
    var stock: [String: Int] = ["item_A": 5]
    
    // BUGGY: reentrancy bisa menyebabkan overselling
    func purchase(item: String) async -> Bool {
        guard let count = stock[item], count > 0 else { return false }
        await processPayment()  // suspension point!
        stock[item]! -= 1      // stock mungkin sudah 0 saat ini
        return true
    }
    
    // BENAR: modifikasi state sebelum suspension
    func purchaseSafe(item: String) async -> Bool {
        guard let count = stock[item], count > 0 else { return false }
        stock[item]! -= 1      // kurangi dulu sebelum await
        await processPayment()
        return true
    }
    
    private func processPayment() async {
        try? await Task.sleep(for: .milliseconds(100))
    }
}
```

### Kesalahan 3: Deadlock dengan Actor + DispatchQueue

```swift
// JANGAN lakukan ini — bisa deadlock
actor MyActor {
    func doWork() {
        DispatchQueue.main.sync {  // ❌ sync dari actor bisa deadlock!
            // update UI
        }
    }
    
    // BENAR: gunakan await MainActor.run
    func doWorkCorrectly() async {
        await MainActor.run {
            // update UI — aman
        }
    }
}
```

### Kesalahan 4: Task.detached Tanpa Perlu

```swift
@MainActor
class VC: UIViewController {
    func loadData() {
        // HINDARI:
        Task.detached {
            let data = await fetchData()
            await MainActor.run { self.display(data) }
        }
        
        // LEBIH BAIK:
        Task {
            let data = await fetchData()
            display(data)  // otomatis di @MainActor
        }
    }
}
```

### Kesalahan 5: Mengabaikan Task Cancellation

```swift
actor DataFetcher {
    func fetchLargeDataset() async throws -> [String] {
        var results: [String] = []
        
        for i in 0..<1000 {
            // SELALU cek cancellation di loop panjang
            try Task.checkCancellation()
            
            let item = await fetchItem(at: i)
            results.append(item)
        }
        
        return results
    }
    
    private func fetchItem(at index: Int) async -> String {
        try? await Task.sleep(for: .milliseconds(10))
        return "item_\(index)"
    }
}
```

---

## Ringkasan: Decision Tree

Gunakan panduan ini ketika bingung memilih tool yang tepat:

```
Apakah datamu mutable?
├── TIDAK (let, immutable) → Tidak butuh isolation. Sendable otomatis.
└── YA (var, bisa berubah)
    │
    Apakah data adalah value type (struct/enum)?
    ├── YA → Value semantics memberikan copy otomatis.
    │         Tidak butuh actor. Sendable otomatis jika semua property Sendable.
    └── TIDAK (class, reference type)
        │
        Apakah hanya diakses dari main thread (UI)?
        ├── YA → Gunakan @MainActor class
        └── TIDAK (diakses dari multiple contexts)
            │
            Apakah satu instance dengan state terlindungi?
            ├── YA → Gunakan actor
            └── Apakah banyak class berbagi satu domain isolation?
                ├── YA → Gunakan @globalActor
                └── Pertimbangkan refactor ke struct atau actor
```

```
Apakah kamu mengirim nilai melewati isolation boundary?
├── Value type (struct/enum) dengan semua Sendable properties → OK otomatis
├── Actor reference → OK, actor selalu Sendable
├── Class dengan hanya immutable (let) properties → final class: Sendable
├── Class dengan mutable (var) properties → Jadikan actor atau struct
└── Non-Sendable yang tidak akan diakses lagi setelah dikirim → sending keyword
```

---

## Ringkasan Konsep Kunci

| Konsep | Mengapa Ada | Kapan Digunakan | Kapan Tidak Digunakan |
|--------|-------------|-----------------|----------------------|
| `actor` | Mencegah concurrent mutation pada reference type | Shared mutable state lintas threads | Value type, single-thread access, butuh inheritance |
| `@MainActor` | UIKit tidak thread-safe | UI layer, ViewModel dengan @Published | Business logic, network/DB layer |
| `nonisolated` | Menghindari overhead hop untuk data immutable | Method hanya akses `let` properties | Jika ada akses ke `var` property |
| `Sendable` | Membuktikan ke compiler bahwa nilai aman dikirim lintas isolation | Semua nilai yang melewati isolation boundary | Tidak perlu eksplisit jika struct/enum dengan all-Sendable properties |
| `@unchecked Sendable` | Bridging ke legacy/C code yang sudah thread-safe | Manual synchronization yang terbukti benar | Jangan untuk menghindari compiler error tanpa alasan |
| `sending` | Transfer ownership tanpa Sendable | Pipeline processing, nilai yang tidak diakses lagi | Jika caller masih butuh nilai setelah call |
| `isolated` param | Utility function yang perlu akses actor state | Helper function terkait actor, testing | Jika function butuh dua actor berbeda sekaligus |
| `@globalActor` | Satu domain isolation untuk banyak tipe | Layer terpisah (DB, Network) yang butuh shared domain | Jika hanya satu instance — gunakan actor biasa |
| `Task` | Unit kerja async yang mewarisi isolation | Default untuk semua async work | Jarang perlu hindari, ini tool utama |
| `Task.detached` | Task benar-benar independent | Tugas yang tidak boleh di-cancel oleh parent | Hampir semua kasus — Task {} sudah cukup |

---

## Referensi

- [Swift Evolution SE-0302 — Sendable](https://github.com/apple/swift-evolution/blob/main/proposals/0302-concurrent-value-and-concurrent-closures.md)
- [Swift Evolution SE-0306 — Actors](https://github.com/apple/swift-evolution/blob/main/proposals/0306-actors.md)
- [Swift Evolution SE-0316 — Global Actors](https://github.com/apple/swift-evolution/blob/main/proposals/0316-global-actors.md)
- [Swift Evolution SE-0414 — Region-Based Isolation](https://github.com/apple/swift-evolution/blob/main/proposals/0414-region-based-isolation.md)
- [Swift 6 Migration Guide — swift.org](https://www.swift.org/migration/documentation/migrationguide/)

---

*Dibuat: 2026-05-08 | Diperbarui: 2026-05-08 | Swift 6.0 | Xcode 16+*
