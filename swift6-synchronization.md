# Swift 6: Synchronization Module — Mutex dan Atomic
### Teori Mendalam, Real Use Case, dan Panduan Penggunaan

---

## Daftar Isi
1. [Masalah yang Dipecahkan](#1-masalah-yang-dipecahkan)
2. [Teori: Mutual Exclusion dan Memory Ordering](#2-teori-mutual-exclusion-dan-memory-ordering)
3. [Mutex\<Value\>](#3-mutexvalue)
4. [Atomic\<Value\>](#4-atomicvalue)
5. [Memory Ordering — Teori Lengkap](#5-memory-ordering--teori-lengkap)
6. [Kapan Harus Digunakan](#6-kapan-harus-digunakan)
7. [Kapan Tidak Digunakan](#7-kapan-tidak-digunakan)
8. [Real Use Cases](#8-real-use-cases)
9. [Mutex vs Atomic vs Actor — Kapan Memilih](#9-mutex-vs-atomic-vs-actor--kapan-memilih)
10. [Ringkasan](#10-ringkasan)

---

## 1. Masalah yang Dipecahkan

### Sebelum Swift 6: Tooling yang Tidak Memadai

Developer iOS/macOS punya beberapa pilihan untuk synchronization, semua dengan masalahnya masing-masing:

**`NSLock` + manual variable**
```swift
// Masalah: lock dan data terpisah — tidak ada yang mencegah akses tanpa lock
class Counter {
    private let lock = NSLock()
    private var value = 0  // tidak terkait dengan lock — bisa diakses langsung!
    
    func increment() {
        lock.lock()
        value += 1
        lock.unlock()  // mudah lupa, apalagi dengan early return atau throw
    }
    
    func dangerousRead() -> Int {
        return value  // ❌ tidak ada lock — developer bisa lupa ini
    }
}
```

**`DispatchQueue.sync`**
```swift
// Masalah: verbose, bisa deadlock jika nested call ke queue yang sama
let queue = DispatchQueue(label: "com.app.counter")
var value = 0

func increment() {
    queue.sync { value += 1 }
}

func nestedCall() {
    queue.sync {           // deadlock jika ini dipanggil dari dalam queue yang sama!
        queue.sync { }     // ❌ deadlock
    }
}
```

**`OSAtomic` (deprecated)**
```swift
// Deprecated di macOS 10.12 / iOS 10
// Tidak ada pengganti resmi yang type-safe di Swift sampai Swift 6
import Darwin
var counter: Int32 = 0
OSAtomicIncrement32(&counter)  // deprecated, Objective-C era
```

**`actor`**
```swift
// Bagus untuk complex state, tapi overhead async untuk operasi sederhana
actor Counter {
    var value = 0
    func increment() { value += 1 }
}
let c = Counter()
Task { await c.increment() }  // overhead async untuk sekadar += 1
```

Swift 6 menghadirkan **`Mutex<Value>`** dan **`Atomic<Value>`** — dua primitive yang dirancang khusus untuk Swift's type system, aman, dan ergonomis.

---

## 2. Teori: Mutual Exclusion dan Memory Ordering

### Mutual Exclusion (Mutex)

**Mutex** (Mutual Exclusion) adalah primitive synchronization yang menjamin hanya satu thread yang bisa mengeksekusi critical section pada satu waktu.

```
Thread A: lock() → critical section → unlock()
Thread B:          [blocked]       →  lock() → critical section → unlock()
```

Properti utama mutex yang benar:
1. **Safety:** Hanya satu thread dalam critical section pada satu waktu
2. **Liveness:** Thread yang menunggu akhirnya akan mendapatkan lock
3. **No starvation:** Tidak ada thread yang menunggu selamanya

**Masalah klasik dengan mutex:**
- **Deadlock:** Thread A menunggu Thread B, Thread B menunggu Thread A
- **Priority inversion:** Thread prioritas tinggi menunggu thread prioritas rendah yang memegang lock
- **Livelock:** Thread terus bergerak tapi tidak ada kemajuan

Swift's `Mutex<Value>` mengikat data langsung ke mutex-nya — ini mencegah akses tanpa lock karena data **hanya bisa diakses melalui `withLock`**.

### Atomic Operations

**Atomik** berarti operasi terjadi sebagai satu unit yang tidak bisa dipotong oleh thread lain. Bayangkan ini seperti "transaksi yang tidak bisa di-interrupt":

```
Thread A: read-modify-write counter (atomic)
Thread B:                              [harus tunggu operasi A selesai]
```

Atomic operations diimplementasikan menggunakan instruksi CPU khusus:
- **CAS (Compare-and-Swap)**: "Ganti nilai ini dari X ke Y, tapi hanya jika masih X"
- **Fetch-and-Add**: "Tambah nilai ini dan kembalikan nilai lama, secara atomik"
- **Load/Store**: "Baca/tulis dengan jaminan atomisitas"

Atomic jauh **lebih cepat dari mutex** untuk operasi sederhana karena:
- Tidak ada OS syscall (mutex mungkin masuk ke kernel)
- Tidak ada thread blocking
- Diimplementasikan langsung sebagai instruksi CPU

Tapi atomic hanya cocok untuk **tipe sederhana** (Int, Bool, pointer) dan **operasi tunggal**. Untuk operasi yang melibatkan banyak variabel atau logika kompleks, kamu butuh mutex.

---

## 3. Mutex\<Value\>

### Konsep Utama: Data dan Lock Tidak Terpisah

Inovasi utama `Mutex<Value>` adalah bahwa **data terkunci langsung di dalam mutex** — tidak ada cara untuk mengakses data tanpa lock:

```swift
import Synchronization

// Value (Int) dibungkus di dalam Mutex
// Tidak ada cara akses langsung ke Int tanpa withLock
let counter = Mutex(0)

// SATU-SATUNYA cara akses: withLock
counter.withLock { value in
    value += 1
}

// Baca nilai
let current = counter.withLock { $0 }
```

### API Lengkap Mutex

```swift
import Synchronization

// Inisialisasi dengan nilai awal
let mutex = Mutex(initialValue)

// withLock: akses eksklusif — parameter adalah inout
mutex.withLock { (value: inout T) in
    // value adalah inout — bisa read dan write
    value = newValue
    let current = value
}

// withLockIfAvailable: non-blocking — returns nil jika lock tidak bisa diperoleh
let result = mutex.withLockIfAvailable { (value: inout T) -> Result in
    // Dieksekusi hanya jika lock tersedia sekarang
    return transform(value)
}
// result adalah Optional — nil jika lock tidak diperoleh
```

### Mutex adalah Sendable

`Mutex<Value>` adalah `Sendable` secara otomatis (jika Value-nya Sendable), membuatnya aman untuk digunakan di seluruh isolation domain:

```swift
struct AppState: Sendable {
    var user: User?
    var settings: Settings
}

// Mutex bisa dibagikan ke Actor, Task, dll
let sharedState = Mutex(AppState(user: nil, settings: .default))

actor NetworkService {
    let state: Mutex<AppState>  // OK: Mutex adalah Sendable
    
    init(state: Mutex<AppState>) {
        self.state = state
    }
    
    func updateUser(_ user: User) {
        state.withLock { $0.user = user }
    }
}
```

### Nested Mutex — Deadlock Risk

```swift
let mutex1 = Mutex(1)
let mutex2 = Mutex(2)

// DEADLOCK RISK: nested withLock
mutex1.withLock { _ in
    mutex2.withLock { _ in  // ⚠️ Bisa deadlock jika thread lain melakukan kebalikannya
        // ...
    }
}

// Solusi: gunakan satu Mutex untuk semua state yang saling terkait
struct CombinedState {
    var value1: Int
    var value2: Int
}
let combined = Mutex(CombinedState(value1: 1, value2: 2))
combined.withLock { state in
    state.value1 += 1
    state.value2 += 2
    // Kedua operasi dalam satu lock — atomic dan tidak deadlock
}
```

---

## 4. Atomic\<Value\>

### Konsep: Lock-Free untuk Tipe Primitif

`Atomic<Value>` untuk operasi pada tipe yang mendukung `AtomicRepresentable`:
- Integer types: `Int`, `Int8`, `Int16`, `Int32`, `Int64`
- Unsigned: `UInt`, `UInt8`, `UInt16`, `UInt32`, `UInt64`
- `Bool`
- `Optional<T>` di mana T adalah pointer/reference

```swift
import Synchronization

// Inisialisasi — harus var, Atomic bukan Sendable sepenuhnya
let requestCount = Atomic(0)

// Load: baca nilai
let count = requestCount.load(ordering: .relaxed)

// Store: tulis nilai
requestCount.store(42, ordering: .relaxed)

// Exchange: tulis nilai baru, dapatkan nilai lama
let old = requestCount.exchange(0, ordering: .acquiring)

// Add (hanya untuk integer)
requestCount.add(1, ordering: .relaxed)
let newValue = requestCount.wrappingAdd(1, ordering: .relaxed)  // overflow-safe

// Compare-and-Exchange (CAS)
let (exchanged, current) = requestCount.compareExchange(
    expected: 5,
    desired: 10,
    ordering: .sequentiallyConsistent
)
// exchanged = true jika nilai sebelumnya adalah 5 dan berhasil di-ganti ke 10
// current = nilai actual setelah operasi
```

### Mengapa `Atomic` Butuh `var` dan Bukan `let`?

```swift
// Atomic<Value> harus disimpan sebagai property yang stable address-nya
// Karena implementasinya menggunakan pointer ke lokasi memori tetap

// Sebagai stored property (OK):
class RequestTracker {
    let count = Atomic(0)  // OK: class memberikan stable address
}

// Sebagai local variable (OK dengan var):
var localCounter = Atomic(0)

// Sebagai let local (mungkin tidak OK tergantung konteks)
// Hindari Atomic sebagai local var yang di-copy
```

---

## 5. Memory Ordering — Teori Lengkap

Ini adalah konsep paling kompleks dari atomic operations. **Memory ordering** menentukan bagaimana operasi atomic berinteraksi dengan operasi memori lain di sekitarnya.

### Mengapa Memory Ordering Penting?

CPU modern melakukan optimasi agresif:
- **Out-of-order execution:** CPU mengeksekusi instruksi tidak berurutan
- **Store buffering:** Writes mungkin tidak langsung terlihat oleh CPU lain
- **Cache coherency:** Setiap CPU punya cache sendiri

Tanpa ordering, operasi yang kamu tulis di kode mungkin tidak terlihat oleh thread lain dalam urutan yang kamu harapkan.

### Hierarchy Memory Ordering (dari paling longgar ke paling ketat)

```swift
// 1. .relaxed — hanya atomisitas, tidak ada ordering guarantees
// Paling cepat, paling sedikit jaminan
// Gunakan: counter statistik, flag yang tidak bergantung pada data lain
requestCount.add(1, ordering: .relaxed)

// 2. .acquiring — operasi ini "melihat" semua writes dari release sebelumnya
// Gunakan: saat membaca flag sebelum membaca data yang dilindungi flag tersebut
let isReady = flag.load(ordering: .acquiring)
if isReady {
    // Dijamin melihat semua data yang di-store sebelum flag di-set
    let data = sharedData  // aman dibaca
}

// 3. .releasing — semua operasi sebelumnya dijamin terlihat sebelum operasi ini
// Gunakan: saat menulis data, lalu set flag "data ready"
sharedData = preparedData   // tulis data dulu
flag.store(true, ordering: .releasing)  // set flag — dijamin setelah data ditulis

// 4. .acquiringAndReleasing — kombinasi acquiring dan releasing
// Gunakan: untuk operasi read-modify-write seperti compare-exchange
let (_, _) = flag.compareExchange(
    expected: false,
    desired: true,
    ordering: .acquiringAndReleasing
)

// 5. .sequentiallyConsistent — ordering paling kuat
// Semua thread melihat semua operasi dalam urutan yang sama
// Paling lambat, paling aman
// Gunakan: default jika tidak yakin, atau untuk correctness yang mutlak
flag.store(true, ordering: .sequentiallyConsistent)
```

### Panduan Praktis Memory Ordering

```
Sedang belajar / tidak yakin → .sequentiallyConsistent
Counter statistik (tidak ada dependensi data) → .relaxed
Publish data (write flag setelah write data) → .releasing
Consume data (read data setelah check flag) → .acquiring
Compare-and-swap untuk lock → .acquiringAndReleasing
```

---

## 6. Kapan Harus Digunakan

### Gunakan `Mutex<Value>` ketika:

**1. State yang perlu banyak field diupdate atomik bersama**
```swift
// Beberapa field harus konsisten satu sama lain
struct NetworkState {
    var isConnected: Bool
    var lastError: Error?
    var reconnectAttempts: Int
}

let networkState = Mutex(NetworkState(isConnected: false, lastError: nil, reconnectAttempts: 0))

// Update semua field dalam satu lock — konsisten
networkState.withLock { state in
    state.isConnected = false
    state.lastError = connectionError
    state.reconnectAttempts += 1
}
```

**2. Synchronous code yang tidak bisa pakai actor (tidak bisa await)**
```swift
// Misalnya delegate methods, init, atau synchronous path
class URLSessionManager: NSObject, URLSessionDelegate {
    private let activeRequests = Mutex([UUID: URLSessionTask]())
    
    // URLSessionDelegate method adalah synchronous — tidak bisa await actor
    func urlSession(_ session: URLSession, task: URLSessionTask, didComplete: Error?) {
        activeRequests.withLock { requests in
            requests = requests.filter { $0.value !== task }
        }
    }
}
```

**3. Non-blocking lock attempt diperlukan**
```swift
let resource = Mutex(Resource())

// Coba tanpa blocking
if let result = resource.withLockIfAvailable({ $0.process() }) {
    use(result)
} else {
    // Resource sedang dipakai — coba lagi nanti atau skip
    scheduleRetry()
}
```

### Gunakan `Atomic<Value>` ketika:

**1. Single primitive yang diakses concurrent**
```swift
let activeConnectionCount = Atomic(0)

// Di setiap connection open:
activeConnectionCount.add(1, ordering: .relaxed)

// Di setiap connection close:
activeConnectionCount.subtract(1, ordering: .relaxed)

// Baca total:
let total = activeConnectionCount.load(ordering: .relaxed)
```

**2. Lock-free flag / once**
```swift
let isInitialized = Atomic(false)

func initialize() {
    // Hanya inisialisasi sekali, bahkan dari concurrent threads
    let (exchanged, _) = isInitialized.compareExchange(
        expected: false,
        desired: true,
        ordering: .acquiringAndReleasing
    )
    
    guard exchanged else { return }  // sudah diinisialisasi thread lain
    
    // Lakukan inisialisasi...
    setupDatabase()
    loadConfiguration()
}
```

**3. Performance-critical counter**
```swift
// Profiling atau metrics di hot path — Atomic jauh lebih cepat dari Mutex untuk ini
struct Metrics {
    let requestCount = Atomic(0)
    let errorCount = Atomic(0)
    let totalLatencyMicros = Atomic(Int64(0))
    
    func recordRequest(latencyMicros: Int64, isError: Bool) {
        requestCount.add(1, ordering: .relaxed)
        if isError { errorCount.add(1, ordering: .relaxed) }
        totalLatencyMicros.add(latencyMicros, ordering: .relaxed)
    }
    
    var averageLatency: Double {
        let count = Double(requestCount.load(ordering: .relaxed))
        let total = Double(totalLatencyMicros.load(ordering: .relaxed))
        return count > 0 ? total / count : 0
    }
}
```

---

## 7. Kapan Tidak Digunakan

### Hindari `Mutex` ketika:

**1. Kamu bisa menggunakan actor (async context tersedia)**
```swift
// JANGAN: menggunakan Mutex jika actor lebih tepat
let stateMutex = Mutex(AppState())
// ... withLock di mana-mana

// LEBIH BAIK: actor untuk async context
actor AppStateManager {
    var state = AppState()
    // ... methods yang jelas isolated
}
```

**2. Operasi di dalam lock sangat lambat (memblokir thread lama)**
```swift
// JANGAN: melakukan async/network call di dalam withLock
mutex.withLock { _ in
    // ❌ Ini memblokir thread selama lock dipegang
    let data = URLSession.shared.synchronousDataTask(...)  // JANGAN
}

// LEBIH BAIK: ambil state dulu, lalu lakukan operasi di luar lock
let localState = mutex.withLock { $0 }
let data = try await fetchData(using: localState)
mutex.withLock { state in
    state.update(with: data)
}
```

**3. Banyak nested lock yang dibutuhkan**
```swift
// JANGAN: nested mutex — deadlock risk tinggi
mutex1.withLock { _ in
    mutex2.withLock { _ in  // ⚠️ deadlock potential
        mutex3.withLock { _ in  // ⚠️⚠️ semakin dalam semakin berbahaya
        }
    }
}

// LEBIH BAIK: desain ulang state agar satu mutex cukup
```

### Hindari `Atomic` ketika:

**1. Operasi melibatkan lebih dari satu variabel**
```swift
// JANGAN: dua atomic yang harus konsisten — bukan atomic bersama!
let balance = Atomic(1000)
let pendingCharges = Atomic(0)

// ❌ Race condition: tidak atomik antar keduanya
let currentBalance = balance.load(ordering: .relaxed)
let pending = pendingCharges.load(ordering: .relaxed)
// Di antara dua load ini, nilai bisa berubah!
let available = currentBalance - pending

// LEBIH BAIK: gunakan Mutex untuk state yang saling terkait
let accountState = Mutex(AccountState(balance: 1000, pendingCharges: 0))
```

**2. Logic kompleks bergantung pada nilai atomic**
```swift
// JANGAN: check-then-act tanpa atomic — TOCTOU race
let stockCount = Atomic(5)
if stockCount.load(ordering: .acquiring) > 0 {
    // ⚠️ Stock bisa berubah setelah check tapi sebelum decrement!
    stockCount.subtract(1, ordering: .releasing)
}

// LEBIH BAIK: gunakan compareExchange untuk atomic check-and-decrement
var current = stockCount.load(ordering: .relaxed)
while current > 0 {
    let (exchanged, newCurrent) = stockCount.compareExchange(
        expected: current,
        desired: current - 1,
        ordering: .acquiringAndReleasing
    )
    if exchanged { break }  // berhasil decrement
    current = newCurrent    // coba lagi dengan nilai terbaru
}
```

---

## 8. Real Use Cases

### Use Case 1: Rate Limiter Thread-Safe

```swift
import Synchronization
import Foundation

// Rate limiter yang bisa dipakai dari banyak thread tanpa actor overhead
final class RateLimiter: Sendable {
    struct State {
        var tokens: Double
        var lastRefill: Date
    }
    
    private let state: Mutex<State>
    private let maxTokens: Double
    private let refillRate: Double  // tokens per second
    
    init(maxTokens: Double, refillRate: Double) {
        self.maxTokens = maxTokens
        self.refillRate = refillRate
        self.state = Mutex(State(tokens: maxTokens, lastRefill: Date()))
    }
    
    // Thread-safe: boleh dipanggil dari mana saja
    func tryConsume(tokens: Double = 1) -> Bool {
        state.withLock { state in
            // Refill berdasarkan waktu yang berlalu
            let now = Date()
            let elapsed = now.timeIntervalSince(state.lastRefill)
            state.tokens = min(maxTokens, state.tokens + elapsed * refillRate)
            state.lastRefill = now
            
            // Cek dan konsumsi token
            guard state.tokens >= tokens else { return false }
            state.tokens -= tokens
            return true
        }
    }
    
    var availableTokens: Double {
        state.withLock { $0.tokens }
    }
}

// Penggunaan di networking layer:
let apiRateLimiter = RateLimiter(maxTokens: 100, refillRate: 10)

func makeAPIRequest() async throws -> Data {
    guard apiRateLimiter.tryConsume() else {
        throw APIError.rateLimitExceeded
    }
    // ... lakukan request
    return Data()
}
```

---

### Use Case 2: Request Deduplication dengan Atomic

```swift
import Synchronization

// Track request IDs secara lock-free untuk deduplication
final class RequestDeduplicator: Sendable {
    // Gunakan Atomic untuk counter sederhana
    private let totalRequests = Atomic(0)
    private let duplicatesBlocked = Atomic(0)
    
    // Gunakan Mutex untuk state yang lebih kompleks
    private let activeRequestIDs = Mutex(Set<String>())
    
    func shouldProcess(requestID: String) -> Bool {
        totalRequests.add(1, ordering: .relaxed)
        
        return activeRequestIDs.withLock { activeIDs in
            guard !activeIDs.contains(requestID) else {
                duplicatesBlocked.add(1, ordering: .relaxed)
                return false
            }
            activeIDs.insert(requestID)
            return true
        }
    }
    
    func complete(requestID: String) {
        activeRequestIDs.withLock { activeIDs in
            activeIDs.remove(requestID)
        }
    }
    
    var stats: (total: Int, duplicatesBlocked: Int) {
        (
            total: totalRequests.load(ordering: .relaxed),
            duplicatesBlocked: duplicatesBlocked.load(ordering: .relaxed)
        )
    }
}
```

---

### Use Case 3: Metrics Collector dengan Atomic

```swift
import Synchronization
import Foundation

// High-performance metrics — Atomic untuk throughput tinggi
final class PerformanceMetrics: Sendable {
    // Gunakan Atomic untuk hot-path counters (dipanggil ribuan kali/detik)
    private let _requestCount = Atomic(Int64(0))
    private let _errorCount = Atomic(Int64(0))
    private let _totalLatencyNanos = Atomic(Int64(0))
    private let _minLatencyNanos = Atomic(Int64.max)
    private let _maxLatencyNanos = Atomic(Int64(0))
    
    func record(latencyNanos: Int64, isError: Bool) {
        _requestCount.add(1, ordering: .relaxed)
        _totalLatencyNanos.add(latencyNanos, ordering: .relaxed)
        
        if isError {
            _errorCount.add(1, ordering: .relaxed)
        }
        
        // Update min/max dengan CAS loop
        var currentMin = _minLatencyNanos.load(ordering: .relaxed)
        while latencyNanos < currentMin {
            let (exchanged, newMin) = _minLatencyNanos.compareExchange(
                expected: currentMin,
                desired: latencyNanos,
                ordering: .relaxed
            )
            if exchanged { break }
            currentMin = newMin
        }
        
        var currentMax = _maxLatencyNanos.load(ordering: .relaxed)
        while latencyNanos > currentMax {
            let (exchanged, newMax) = _maxLatencyNanos.compareExchange(
                expected: currentMax,
                desired: latencyNanos,
                ordering: .relaxed
            )
            if exchanged { break }
            currentMax = newMax
        }
    }
    
    var snapshot: MetricsSnapshot {
        let count = _requestCount.load(ordering: .acquiring)
        return MetricsSnapshot(
            requestCount: count,
            errorCount: _errorCount.load(ordering: .relaxed),
            averageLatencyMs: count > 0
                ? Double(_totalLatencyNanos.load(ordering: .relaxed)) / Double(count) / 1_000_000
                : 0,
            minLatencyMs: Double(_minLatencyNanos.load(ordering: .relaxed)) / 1_000_000,
            maxLatencyMs: Double(_maxLatencyNanos.load(ordering: .relaxed)) / 1_000_000
        )
    }
}

struct MetricsSnapshot: Sendable {
    let requestCount: Int64
    let errorCount: Int64
    let averageLatencyMs: Double
    let minLatencyMs: Double
    let maxLatencyMs: Double
}
```

---

### Use Case 4: Plugin Registry dengan Mutex

```swift
import Synchronization

// Registry yang diisi dari berbagai thread (plugin loading), dibaca dari banyak tempat
final class PluginRegistry: Sendable {
    struct RegistryState {
        var plugins: [String: any Plugin] = [:]
        var loadOrder: [String] = []
    }
    
    private let state = Mutex(RegistryState())
    
    func register(_ plugin: any Plugin, for key: String) {
        state.withLock { registry in
            registry.plugins[key] = plugin
            registry.loadOrder.append(key)
        }
        print("Plugin registered: \(key)")
    }
    
    func plugin(for key: String) -> (any Plugin)? {
        state.withLock { registry in
            registry.plugins[key]
        }
    }
    
    func allPlugins() -> [any Plugin] {
        state.withLock { registry in
            registry.loadOrder.compactMap { registry.plugins[$0] }
        }
    }
    
    func unregister(key: String) {
        state.withLock { registry in
            registry.plugins.removeValue(forKey: key)
            registry.loadOrder.removeAll { $0 == key }
        }
    }
    
    // Non-blocking: cek apakah plugin ada tanpa menunggu
    func hasPlugin(for key: String) -> Bool {
        state.withLockIfAvailable { registry in
            registry.plugins[key] != nil
        } ?? false  // jika lock tidak tersedia, anggap tidak ada
    }
}

protocol Plugin {}
```

---

## 9. Mutex vs Atomic vs Actor — Kapan Memilih

```
State-mu berupa primitif tunggal (Int, Bool, pointer)?
├── YA → Atomic<Value>
│         Keuntungan: lock-free, sangat cepat
│         Contoh: counter, flag, reference count
└── TIDAK

State-mu kompleks (struct, multiple fields) atau logika non-trivial?
├── Semua akses dari async context (await tersedia)?
│   ├── YA → Actor
│   │         Keuntungan: natural Swift concurrency, cancellation support
│   │         Contoh: service layers, data stores dalam app
│   └── TIDAK (sync callback, init, atau perlu non-blocking)
│       └── Mutex<Value>
│             Keuntungan: synchronous, tidak butuh await, withLockIfAvailable
│             Contoh: delegate callbacks, plugin registry, metrics
```

### Perbandingan Performa (Approximasi)

| Primitive | Overhead | Thread Block? | Async? |
|---|---|---|---|
| `Atomic` | ~1-5 ns | Tidak | Tidak |
| `Mutex.withLock` | ~10-50 ns | Ya (OS lock) | Tidak |
| `actor` hop | ~100-500 ns | Tidak (suspend) | Ya (butuh await) |
| `DispatchQueue.sync` | ~500-2000 ns | Ya | Tidak |

*Angka sangat bervariasi tergantung contention dan hardware.*

---

## 10. Ringkasan

### Quick Reference

```swift
// Counter sederhana — gunakan Atomic
let counter = Atomic(0)
counter.add(1, ordering: .relaxed)

// State dengan beberapa field — gunakan Mutex
let state = Mutex(MyState())
state.withLock { $0.field = newValue }

// Non-blocking attempt
if let result = mutex.withLockIfAvailable({ $0.value }) {
    use(result)
}

// CAS untuk lock-free check-and-update
let (success, _) = atomic.compareExchange(
    expected: oldValue,
    desired: newValue,
    ordering: .acquiringAndReleasing
)
```

### Memory Ordering Cheat Sheet

| Situasi | Ordering |
|---|---|
| Counter statistik (tidak ada dependensi data lain) | `.relaxed` |
| Write data, lalu set flag "ready" | `.releasing` |
| Check flag "ready", lalu read data | `.acquiring` |
| CAS untuk lock implementation | `.acquiringAndReleasing` |
| Tidak yakin / butuh maximum safety | `.sequentiallyConsistent` |

---

*Dibuat: 2026-05-08 | Swift 6.0 | import Synchronization*
