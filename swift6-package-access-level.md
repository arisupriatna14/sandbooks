# Swift 6: Package Access Level
### Teori Mendalam, Real Use Case, dan Panduan Penggunaan

---

## Daftar Isi
1. [Masalah yang Dipecahkan](#1-masalah-yang-dipecahkan)
2. [Teori: Access Control di Swift](#2-teori-access-control-di-swift)
3. [Sintaks dan Cara Kerja](#3-sintaks-dan-cara-kerja)
4. [Kapan Harus Digunakan](#4-kapan-harus-digunakan)
5. [Kapan Tidak Digunakan](#5-kapan-tidak-digunakan)
6. [Real Use Cases](#6-real-use-cases)
7. [Ringkasan](#7-ringkasan)

---

## 1. Masalah yang Dipecahkan

### Gap antara `internal` dan `public`

Sebelum Swift 6, Swift punya 5 access level: `private`, `fileprivate`, `internal`, `public`, `open`. Ada gap besar antara `internal` dan `public` yang sangat terasa ketika membangun SDK atau framework multi-modul.

**Skenario: Library multi-modul dengan Swift Package Manager**

```
MySDK (Swift Package)
├── Core/          (Module A)
│   ├── Networking.swift
│   └── Authentication.swift
├── Analytics/     (Module B)  
│   └── Tracker.swift
└── UI/            (Module C)
    └── Components.swift
```

Masalah: `Analytics` butuh mengakses fungsi internal `Core`, tapi kamu tidak ingin fungsi itu accessible oleh pengguna SDK.

```swift
// Core/Networking.swift
public func fetchData() { ... }     // terlalu public — user bisa pakai ini
internal func _internalHelper() { } // tidak bisa dipakai oleh Analytics!

// Analytics/Tracker.swift
// _internalHelper() tidak visible dari sini karena berbeda module
// Terpaksa jadikan public:
// public func _internalHelper() { }  // ❌ "public" tapi naming dengan _ = kebingungan
```

**Dilema klasik:**
- Jadikan `internal` → tidak bisa dipakai module lain dalam paket yang sama
- Jadikan `public` → accessible oleh SEMUA pengguna, termasuk end-user yang tidak seharusnya tahu
- Solusi sebelumnya: naming convention `_` prefix (tidak enforced compiler), `@_spi` (unofficial), atau modul monolith yang besar

**Dengan `package`:**
```swift
// Core/Networking.swift
package func fetchData() { }  // visible ke semua module dalam satu Swift Package

// Analytics/Tracker.swift — module berbeda, PAKET YANG SAMA
import Core
fetchData()  // ✅ OK: package access level
```

---

## 2. Teori: Access Control di Swift

### Hierarchy Access Control

Swift 6 kini punya 6 level, dari paling terbatas ke paling luas:

```
private      → hanya dalam declaration + extension di file yang sama
fileprivate  → seluruh file
internal     → seluruh module (default)
package      → seluruh Swift Package (NEW in Swift 6)
public       → accessible dari luar, tapi tidak bisa di-subclass/override dari luar
open         → accessible dan bisa di-subclass/override dari luar
```

### Apa itu "Package" dalam Konteks ini?

Dalam Swift Package Manager, sebuah **package** adalah unit distribusi yang didefinisikan oleh `Package.swift`. Satu package bisa punya beberapa **targets** (yang menjadi modules terpisah).

```swift
// Package.swift
let package = Package(
    name: "MySDK",
    targets: [
        .target(name: "Core"),       // Module 1
        .target(name: "Analytics",   // Module 2
            dependencies: ["Core"]),
        .target(name: "UI",          // Module 3
            dependencies: ["Core"]),
    ]
)
```

`package` access level berarti: **visible ke semua targets dalam satu `Package.swift`**.

### Boundary yang Jelas

```
Consumer App ──────────────────────────────────────────────────────
│  (tidak bisa akses `package` symbols dari MySDK)                 │
└──────────────────────────────────────────────────────────────────┘
                           ↕ public / open only
┌──────────────────────────────────────────────────────────────────┐
│  MySDK Package                                                   │
│  ┌─────────────┐   package   ┌───────────────┐                  │
│  │    Core     │ ←─────────→ │   Analytics   │                  │
│  │   Module    │   access    │    Module     │                  │
│  └─────────────┘             └───────────────┘                  │
│         ↕ package                   ↕ package                   │
│  ┌─────────────┐                                                 │
│  │     UI      │                                                 │
│  │   Module    │                                                 │
│  └─────────────┘                                                 │
└──────────────────────────────────────────────────────────────────┘
```

---

## 3. Sintaks dan Cara Kerja

### Deklarasi

```swift
// Variabel
package var internalConfig = Configuration()

// Fungsi
package func setupDatabase() { }

// Class / Struct / Enum
package class PackageRouter { }
package struct PackageConfiguration { }
package enum PackageState { case ready, loading, error }

// Protocol
package protocol PackageServiceProtocol { }

// Extension
package extension String {
    func trimmedAndLowercased() -> String {
        trimmingCharacters(in: .whitespaces).lowercased()
    }
}

// Init
package class Service {
    package init(config: Configuration) { }
    package var config: Configuration
    package func start() { }
    
    // Internal — hanya dalam module ini
    func internalOperation() { }
}
```

### Mengaktifkan di Package.swift

```swift
// Package.swift — tidak perlu setting khusus untuk Swift 6
// Package access level otomatis tersedia

let package = Package(
    name: "MyApp",
    targets: [
        .target(
            name: "Core",
            swiftSettings: [
                .swiftLanguageVersion(.v6)  // aktifkan Swift 6
            ]
        ),
        .target(
            name: "Features",
            dependencies: ["Core"],
            swiftSettings: [
                .swiftLanguageVersion(.v6)
            ]
        )
    ]
)
```

### Package vs Internal: Perbedaan

```swift
// Module A (dalam Package yang sama)
public class PublicClass {
    public var publicProp = 0       // accessible dari mana saja
    package var packageProp = 0     // accessible dari semua module dalam package ini
    internal var internalProp = 0   // hanya dalam Module A
    private var privateProp = 0     // hanya dalam class ini
}

// Module B (dalam Package yang sama)
import ModuleA

let obj = PublicClass()
print(obj.publicProp)    // ✅ OK
print(obj.packageProp)   // ✅ OK (same package)
print(obj.internalProp)  // ❌ ERROR (different module)

// Consumer App (di luar Package)
import ModuleA

let obj = PublicClass()
print(obj.publicProp)    // ✅ OK
print(obj.packageProp)   // ❌ ERROR (different package)
```

---

## 4. Kapan Harus Digunakan

### Gunakan `package` ketika:

**1. Cross-module helpers dalam satu SDK/library**
```swift
// Core module: helper yang dibutuhkan Analytics tapi bukan end-user API
package extension URLRequest {
    package func addAuthHeaders(_ token: String) -> URLRequest {
        var request = self
        request.setValue("Bearer \(token)", forHTTPHeaderField: "Authorization")
        return request
    }
}

// Analytics module: pakai helper ini
import Core
let authenticated = request.addAuthHeaders(token)  // ✅ OK
```

**2. Shared types antar feature modules**
```swift
// SharedTypes module (dalam paket yang sama)
package struct AnalyticsEvent {
    package let name: String
    package let properties: [String: String]
    package let timestamp: Date
    
    package init(name: String, properties: [String: String] = [:]) {
        self.name = name
        self.properties = properties
        self.timestamp = Date()
    }
}

// UserFeature module
import SharedTypes
let event = AnalyticsEvent(name: "user_login")  // ✅ OK

// Consumer App: tidak bisa akses AnalyticsEvent langsung
// AnalyticsEvent adalah implementation detail dari SDK
```

**3. Testing yang butuh akses ke internal implementation**
```swift
// Core module: implementation detail
package class RequestBuilder {
    package var baseURL: URL
    package var headers: [String: String] = [:]
    
    package init(baseURL: URL) { self.baseURL = baseURL }
    package func build() -> URLRequest { URLRequest(url: baseURL) }
}

// CoreTests target (dalam paket yang sama):
import Core
// Bisa test RequestBuilder secara detail tanpa menjadikannya public
let builder = RequestBuilder(baseURL: testURL)
builder.headers["X-Test"] = "true"
let request = builder.build()
```

**4. Plugin/extension point antar module**
```swift
// PluginCore module
package protocol FeaturePlugin {
    package var identifier: String { get }
    package func activate(with context: PluginContext)
}

package class PluginRegistry {
    package static let shared = PluginRegistry()
    package func register(_ plugin: any FeaturePlugin) { }
}

// FeatureA module
import PluginCore

struct FeatureAPlugin: FeaturePlugin {
    package var identifier = "com.app.featureA"
    package func activate(with context: PluginContext) { }
}

// Registration — tanpa end-user bisa melakukan hal yang sama
PluginRegistry.shared.register(FeatureAPlugin())
```

---

## 5. Kapan Tidak Digunakan

### Hindari `package` ketika:

**1. Single-module project (app tanpa library)**
```swift
// App target biasa — hanya satu module
// 'package' = 'internal' di sini karena tidak ada module lain dalam "package"
// Gunakan 'internal' saja — lebih jelas intent-nya

// JANGAN: penggunaan tidak berguna
package class ViewModel { }  // sama saja dengan internal di single-module app

// GUNAKAN: internal (default)
class ViewModel { }  // otomatis internal
```

**2. API yang memang harus public**
```swift
// Jika end-user SDK perlu menggunakan type ini, harus public
// 'package' menyembunyikannya dari end-user

// JANGAN jika ini adalah public API:
package class UserProfile { }  // end-user tidak bisa pakai!

// GUNAKAN: public
public class UserProfile { }
```

**3. Implementation detail yang hanya relevan dalam satu module**
```swift
// Ini cukup internal — tidak perlu diakses dari module lain dalam paket

// JANGAN: terlalu luas
package func validateEmail(_ email: String) -> Bool { }  // tidak perlu lintas module

// GUNAKAN: internal (atau private)
func validateEmail(_ email: String) -> Bool { }  // default internal
```

**4. Ketika package boundary tidak sesuai dengan logical boundary**
```swift
// Hati-hati: "package" berarti SEMUA module dalam Package.swift
// Jika package-mu sangat besar, ini mungkin terlalu luas

// Lebih baik rancang module boundary yang tepat daripada
// menggunakan 'package' sebagai workaround untuk design yang buruk
```

---

## 6. Real Use Cases

### Use Case 1: SDK Multi-Modul — Networking + Analytics

```
NetworkSDK (Package.swift)
├── NetworkCore/     — request, response, auth
├── Caching/         — response cache
├── Analytics/       — telemetry dan logging
└── NetworkUI/       — loading indicators, retry UI
```

```swift
// NetworkCore/Authentication.swift
// package: dibutuhkan oleh Analytics untuk attach auth info ke events
package struct AuthContext {
    package let userID: String?
    package let sessionID: String
    package let deviceID: String
    
    package static var current: AuthContext {
        AuthContext(
            userID: AuthStore.shared.currentUserID,
            sessionID: SessionManager.shared.sessionID,
            deviceID: DeviceIdentifier.current
        )
    }
}

// NetworkCore/RequestPipeline.swift
// package: Analytics perlu hook ke pipeline untuk record timing
package protocol RequestMiddleware {
    package func process(_ request: URLRequest) async throws -> URLRequest
    package func process(_ response: HTTPURLResponse, data: Data) async throws -> Data
}

package class RequestPipeline {
    package var middlewares: [any RequestMiddleware] = []
    
    package func execute(_ request: URLRequest) async throws -> (Data, HTTPURLResponse) {
        var mutableRequest = request
        for middleware in middlewares {
            mutableRequest = try await middleware.process(mutableRequest)
        }
        // ... execute request
        return (Data(), HTTPURLResponse())
    }
}

// Analytics/NetworkAnalyticsMiddleware.swift
import NetworkCore

// Analytics bisa inject middleware ke dalam pipeline — tanpa end-user tahu cara ini
package class NetworkAnalyticsMiddleware: RequestMiddleware {
    package init() {}
    
    package func process(_ request: URLRequest) async throws -> URLRequest {
        let auth = AuthContext.current  // ✅ package access — sama paket
        recordRequestStart(url: request.url, userID: auth.userID)
        return request
    }
    
    package func process(_ response: HTTPURLResponse, data: Data) async throws -> Data {
        recordRequestEnd(statusCode: response.statusCode, size: data.count)
        return data
    }
    
    private func recordRequestStart(url: URL?, userID: String?) {
        print("Request started: \(url?.absoluteString ?? "unknown")")
    }
    
    private func recordRequestEnd(statusCode: Int, size: Int) {
        print("Request ended: \(statusCode), \(size) bytes")
    }
}

// NetworkSDK/Setup.swift — entry point yang public
import NetworkCore
import Analytics

public class NetworkSDK {
    public static let shared = NetworkSDK()
    
    private let pipeline = RequestPipeline()
    
    private init() {
        // Setup internal — end-user tidak perlu tahu ini
        pipeline.middlewares.append(NetworkAnalyticsMiddleware())
    }
    
    // Ini yang public — clean API untuk end-user
    public func fetch(url: URL) async throws -> Data {
        let request = URLRequest(url: url)
        let (data, _) = try await pipeline.execute(request)
        return data
    }
}
```

---

### Use Case 2: Shared Test Infrastructure

```
AppTests (Test Targets dalam satu Package)
├── CoreTests/
├── FeatureTests/
└── TestHelpers/     — shared test utilities
```

```swift
// TestHelpers/MockFactory.swift
package class MockFactory {
    package static func makeUser(
        id: String = "test_user",
        name: String = "Test User",
        role: UserRole = .basic
    ) -> User {
        User(id: id, name: name, role: role)
    }
    
    package static func makeOrder(
        id: String = "test_order",
        items: [OrderItem] = [makeOrderItem()]
    ) -> Order {
        Order(id: id, items: items, createdAt: Date())
    }
    
    package static func makeOrderItem(
        productID: String = "prod_1",
        quantity: Int = 1,
        price: Decimal = 10.0
    ) -> OrderItem {
        OrderItem(productID: productID, quantity: quantity, price: price)
    }
}

// TestHelpers/AssertionHelpers.swift
import Testing

// Custom assertion helpers yang di-share semua test targets
package func assertUserEquals(
    _ actual: User,
    _ expected: User,
    sourceLocation: SourceLocation = #_sourceLocation
) {
    #expect(actual.id == expected.id, sourceLocation: sourceLocation)
    #expect(actual.name == expected.name, sourceLocation: sourceLocation)
    #expect(actual.role == expected.role, sourceLocation: sourceLocation)
}

// CoreTests/UserServiceTests.swift
import TestHelpers
import Testing

@Suite("UserService")
struct UserServiceTests {
    @Test
    func createUser() async throws {
        let input = MockFactory.makeUser(name: "Ari")  // ✅ package access
        let created = try await UserService().create(input)
        assertUserEquals(created, input)  // ✅ package access
    }
}

// FeatureTests/CheckoutTests.swift
import TestHelpers
import Testing

@Suite("Checkout")
struct CheckoutTests {
    @Test
    func checkout() async throws {
        let user = MockFactory.makeUser(role: .premium)  // ✅ re-use mock factory
        let order = MockFactory.makeOrder()
        // ...
    }
}
```

---

### Use Case 3: Feature Flags System

```swift
// FeatureFlags module (dalam SDK package)
package class FeatureFlagRegistry {
    package static let shared = FeatureFlagRegistry()
    
    private var flags: [String: Bool] = [:]
    
    package func register(flag: String, defaultValue: Bool) {
        flags[flag] = defaultValue
    }
    
    package func isEnabled(_ flag: String) -> Bool {
        flags[flag] ?? false
    }
    
    // Internal ke SDK — tidak untuk end-user
    package func override(flag: String, value: Bool) {
        flags[flag] = value
    }
}

// Feature modules — register flags saat init
// FeatureA module
import FeatureFlags

package enum FeatureAFlags {
    package static let newOnboarding = "feature_a.new_onboarding"
    package static let betaCheckout = "feature_a.beta_checkout"
}

package func setupFeatureA() {
    FeatureFlagRegistry.shared.register(flag: FeatureAFlags.newOnboarding, defaultValue: false)
    FeatureFlagRegistry.shared.register(flag: FeatureAFlags.betaCheckout, defaultValue: false)
}

// Public entry point — end-user hanya perlu ini
public class AppSDK {
    public static func initialize() {
        setupFeatureA()      // package — end-user tidak bisa panggil langsung
        setupFeatureB()
        setupFeatureC()
    }
    
    // End-user API: hanya bisa cek flag via nama yang di-expose
    public func isFeatureEnabled(_ name: PublicFeatureFlag) -> Bool {
        FeatureFlagRegistry.shared.isEnabled(name.rawValue)
    }
}

public enum PublicFeatureFlag: String {
    case newOnboarding = "feature_a.new_onboarding"
    // Hanya expose flag yang memang boleh dikontrol end-user
}
```

---

## 7. Ringkasan

### Hierarchy Lengkap Swift Access Control

```
┌─────────────────────────────────────────────────────────────────────┐
│ open           — subclassable dari luar module                      │
├─────────────────────────────────────────────────────────────────────┤
│ public         — accessible dari luar package                       │
├─────────────────────────────────────────────────────────────────────┤
│ package ★ NEW  — accessible semua module dalam satu Package.swift   │
├─────────────────────────────────────────────────────────────────────┤
│ internal       — accessible dalam satu module (DEFAULT)             │
├─────────────────────────────────────────────────────────────────────┤
│ fileprivate    — accessible dalam satu file                         │
├─────────────────────────────────────────────────────────────────────┤
│ private        — accessible dalam declaration + same-file extension │
└─────────────────────────────────────────────────────────────────────┘
```

### Decision Guide

```
Siapa yang perlu mengakses ini?
├── Hanya deklarasi saat ini → private
├── Hanya file saat ini → fileprivate
├── Hanya module saat ini → internal (default, tidak perlu tulis)
├── Semua module dalam Package.swift yang sama → package ★
├── Pengguna library/SDK tapi tidak bisa di-subclass → public
└── Pengguna library/SDK dan bisa di-subclass → open
```

### Kapan `package` vs Alternatif

| Situasi | Rekomendasi |
|---|---|
| Cross-module helper dalam satu SDK | `package` |
| End-user facing API | `public` / `open` |
| Test helper yang di-share antar test target | `package` |
| Implementation detail satu module | `internal` (default) |
| Single-module app | `internal` (package tidak berguna) |
| Workaround untuk desain modul yang buruk | Desain ulang modul-nya |

---

*Dibuat: 2026-05-08 | Swift 6.0 | Swift Package Manager*
