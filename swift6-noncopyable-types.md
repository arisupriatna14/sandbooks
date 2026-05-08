# Swift 6: Noncopyable Types (`~Copyable`)
### Teori Mendalam, Real Use Case, dan Panduan Penggunaan

---

## Daftar Isi
1. [Masalah yang Dipecahkan](#1-masalah-yang-dipecahkan)
2. [Teori: Value Semantics vs Ownership](#2-teori-value-semantics-vs-ownership)
3. [Sintaks Dasar](#3-sintaks-dasar)
4. [consuming, borrowing, dan copy](#4-consuming-borrowing-dan-copy)
5. [Noncopyable dengan Generics](#5-noncopyable-dengan-generics)
6. [Kapan Harus Digunakan](#6-kapan-harus-digunakan)
7. [Kapan Tidak Digunakan](#7-kapan-tidak-digunakan)
8. [Real Use Cases](#8-real-use-cases)
9. [Ringkasan](#9-ringkasan)

---

## 1. Masalah yang Dipecahkan

### Implicit Copying: Nyaman tapi Berbahaya untuk Resources

Di Swift (dan hampir semua bahasa modern), value types di-copy secara implisit. Ini membuat kode mudah dan aman — untuk data biasa:

```swift
var point1 = CGPoint(x: 1, y: 2)
var point2 = point1  // copy — tidak ada masalah, CGPoint adalah "data murni"
point2.x = 10
// point1.x masih 1 — aman
```

Tapi ada kategori nilai yang **tidak boleh di-copy**: resource handles, unique ownership tokens, dan objects dengan lifecycle yang ketat.

```swift
// Swift 5: masalah ini tidak terdeteksi compiler
struct FileDescriptor {
    let fd: Int32
    
    func close() {
        Darwin.close(fd)
    }
}

let file = FileDescriptor(fd: open("/tmp/data.txt", O_RDONLY))
let copy = file  // COPY terjadi — dua FileDescriptor, satu fd yang sama!

file.close()     // menutup fd
copy.close()     // menutup fd yang SAMA lagi → undefined behavior (double close)
```

**Double free, double close, use-after-free** — semua ini adalah bug yang umum dan berbahaya, dan sebelumnya tidak bisa dicegah di level type system Swift.

### Mengapa Ini Penting?

Kategori resource yang sering punya masalah ini:
- **File handles** — harus ditutup tepat sekali
- **Database connections** — harus dikembalikan ke pool tepat sekali
- **Network sockets** — harus ditutup dengan benar
- **Mutex locks / critical section guards** — harus di-release tepat sekali
- **Memory allocations** — harus di-free tepat sekali
- **Kriptografi keys** — harus di-clear dari memori tepat sekali
- **Transaction handles** — harus di-commit atau di-rollback tepat sekali

---

## 2. Teori: Value Semantics vs Ownership

### Tiga Model Memori di Berbagai Bahasa

**Model 1: Garbage Collection (Java, Go, C#)**
```
Objek hidup di heap, GC menentukan kapan di-free.
Pro: sederhana bagi developer.
Cons: tidak ada kontrol lifecycle, pause, overhead memori.
```

**Model 2: Reference Counting (Swift, Objective-C, Python)**
```
Setiap objek punya counter. Counter = 0 → deallocated.
Pro: deterministik (deinit dipanggil langsung saat count = 0).
Cons: retain cycles, overhead counter, tidak mencegah logical errors.
```

**Model 3: Ownership / Borrow Checker (Rust)**
```
Compiler melacak siapa "pemilik" setiap nilai.
Nilai hanya punya SATU pemilik pada satu waktu.
Ketika pemilik keluar scope → nilai di-destroy.
Pro: zero overhead, compile-time safety, tidak ada double-free.
Cons: learning curve tinggi, borrow checker kadang terlalu ketat.
```

Swift dengan `~Copyable` mengadopsi **model ownership yang pragmatis** — tidak sekompleks Rust's borrow checker, tapi memberikan jaminan ownership yang cukup untuk sebagian besar resource management use cases.

### Apa itu "Copyable"?

Di Swift 6, semua tipe secara default conform ke protokol `Copyable` (secara implisit). Ini berarti mereka bisa di-copy secara bebas.

`~Copyable` (dibaca: "tilde Copyable" atau "noncopyable") adalah **suppression syntax** — kamu menghapus conformance default ini:

```
Copyable (default):  nilai bisa di-copy kapan saja, bebas
~Copyable:           nilai tidak bisa di-copy; compiler melacak ownership-nya
```

Ketika sebuah tipe `~Copyable`:
1. Assignment menjadi **move** bukan **copy**
2. Melewatkan ke fungsi bisa **consume** (pindah ownership) atau **borrow** (baca saja)
3. `deinit` dipanggil **tepat sekali** — saat pemilik terakhir keluar scope
4. Compiler mencegah penggunaan setelah ownership dipindah

---

## 3. Sintaks Dasar

### Mendefinisikan Noncopyable Type

```swift
// Tambahkan ~Copyable untuk opt-out dari default Copyable
struct FileHandle: ~Copyable {
    private let descriptor: Int32
    
    init?(path: String, flags: Int32 = O_RDONLY) {
        let fd = Darwin.open(path, flags)
        guard fd != -1 else { return nil }
        self.descriptor = fd
    }
    
    // deinit DIJAMIN dipanggil tepat sekali — tidak ada double close
    deinit {
        Darwin.close(descriptor)
    }
    
    // 'consuming' method: menghabiskan ownership — instance tidak bisa dipakai lagi
    consuming func close() {
        // deinit akan memanggil Darwin.close setelah method ini selesai
        // Bisa lakukan cleanup tambahan di sini
    }
    
    // 'borrowing' method: hanya membaca — tidak mengambil ownership
    borrowing func read(count: Int) -> Data? {
        var buffer = [UInt8](repeating: 0, count: count)
        let bytesRead = Darwin.read(descriptor, &buffer, count)
        guard bytesRead > 0 else { return nil }
        return Data(buffer.prefix(bytesRead))
    }
}

// Penggunaan:
func processFile() {
    guard let file = FileHandle(path: "/tmp/data.txt") else { return }
    
    let data = file.read(count: 1024)  // borrow — file masih valid
    process(data)
    
    // let copy = file  // ❌ ERROR: FileHandle tidak bisa di-copy
    
    // file di-close otomatis saat keluar scope — TEPAT SEKALI
}
```

### Move dengan `consume`

```swift
func processFile() {
    guard var file = FileHandle(path: "/tmp/data.txt") else { return }
    
    // 'consume' memindahkan ownership secara eksplisit
    let movedFile = consume file
    
    // ❌ ERROR: file tidak bisa dipakai lagi setelah consume
    // let data = file.read(count: 100)
    
    // movedFile sekarang pemilik sah
    let data = movedFile.read(count: 100)
    // movedFile di-close saat keluar scope
}
```

### Noncopyable Class

```swift
// Class juga bisa noncopyable — mencegah reference sharing yang tidak diinginkan
final class UniqueConnection: ~Copyable {
    private let socket: Int32
    
    init(host: String, port: Int) {
        // setup socket connection
        self.socket = 0  // placeholder
    }
    
    deinit {
        // disconnect dan cleanup
        Darwin.close(socket)
    }
    
    func send(_ data: Data) {
        // write ke socket
    }
}

// Ownership sekarang jelas — tidak ada dua variabel yang bisa memegang koneksi yang sama
func handleRequest() {
    let connection = UniqueConnection(host: "api.example.com", port: 443)
    // let shared = connection  // ❌ ERROR
    
    connection.send(requestData)
    // connection di-disconnect saat keluar scope
}
```

---

## 4. consuming, borrowing, dan copy

### `consuming` — Transfer Ownership ke Callee

```swift
struct Token: ~Copyable {
    let value: String
    let expiresAt: Date
    
    // consuming: method ini menghabiskan token setelah dipanggil
    consuming func use() -> AuthResult {
        // Token tidak bisa dipakai lagi setelah ini
        return AuthResult(token: self.value)
    }
}

// consuming parameter: caller tidak bisa pakai 'token' setelah memanggil ini
func authenticate(with token: consuming Token) async -> Bool {
    let result = token.use()  // ownership berpindah ke use()
    // token tidak bisa dipakai lagi
    return result.isValid
}

func loginFlow() async {
    let token = Token(value: "abc123", expiresAt: .now.addingTimeInterval(3600))
    let success = await authenticate(with: token)  // token di-consume di sini
    // ❌ ERROR: token sudah di-consume
    // print(token.value)
}
```

### `borrowing` — Pinjam Tanpa Ambil Ownership

```swift
// borrowing parameter: caller tetap memegang ownership setelah call
func inspect(handle: borrowing FileHandle) -> FileInfo {
    // Bisa baca dari handle, tapi tidak bisa consume atau mutate
    let size = getFileSize(handle)
    return FileInfo(size: size)
}

func processFile() {
    guard let file = FileHandle(path: "/data.bin") else { return }
    
    let info = inspect(handle: file)  // file dipinjam, tidak di-consume
    print("File size: \(info.size)")
    
    // file masih valid di sini!
    if let data = file.read(count: info.size) {
        process(data)
    }
    // file di-close saat keluar scope
}
```

### `copy` Keyword — Explicit Copy untuk Noncopyable Fields

```swift
struct Wrapper<T: ~Copyable>: ~Copyable {
    var value: T
    
    // Mengakses field noncopyable butuh borrowing atau consuming
    borrowing func inspect() -> String {
        // 'copy' keyword tidak tersedia untuk noncopyable — tapi untuk copyable fields:
        let description = value  // ini borrow, bukan copy
        return "\(description)"
    }
}
```

---

## 5. Noncopyable dengan Generics

```swift
// Generic function yang menerima noncopyable type
func withResource<R: ~Copyable, Result>(
    _ resource: consuming R,
    body: (borrowing R) throws -> Result
) rethrows -> Result {
    defer { _ = consume resource }  // cleanup setelah body
    return try body(resource)
}

// Penggunaan:
let result = withResource(FileHandle(path: "/tmp/data.txt")!) { handle in
    handle.read(count: 100)
}
// FileHandle otomatis di-close setelah withResource selesai
```

---

## 6. Kapan Harus Digunakan

### Gunakan `~Copyable` ketika:

**1. Resource yang harus di-release tepat sekali**
```swift
struct DatabaseTransaction: ~Copyable {
    private let connectionID: Int
    private var committed = false
    
    consuming func commit() {
        committed = true
        // ... commit transaction
    }
    
    deinit {
        if !committed {
            // Otomatis rollback jika tidak di-commit — tidak ada resource leak
            rollback(connectionID)
        }
    }
}
```

**2. Unique ownership token / capability**
```swift
// Hanya satu holder yang boleh menulis ke file pada satu waktu
struct WritePermission: ~Copyable {
    let fileID: String
    
    deinit {
        releaseWriteLock(fileID)  // lock pasti di-release
    }
}

func getWritePermission(for fileID: String) -> WritePermission? {
    guard acquireWriteLock(fileID) else { return nil }
    return WritePermission(fileID: fileID)
}
```

**3. Kriptografi dan sensitive data**
```swift
struct PrivateKey: ~Copyable {
    private var keyBytes: [UInt8]
    
    deinit {
        // Zero-out memori saat key tidak dibutuhkan lagi
        keyBytes = Array(repeating: 0, count: keyBytes.count)
    }
    
    borrowing func sign(_ data: Data) -> Signature {
        // Gunakan keyBytes untuk signing
        return Signature(bytes: [])
    }
}
```

**4. Mencegah use-after-free bug**
```swift
struct UnsafeBuffer: ~Copyable {
    private let pointer: UnsafeMutablePointer<UInt8>
    private let count: Int
    
    init(count: Int) {
        pointer = .allocate(capacity: count)
        self.count = count
    }
    
    deinit {
        pointer.deallocate()  // dijamin dipanggil tepat sekali
    }
}
```

---

## 7. Kapan Tidak Digunakan

### Hindari `~Copyable` ketika:

**1. Data murni yang tidak punya lifecycle**
```swift
// JANGAN: tidak ada manfaat noncopyable di sini
// struct Point: ~Copyable { var x, y: Double }

// GUNAKAN: Copyable biasa — CGPoint, CLLocationCoordinate2D, dll
struct Point { var x, y: Double }
```

**2. Data yang memang perlu dibagi ke banyak tempat**
```swift
// JANGAN: User perlu dibaca dari banyak tempat
// struct User: ~Copyable { var name: String }

// GUNAKAN: struct biasa atau actor
struct User { var name: String }  // di-copy saat diteruskan — aman
```

**3. Protocol conformances yang butuh Copyable**
```swift
// Banyak protokol Swift (Hashable, Equatable, Codable) membutuhkan Copyable
// Jika tipe kamu perlu conform ke protokol ini, tidak bisa ~Copyable

// Bukan noncopyable karena perlu Hashable (untuk dipakai sebagai Dictionary key)
struct UserID: Hashable {
    let value: UUID
}
```

**4. Saat kompleksitas melebihi manfaatnya**
```swift
// Jika tidak ada resource leak, double-free, atau use-after-free risk,
// noncopyable hanya menambah kerumitan tanpa benefit nyata
```

---

## 8. Real Use Cases

### Use Case 1: Database Transaction Guard

```swift
import Foundation

enum TransactionError: Error {
    case connectionFailed
    case constraintViolation(String)
    case deadlock
    case rollbackFailed
}

// Transaction yang PASTI di-commit atau di-rollback — tidak ada yang lolos
struct DatabaseTransaction: ~Copyable {
    private let id: UUID
    private let connection: DatabaseConnection
    private var state: State = .active
    
    enum State { case active, committed, rolledBack }
    
    init(connection: DatabaseConnection) throws(TransactionError) {
        self.id = UUID()
        self.connection = connection
        try connection.beginTransaction(id: id)
    }
    
    // consuming: setelah commit, transaction tidak bisa dipakai
    consuming func commit() throws(TransactionError) {
        guard state == .active else { return }
        state = .committed
        try connection.commitTransaction(id: id)
    }
    
    consuming func rollback() {
        guard state == .active else { return }
        state = .rolledBack
        try? connection.rollbackTransaction(id: id)
    }
    
    // borrowing: execute query tanpa mengambil ownership transaction
    borrowing func execute(_ sql: String) throws(TransactionError) {
        guard state == .active else {
            throw TransactionError.connectionFailed
        }
        try connection.execute(sql, in: id)
    }
    
    deinit {
        // Safety net: jika keluar scope tanpa commit/rollback
        if state == .active {
            try? connection.rollbackTransaction(id: id)
            print("⚠️ Transaction \(id) di-rollback otomatis karena tidak di-commit")
        }
    }
}

// Helper untuk RAII-style transaction management
func withTransaction<Result>(
    on connection: DatabaseConnection,
    body: (borrowing DatabaseTransaction) throws -> Result
) throws -> Result {
    var tx = try DatabaseTransaction(connection: connection)
    
    do {
        let result = try body(tx)
        try (consume tx).commit()
        return result
    } catch {
        (consume tx).rollback()
        throw error
    }
}

// Penggunaan — transaction PASTI di-commit atau di-rollback
func transferFunds(from sourceID: String, to destID: String, amount: Double) async throws {
    let connection = try await DatabaseConnection.connect()
    
    try withTransaction(on: connection) { tx in
        try tx.execute("UPDATE accounts SET balance = balance - \(amount) WHERE id = '\(sourceID)'")
        try tx.execute("UPDATE accounts SET balance = balance + \(amount) WHERE id = '\(destID)'")
        try tx.execute("INSERT INTO transfers VALUES ('\(sourceID)', '\(destID)', \(amount))")
        // commit terjadi otomatis saat withTransaction selesai
    }
    // Jika ada exception: rollback otomatis via deinit
}
```

---

### Use Case 2: Mutex Lock Guard (RAII Pattern)

```swift
// RAII: Resource Acquisition Is Initialization
// Lock diperoleh saat init, dilepas saat deinit — tidak bisa bocor

struct MutexGuard<Value>: ~Copyable {
    private let mutex: NSLock
    private let valuePtr: UnsafeMutablePointer<Value>
    
    fileprivate init(mutex: NSLock, valuePtr: UnsafeMutablePointer<Value>) {
        self.mutex = mutex
        self.valuePtr = valuePtr
        mutex.lock()  // acquire saat init
    }
    
    // Akses nilai yang dilindungi
    var value: Value {
        get { valuePtr.pointee }
        set { valuePtr.pointee = newValue }
    }
    
    deinit {
        mutex.unlock()  // release PASTI terjadi — tidak ada lock leak
    }
}

final class Protected<Value> {
    private let mutex = NSLock()
    private var _value: Value
    
    init(_ value: Value) {
        self._value = value
    }
    
    func withLock<Result>(_ body: (inout Value) throws -> Result) rethrows -> Result {
        mutex.lock()
        defer { mutex.unlock() }
        return try body(&_value)
    }
    
    // Menggunakan MutexGuard untuk ownership yang jelas
    func lock() -> MutexGuard<Value> {
        MutexGuard(mutex: mutex, valuePtr: &_value)
    }
}

// Penggunaan:
let counter = Protected(0)

// Cara 1: withLock closure
counter.withLock { value in
    value += 1
}

// Cara 2: MutexGuard — lock hidup selama guard dalam scope
func incrementAndRead() -> Int {
    var guard_ = counter.lock()   // lock acquired
    guard_.value += 1
    let result = guard_.value
    return result
    // guard_ keluar scope → lock released OTOMATIS
}
```

---

### Use Case 3: Cryptographic Key Management

```swift
import CryptoKit

// Private key yang harus di-zero saat tidak dibutuhkan
struct SecurePrivateKey: ~Copyable {
    // P256.Signing.PrivateKey sudah dikelola CryptoKit
    // Tapi kita bisa wrap untuk kontrol ownership yang lebih ketat
    private var key: P256.Signing.PrivateKey
    private let keyID: String
    
    init() {
        self.key = P256.Signing.PrivateKey()
        self.keyID = UUID().uuidString
        logKeyCreation(keyID)
    }
    
    // borrowing: sign tanpa memberikan akses ke raw key material
    borrowing func sign(_ data: Data) throws -> P256.Signing.ECDSASignature {
        return try key.signature(for: data)
    }
    
    // Public key aman untuk di-copy dan disebarkan
    borrowing func publicKey() -> P256.Signing.PublicKey {
        return key.publicKey
    }
    
    // Export — consuming karena setelah export key dianggap "habis"
    consuming func exportEncrypted(with wrappingKey: SymmetricKey) -> Data {
        // Encrypt dan return
        // Key material akan di-clear oleh deinit
        return Data()
    }
    
    deinit {
        // Key material di-clear dari memori
        // P256.PrivateKey sudah melakukan ini, tapi log kita tambahkan
        logKeyDestruction(keyID)
        // Dalam implementasi nyata: zero-out semua sensitive bytes
    }
    
    private func logKeyCreation(_ id: String) { print("Key created: \(id)") }
    private func logKeyDestruction(_ id: String) { print("Key destroyed: \(id)") }
}

// Penggunaan:
func signDocument(_ document: Data) async throws -> SignedDocument {
    // Key hanya ada dalam scope ini — tidak bisa bocor ke luar
    let key = SecurePrivateKey()
    
    let signature = try key.sign(document)
    let publicKey = key.publicKey()
    
    // let exportedKey = key  // ❌ ERROR: tidak bisa copy key
    
    return SignedDocument(
        data: document,
        signature: signature,
        signerPublicKey: publicKey
    )
    // key di-destroy di sini — zero-out dari memori
}
```

---

### Use Case 4: Connection Pool dengan Guaranteed Return

```swift
// Connection yang PASTI dikembalikan ke pool setelah dipakai

final class ConnectionPool {
    private var available: [Connection] = []
    private let mutex = NSLock()
    
    // Mengembalikan ticket — bukan connection langsung
    func acquire() -> PooledConnection? {
        mutex.lock()
        defer { mutex.unlock() }
        guard let connection = available.popLast() else { return nil }
        return PooledConnection(connection: connection, pool: self)
    }
    
    fileprivate func returnConnection(_ connection: Connection) {
        mutex.lock()
        defer { mutex.unlock() }
        available.append(connection)
    }
}

// Noncopyable: connection tidak bisa di-share atau di-copy
struct PooledConnection: ~Copyable {
    private let connection: Connection
    private weak var pool: ConnectionPool?
    
    fileprivate init(connection: Connection, pool: ConnectionPool) {
        self.connection = connection
        self.pool = pool
    }
    
    borrowing func query(_ sql: String) throws -> [Row] {
        try connection.execute(sql)
    }
    
    deinit {
        // PASTI dikembalikan ke pool — tidak ada connection leak
        pool?.returnConnection(connection)
    }
}

// Penggunaan:
func handleRequest(pool: ConnectionPool) async throws -> Response {
    guard let conn = pool.acquire() else {
        throw ServiceError.noConnectionAvailable
    }
    
    // conn tidak bisa di-copy, tidak bisa keluar scope tanpa dikembalikan ke pool
    let rows = try conn.query("SELECT * FROM users LIMIT 10")
    let response = Response(data: rows)
    
    return response
    // conn dikembalikan ke pool OTOMATIS di sini — tidak perlu manual return
}
```

---

## 9. Ringkasan

### Decision Guide

```
Type-mu punya resource yang harus di-release tepat sekali?
├── YA → Pertimbangkan ~Copyable + deinit untuk cleanup
│         Tanyakan: "Apa yang terjadi jika type ini di-copy?"
│         ├── "Double free/close/release" → HARUS ~Copyable
│         └── "Tidak masalah" → Copyable biasa
└── TIDAK → Gunakan Copyable biasa (default)

Type-mu merepresentasikan "kepemilikan eksklusif" atas sesuatu?
├── YA → ~Copyable adalah pilihan natural
└── TIDAK → Copyable biasa

Type-mu perlu conform ke Hashable/Equatable/Codable?
└── YA → Tidak bisa ~Copyable (protokol-protokol ini butuh Copyable)
```

### Perbandingan dengan Alternatif

| Pendekatan | Pro | Cons |
|---|---|---|
| `~Copyable` | Compile-time safety, zero overhead | Learning curve, limitasi protocol conformance |
| `class` + `deinit` | Familiar, flexible | Reference semantics, bisa di-copy pointer |
| `actor` | Thread-safe, familiar | Async overhead, bukan untuk resource ownership |
| Manual (`defer { resource.close() }`) | Simple | Error-prone, mudah lupa |
| `withResource { }` pattern | Explicit scope | Butuh closure, nesting bisa dalam |

---

*Dibuat: 2026-05-08 | Swift 6.0*
