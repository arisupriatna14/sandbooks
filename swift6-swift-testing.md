# Swift 6: Swift Testing Framework
### Teori Mendalam, Real Use Case, dan Panduan Penggunaan

---

## Daftar Isi
1. [Masalah yang Dipecahkan](#1-masalah-yang-dipecahkan)
2. [Teori: Desain Swift Testing](#2-teori-desain-swift-testing)
3. [@Test dan @Suite — Dasar](#3-test-dan-suite--dasar)
4. [Expectations — #expect dan #require](#4-expectations--expect-dan-require)
5. [Parameterized Tests](#5-parameterized-tests)
6. [Tags dan Traits](#6-tags-dan-traits)
7. [Async Testing](#7-async-testing)
8. [Kapan Harus Digunakan](#8-kapan-harus-digunakan)
9. [Kapan Tidak Digunakan](#9-kapan-tidak-digunakan)
10. [Real Use Cases](#10-real-use-cases)
11. [Migrasi dari XCTest](#11-migrasi-dari-xctest)
12. [Ringkasan](#12-ringkasan)

---

## 1. Masalah yang Dipecahkan

### XCTest: Warisan Objective-C yang Mulai Menua

XCTest didesain pada era Objective-C dan membawa banyak pattern dari sana ke Swift, menciptakan beberapa friction:

**1. Harus pakai class dan mewarisi XCTestCase**
```swift
// XCTest: boilerplate inheritance dan class requirement
class UserServiceTests: XCTestCase {
    var sut: UserService!  // harus var + implicitly unwrapped Optional
    
    override func setUp() { super.setUp()  // harus panggil super
        sut = UserService()
    }
    
    override func tearDown() {
        sut = nil
        super.tearDown()  // harus panggil super
    }
    
    func testLogin() {  // harus prefix 'test' — konvensi runtime, bukan compile-time
        // ...
    }
}
```

**2. Assertion messages yang buruk**
```swift
// XCTest: pesan error tidak informatif
XCTAssertEqual(user.name, "Ari Supriatna")
// Gagal: XCTAssertEqual failed: ("Ari") is not equal to ("Ari Supriatna")
// Tidak ada konteks: di mana? Nilai yang ada apa?

// Swift Testing: pesan yang kaya informasi
#expect(user.name == "Ari Supriatna")
// Gagal: Expectation failed: (user.name → "Ari") == "Ari Supriatna"
// Jelas: kita tahu nilai aktual dan yang diharapkan
```

**3. Tidak ada parameterized test built-in**
```swift
// XCTest: harus tulis manual atau pakai library eksternal
func testValidateEmail_valid() { XCTAssertTrue(validate("a@b.com")) }
func testValidateEmail_noAt() { XCTAssertFalse(validate("ab.com")) }
func testValidateEmail_noDomain() { XCTAssertFalse(validate("a@")) }
// Banyak fungsi, banyak duplikasi

// Swift Testing: satu fungsi dengan banyak input
@Test("Email validation", arguments: ["a@b.com", "x@y.z", "user@domain.org"])
func validateValidEmails(email: String) {
    #expect(isValidEmail(email))
}
```

**4. Async testing yang verbose**
```swift
// XCTest: expectation + wait + timeout
func testAsyncFetch() {
    let expectation = self.expectation(description: "fetch completes")
    service.fetch { result in
        XCTAssertNotNil(result)
        expectation.fulfill()
    }
    wait(for: [expectation], timeout: 5.0)
}

// Swift Testing: native async/await
@Test
func fetchUser() async throws {
    let user = try await service.fetch(id: "123")
    #expect(user.id == "123")
}
```

---

## 2. Teori: Desain Swift Testing

### Prinsip Desain

Swift Testing didesain dengan prinsip-prinsip:

1. **Macro-based** — menggunakan Swift macros (`@Test`, `#expect`) alih-alih runtime introspection
2. **Value-oriented** — test bisa berupa `struct` atau `class`, tidak harus inherit
3. **Parallel by default** — test dijalankan secara paralel kecuali ada hambatan eksplisit
4. **Expressive failures** — pesan error yang kaya dengan nilai aktual dan ekspresi aslinya
5. **Composable traits** — behavior test dikonfigurasi melalui traits, bukan override method

### Cara Kerja Macro `@Test`

`@Test` adalah Swift macro yang di-expand oleh compiler. Secara konseptual:

```swift
// Kamu tulis:
@Test("Login berhasil")
func loginSucceeds() async throws { ... }

// Compiler expand menjadi:
// (Simplistik — implementasi actual lebih kompleks)
func loginSucceeds() async throws { ... }
Testing.__registerTest(
    name: "Login berhasil",
    function: loginSucceeds,
    sourceLocation: .init(file: #file, line: #line)
)
```

Ini berbeda dari XCTest yang menggunakan **Objective-C runtime introspection** untuk menemukan `func test*()` — Swift Testing menggunakan compile-time registration. Artinya:
- Lebih cepat (tidak perlu scan runtime)
- Lebih portable (tidak bergantung Objective-C runtime)
- Bisa berjalan di Linux dan Windows

### Cara Kerja `#expect`

`#expect` adalah expression macro yang di-expand untuk menangkap informasi lengkap tentang ekspresi:

```swift
// Kamu tulis:
#expect(user.age > 18 && user.isVerified)

// Compiler menangkap sub-expressions:
// - user.age → nilai aktual (misalnya 15)
// - user.isVerified → nilai aktual (misalnya true)
// - user.age > 18 → false
// - seluruh ekspresi → false

// Pesan gagal:
// Expectation failed: (user.age → 15) > 18 && (user.isVerified → true)
// Jelas terlihat bahwa 'age' yang jadi masalah, bukan 'isVerified'
```

---

## 3. @Test dan @Suite — Dasar

### @Test

```swift
import Testing

// Minimal: nama diambil dari nama fungsi
@Test
func addTwoNumbers() {
    let result = Calculator.add(2, 3)
    #expect(result == 5)
}

// Dengan display name kustom
@Test("Kalkulasi persentase dengan pembulatan")
func percentageCalculation() {
    let result = Calculator.percentage(of: 33.33, from: 100)
    #expect(result ≈ 33.33)  // custom operator atau approximate comparison
}

// Async test — native, tidak perlu expectation
@Test("Fetch user dari API")
func fetchUser() async throws {
    let user = try await UserService().fetch(id: "user_123")
    #expect(user.id == "user_123")
    #expect(!user.name.isEmpty)
}

// Test yang diharapkan gagal (known bug/todo)
@Test("Fitur X belum diimplementasi", .disabled("Menunggu backend API"))
func featureX() {
    // Test ini di-skip
}
```

### @Suite

```swift
// Suite mengelompokkan test yang terkait
// Bisa struct, class, atau enum — tidak harus inherit apapun!
@Suite("Authentication")
struct AuthTests {
    // Properti bisa diinisialisasi langsung — tidak perlu setUp()
    let service = AuthService(environment: .testing)
    
    @Test("Login dengan kredensial valid")
    func loginWithValidCredentials() async throws {
        let session = try await service.login(username: "ari", password: "secret123")
        #expect(session.isValid)
        #expect(session.userId == "user_123")
    }
    
    @Test("Login dengan password salah")
    func loginWithWrongPassword() async throws {
        await #expect(throws: AuthError.invalidCredentials) {
            try await service.login(username: "ari", password: "wrong")
        }
    }
    
    // Suite bersarang
    @Suite("Session Management")
    struct SessionTests {
        @Test("Session expire setelah timeout")
        func sessionExpires() async throws {
            // ...
        }
    }
}
```

### Init/Deinit sebagai setUp/tearDown

```swift
@Suite("Database Tests")
struct DatabaseTests {
    let db: TestDatabase
    
    // init() adalah setUp — dijalankan sebelum setiap test
    init() async throws {
        db = try await TestDatabase.create()
        try await db.migrate()
    }
    
    // deinit adalah tearDown — dijalankan setelah setiap test
    deinit {
        db.destroyAll()  // cleanup
    }
    
    @Test
    func insertUser() async throws {
        try await db.insert(User(name: "Ari"))
        let users = try await db.fetchAll(User.self)
        #expect(users.count == 1)
    }
}
```

---

## 4. Expectations — #expect dan #require

### `#expect` — Soft Assertion (Test Lanjut jika Gagal)

```swift
@Test
func userProfile() async throws {
    let user = try await fetchUser(id: "123")
    
    // Test LANJUT meskipun salah satu #expect gagal
    #expect(user.id == "123")          // cek, lanjut
    #expect(!user.name.isEmpty)        // cek, lanjut
    #expect(user.email.contains("@"))  // cek, lanjut
    // Semua failure dikumpulkan — satu run, banyak laporan
}
```

### `#require` — Hard Assertion (Stop jika Gagal)

```swift
@Test
func processOrder() async throws {
    let order = try await fetchOrder(id: "ORD-001")
    
    // Jika ini gagal, test BERHENTI (throw)
    let lineItems = try #require(order.lineItems)
    // lineItems di sini pasti non-nil (unwrapped)
    
    #expect(lineItems.count > 0)
    #expect(lineItems.first?.price ?? 0 > 0)
}

// #require dengan Optional unwrapping
@Test
func configurationLoaded() throws {
    let config = loadConfiguration()
    let apiKey = try #require(config.apiKey)  // unwrap atau stop
    #expect(apiKey.count == 32)
}
```

### Mengharapkan Error

```swift
@Test
func divisionByZero() {
    // #expect(throws:) — verifikasi bahwa throw terjadi
    #expect(throws: MathError.divisionByZero) {
        _ = try Calculator.divide(10, by: 0)
    }
}

// Dengan inspeksi error
@Test
func networkFailure() async {
    await #expect(throws: NetworkError.self) {
        try await NetworkClient().fetch(badURL)
    }
}

// Ekspektasi error dengan kondisi lebih spesifik
@Test
func rateLimitError() async throws {
    let error = try await #require(throws: APIError.self) {
        try await client.fetch(rateLimitedEndpoint)
    }
    // error sekarang adalah APIError yang sudah di-unwrap
    guard case .rateLimited(let retryAfter) = error else {
        Issue.record("Expected rateLimited error")
        return
    }
    #expect(retryAfter > 0)
}
```

---

## 5. Parameterized Tests

### Argumen Sederhana

```swift
// Satu parameter — test dijalankan untuk setiap nilai
@Test("Validate email format", arguments: [
    "user@example.com",
    "name.surname@domain.org",
    "user+tag@mail.co.id"
])
func validEmailFormats(email: String) {
    #expect(EmailValidator.isValid(email))
}

@Test("Reject invalid emails", arguments: [
    "notanemail",
    "@domain.com",
    "user@",
    "",
    "user @domain.com"  // spasi tidak valid
])
func invalidEmailFormats(email: String) {
    #expect(!EmailValidator.isValid(email))
}
```

### Multiple Parameters dengan `zip`

```swift
// Dua array yang di-zip — pasangan (input, expected)
@Test(
    "Temperature conversion",
    arguments: zip(
        [0.0, 100.0, -40.0, 37.0],    // Celsius
        [32.0, 212.0, -40.0, 98.6]    // Fahrenheit
    )
)
func celsiusToFahrenheit(celsius: Double, expectedFahrenheit: Double) {
    let result = Temperature.celsiusToFahrenheit(celsius)
    #expect(abs(result - expectedFahrenheit) < 0.01)
}
```

### Cartesian Product (semua kombinasi)

```swift
// Tanpa zip = cartesian product — semua kombinasi
@Test(
    "HTTP methods dan endpoints",
    arguments: ["GET", "POST", "PUT", "DELETE"],
             ["/users", "/orders", "/products"]
)
func apiEndpointAccessControl(method: String, path: String) async throws {
    let response = try await client.request(method: method, path: path)
    // Test bahwa auth bekerja untuk semua kombinasi
    #expect(response.statusCode != 401)
}
// Total: 4 × 3 = 12 test cases — semua dijalankan paralel!
```

### Argumen dengan Custom Type

```swift
struct TestCase {
    let input: String
    let expected: Int
    let description: String
}

@Test("Parse integer", arguments: [
    TestCase(input: "42", expected: 42, description: "positif"),
    TestCase(input: "-10", expected: -10, description: "negatif"),
    TestCase(input: "0", expected: 0, description: "nol"),
])
func parseInteger(_ testCase: TestCase) throws {
    let result = try parseInt(testCase.input)
    #expect(result == testCase.expected, "\(testCase.description) gagal")
}
```

---

## 6. Tags dan Traits

### Mendefinisikan Tags

```swift
// Tags untuk kategorisasi
extension Tag {
    @Tag static var critical: Self
    @Tag static var network: Self
    @Tag static var ui: Self
    @Tag static var slow: Self
    @Tag static var regression: Self
}

// Gunakan di test
@Test("Payment processing", .tags(.critical, .network))
func processPayment() async throws {
    // ...
}

@Test("User list rendering", .tags(.ui))
func userListRenders() {
    // ...
}
```

### Traits Lainnya

```swift
// .disabled: skip test dengan alasan
@Test("Feature flag integration", .disabled("Menunggu feature flag service"))
func featureFlagTest() { }

// .bug: link ke bug tracker
@Test("Handle null response", .bug("https://github.com/org/repo/issues/42"))
func nullResponseHandling() async throws {
    // Test untuk bug yang sudah difix
}

// .timeLimit: batas waktu eksekusi
@Test("Cache warming", .timeLimit(.minutes(2)))
func cacheWarmingCompletes() async throws {
    try await cacheService.warmUp()
}

// .serialized: jalankan secara serial (tidak paralel)
@Suite("Integration Tests", .serialized)
struct IntegrationTests {
    // Test di suite ini tidak dijalankan paralel satu sama lain
    // (tapi masih paralel dengan suite lain)
}
```

### Menjalankan Test Berdasarkan Tag

```bash
# Di command line:
swift test --filter ".tags(.critical)"
swift test --filter ".tags(.network)"
swift test --skip ".tags(.slow)"

# Di Xcode: gunakan Test Plan untuk filter berdasarkan tag
```

---

## 7. Async Testing

### Native async/await

```swift
@Test("Fetch dan cache user")
func fetchAndCacheUser() async throws {
    let service = UserService()
    
    // Pertama fetch — dari network
    let user1 = try await service.fetchUser(id: "123")
    #expect(user1.id == "123")
    
    // Kedua fetch — dari cache
    let user2 = try await service.fetchUser(id: "123")
    #expect(user2.id == user1.id)
    
    // Verifikasi sumber
    #expect(service.lastFetchSource == .cache)
}
```

### Menguji Concurrent Code

```swift
@Test("Concurrent requests tidak menyebabkan data race")
func concurrentRequests() async throws {
    let counter = Counter()
    
    // Jalankan 100 increment secara bersamaan
    await withTaskGroup(of: Void.self) { group in
        for _ in 0..<100 {
            group.addTask {
                await counter.increment()
            }
        }
    }
    
    let finalCount = await counter.value
    #expect(finalCount == 100)
}
```

### Menguji AsyncStream dan AsyncSequence

```swift
@Test("EventBus menerima events")
func eventBusReceivesEvents() async throws {
    let bus = EventBus<String>()
    var received: [String] = []
    
    // Subscribe
    let (_, stream) = await bus.subscribe()
    
    // Publish dalam task terpisah
    let publishTask = Task {
        await bus.publish("event1")
        await bus.publish("event2")
        await bus.publish("event3")
    }
    
    // Collect events (dengan timeout implisit dari .timeLimit trait)
    for await event in stream.prefix(3) {
        received.append(event)
    }
    
    publishTask.cancel()
    
    #expect(received == ["event1", "event2", "event3"])
}
```

---

## 8. Kapan Harus Digunakan

### Gunakan Swift Testing untuk:

**1. Semua unit test baru**
```swift
// Default choice untuk test baru di Swift 6+
@Test("Business logic")
func calculateDiscount() {
    let discount = PricingEngine.discount(for: .premium, amount: 100)
    #expect(discount == 20)
}
```

**2. Test dengan banyak kombinasi input**
```swift
// Parameterized test jauh lebih expressive dari XCTest
@Test("Discount tiers", arguments: [
    (tier: UserTier.basic, amount: 100.0, expectedDiscount: 5.0),
    (tier: .premium, amount: 100.0, expectedDiscount: 20.0),
    (tier: .vip, amount: 100.0, expectedDiscount: 35.0),
])
func discountByTier(tier: UserTier, amount: Double, expected: Double) {
    #expect(PricingEngine.discount(for: tier, amount: amount) == expected)
}
```

**3. Test yang butuh grouping dan tagging yang baik**
```swift
@Suite("Payment", .tags(.critical))
struct PaymentTests { ... }
```

**4. Test di Swift Package (cross-platform)**
```swift
// Swift Testing berjalan di macOS, iOS, Linux, Windows
// XCTest tidak berjalan di Linux/Windows
```

---

## 9. Kapan Tidak Digunakan

### Tetap Gunakan XCTest untuk:

**1. UI Testing (XCUITest)**
```swift
// Swift Testing tidak punya pengganti untuk XCUITest
class UITests: XCTestCase {
    func testLoginFlow() {
        let app = XCUIApplication()
        app.launch()
        // ... UI interactions
    }
}
```

**2. Performance Testing**
```swift
// XCTest punya measure {} yang belum ada di Swift Testing
func testParsingPerformance() {
    measure {
        _ = parser.parse(largeDataset)
    }
}
```

**3. Test yang harus integrate dengan CI yang hanya support XCTest**
```swift
// Beberapa CI tools dan coverage tools belum support Swift Testing sepenuhnya
// Cek kompatibilitas toolchain sebelum migrasi total
```

**4. Codebase yang harus support Swift 5.9 atau lebih lama**
```swift
// Swift Testing membutuhkan Swift 5.10+ (stabil di Swift 6)
// Gunakan #if canImport(Testing) untuk conditional compilation
#if canImport(Testing)
import Testing
@Test func newTest() { }
#else
import XCTest
class NewTests: XCTestCase {
    func testNewTest() { }
}
#endif
```

---

## 10. Real Use Cases

### Use Case 1: Testing Validation Layer

```swift
import Testing

@Suite("User Registration Validation")
struct RegistrationValidationTests {
    let validator = RegistrationValidator()
    
    // Grup: valid inputs
    @Suite("Valid Inputs")
    struct ValidInputs {
        let validator = RegistrationValidator()
        
        @Test("Username valid", arguments: [
            "ari_supriatna",
            "budi123",
            "user-name",
            "a",           // minimum 1 karakter
            String(repeating: "a", count: 30)  // maximum
        ])
        func validUsernames(username: String) throws {
            #expect(throws: Never.self) {
                try validator.validateUsername(username)
            }
        }
    }
    
    // Grup: invalid inputs dengan pesan error spesifik
    @Suite("Invalid Inputs")
    struct InvalidInputs {
        let validator = RegistrationValidator()
        
        @Test("Username terlalu panjang")
        func usernameTooLong() throws {
            let longUsername = String(repeating: "a", count: 31)
            #expect(throws: ValidationError.tooLong) {
                try validator.validateUsername(longUsername)
            }
        }
        
        @Test("Karakter tidak valid dalam username", arguments: [
            "user name",   // spasi
            "user@name",   // @
            "user.name!",  // !
        ])
        func usernameInvalidCharacters(username: String) {
            #expect(throws: ValidationError.invalidCharacters) {
                try validator.validateUsername(username)
            }
        }
    }
    
    // Test integrasi: seluruh form
    @Test("Form valid diterima")
    func validFormAccepted() throws {
        let form = RegistrationForm(
            username: "ari123",
            email: "ari@example.com",
            password: "SecurePass123!"
        )
        
        try validator.validate(form)  // tidak throws = sukses
    }
    
    @Test("Form invalid ditolak dengan semua errors", .tags(.regression))
    func invalidFormReturnsAllErrors() {
        let form = RegistrationForm(username: "", email: "invalid", password: "123")
        let errors = validator.collectErrors(form)
        
        #expect(errors.contains { $0 == .requiredField("username") })
        #expect(errors.contains { $0 == .invalidFormat("email") })
        #expect(errors.contains { $0 == .tooShort(field: "password", minimum: 8, actual: 3) })
        #expect(errors.count == 3)
    }
}
```

---

### Use Case 2: Testing Networking Layer

```swift
import Testing

// Mock untuk testing
actor MockHTTPClient {
    var responses: [URL: Result<Data, Error>] = [:]
    var requestLog: [(url: URL, method: String)] = []
    
    func stub(url: URL, response: Result<Data, Error>) {
        responses[url] = response
    }
    
    func request(url: URL, method: String) async throws -> Data {
        requestLog.append((url: url, method: method))
        switch responses[url] {
        case .success(let data): return data
        case .failure(let error): throw error
        case .none: throw URLError(.badURL)
        }
    }
}

@Suite("User API Client", .serialized)
struct UserAPIClientTests {
    let mockClient: MockHTTPClient
    let apiClient: UserAPIClient
    
    init() async {
        mockClient = MockHTTPClient()
        apiClient = UserAPIClient(httpClient: mockClient)
    }
    
    @Test("Fetch user sukses")
    func fetchUserSuccess() async throws {
        let userData = """
        {"id": "123", "name": "Ari Supriatna", "email": "ari@example.com"}
        """.data(using: .utf8)!
        
        await mockClient.stub(
            url: URL(string: "https://api.example.com/users/123")!,
            response: .success(userData)
        )
        
        let user = try await apiClient.fetchUser(id: "123")
        
        #expect(user.id == "123")
        #expect(user.name == "Ari Supriatna")
        #expect(user.email == "ari@example.com")
    }
    
    @Test("Fetch user not found returns error")
    func fetchUserNotFound() async throws {
        await mockClient.stub(
            url: URL(string: "https://api.example.com/users/999")!,
            response: .failure(URLError(.fileDoesNotExist))
        )
        
        await #expect(throws: UserAPIError.notFound) {
            try await apiClient.fetchUser(id: "999")
        }
    }
    
    @Test("Retry on server error", .timeLimit(.seconds(10)))
    func retryOnServerError() async throws {
        var callCount = 0
        
        // Pertama dua kali gagal, ketiga sukses
        await mockClient.stub(
            url: URL(string: "https://api.example.com/users/123")!,
            response: .failure(URLError(.timedOut))
        )
        
        // Test retry behavior
        await #expect(throws: Never.self) {
            try await apiClient.fetchUserWithRetry(id: "123", maxRetries: 3)
        }
    }
    
    @Test("Concurrent requests tidak interferensi", .tags(.critical))
    func concurrentRequestsIndependent() async throws {
        let userIDs = ["1", "2", "3", "4", "5"]
        
        for id in userIDs {
            let data = """{"id": "\(id)", "name": "User \(id)", "email": "\(id)@test.com"}"""
                .data(using: .utf8)!
            await mockClient.stub(
                url: URL(string: "https://api.example.com/users/\(id)")!,
                response: .success(data)
            )
        }
        
        // Fetch semua secara concurrent
        let users = try await withThrowingTaskGroup(of: User.self) { group in
            for id in userIDs {
                group.addTask { try await self.apiClient.fetchUser(id: id) }
            }
            return try await group.reduce(into: []) { $0.append($1) }
        }
        
        #expect(users.count == userIDs.count)
        #expect(Set(users.map { $0.id }) == Set(userIDs))
    }
}
```

---

### Use Case 3: Testing Actor dan Concurrency

```swift
import Testing

@Suite("Cache Actor")
struct CacheActorTests {
    
    @Test("Set dan get value")
    func setAndGetValue() async {
        let cache = Cache<String, Int>(ttl: .seconds(60))
        
        await cache.set(42, for: "answer")
        let retrieved = await cache.get(for: "answer")
        
        #expect(retrieved == 42)
    }
    
    @Test("Cache miss returns nil")
    func cacheMissReturnsNil() async {
        let cache = Cache<String, Int>(ttl: .seconds(60))
        let value = await cache.get(for: "nonexistent")
        #expect(value == nil)
    }
    
    @Test("TTL expiry — entry kadaluarsa")
    func ttlExpiry() async throws {
        let cache = Cache<String, Int>(ttl: .milliseconds(100))
        
        await cache.set(42, for: "key")
        try await Task.sleep(for: .milliseconds(150))
        
        let value = await cache.get(for: "key")
        #expect(value == nil, "Entry harus sudah kadaluarsa")
    }
    
    @Test("Concurrent writes tidak corrupt state", .tags(.critical))
    func concurrentWritesSafe() async {
        let cache = Cache<String, Int>(maxSize: 1000, ttl: .seconds(60))
        
        await withTaskGroup(of: Void.self) { group in
            for i in 0..<100 {
                group.addTask {
                    await cache.set(i, for: "key_\(i)")
                }
            }
        }
        
        // Verifikasi semua values tersimpan dengan benar
        var successCount = 0
        for i in 0..<100 {
            if await cache.get(for: "key_\(i)") == i {
                successCount += 1
            }
        }
        
        #expect(successCount == 100)
    }
    
    // Parameterized test untuk TTL values
    @Test("Max size enforcement", arguments: [10, 50, 100, 500])
    func maxSizeEnforcement(maxSize: Int) async {
        let cache = Cache<Int, Int>(maxSize: maxSize, ttl: .seconds(60))
        
        for i in 0..<(maxSize + 50) {
            await cache.set(i, for: i)
        }
        
        // Cache tidak boleh melebihi maxSize
        let count = await cache.count
        #expect(count <= maxSize)
    }
}
```

---

## 11. Migrasi dari XCTest

### Mapping XCTest → Swift Testing

| XCTest | Swift Testing | Catatan |
|---|---|---|
| `XCTestCase` class | `@Suite` struct | Tidak perlu class |
| `func testFoo()` | `@Test func foo()` | Tidak perlu prefix "test" |
| `setUp()` | `init()` | Lebih natural |
| `tearDown()` | `deinit` | Lebih natural |
| `XCTAssertEqual(a, b)` | `#expect(a == b)` | Ekspresi arbitrary |
| `XCTAssertNil(x)` | `#expect(x == nil)` | Sama |
| `XCTAssertNotNil(x)` | `try #require(x)` | Unwrap atau stop |
| `XCTAssertThrowsError` | `#expect(throws:)` | Lebih expressive |
| `XCTAssertNoThrow` | `#expect(throws: Never.self)` | Eksplisit |
| `XCTFail("message")` | `Issue.record("message")` | Tidak stop test |

### Strategi Migrasi Bertahap

```swift
// File lama: tetap jalan dengan XCTest
class OldUserTests: XCTestCase {
    func testLegacy() { XCTAssertTrue(true) }
}

// File baru: Swift Testing
@Suite("User Tests")
struct NewUserTests {
    @Test func newBehavior() { #expect(true) }
}

// Keduanya bisa ada bersamaan — migrasi bertahap
```

---

## 12. Ringkasan

### Quick Reference

```swift
// Struktur dasar
@Suite("Feature Name")
struct FeatureTests {
    let sut = SubjectUnderTest()  // tidak perlu setUp/tearDown sederhana
    
    @Test("Deskripsi test")
    func testBehavior() throws {
        let result = sut.doSomething()
        #expect(result == expected)
    }
    
    @Test("Async", .timeLimit(.seconds(5)))
    func asyncTest() async throws {
        let result = try await sut.asyncOperation()
        let value = try #require(result.optionalValue)
        #expect(value > 0)
    }
    
    @Test("Parameterized", arguments: inputs)
    func paramTest(input: InputType) {
        #expect(sut.process(input) == expected(input))
    }
}
```

### Kapan Pilih Apa

| Situasi | Pilihan |
|---|---|
| Unit test baru | Swift Testing |
| UI Test (XCUITest) | XCTest (tidak ada alternatif) |
| Performance test (`measure {}`) | XCTest |
| Parameterized test | Swift Testing |
| Cross-platform (Linux/Windows) | Swift Testing |
| Codebase Swift 5.9 ke bawah | XCTest |
| Test yang perlu tag/filter | Swift Testing |

---

*Dibuat: 2026-05-08 | Swift 6.0 | import Testing*
