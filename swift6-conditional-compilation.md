# Swift 6: Conditional Compilation untuk Project Swift 5
### Panduan Migrasi Bertahap — Teori, Konfigurasi, dan Real Use Case

---

## Daftar Isi

1. [Konsep Dasar: Language Version vs Compiler Version](#1-konsep-dasar-language-version-vs-compiler-version)
2. [Directive Conditional Compilation](#2-directive-conditional-compilation)
3. [Konfigurasi di Xcode](#3-konfigurasi-di-xcode)
4. [Konfigurasi di Swift Package Manager](#4-konfigurasi-di-swift-package-manager)
5. [Strategi Migrasi Bertahap](#5-strategi-migrasi-bertahap)
6. [Compatibility Layer Pattern](#6-compatibility-layer-pattern)
7. [Kapan Menggunakan Setiap Pendekatan](#7-kapan-menggunakan-setiap-pendekatan)
8. [Real Use Cases](#8-real-use-cases)
9. [Troubleshooting Error Umum](#9-troubleshooting-error-umum)
10. [Ringkasan dan Checklist Migrasi](#10-ringkasan-dan-checklist-migrasi)

---

## 1. Konsep Dasar: Language Version vs Compiler Version

Sebelum masuk ke konfigurasi, pahami dua hal yang sering tertukar:

### Language Version ≠ Compiler Version

```
Xcode 16 (Swift 6 Compiler)
│
├── Target A → SWIFT_VERSION = 6  ← Swift 6 language mode
│             strict concurrency, typed throws, dll aktif sebagai ERROR
│
├── Target B → SWIFT_VERSION = 5  ← Swift 5 language mode
│             Swift 6 compiler, tapi bahasa tetap Swift 5
│             Fitur Swift 6 TIDAK aktif secara default
│
└── Target C → SWIFT_VERSION = 5 + upcoming features
              Swift 5 mode + beberapa fitur Swift 6 di-opt-in satu per satu
```

**Artinya:** Kamu bisa upgrade ke Xcode 16 tanpa satu baris kode pun berubah, selama `SWIFT_VERSION` tetap `5`. Compiler Swift 6 sepenuhnya backward-compatible dengan kode Swift 5.

### Tiga Kondisi Runtime Compilation

```
Kondisi 1: Swift 5 compiler + Swift 5 mode
           → semua #if swift(<6.0) aktif, #if swift(>=6.0) di-skip

Kondisi 2: Swift 6 compiler + Swift 5 mode (SWIFT_VERSION=5)
           → sama seperti kondisi 1 dari sisi bahasa
           → #if compiler(>=6.0) aktif

Kondisi 3: Swift 6 compiler + Swift 6 mode (SWIFT_VERSION=6)
           → semua #if swift(>=6.0) aktif
           → strict concurrency sebagai error
```

---

## 2. Directive Conditional Compilation

### 2.1 `#if swift(>=X.Y)` — Berdasarkan Language Version

Paling umum. Bergantung pada nilai `SWIFT_VERSION` di Build Settings, **bukan** versi Xcode:

```swift
// Guard syntax dan fitur language baru
#if swift(>=6.0)
// Kode ini hanya dikompilasi saat SWIFT_VERSION = 6
func parse(_ data: Data) throws(ParseError) -> User {
    // typed throws — hanya ada di Swift 6
}
#else
// Kode ini dikompilasi saat SWIFT_VERSION = 5
func parse(_ data: Data) throws -> User {
    // untyped throws — kompatibel Swift 5
}
#endif
```

```swift
// Sendable conformance yang di-enforce
#if swift(>=6.0)
struct AppConfig: Sendable {
    let apiURL: URL
    let timeout: TimeInterval
}
#else
struct AppConfig {
    let apiURL: URL
    let timeout: TimeInterval
}
#endif
```

### 2.2 `#if compiler(>=X.Y)` — Berdasarkan Versi Toolchain

Bergantung pada versi compiler (Xcode), **bukan** language version. Berguna untuk API compiler yang baru tapi tidak terkait language version:

```swift
// Compiler version check — independen dari SWIFT_VERSION
#if compiler(>=6.0)
// Tersedia saat menggunakan Xcode 16+ / Swift 6 toolchain
// Meski SWIFT_VERSION masih = 5
import Observation  // @Observable macro tersedia
#else
import Combine      // fallback ke Combine untuk toolchain lama
#endif
```

**Perbedaan kritis:**

```swift
// Skenario: Xcode 16, SWIFT_VERSION = 5

#if swift(>=6.0)
    // TIDAK dikompilasi — language version masih 5
#endif

#if compiler(>=6.0)
    // DIKOMPILASI — compiler adalah Xcode 16 (Swift 6 toolchain)
#endif
```

### 2.3 `#if canImport()` — Berdasarkan Ketersediaan Module

Paling robust untuk library/module baru. Tidak peduli versi — jika module ada, gunakan:

```swift
// Module Synchronization hanya ada di Swift 6 / macOS 15 / iOS 18
#if canImport(Synchronization)
import Synchronization

struct ThreadSafeCounter {
    private let value = Mutex(0)
    
    func increment() {
        value.withLock { $0 += 1 }
    }
    
    func current() -> Int {
        value.withLock { $0 }
    }
}

#else
import Foundation

final class ThreadSafeCounter {
    private let lock = NSLock()
    private var _value = 0
    
    func increment() {
        lock.withLock { _value += 1 }
    }
    
    func current() -> Int {
        lock.withLock { _value }
    }
}
#endif
```

```swift
// Observation framework — @Observable macro
#if canImport(Observation)
import Observation

@Observable
class UserViewModel {
    var name: String = ""
    var isLoading: Bool = false
}

#else
import Combine

class UserViewModel: ObservableObject {
    @Published var name: String = ""
    @Published var isLoading: Bool = false
}
#endif
```

### 2.4 `#if hasFeature()` — Berdasarkan Fitur Language Spesifik

Cara paling granular — guard satu fitur tanpa terikat versi angka. Berguna saat fitur di-backport atau di-enable secara selektif:

```swift
// Guard fitur noncopyable types
#if hasFeature(NoncopyableGenerics)
struct FileDescriptor: ~Copyable {
    private let fd: Int32
    
    init?(path: String) {
        let descriptor = Darwin.open(path, O_RDONLY)
        guard descriptor != -1 else { return nil }
        self.fd = descriptor
    }
    
    deinit { Darwin.close(fd) }
    
    borrowing func read(count: Int) -> Data? {
        var buffer = [UInt8](repeating: 0, count: count)
        let n = Darwin.read(fd, &buffer, count)
        return n > 0 ? Data(buffer.prefix(n)) : nil
    }
}
#else
// Fallback: class dengan manual lifecycle
final class FileDescriptor {
    private let fd: Int32
    
    init?(path: String) {
        let descriptor = Darwin.open(path, O_RDONLY)
        guard descriptor != -1 else { return nil }
        self.fd = descriptor
    }
    
    deinit { Darwin.close(fd) }
    
    func read(count: Int) -> Data? {
        var buffer = [UInt8](repeating: 0, count: count)
        let n = Darwin.read(fd, &buffer, count)
        return n > 0 ? Data(buffer.prefix(n)) : nil
    }
}
#endif
```

```swift
// Guard typed throws
#if hasFeature(TypedThrows)
func validateAge(_ value: Int) throws(ValidationError) -> Int {
    guard value >= 0 else { throw ValidationError.negative }
    guard value <= 150 else { throw ValidationError.tooLarge }
    return value
}
#else
func validateAge(_ value: Int) throws -> Int {
    guard value >= 0 else { throw ValidationError.negative }
    guard value <= 150 else { throw ValidationError.tooLarge }
    return value
}
#endif
```

### 2.5 `@available` — Untuk OS Version (Bukan Language Version)

Berbeda dari `#if` — ini adalah runtime check dan hanya untuk API ketersediaan OS:

```swift
// Bukan untuk kondisi compile — ini runtime
if #available(iOS 18.0, macOS 15.0, *) {
    // Gunakan API baru
    let _ = Mutex(0)
} else {
    // Fallback
    let _ = NSLock()
}

// Untuk deklarasi tipe: @available sebagai annotation
@available(iOS 18.0, macOS 15.0, *)
struct NewAPIWrapper {
    // Hanya bisa dipakai di iOS 18+ / macOS 15+
}
```

---

## 3. Konfigurasi di Xcode

### 3.1 Set Swift Language Version per Target

```
Xcode → Project → Target → Build Settings
└── Cari: "Swift Language Version"
    ├── Pilih "Swift 5"   → tetap Swift 5
    └── Pilih "Swift 6"   → aktifkan Swift 6 mode
```

Screenshot path di Build Settings:
```
Build Settings
├── Swift Compiler - Language
│   ├── Swift Language Version    [Swift 5] ← ubah ini
│   └── ...
└── Swift Compiler - Upcoming Features
    ├── Strict Concurrency Checking  [Minimal/Targeted/Complete]
    └── ...
```

### 3.2 Strict Concurrency Checking (Tanpa Full Swift 6)

Untuk dapat **warning** concurrency tanpa mengaktifkan Swift 6 sepenuhnya:

```
Build Settings → Swift Compiler - Upcoming Features
└── Strict Concurrency Checking
    ├── Minimal   → hanya warning minimal (hampir tidak ada)
    ├── Targeted  → warning pada kode yang jelas concurrent ← REKOMENDASI untuk migrasi
    └── Complete  → semua warning Swift 6 (tapi masih warning, bukan error)
```

Setara di `xcconfig` atau `Other Swift Flags`:
```
// xcconfig file
SWIFT_STRICT_CONCURRENCY = targeted
```

### 3.3 Aktifkan Upcoming Features Satu per Satu di Xcode

Di **Other Swift Flags** (Build Settings):
```
OTHER_SWIFT_FLAGS = -enable-upcoming-feature StrictConcurrency
                    -enable-upcoming-feature TypedThrows
                    -enable-upcoming-feature ExistentialAny
```

Atau di `.xcconfig`:
```
// MyApp.xcconfig
OTHER_SWIFT_FLAGS = $(inherited) -enable-upcoming-feature StrictConcurrency
```

### 3.4 Struktur Target yang Direkomendasikan untuk Migrasi

```
MyApp (Xcode Project)
├── MyApp Target          → SWIFT_VERSION = 5  (UI layer, belum disentuh)
├── MyAppCore Target      → SWIFT_VERSION = 6  (business logic, sudah dimigrasikan)
│   └── Dependencies: MyAppLegacy
├── MyAppLegacy Target    → SWIFT_VERSION = 5  (kode lama yang belum dimigrasikan)
└── MyAppTests Target     → SWIFT_VERSION = 6  (test baru pakai Swift Testing)
```

---

## 4. Konfigurasi di Swift Package Manager

### 4.1 Swift Language Version per Target

```swift
// Package.swift
// swift-tools-version: 5.10  ← minimum tools version

import PackageDescription

let package = Package(
    name: "MyPackage",
    platforms: [
        .iOS(.v16),
        .macOS(.v13),
    ],
    products: [
        .library(name: "MyPackage", targets: ["Core", "Features"]),
    ],
    targets: [
        // Target dengan Swift 6
        .target(
            name: "Core",
            swiftSettings: [
                .swiftLanguageVersion(.v6)
            ]
        ),
        
        // Target tetap Swift 5
        .target(
            name: "LegacySupport",
            swiftSettings: [
                .swiftLanguageVersion(.v5)
            ]
        ),
        
        // Target migrasi: Swift 5 + strict concurrency warning
        .target(
            name: "Features",
            dependencies: ["Core", "LegacySupport"],
            swiftSettings: [
                .swiftLanguageVersion(.v5),
                .enableUpcomingFeature("StrictConcurrency"),
                .enableUpcomingFeature("ExistentialAny"),
            ]
        ),
        
        // Test target dengan Swift Testing
        .testTarget(
            name: "CoreTests",
            dependencies: ["Core"],
            swiftSettings: [
                .swiftLanguageVersion(.v6)
            ]
        ),
    ]
)
```

### 4.2 Upcoming Features yang Bisa Di-enable di Swift 5

Daftar fitur yang bisa di-enable satu per satu sebelum full Swift 6:

```swift
// Package.swift — semua upcoming features yang tersedia
.target(
    name: "MyTarget",
    swiftSettings: [
        // Concurrency
        .enableUpcomingFeature("StrictConcurrency"),
        
        // Type system
        .enableUpcomingFeature("ExistentialAny"),         // 'any Protocol' wajib
        .enableUpcomingFeature("ImplicitOpenExistentials"),
        
        // Language
        .enableUpcomingFeature("BareSlashRegexLiterals"), // /regex/ syntax
        .enableUpcomingFeature("ConciseMagicFile"),       // #file lebih ringkas
        .enableUpcomingFeature("ForwardTrailingClosures"),
        .enableUpcomingFeature("DisableOutwardActorInference"),
        
        // Swift 6 fitur spesifik
        .enableUpcomingFeature("TypedThrows"),
    ]
)
```

### 4.3 Experimental Features (Hati-hati)

```swift
// Fitur yang masih experimental — bisa berubah
.target(
    name: "ExperimentalTarget",
    swiftSettings: [
        .enableExperimentalFeature("StrictConcurrency"),  // versi lama
        .enableExperimentalFeature("VariadicGenerics"),
    ]
)
```

> **Peringatan:** `enableExperimentalFeature` bisa break antar minor version compiler. Gunakan `enableUpcomingFeature` untuk fitur yang sudah committed ke roadmap.

### 4.4 Conditional Dependencies Berdasarkan Platform

```swift
// Package.swift
targets: [
    .target(
        name: "Networking",
        dependencies: [
            // Dependency berbeda berdasarkan platform
            .product(
                name: "AsyncHTTPClient",
                package: "async-http-client",
                condition: .when(platforms: [.linux])
            ),
        ]
    )
]
```

---

## 5. Strategi Migrasi Bertahap

### Fase 1: Upgrade Xcode, Keep Swift 5 (Zero Risk)

```
Timeline: Minggu 1-2
Risiko:   Sangat Rendah
Action:   Upgrade ke Xcode 16, set SWIFT_VERSION = 5 di semua target
Hasil:    Swift 6 compiler, bahasa tetap Swift 5 — tidak ada perubahan perilaku
```

```swift
// Package.swift — tidak ada perubahan di fase ini
.target(
    name: "MyApp"
    // Tidak ada swiftSettings — default ke Swift 5
)
```

### Fase 2: Aktifkan Warning Concurrency (Targeted)

```
Timeline: Minggu 2-4
Risiko:   Rendah (hanya warning, bukan error)
Action:   Aktifkan StrictConcurrency = targeted
Hasil:    Lihat warning, mulai fix satu per satu
```

```swift
// Package.swift
.target(
    name: "MyApp",
    swiftSettings: [
        .swiftLanguageVersion(.v5),
        .enableUpcomingFeature("StrictConcurrency"),
    ]
)
```

```swift
// xcconfig atau Build Settings (Xcode project)
SWIFT_STRICT_CONCURRENCY = targeted
```

### Fase 3: Fix Warning Per Modul

```
Timeline: Minggu 4-8 (bergantung ukuran codebase)
Risiko:   Sedang
Action:   Fix warning satu module/feature pada satu waktu
          Mulai dari layer paling dalam (model, service) → atas (UI)
```

**Urutan yang direkomendasikan:**
```
1. Models & Entities     → jadikan Sendable struct
2. Worker / Service      → jadikan actor
3. Repository            → jadikan actor atau @MainActor
4. ViewModel             → @MainActor
5. ViewController / View → @MainActor
6. AppDelegate/SceneDelegate → @MainActor
```

### Fase 4: Upgrade Target Satu per Satu ke Swift 6

```
Timeline: Minggu 8-12
Risiko:   Sedang-Tinggi (warning jadi error)
Action:   Set SWIFT_VERSION = 6 per target, mulai dari yang paling sedikit warning
```

```swift
// Package.swift — migrasi bertahap per target
.target(
    name: "Core",           // paling sedikit dependency, migrasikan duluan
    swiftSettings: [.swiftLanguageVersion(.v6)]
),
.target(
    name: "Features",
    dependencies: ["Core"],
    swiftSettings: [.swiftLanguageVersion(.v5),  // belum Swift 6
                    .enableUpcomingFeature("StrictConcurrency")]
),
.target(
    name: "UI",
    dependencies: ["Features"],
    swiftSettings: [.swiftLanguageVersion(.v5)]  // terakhir dimigrasikan
),
```

### Fase 5: Full Swift 6

```
Timeline: Bulan 3-4
Risiko:   Sesuai seberapa jauh persiapan di fase sebelumnya
Action:   Semua target ke Swift 6
```

```swift
// Package.swift — semua target sudah Swift 6
let sharedSettings: [SwiftSetting] = [
    .swiftLanguageVersion(.v6)
]

.target(name: "Core", swiftSettings: sharedSettings),
.target(name: "Features", swiftSettings: sharedSettings),
.target(name: "UI", swiftSettings: sharedSettings),
```

---

## 6. Compatibility Layer Pattern

Pattern ini memungkinkan satu API yang sama bekerja di Swift 5 dan Swift 6:

### 6.1 Sendable Compatibility

```swift
// Sources/Compat/Sendable+Compat.swift

// Di Swift 6, Sendable conformance di-enforce compiler.
// Di Swift 5, kita bisa tambahkan secara retroactive sebagai no-op.
#if swift(<6.0)
// Retroactive conformance untuk tipe yang kita tahu thread-safe
// tapi Swift 5 tidak bisa verifikasi
extension URLRequest: @unchecked Sendable {}
extension JSONDecoder: @unchecked Sendable {}
#endif
// Di Swift 6, conformance ini sudah ada di stdlib — tidak perlu deklarasi
```

### 6.2 Mutex / Thread-Safety Compatibility

```swift
// Sources/Compat/Protected.swift

#if canImport(Synchronization)
import Synchronization

// Alias langsung ke Mutex Swift 6
public struct Protected<Value> {
    private let mutex: Mutex<Value>
    
    public init(_ value: Value) {
        self.mutex = Mutex(value)
    }
    
    public func withLock<R>(_ body: (inout Value) throws -> R) rethrows -> R {
        try mutex.withLock(body)
    }
    
    public func withLockIfAvailable<R>(_ body: (inout Value) throws -> R) rethrows -> R? {
        try mutex.withLockIfAvailable(body)
    }
}

#else
import Foundation

// Fallback: NSLock untuk Swift 5 / OS lama
public final class Protected<Value>: @unchecked Sendable {
    private let lock = NSLock()
    private var value: Value
    
    public init(_ value: Value) {
        self.value = value
    }
    
    public func withLock<R>(_ body: (inout Value) throws -> R) rethrows -> R {
        lock.lock()
        defer { lock.unlock() }
        return try body(&value)
    }
    
    public func withLockIfAvailable<R>(_ body: (inout Value) throws -> R) rethrows -> R? {
        guard lock.try() else { return nil }
        defer { lock.unlock() }
        return try body(&value)
    }
}
#endif

// Penggunaan di kode lain — SAMA, tidak peduli Swift 5 atau 6:
let counter = Protected(0)
counter.withLock { $0 += 1 }
```

### 6.3 Observable / ObservableObject Compatibility

```swift
// Sources/Compat/Observable+Compat.swift

#if canImport(Observation)
import Observation

// Swift 6 / iOS 17+ / macOS 14+: gunakan @Observable
public typealias ObservableBase = AnyObject  // tidak perlu inherit apapun

// Macro untuk menandai class sebagai observable
@attached(member, names: ...)
@attached(memberAttribute)
@attached(conformance)
public macro Observable() = #externalMacro(module: "ObservationMacros", type: "ObservableMacro")

#else
import Combine

// Swift 5 / iOS 16 / macOS 13: gunakan ObservableObject
public typealias ObservableBase = ObservableObject
#endif
```

```swift
// ViewModel yang kompatibel dengan keduanya:

#if canImport(Observation)
import Observation

@Observable
class UserViewModel {
    var name: String = ""
    var email: String = ""
    var isLoading: Bool = false
}

#else
import Combine

class UserViewModel: ObservableObject {
    @Published var name: String = ""
    @Published var email: String = ""
    @Published var isLoading: Bool = false
}
#endif
```

### 6.4 Typed Throws Compatibility

```swift
// Sources/Compat/Error+Compat.swift

enum NetworkError: Error {
    case timeout
    case noConnection
    case serverError(Int)
    case invalidResponse
}

// Fungsi dengan typed throws di Swift 6, untyped di Swift 5
// Caller API tetap sama — hanya internals yang berbeda
#if hasFeature(TypedThrows)
func fetchData(from url: URL) async throws(NetworkError) -> Data {
    do {
        let (data, response) = try await URLSession.shared.data(from: url)
        guard let http = response as? HTTPURLResponse else {
            throw NetworkError.invalidResponse
        }
        guard (200...299).contains(http.statusCode) else {
            throw NetworkError.serverError(http.statusCode)
        }
        return data
    } catch is URLError {
        throw NetworkError.timeout
    } catch let error as NetworkError {
        throw error
    } catch {
        throw NetworkError.invalidResponse
    }
}

#else
func fetchData(from url: URL) async throws -> Data {
    do {
        let (data, response) = try await URLSession.shared.data(from: url)
        guard let http = response as? HTTPURLResponse else {
            throw NetworkError.invalidResponse
        }
        guard (200...299).contains(http.statusCode) else {
            throw NetworkError.serverError(http.statusCode)
        }
        return data
    } catch is URLError {
        throw NetworkError.timeout
    }
}
#endif
```

### 6.5 Swift Testing / XCTest Compatibility

```swift
// Tests/Compat/TestingCompat.swift

#if canImport(Testing)
import Testing

// Gunakan Swift Testing
public typealias SuiteBase = Any  // struct, tidak perlu inherit

// Macro wrappers yang tersedia langsung dari Testing framework

#else
import XCTest

// Fallback ke XCTest
public typealias SuiteBase = XCTestCase
#endif
```

```swift
// Tests/UserServiceTests.swift

#if canImport(Testing)
import Testing

@Suite("UserService Tests")
struct UserServiceTests {
    let service = UserService()
    
    @Test("Fetch user sukses")
    func fetchUserSuccess() async throws {
        let user = try await service.fetchUser(id: "123")
        #expect(user.id == "123")
    }
}

#else
import XCTest

class UserServiceTests: XCTestCase {
    var service: UserService!
    
    override func setUp() {
        super.setUp()
        service = UserService()
    }
    
    func testFetchUserSuccess() async throws {
        let user = try await service.fetchUser(id: "123")
        XCTAssertEqual(user.id, "123")
    }
}
#endif
```

---

## 7. Kapan Menggunakan Setiap Pendekatan

### Decision Tree

```
Apa yang ingin kamu guard?
│
├── Syntax / fitur language (typed throws, ~Copyable)
│   └── Gunakan: #if swift(>=6.0) atau #if hasFeature(FeatureName)
│
├── Module baru (Synchronization, Observation)
│   └── Gunakan: #if canImport(ModuleName)
│
├── Versi toolchain/compiler (Xcode version)
│   └── Gunakan: #if compiler(>=6.0)
│
├── API yang tersedia di OS versi tertentu
│   └── Gunakan: @available(iOS X, *) atau #available(iOS X, *)
│
└── Ingin aktifkan fitur Swift 6 satu per satu (tanpa full migration)
    └── Gunakan: enableUpcomingFeature di Package.swift / Build Settings
```

### Tabel Perbandingan

| Directive | Berdasarkan | Compile-time? | Contoh penggunaan |
|---|---|---|---|
| `#if swift(>=6.0)` | Language version (SWIFT_VERSION) | Ya | Typed throws, Sendable |
| `#if compiler(>=6.0)` | Toolchain version (Xcode) | Ya | API compiler baru |
| `#if canImport(X)` | Ketersediaan module | Ya | Synchronization, Observation |
| `#if hasFeature(X)` | Fitur language spesifik | Ya | NoncopyableGenerics |
| `@available(iOS X)` | OS version | Runtime | API OS baru |
| `#available(iOS X)` | OS version | Runtime | Branching runtime |
| `enableUpcomingFeature` | Opt-in per fitur | Build setting | Migrasi bertahap |

---

## 8. Real Use Cases

### Use Case 1: Library yang Support Swift 5 dan Swift 6

```swift
// Package.swift untuk library publik
// swift-tools-version: 5.10

let package = Package(
    name: "MyNetworkingLibrary",
    platforms: [.iOS(.v15), .macOS(.v12)],
    products: [
        .library(name: "MyNetworking", targets: ["MyNetworking"])
    ],
    targets: [
        .target(
            name: "MyNetworking",
            swiftSettings: [
                // Support Swift 5 dan 6 dengan fitur bertahap
                .swiftLanguageVersion(.v5),
                .enableUpcomingFeature("StrictConcurrency"),
                .enableUpcomingFeature("ExistentialAny"),
            ]
        ),
        .testTarget(
            name: "MyNetworkingTests",
            dependencies: ["MyNetworking"],
            swiftSettings: [
                // Test pakai Swift 6 penuh untuk tangkap issue lebih awal
                .swiftLanguageVersion(.v6)
            ]
        )
    ]
)
```

```swift
// Sources/MyNetworking/NetworkClient.swift

import Foundation

// Sendable: kompatibel Swift 5 dan 6
public struct RequestConfig {
    public let url: URL
    public let method: String
    public let headers: [String: String]
    public let timeout: TimeInterval
    
    public init(url: URL, method: String = "GET",
                headers: [String: String] = [:],
                timeout: TimeInterval = 30) {
        self.url = url
        self.method = method
        self.headers = headers
        self.timeout = timeout
    }
}

#if swift(>=6.0)
// Swift 6: Sendable explicit, di-enforce compiler
extension RequestConfig: Sendable {}
#else
// Swift 5: @unchecked Sendable karena struct dengan value types sudah aman
// tapi compiler tidak enforce ini
extension RequestConfig: @unchecked Sendable {}
#endif

// Actor untuk thread-safety: syntax sama di Swift 5 dan 6
// Tapi di Swift 6 enforcement lebih ketat
public actor NetworkClient {
    private let session: URLSession
    private var activeTasks: [UUID: Task<Data, Error>] = [:]
    
    public init(configuration: URLSessionConfiguration = .default) {
        self.session = URLSession(configuration: configuration)
    }
    
    public func fetch(_ config: RequestConfig) async throws -> Data {
        var request = URLRequest(url: config.url, timeoutInterval: config.timeout)
        request.httpMethod = config.method
        config.headers.forEach { request.setValue($1, forHTTPHeaderField: $0) }
        
        let (data, response) = try await session.data(for: request)
        
        guard let http = response as? HTTPURLResponse,
              (200...299).contains(http.statusCode) else {
            throw NetworkError.invalidResponse
        }
        
        return data
    }
}

public enum NetworkError: Error, Sendable {
    case invalidResponse
    case timeout
    case noConnection
}
```

---

### Use Case 2: App dengan Target Campuran

```
MyApp (Xcode Project)
├── MyAppCore.framework   → Swift 6 (business logic)
├── MyAppUI.framework     → Swift 5 + StrictConcurrency warning
└── MyApp target          → Swift 5 (app delegate, scene delegate)
```

```swift
// MyAppCore — Swift 6, fully migrated
// Sources/MyAppCore/UserRepository.swift

import Foundation

// Swift 6: actor + Sendable sudah di-enforce
public actor UserRepository {
    private var cache: [String: User] = [:]
    
    public func fetchUser(id: String) async throws -> User {
        if let cached = cache[id] { return cached }
        let user = try await loadFromAPI(id: id)
        cache[id] = user
        return user
    }
    
    private func loadFromAPI(id: String) async throws -> User {
        // ... network call
        return User(id: id, name: "Test")
    }
}

public struct User: Sendable {
    public let id: String
    public let name: String
    
    public init(id: String, name: String) {
        self.id = id
        self.name = name
    }
}
```

```swift
// MyAppUI — Swift 5 + StrictConcurrency = targeted (warning only)
// Sources/MyAppUI/UserListViewModel.swift

import Foundation
import MyAppCore

// Warning di Swift 5 mode: "class should be marked @MainActor"
// Tidak error — kita lihat dan fix satu per satu
@MainActor  // fix warning ini
public final class UserListViewModel: ObservableObject {
    @Published public private(set) var users: [User] = []
    @Published public private(set) var isLoading = false
    
    private let repository: UserRepository
    
    public init(repository: UserRepository) {
        self.repository = repository
    }
    
    public func load(userID: String) async {
        isLoading = true
        defer { isLoading = false }
        
        do {
            // UserRepository adalah actor dari MyAppCore (Swift 6)
            // Crossing isolation boundary — butuh await
            let user = try await repository.fetchUser(id: userID)
            users = [user]
        } catch {
            print("Error: \(error)")
        }
    }
}
```

```swift
// MyApp target — Swift 5, belum dimigrasikan
// App/SceneDelegate.swift

import UIKit
import MyAppUI
import MyAppCore

// Di Swift 5 mode: tidak ada error concurrency, tapi ada warning dengan StrictConcurrency
class SceneDelegate: UIResponder, UIWindowSceneDelegate {
    var window: UIWindow?
    
    func scene(_ scene: UIScene,
                willConnectTo session: UISceneSession,
                options: UIScene.ConnectionOptions) {
        guard let windowScene = scene as? UIWindowScene else { return }
        
        let repository = UserRepository()  // OK: actor, Sendable
        let viewModel = UserListViewModel(repository: repository)  // OK: @MainActor class
        
        let vc = UserListViewController(viewModel: viewModel)
        window = UIWindow(windowScene: windowScene)
        window?.rootViewController = UINavigationController(rootViewController: vc)
        window?.makeKeyAndVisible()
    }
}
```

---

### Use Case 3: SPM Package dengan Conditional Features

```swift
// Package.swift — library dengan feature flags untuk Swift 5/6
// swift-tools-version: 5.10

import PackageDescription

let swiftSettings: [SwiftSetting] = [
    // Base: kompatibel Swift 5
    .swiftLanguageVersion(.v5),
    
    // Opt-in fitur yang sudah stabil
    .enableUpcomingFeature("StrictConcurrency"),
    .enableUpcomingFeature("ExistentialAny"),
]

let package = Package(
    name: "DataPipeline",
    platforms: [.iOS(.v16), .macOS(.v13)],
    products: [
        .library(name: "DataPipeline", targets: ["DataPipeline"]),
        .library(name: "DataPipelineCompat", targets: ["DataPipelineCompat"]),
    ],
    targets: [
        // Core — Swift 6
        .target(
            name: "DataPipelineCore",
            swiftSettings: [.swiftLanguageVersion(.v6)]
        ),
        
        // Public API — Swift 5 + compat layer
        .target(
            name: "DataPipeline",
            dependencies: ["DataPipelineCore"],
            swiftSettings: swiftSettings
        ),
        
        // Compatibility wrapper untuk Swift 5 consumer
        .target(
            name: "DataPipelineCompat",
            dependencies: ["DataPipeline"],
            swiftSettings: [.swiftLanguageVersion(.v5)]
        ),
        
        // Tests — Swift 6 untuk catch issues lebih awal
        .testTarget(
            name: "DataPipelineTests",
            dependencies: ["DataPipeline"],
            swiftSettings: [.swiftLanguageVersion(.v6)]
        ),
    ]
)
```

---

## 9. Troubleshooting Error Umum

### Error 1: "Sending X risks causing data races"

```swift
// ❌ Error di Swift 6, Warning di Swift 5 + StrictConcurrency
class SharedData {
    var value = 0
}

let data = SharedData()
Task { data.value += 1 }  // Error: non-Sendable type

// ✅ Solusi A: jadikan struct (jika value semantics cukup)
struct SharedData {
    var value = 0
}

// ✅ Solusi B: jadikan actor
actor SharedData {
    var value = 0
    func increment() { value += 1 }
}

// ✅ Solusi C: @unchecked Sendable (jika sudah thread-safe manual)
final class SharedData: @unchecked Sendable {
    private let lock = NSLock()
    private var _value = 0
    var value: Int {
        get { lock.withLock { _value } }
        set { lock.withLock { _value = newValue } }
    }
}
```

### Error 2: "Main actor-isolated can not be referenced from nonisolated context"

```swift
// ❌ Error: mengakses @MainActor property dari background
class ViewController: UIViewController {
    var label: UILabel = UILabel()
}

Task.detached {
    viewController.label.text = "hello"  // Error!
}

// ✅ Solusi A: tandai ViewController sebagai @MainActor
@MainActor
class ViewController: UIViewController {
    var label: UILabel = UILabel()
}

// ✅ Solusi B: gunakan MainActor.run untuk crossing boundary
Task.detached {
    await MainActor.run {
        viewController.label.text = "hello"
    }
}

// ✅ Solusi C: gunakan Task biasa (mewarisi @MainActor dari context)
// Jika sudah dalam @MainActor context:
Task {  // bukan Task.detached
    label.text = "hello"  // OK: Task mewarisi isolation
}
```

### Error 3: "Protocol requires Sendable conformance"

```swift
// ❌ Error: protocol yang butuh Sendable tapi conforming type bukan Sendable
protocol Service: Sendable {
    func execute() async
}

class MyService: Service {  // Error: class bukan Sendable
    func execute() async { }
}

// ✅ Solusi A: gunakan actor
actor MyService: Service {
    func execute() async { }
}

// ✅ Solusi B: jadikan final class + @unchecked Sendable
final class MyService: Service, @unchecked Sendable {
    private let lock = NSLock()
    func execute() async { }
}

// ✅ Solusi C: hapus Sendable dari protocol jika tidak diperlukan
protocol Service {
    func execute() async
}
```

### Error 4: "Stored property cannot be declared @MainActor in non-isolated context"

```swift
// ❌ Masalah: property @MainActor di struct yang tidak @MainActor
struct AppState {
    @MainActor var currentUser: User?  // Error di Swift 6
}

// ✅ Solusi A: jadikan seluruh struct @MainActor
@MainActor
struct AppState {
    var currentUser: User?
}

// ✅ Solusi B: pindahkan ke class/actor yang sudah @MainActor
@MainActor
class AppState: ObservableObject {
    @Published var currentUser: User?
}
```

### Error 5: "`any` keyword required for existential"

```swift
// ❌ Error di Swift 6 (warning di Swift 5 + ExistentialAny)
let service: MyProtocol = ConcreteService()  // harus pakai 'any'

// ✅ Solusi: tambahkan 'any' keyword
let service: any MyProtocol = ConcreteService()

// Untuk conditional compilation:
#if swift(>=6.0) || hasFeature(ExistentialAny)
let service: any MyProtocol = ConcreteService()
#else
let service: MyProtocol = ConcreteService()
#endif
```

---

## 10. Ringkasan dan Checklist Migrasi

### Checklist Fase per Fase

**Fase 1 — Setup (Minggu 1)**
- [ ] Upgrade Xcode ke versi terbaru (Xcode 16+)
- [ ] Pastikan semua target masih `SWIFT_VERSION = 5`
- [ ] Build dan pastikan tidak ada error baru
- [ ] Aktifkan `Strict Concurrency = Minimal` untuk baseline

**Fase 2 — Observasi Warning (Minggu 2-3)**
- [ ] Naikkan `Strict Concurrency = Targeted`
- [ ] Catat semua warning yang muncul
- [ ] Kategorikan: mana yang mudah fix, mana yang kompleks
- [ ] Jangan fix dulu — pahami polanya dulu

**Fase 3 — Fix Layer per Layer (Minggu 4-8)**
- [ ] Models/Entities: jadikan `Sendable struct`
- [ ] Service/Worker: jadikan `actor`
- [ ] Repository: jadikan `actor` atau `@MainActor`
- [ ] ViewModel: anotasi `@MainActor`
- [ ] View/ViewController: anotasi `@MainActor`
- [ ] Callback/Delegate: refactor ke `async/await` atau `@MainActor`

**Fase 4 — Upgrade Target (Minggu 8-12)**
- [ ] Mulai dari target paling sedikit dependency
- [ ] Set `SWIFT_VERSION = 6` satu target pada satu waktu
- [ ] Fix error yang muncul (warning sudah jadi error)
- [ ] Lanjut ke target berikutnya setelah yang sebelumnya green

**Fase 5 — Full Swift 6 (Bulan 3+)**
- [ ] Semua target sudah Swift 6
- [ ] Hapus `#if swift(<6.0)` compatibility blocks yang tidak lagi diperlukan
- [ ] Refactor `@unchecked Sendable` ke implementasi yang proper
- [ ] Aktifkan Swift Testing untuk test baru

### Quick Reference Directive

```swift
// Guard berdasarkan:
#if swift(>=6.0)           // Language version (SWIFT_VERSION build setting)
#if compiler(>=6.0)        // Toolchain version (Xcode version)
#if canImport(Module)      // Ketersediaan module di platform
#if hasFeature(Feature)    // Fitur language spesifik
#available(iOS 18, *)      // OS version (runtime check)
@available(iOS 18, *)      // OS version (declaration annotation)
```

```swift
// Package.swift settings:
.swiftLanguageVersion(.v5)                       // tetap Swift 5
.swiftLanguageVersion(.v6)                       // upgrade ke Swift 6
.enableUpcomingFeature("StrictConcurrency")      // opt-in warning concurrency
.enableUpcomingFeature("TypedThrows")            // opt-in typed throws
.enableUpcomingFeature("ExistentialAny")         // opt-in any keyword
```

```
// Xcode Build Settings:
SWIFT_VERSION = 5                                // tetap Swift 5
SWIFT_VERSION = 6                                // Swift 6 mode
SWIFT_STRICT_CONCURRENCY = minimal               // warning minimal
SWIFT_STRICT_CONCURRENCY = targeted              // warning relevan
SWIFT_STRICT_CONCURRENCY = complete              // semua warning Swift 6
```

---

*Dibuat: 2026-05-09 | Swift 5 → Swift 6 Migration | Xcode 16+*
