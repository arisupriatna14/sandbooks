# Swift 6: Typed Throws
### Teori Mendalam, Real Use Case, dan Panduan Penggunaan

---

## Daftar Isi
1. [Masalah yang Dipecahkan](#1-masalah-yang-dipecahkan)
2. [Teori: Cara Kerja Typed Throws](#2-teori-cara-kerja-typed-throws)
3. [Sintaks Dasar](#3-sintaks-dasar)
4. [Typed Throws dengan Generics](#4-typed-throws-dengan-generics)
5. [Kapan Harus Digunakan](#5-kapan-harus-digunakan)
6. [Kapan Tidak Digunakan](#6-kapan-tidak-digunakan)
7. [Real Use Cases](#7-real-use-cases)
8. [Migrasi dari Untyped Throws](#8-migrasi-dari-untyped-throws)
9. [Ringkasan](#9-ringkasan)

---

## 1. Masalah yang Dipecahkan

### Error Handling di Swift 5: Kehilangan Informasi Tipe

Sebelum Swift 6, `throws` adalah **untyped** — compiler tidak tahu dan tidak peduli error apa yang dilempar. Semua error diperlakukan sebagai `any Error`:

```swift
// Swift 5: apa pun bisa keluar dari sini
func parseJSON(_ data: Data) throws -> User {
    // Bisa lempar: DecodingError, NetworkError, CustomError, apapun
}

// Caller terpaksa menebak atau catch-all:
do {
    let user = try parseJSON(data)
} catch let error as DecodingError {
    // mungkin ini
} catch let error as NetworkError {
    // mungkin ini
} catch {
    // atau apapun yang lain — ini selalu dibutuhkan
    // karena compiler tidak tahu apa yang mungkin dilempar
}
```

**Masalah nyata dari untyped throws:**

1. **False safety:** Kamu bisa menangkap tipe error yang salah dan compiler tidak memperingatkanmu
2. **Documentation rot:** Dokumentasi "throws NetworkError" bisa berbohong — compiler tidak enforces ini
3. **Overhead runtime:** Error selalu di-box sebagai existential `any Error` — ada heap allocation
4. **Library design:** Library tidak bisa mengekspresikan "fungsi ini HANYA bisa lempar tipe X"
5. **Generic functions:** Tidak bisa menulis `map` yang meneruskan error type dari transform ke caller

```swift
// Swift 5: ini compile, bahkan jika parseJSON tidak pernah lempar NetworkError
do {
    let user = try parseJSON(data)
} catch let error as NetworkError {
    // block ini tidak pernah dieksekusi, tapi compiler tidak tahu itu
    handleNetworkError(error)
}
```

### Apa yang Diinginkan Developer

Yang diinginkan adalah ekspresi yang **jujur dan verifiable di compile time**:
- "Fungsi ini hanya bisa lempar `NetworkError`"
- "Fungsi ini tidak bisa lempar sama sekali"
- "Fungsi generic ini meneruskan error type dari parameter-nya"

---

## 2. Teori: Cara Kerja Typed Throws

### Unified Error Handling Model

Swift 6 menyatukan semua varian error handling dalam satu sintaks `throws(E)`:

```
throws(Never)  ≡  tidak throws (non-throwing)
throws         ≡  throws(any Error)  — backward compatible
throws(E)      ≡  hanya bisa lempar tipe E yang concrete
```

Ini bukan fitur terpisah — ini **generalisasi** dari sistem yang sudah ada. `throws` biasa sekarang secara konseptual adalah `throws(any Error)`.

### Memory Model: Menghindari Existential Boxing

Di Swift 5, semua error di-*box* sebagai `any Error` existential:
```
Error value → heap allocation → existential container → catch block
```

Dengan typed throws menggunakan concrete type:
```
Error value → langsung di stack/register → catch block
```

Untuk kode yang performance-critical (misalnya parsing loop yang memproses jutaan record), perbedaan ini signifikan karena menghilangkan heap allocation per-error.

### Subtyping: throws adalah supertipe

Karena `throws` = `throws(any Error)`, fungsi `throws(MyError)` adalah **subtype** dari `throws(any Error)`:

```swift
func specific() throws(NetworkError) -> Int { 42 }
func general() throws -> Int { 42 }

// Aman: assign more specific ke more general
let fn: () throws -> Int = specific  // OK
// Tidak aman sebaliknya:
// let fn2: () throws(NetworkError) -> Int = general  // ERROR
```

### `Never` sebagai Error Type

`throws(Never)` adalah cara formal menyatakan "tidak pernah throws" — secara semantik sama dengan tidak menulis `throws` sama sekali, tapi berguna dalam konteks generic:

```swift
// Generic function yang bisa throwing atau non-throwing
func transform<T, E: Error>(_ value: T, using fn: (T) throws(E) -> T) throws(E) -> T {
    try fn(value)
}

// Penggunaan non-throwing: E = Never, tidak perlu try
let result = transform(5, using: { $0 * 2 })  // tidak throws, tidak perlu try

// Penggunaan throwing: E = ValidationError
let result2 = try transform("hello", using: validateInput)
```

---

## 3. Sintaks Dasar

### Deklarasi Fungsi dengan Typed Throws

```swift
enum ParseError: Error {
    case invalidFormat(String)
    case missingField(String)
    case valueOutOfRange(field: String, value: Int, range: ClosedRange<Int>)
}

// Deklarasi: hanya bisa lempar ParseError
func parseAge(_ string: String) throws(ParseError) -> Int {
    guard let number = Int(string) else {
        throw ParseError.invalidFormat("'\(string)' bukan angka")
    }
    
    guard (0...150).contains(number) else {
        throw ParseError.valueOutOfRange(field: "age", value: number, range: 0...150)
    }
    
    return number
}

// Caller: catch block punya tipe konkret, tidak perlu cast
do {
    let age = try parseAge("abc")
} catch {
    // 'error' di sini adalah ParseError — langsung akses properties-nya
    switch error {
    case .invalidFormat(let msg):
        print("Format salah: \(msg)")
    case .missingField(let field):
        print("Field hilang: \(field)")
    case .valueOutOfRange(let field, let value, let range):
        print("\(field): \(value) di luar \(range)")
    }
    // Tidak perlu 'default' — compiler tahu semua case sudah ter-cover!
}
```

### Typed Throws + try?

```swift
// try? menghasilkan Optional — error di-discard
let age: Int? = try? parseAge("25")  // Optional(25)
let bad: Int? = try? parseAge("abc") // nil
```

### Typed Throws + try!

```swift
// try! crash jika throw — gunakan hanya jika yakin tidak bisa gagal
let age = try! parseAge("25")  // 25
// let bad = try! parseAge("abc")  // CRASH: ParseError.invalidFormat
```

---

## 4. Typed Throws dengan Generics

Ini adalah killer feature dari typed throws — memungkinkan generic functions yang meneruskan error type secara transparan:

### Pattern: Error Propagation di Generic Function

```swift
// Sebelum Swift 6: harus throws(any Error), kehilangan tipe error
// extension Array {
//     func myMap<T>(_ transform: (Element) throws -> T) throws -> [T] { ... }
// }

// Swift 6: error type dari transform di-propagate secara tepat
extension Array {
    func myMap<T, E: Error>(_ transform: (Element) throws(E) -> T) throws(E) -> [T] {
        var result: [T] = []
        result.reserveCapacity(count)
        for element in self {
            result.append(try transform(element))
        }
        return result
    }
}

// Penggunaan 1: non-throwing transform → myMap tidak throws
let doubled = [1, 2, 3].myMap { $0 * 2 }  // tidak perlu try!
print(doubled)  // [2, 4, 6]

// Penggunaan 2: typed throwing transform → myMap propagates error type
enum ConversionError: Error { case overflow }
let parsed = try [" 1", "2", "3"].myMap { str throws(ConversionError) -> Int in
    guard let n = Int(str.trimmingCharacters(in: .whitespaces)) else {
        throw ConversionError.overflow
    }
    return n
}
// catch block tahu persis error-nya adalah ConversionError
```

### Pattern: Result Type dengan Typed Error

```swift
// Result<Value, E> sudah ada — sekarang bisa berpasangan dengan typed throws
func riskyOperation() throws(DomainError) -> Value {
    // ...
}

// Konversi mudah antara throws dan Result
let result: Result<Value, DomainError> = Result { try riskyOperation() }

// Dan sebaliknya
let value = try result.get()  // throws(DomainError)
```

### Pattern: Async + Typed Throws

```swift
actor NetworkClient {
    enum RequestError: Error {
        case noConnection
        case timeout
        case serverError(statusCode: Int)
        case invalidResponse
    }
    
    func fetch(url: URL) async throws(RequestError) -> Data {
        guard isConnected else { throw RequestError.noConnection }
        
        do {
            let (data, response) = try await URLSession.shared.data(from: url)
            
            guard let http = response as? HTTPURLResponse else {
                throw RequestError.invalidResponse
            }
            
            guard (200...299).contains(http.statusCode) else {
                throw RequestError.serverError(statusCode: http.statusCode)
            }
            
            return data
        } catch is URLError {
            throw RequestError.timeout
        } catch {
            throw RequestError.invalidResponse
        }
    }
    
    private var isConnected: Bool { true }
}

// Penggunaan:
let client = NetworkClient()
do {
    let data = try await client.fetch(url: someURL)
    process(data)
} catch {
    // error adalah NetworkClient.RequestError — tanpa cast
    switch error {
    case .noConnection: showOfflineUI()
    case .timeout: showRetryUI()
    case .serverError(let code): showServerError(code)
    case .invalidResponse: showParseError()
    }
}
```

---

## 5. Kapan Harus Digunakan

### Gunakan Typed Throws ketika:

**1. Domain layer dengan error yang terdefinisi dengan baik**
```swift
// Business logic yang punya set error terbatas dan diketahui
enum AuthError: Error {
    case invalidCredentials
    case accountLocked(until: Date)
    case sessionExpired
    case insufficientPermissions(required: Permission)
}

func authenticate(username: String, password: String) async throws(AuthError) -> Session {
    // Caller tahu persis apa yang bisa salah
}
```

**2. Library/SDK public API**
```swift
// Pengguna library harus tahu persis error apa yang mungkin terjadi
public enum JSONParserError: Error {
    case malformedJSON(offset: Int)
    case unexpectedToken(found: Character, expected: String)
    case depthLimitExceeded(limit: Int)
}

public func parse(_ data: Data) throws(JSONParserError) -> JSONValue {
    // Contract yang jelas dan enforced
}
```

**3. Generic functions yang harus meneruskan error type**
```swift
// map, filter, reduce, flatMap yang type-preserving
func validate<T, E: Error>(_ values: [T], using validator: (T) throws(E) -> Void) throws(E) {
    for value in values {
        try validator(value)
    }
}
```

**4. Performance-critical code dengan banyak error path**
```swift
// Parser yang memproses jutaan record — eliminasi heap allocation per-error
func parseRecord(_ bytes: UnsafeBufferPointer<UInt8>) throws(BinaryParseError) -> Record {
    // Setiap error tidak perlu boxing — langsung di stack
}
```

**5. Ketika exhaustive handling adalah requirement**
```swift
// Kamu ingin memastikan caller menangani SETIAP kemungkinan error
// Switch tanpa default — compiler memperingatkan jika ada case baru ditambahkan
```

---

## 6. Kapan Tidak Digunakan

### Hindari Typed Throws ketika:

**1. Error bisa datang dari banyak sumber berbeda**
```swift
// JANGAN: memaksa wrap dari banyak sumber ke satu tipe
enum ServiceError: Error {
    case network(URLError)         // dari URLSession
    case database(SQLiteError)     // dari SQLite
    case parsing(DecodingError)    // dari JSONDecoder
    case unknown(any Error)        // catch-all yang mengalahkan tujuan typed throws
}

// Lebih baik: gunakan throws biasa atau pisahkan ke fungsi-fungsi terpisah
func fetchAndSave() throws -> User { ... }  // throws(any Error) — jujur tentang kompleksitas
```

**2. Prototyping atau early development**
```swift
// Saat masih eksplorasi, error types sering berubah
// Gunakan throws biasa dulu, refactor ke typed throws saat API stabil
func experimentalFeature() throws -> Result { ... }
```

**3. Fungsi yang memanggil banyak library eksternal**
```swift
// Library eksternal punya error types mereka sendiri
// Wrapping semua ke satu typed error sering kali sia-sia
func processPDF() throws -> Document {
    // Memanggil PDFKit, Network, CoreData — semua punya error berbeda
    // throws biasa lebih jujur di sini
}
```

**4. API yang perlu backward compatibility dengan Swift 5**
```swift
// Jika codebase harus compile di Swift 5, typed throws tidak tersedia
// Tandai dengan @available atau gunakan conditional compilation
```

**5. Ketika error type akan terus berkembang**
```swift
// Jika error enum sering mendapat case baru, setiap penambahan
// akan break semua switch statement caller — trade-off yang berat
```

---

## 7. Real Use Cases

### Use Case 1: Validation Pipeline

```swift
// Validation layer yang ketat — setiap validator hanya bisa gagal dengan satu jenis error

enum ValidationError: Error, LocalizedError {
    case tooShort(field: String, minimum: Int, actual: Int)
    case tooLong(field: String, maximum: Int, actual: Int)
    case invalidCharacters(field: String, allowed: CharacterSet)
    case alreadyTaken(field: String, value: String)
    case requiredField(String)
    
    var errorDescription: String? {
        switch self {
        case .tooShort(let field, let min, let actual):
            return "\(field) minimal \(min) karakter (sekarang \(actual))"
        case .tooLong(let field, let max, let actual):
            return "\(field) maksimal \(max) karakter (sekarang \(actual))"
        case .invalidCharacters(let field, _):
            return "\(field) mengandung karakter tidak valid"
        case .alreadyTaken(let field, let value):
            return "\(field) '\(value)' sudah digunakan"
        case .requiredField(let field):
            return "\(field) wajib diisi"
        }
    }
}

struct UserRegistrationForm {
    let username: String
    let email: String
    let password: String
}

// Setiap validator hanya throws ValidationError — tidak lebih
func validateUsername(_ username: String) throws(ValidationError) {
    guard !username.isEmpty else { throw ValidationError.requiredField("username") }
    guard username.count >= 3 else {
        throw ValidationError.tooShort(field: "username", minimum: 3, actual: username.count)
    }
    guard username.count <= 30 else {
        throw ValidationError.tooLong(field: "username", maximum: 30, actual: username.count)
    }
    
    let allowed = CharacterSet.alphanumerics.union(CharacterSet(charactersIn: "_-"))
    guard username.unicodeScalars.allSatisfy({ allowed.contains($0) }) else {
        throw ValidationError.invalidCharacters(field: "username", allowed: allowed)
    }
}

func validateEmail(_ email: String) throws(ValidationError) {
    guard !email.isEmpty else { throw ValidationError.requiredField("email") }
    guard email.contains("@") && email.contains(".") else {
        throw ValidationError.invalidCharacters(field: "email", allowed: .init())
    }
}

func validatePassword(_ password: String) throws(ValidationError) {
    guard !password.isEmpty else { throw ValidationError.requiredField("password") }
    guard password.count >= 8 else {
        throw ValidationError.tooShort(field: "password", minimum: 8, actual: password.count)
    }
}

// Aggregator — mengumpulkan semua validation errors
func validateForm(_ form: UserRegistrationForm) -> [ValidationError] {
    var errors: [ValidationError] = []
    
    if let error = try? { try validateUsername(form.username) }() {
        // Tip: closure trick untuk catch typed error ke optional
    }
    
    // Pattern yang lebih bersih:
    do { try validateUsername(form.username) } catch { errors.append(error) }
    do { try validateEmail(form.email) } catch { errors.append(error) }
    do { try validatePassword(form.password) } catch { errors.append(error) }
    
    return errors
}

// Penggunaan di ViewModel
@MainActor
class RegistrationViewModel: ObservableObject {
    @Published var validationErrors: [ValidationError] = []
    
    func submit(form: UserRegistrationForm) async {
        let errors = validateForm(form)
        validationErrors = errors
        
        guard errors.isEmpty else { return }
        
        do {
            try await registerUser(form)
        } catch {
            // error di sini adalah RegistrationError — typed
            handleRegistrationError(error)
        }
    }
}
```

---

### Use Case 2: File Parser dengan Typed Error

```swift
// CSV Parser — contoh typed throws di parsing pipeline

enum CSVError: Error {
    case emptyInput
    case invalidEncoding
    case malformedRow(line: Int, content: String)
    case missingColumn(name: String, availableColumns: [String])
    case typeMismatch(column: String, expected: String, got: String)
}

struct CSVParser {
    let separator: Character
    
    init(separator: Character = ",") {
        self.separator = separator
    }
    
    func parse(_ content: String) throws(CSVError) -> [[String: String]] {
        guard !content.isEmpty else { throw CSVError.emptyInput }
        
        var lines = content.components(separatedBy: "\n")
            .map { $0.trimmingCharacters(in: .whitespaces) }
            .filter { !$0.isEmpty }
        
        guard !lines.isEmpty else { throw CSVError.emptyInput }
        
        let headers = lines.removeFirst()
            .split(separator: separator)
            .map(String.init)
        
        return try lines.enumerated().map { lineIndex, line throws(CSVError) in
            let values = line.split(separator: separator, omittingEmptySubsequences: false)
                .map(String.init)
            
            guard values.count == headers.count else {
                throw CSVError.malformedRow(line: lineIndex + 2, content: line)
            }
            
            return Dictionary(uniqueKeysWithValues: zip(headers, values))
        }
    }
    
    // Typed conversion — generic, propagates error
    func column<T, E: Error>(
        named name: String,
        from row: [String: String],
        as type: T.Type,
        converter: (String) throws(E) -> T
    ) throws(CSVError) -> T {
        guard let raw = row[name] else {
            throw CSVError.missingColumn(name: name, availableColumns: Array(row.keys))
        }
        
        do {
            return try converter(raw)
        } catch {
            throw CSVError.typeMismatch(column: name, expected: "\(T.self)", got: raw)
        }
    }
}

// Penggunaan:
let parser = CSVParser()

do {
    let csvContent = """
    name,age,email
    Ari,28,ari@example.com
    Budi,30,budi@example.com
    """
    
    let rows = try parser.parse(csvContent)
    
    for row in rows {
        let name = row["name"] ?? ""
        // Kalau ada error, langsung tahu itu CSVError — tidak perlu cast
        print("User: \(name)")
    }
} catch {
    // exhaustive switch — compiler pastikan semua case ter-cover
    switch error {
    case .emptyInput:
        print("File CSV kosong")
    case .invalidEncoding:
        print("Encoding tidak valid")
    case .malformedRow(let line, let content):
        print("Baris \(line) tidak valid: \(content)")
    case .missingColumn(let name, let available):
        print("Kolom '\(name)' tidak ada. Tersedia: \(available.joined(separator: ", "))")
    case .typeMismatch(let col, let expected, let got):
        print("Kolom '\(col)': expected \(expected), got '\(got)'")
    }
}
```

---

### Use Case 3: Repository Pattern dengan Domain Errors

```swift
// Repository dengan typed errors yang jelas di setiap operation

enum UserRepositoryError: Error {
    case notFound(id: String)
    case duplicateEmail(email: String)
    case constraintViolation(field: String, reason: String)
    case storageFailure(underlying: String)
}

enum QueryError: Error {
    case invalidFilter(String)
    case resultTooLarge(count: Int, limit: Int)
    case timeout(after: Duration)
}

protocol UserRepository {
    func find(by id: String) async throws(UserRepositoryError) -> User
    func save(_ user: User) async throws(UserRepositoryError)
    func delete(id: String) async throws(UserRepositoryError)
    func query(_ filter: UserFilter) async throws(QueryError) -> [User]
}

// Interactor yang tahu persis error dari setiap operation
struct UserInteractor {
    let repository: any UserRepository
    
    func getUser(id: String) async -> Result<User, UserRepositoryError> {
        do {
            let user = try await repository.find(by: id)
            return .success(user)
        } catch {
            // error PASTI UserRepositoryError — tidak perlu if let / as?
            return .failure(error)
        }
    }
    
    func registerUser(_ form: RegistrationForm) async throws(UserRepositoryError) {
        let user = User(from: form)
        try await repository.save(user)  // typed error propagates
    }
}
```

---

## 8. Migrasi dari Untyped Throws

### Strategi Bertahap

```swift
// Langkah 1: Identifikasi semua error yang bisa dilempar
// Langkah 2: Buat enum yang mencakup semua kasus
// Langkah 3: Wrap existing throws ke typed throws

// SEBELUM:
func processPayment(amount: Double) throws -> Receipt {
    // bisa lempar: PaymentError, NetworkError, DatabaseError, ...
}

// SESUDAH — pendekatan pragmatis:
enum PaymentProcessingError: Error {
    case insufficientFunds(available: Double, required: Double)
    case cardDeclined(reason: String)
    case networkUnavailable
    case processingFailed(String)  // catch-all untuk yang tidak terduga
}

func processPayment(amount: Double) async throws(PaymentProcessingError) -> Receipt {
    do {
        let balance = try await checkBalance()
        guard balance >= amount else {
            throw PaymentProcessingError.insufficientFunds(available: balance, required: amount)
        }
        
        let receipt = try await chargeCard(amount: amount)
        try await saveReceipt(receipt)
        return receipt
        
    } catch let error as PaymentProcessingError {
        throw error  // re-throw typed
    } catch is URLError {
        throw PaymentProcessingError.networkUnavailable
    } catch {
        throw PaymentProcessingError.processingFailed(error.localizedDescription)
    }
}
```

---

## 9. Ringkasan

### Decision Guide

```
Fungsi throws error?
├── TIDAK → Tidak perlu throws
└── YA
    │
    Error datang dari satu domain yang terdefinisi?
    ├── YA dan set error terbatas → throws(DomainError)
    ├── YA tapi set bisa berkembang banyak → pertimbangkan throws biasa
    └── TIDAK (multiple sources, external libs) → throws biasa (any Error)
    
Apakah fungsi ini generic dan butuh meneruskan error type?
└── YA → throws(E) dengan E: Error — ini killer use case-nya
```

### Perbandingan Cepat

| Situasi | Rekomendasi |
|---|---|
| Domain error yang jelas dan terbatas | `throws(DomainError)` |
| Generic function (map, filter) | `throws(E) where E: Error` |
| Fungsi memanggil banyak library eksternal | `throws` biasa |
| Non-throwing (tapi dalam konteks generic) | `throws(Never)` |
| Public API library | `throws(LibraryError)` |
| Prototyping | `throws` biasa dulu |

---

*Dibuat: 2026-05-08 | Swift 6.0*
