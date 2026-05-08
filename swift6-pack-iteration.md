# Swift 6: Pack Iteration dan Parameter Packs
### Teori Mendalam, Real Use Case, dan Panduan Penggunaan

---

## Daftar Isi
1. [Masalah yang Dipecahkan](#1-masalah-yang-dipecahkan)
2. [Teori: Parameter Packs dan Value Packs](#2-teori-parameter-packs-dan-value-packs)
3. [Sintaks Dasar](#3-sintaks-dasar)
4. [Pack Iteration — Swift 6](#4-pack-iteration--swift-6)
5. [Kapan Harus Digunakan](#5-kapan-harus-digunakan)
6. [Kapan Tidak Digunakan](#6-kapan-tidak-digunakan)
7. [Real Use Cases](#7-real-use-cases)
8. [Ringkasan](#8-ringkasan)

---

## 1. Masalah yang Dipecahkan

### Masalah: Variadic Generics yang Tidak Ekspresif

Sebelum Swift 5.9, jika kamu ingin menulis fungsi yang menerima beberapa argumen dengan tipe berbeda, kamu punya beberapa pilihan buruk:

**Pilihan 1: Overloading manual — tidak skalabel**
```swift
// Harus tulis versi untuk setiap jumlah argumen
func zip<A, B>(_ a: A, _ b: B) -> (A, B) { (a, b) }
func zip<A, B, C>(_ a: A, _ b: B, _ c: C) -> (A, B, C) { (a, b, c) }
func zip<A, B, C, D>(_ a: A, _ b: B, _ c: C, _ d: D) -> (A, B, C, D) { (a, b, c, d) }
// ... terus sampai berapa? 5? 10? 20?
// Swift standard library punya versi hingga 6 argumen untuk beberapa fungsi
```

**Pilihan 2: `Any` atau existential — kehilangan type safety**
```swift
func process(_ values: Any...) {
    for value in values {
        // Harus cast manual — kehilangan type information
        if let int = value as? Int { handleInt(int) }
        else if let string = value as? String { handleString(string) }
        // ...
    }
}
```

**Pilihan 3: Tuple — tidak fleksibel**
```swift
// Hanya bisa tuple dengan jumlah elemen fixed
func process<A, B>(_ pair: (A, B)) { }
// Tidak bisa generic terhadap jumlah elemen
```

**Apa yang diinginkan:**
```swift
// Ideanya: fungsi yang menerima "berapapun" argumen dengan tipe berbeda
// dan melakukan operasi type-safe pada masing-masing
func validate<each T>(_ values: repeat each T, using validators: repeat Validator<each T>) -> Bool
```

---

## 2. Teori: Parameter Packs dan Value Packs

### Apa itu Pack?

**Pack** adalah kumpulan tipe atau nilai yang diproses bersama-sama. Bayangkan ini seperti tuple yang bisa punya panjang arbiter, dengan semua tipe tersimpan di level compile-time.

```
Normal generics:    func foo<T>(_ value: T)
                              ─ T adalah SATU tipe

Parameter packs:    func foo<each T>(_ values: repeat each T)
                              ───── T adalah KUMPULAN tipe (pack)
                                          ───────────── repeat: expand pack
```

### Dua Jenis Pack

**Type Pack** — kumpulan tipe:
```swift
// each T adalah type pack: bisa (Int), (Int, String), (Int, String, Bool), dll
func example<each T>(...) { }
```

**Value Pack** — kumpulan nilai yang tipe-tipenya didefinisikan oleh type pack:
```swift
// repeat each value adalah value pack: satu nilai untuk setiap T dalam type pack
func example<each T>(_ values: repeat each T) { }
```

### Pack Expansion

`repeat each X` adalah **pack expansion** — operasi yang "membuka" pack dan menerapkan sesuatu untuk setiap elemen:

```swift
// Type-level expansion:
(repeat each T)        // expand type pack T menjadi tuple
                       // (T₁, T₂, T₃, ...) jika T = (T₁, T₂, T₃)

// Value-level expansion:
repeat someFunc(each value)  // panggil someFunc untuk setiap value dalam pack
```

### Compile-Time Unrolling

Penting untuk dipahami: parameter packs di-resolve **sepenuhnya di compile time**. Tidak ada overhead runtime untuk "iterasi" — compiler men-generate kode untuk setiap elemen secara terpisah:

```swift
func printAll<each T>(_ values: repeat each T) {
    repeat print(each values)
}

// Ketika dipanggil dengan (1, "hello", true):
// Compiler generate code setara dengan:
// print(1)
// print("hello")
// print(true)
```

Ini seperti template metaprogramming di C++, tapi dengan syntax yang jauh lebih bersih.

---

## 3. Sintaks Dasar

### Deklarasi Type Pack

```swift
// 'each' dalam generic parameter list = type pack
func example<each T>() { }

// Dengan constraint — semua tipe dalam pack harus conform
func example<each T: Equatable>() { }

// Multiple packs
func zip<each A, each B>() { }
```

### Pack Parameter

```swift
// 'repeat each T' sebagai parameter type = value pack
func printAll<each T>(_ values: repeat each T) {
    repeat print(each values)  // pack expansion dalam body
}

// Pemanggilan:
printAll(1, "hello", true, 3.14)
//       ─  ───────  ────  ────
//    Int   String   Bool  Double  ← T = (Int, String, Bool, Double)
```

### Pack dalam Tuple

```swift
// (repeat each T) adalah tuple type yang di-expand dari pack
func asTyple<each T>(_ values: repeat each T) -> (repeat each T) {
    return (repeat each values)  // pack expansion dalam return
}

let tuple = asTuple(1, "hello", true)  // (Int, String, Bool)
print(tuple.0)  // 1
print(tuple.1)  // "hello"
print(tuple.2)  // true
```

---

## 4. Pack Iteration — Swift 6

Swift 5.9 memperkenalkan parameter packs, tapi hanya untuk ekspresi tunggal. Swift 6 menambahkan kemampuan untuk **iterasi penuh** dengan statements:

### `for ... in repeat each` — Swift 6

```swift
// Swift 6: iterasi atas pack dengan full statement body
func logAll<each T>(_ values: repeat each T) {
    for value in repeat each values {
        // 'value' punya tipe yang berbeda untuk setiap iterasi
        // body bisa berisi banyak statements
        let description = "\(value)"
        let type = "\(type(of: value))"
        print("[\(type)]: \(description)")
    }
}

logAll(42, "hello", true, 3.14)
// [Int]: 42
// [String]: hello
// [Bool]: true
// [Double]: 3.14
```

### Collecting Results dari Pack Iteration

```swift
// Kumpulkan hasil dari setiap elemen pack
func descriptions<each T>(_ values: repeat each T) -> [String] {
    var result: [String] = []
    for value in repeat each values {
        result.append("\(value)")
    }
    return result
}

let desc = descriptions(1, "two", 3.0, true)
// ["1", "two", "3.0", "true"]
```

### Pack Iteration dengan Constraint

```swift
// Iterate dan panggil protocol method pada setiap elemen
func validateAll<each T: Validatable>(_ values: repeat each T) -> [ValidationResult] {
    var results: [ValidationResult] = []
    for value in repeat each values {
        results.append(value.validate())  // type-safe: value conform Validatable
    }
    return results
}

protocol Validatable {
    func validate() -> ValidationResult
}

struct ValidationResult {
    let isValid: Bool
    let message: String?
}
```

### Zipping Multiple Packs

```swift
// Zip dua pack bersama — seperti zip() di functional programming
func zipWith<each A, each B, Result>(
    _ firstValues: repeat each A,
    _ secondValues: repeat each B,
    combine: (repeat each A, repeat each B) -> Result
) -> Result {
    combine(repeat each firstValues, repeat each secondValues)
}
```

---

## 5. Kapan Harus Digunakan

### Gunakan Parameter Packs + Pack Iteration ketika:

**1. Library function yang harus bekerja dengan beberapa tipe berbeda**
```swift
// Validation, transformation, atau combination dari beberapa nilai berbeda tipe
func allSatisfy<each T: Validatable>(_ values: repeat each T) -> Bool {
    for value in repeat each values {
        guard value.validate().isValid else { return false }
    }
    return true
}

// Penggunaan:
let isFormValid = allSatisfy(username, email, password, phoneNumber)
```

**2. Type-safe tuple operations**
```swift
// Map atas tuple dengan tipe berbeda
func mapTuple<each Input, each Output>(
    _ values: repeat each Input,
    transform: (repeat each Input) -> (repeat each Output)
) -> (repeat each Output) {
    transform(repeat each values)
}
```

**3. Menggantikan overloading yang repetitif**
```swift
// SEBELUM: harus tulis banyak overload
func assertEqual<T: Equatable>(_ a: T, _ b: T) { }
func assertAllEqual<T: Equatable>(_ a: T, _ b: T, _ c: T) { }
func assertAllEqual<T: Equatable>(_ a: T, _ b: T, _ c: T, _ d: T) { }

// SESUDAH: satu fungsi dengan pack
func assertAllEqual<each T: Equatable>(_ pairs: repeat (each T, each T)) {
    for (a, b) in repeat each pairs {
        assert(a == b, "\(a) != \(b)")
    }
}
```

**4. Type-safe dependency injection**
```swift
// Factory yang menginstansiasi banyak tipe dari pack
func makeAll<each T: Service>(types: repeat (each T).Type) -> (repeat each T) {
    (repeat (each types).init())
}
```

---

## 6. Kapan Tidak Digunakan

### Hindari Pack Iteration ketika:

**1. Semua elemen punya tipe yang sama — gunakan Array**
```swift
// JANGAN: overkill untuk homogenous collection
func sumAll<each T: Numeric>(_ values: repeat each T) { }  // overcomplicated

// GUNAKAN: Array lebih sederhana dan jelas
func sumAll(_ values: [Int]) -> Int { values.reduce(0, +) }
```

**2. Jumlah argumen selalu fixed**
```swift
// JANGAN: gunakan pack jika selalu 2 argumen
func combine<each A, each B>(_ a: repeat each A, _ b: repeat each B) { }  // overkill

// GUNAKAN: generic biasa
func combine<A, B>(_ a: A, _ b: B) -> (A, B) { (a, b) }
```

**3. Kamu belum familiar dengan generic programming**
```swift
// Pack iteration punya learning curve tinggi
// Jika bisa solve dengan overloading atau Array, lakukan itu dulu
// Gunakan pack ketika manfaatnya jelas melebihi kompleksitasnya
```

**4. Operasi yang butuh runtime flexibility**
```swift
// Pack di-resolve compile-time — jika kamu butuh menambah/hapus elemen saat runtime,
// gunakan Array atau Collection
var dynamicItems: [any Processable] = []  // runtime flexibility
dynamicItems.append(contentsOf: newItems)
```

---

## 7. Real Use Cases

### Use Case 1: Multi-Parameter Validator

```swift
import Foundation

// Protocol untuk semua validatable types
protocol Validatable {
    associatedtype Value
    func validate(_ value: Value) -> ValidationResult
}

struct ValidationResult {
    let isValid: Bool
    let fieldName: String
    let errorMessage: String?
    
    static func success(_ field: String) -> ValidationResult {
        ValidationResult(isValid: true, fieldName: field, errorMessage: nil)
    }
    
    static func failure(_ field: String, message: String) -> ValidationResult {
        ValidationResult(isValid: false, fieldName: field, errorMessage: message)
    }
}

struct StringValidator: Validatable {
    let fieldName: String
    let minLength: Int
    let maxLength: Int
    
    func validate(_ value: String) -> ValidationResult {
        if value.count < minLength {
            return .failure(fieldName, message: "Minimal \(minLength) karakter")
        }
        if value.count > maxLength {
            return .failure(fieldName, message: "Maksimal \(maxLength) karakter")
        }
        return .success(fieldName)
    }
}

struct RangeValidator<T: Comparable>: Validatable {
    let fieldName: String
    let range: ClosedRange<T>
    
    func validate(_ value: T) -> ValidationResult {
        range.contains(value)
            ? .success(fieldName)
            : .failure(fieldName, message: "Harus di antara \(range.lowerBound) dan \(range.upperBound)")
    }
}

// Form validation engine menggunakan pack
struct FormValidator {
    
    // Validate banyak field sekaligus dengan tipe berbeda
    static func validate<each V: Validatable>(
        pairs: repeat (each V, (each V).Value)
    ) -> [ValidationResult] {
        var results: [ValidationResult] = []
        for (validator, value) in repeat each pairs {
            results.append(validator.validate(value))
        }
        return results
    }
    
    static func allValid(_ results: [ValidationResult]) -> Bool {
        results.allSatisfy { $0.isValid }
    }
}

// Penggunaan:
struct RegistrationForm {
    var username: String
    var age: Int
    var bio: String
}

func validateRegistrationForm(_ form: RegistrationForm) -> [ValidationResult] {
    let usernameValidator = StringValidator(fieldName: "username", minLength: 3, maxLength: 30)
    let ageValidator = RangeValidator(fieldName: "age", range: 13...120)
    let bioValidator = StringValidator(fieldName: "bio", minLength: 0, maxLength: 500)
    
    return FormValidator.validate(pairs:
        (usernameValidator, form.username),
        (ageValidator, form.age),
        (bioValidator, form.bio)
    )
}

let form = RegistrationForm(username: "ari", age: 28, bio: "iOS Developer")
let results = validateRegistrationForm(form)
let errors = results.filter { !$0.isValid }
print(errors.isEmpty ? "Form valid!" : "Errors: \(errors.map { $0.errorMessage ?? "" })")
```

---

### Use Case 2: Type-Safe Observable Properties

```swift
// Observer pattern untuk multiple property types
protocol PropertyObserver: AnyObject {
    associatedtype Value
    func valueChanged(to newValue: Value)
}

class PropertyStore {
    
    // Notifikasi multiple observer dengan tipe berbeda sekaligus
    func notifyObservers<each O: PropertyObserver>(
        _ observers: repeat each O,
        with values: repeat (each O).Value
    ) {
        for (observer, value) in repeat each (observers, values) {
            observer.valueChanged(to: value)
        }
    }
}

// Konkret observer types
class IntObserver: PropertyObserver {
    var lastValue: Int?
    func valueChanged(to newValue: Int) { lastValue = newValue }
}

class StringObserver: PropertyObserver {
    var lastValue: String?
    func valueChanged(to newValue: String) { lastValue = newValue }
}

class BoolObserver: PropertyObserver {
    var lastValue: Bool?
    func valueChanged(to newValue: Bool) { lastValue = newValue }
}

// Penggunaan
let store = PropertyStore()
let countObserver = IntObserver()
let nameObserver = StringObserver()
let activeObserver = BoolObserver()

store.notifyObservers(
    countObserver, nameObserver, activeObserver,
    with: 42, "Ari", true
)

print(countObserver.lastValue!)   // 42
print(nameObserver.lastValue!)    // "Ari"
print(activeObserver.lastValue!)  // true
```

---

### Use Case 3: Dependency Container

```swift
// Type-safe dependency injection container
protocol Injectable {
    init()
}

class DependencyContainer {
    
    // Resolve multiple dependencies sekaligus, type-safe
    func resolve<each D: Injectable>(_ types: repeat (each D).Type) -> (repeat each D) {
        (repeat (each types).init())
    }
    
    // Make dan configure
    func make<each D: Injectable & Configurable>(
        _ types: repeat (each D).Type,
        configurations: repeat Configuration<each D>
    ) -> (repeat each D) {
        var instances = (repeat (each types).init())
        for (instance, config) in repeat each (instances, configurations) {
            // Ini tidak langsung bisa karena tuple mutation
            // Tapi konsepnya benar
        }
        return instances
    }
}

protocol Configurable {
    mutating func configure(with config: [String: Any])
}

struct Configuration<T> {
    let values: [String: Any]
}

// Services
struct UserService: Injectable {
    init() { print("UserService initialized") }
}

struct AuthService: Injectable {
    init() { print("AuthService initialized") }
}

struct AnalyticsService: Injectable {
    init() { print("AnalyticsService initialized") }
}

// Penggunaan
let container = DependencyContainer()

let (userService, authService, analyticsService) = container.resolve(
    UserService.self,
    AuthService.self,
    AnalyticsService.self
)
// Semua diinstansiasi, tipe-nya diketahui compiler
```

---

### Use Case 4: Assertion Helper untuk Testing

```swift
import Testing

// Type-safe assertion untuk banyak nilai sekaligus
func assertAll<each T: Equatable>(
    _ label: String,
    pairs: repeat (actual: each T, expected: each T),
    sourceLocation: SourceLocation = #_sourceLocation
) {
    var allPassed = true
    var failureMessages: [String] = []
    
    for (actual, expected) in repeat each pairs {
        if actual != expected {
            allPassed = false
            failureMessages.append("  Expected \(expected), got \(actual)")
        }
    }
    
    if !allPassed {
        let message = "\(label) assertion(s) failed:\n" + failureMessages.joined(separator: "\n")
        Issue.record(message, sourceLocation: sourceLocation)
    }
}

// Penggunaan dalam test
@Test("User creation sets all properties correctly")
func userCreation() {
    let user = User(name: "Ari", age: 28, email: "ari@example.com", isActive: true)
    
    // Assert banyak properti sekaligus, semua type-safe
    assertAll("User properties",
        pairs:
            (actual: user.name, expected: "Ari"),
            (actual: user.age, expected: 28),
            (actual: user.email, expected: "ari@example.com"),
            (actual: user.isActive, expected: true)
    )
    // Jika beberapa gagal, semua failure dilaporkan sekaligus
}

struct User {
    var name: String
    var age: Int
    var email: String
    var isActive: Bool
}
```

---

## 8. Ringkasan

### Kapan Pack Iteration vs Alternatif

```
Semua elemen punya tipe yang sama?
└── YA → Gunakan Array<T> atau [any Protocol]
    Lebih simple, runtime flexible

Tipe berbeda, jumlah FIXED?
└── YA → Gunakan tuple atau multiple parameters
    func process(_ a: Int, _ b: String, _ c: Bool) { }

Tipe berbeda, jumlah VARIABEL, butuh type-safety?
└── YA → Parameter Packs + Pack Iteration ← use case utama
    Contoh: validation framework, DI container, assertion helpers

Butuh tambah/hapus elemen saat runtime?
└── YA → Array<any Protocol> atau existential collection
    Pack di-resolve compile-time, tidak bisa dinamis
```

### Sintaks Cheat Sheet

```swift
// Deklarasi
func f<each T>(_ values: repeat each T) { }

// Dengan constraint
func f<each T: Equatable>(_ values: repeat each T) { }

// Return pack
func f<each T>(_ values: repeat each T) -> (repeat each T) {
    (repeat each values)
}

// Pack iteration (Swift 6)
for value in repeat each values {
    print(value)
}

// Collect ke Array
var results: [String] = []
for value in repeat each values {
    results.append("\(value)")
}

// Zip dua pack
for (a, b) in repeat each (firstPack, secondPack) {
    process(a, b)
}
```

### Learning Path

1. **Mulai dengan:** Generic functions biasa (`<T>`)
2. **Lanjut ke:** Multiple type parameters (`<T, U>`)
3. **Kemudian:** Parameter packs dasar (`<each T>`, `repeat each T`)
4. **Akhirnya:** Pack iteration (Swift 6) untuk logika yang lebih kompleks

Pack iteration adalah fitur yang **jarang dibutuhkan** di aplikasi sehari-hari, tapi menjadi **sangat powerful** saat membangun framework, testing library, atau validation engine yang harus bekerja dengan multiple tipe secara type-safe.

---

*Dibuat: 2026-05-08 | Swift 6.0*
