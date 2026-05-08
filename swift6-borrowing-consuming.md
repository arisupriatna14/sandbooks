# Swift 6: Borrowing dan Consuming Parameter Ownership
### Teori Mendalam, Real Use Case, dan Panduan Penggunaan

---

## Daftar Isi
1. [Masalah yang Dipecahkan](#1-masalah-yang-dipecahkan)
2. [Teori: Ownership dan Copy Semantics](#2-teori-ownership-dan-copy-semantics)
3. [Tiga Mode Parameter Ownership](#3-tiga-mode-parameter-ownership)
4. [borrowing — Read-Only Access](#4-borrowing--read-only-access)
5. [consuming — Transfer Ownership](#5-consuming--transfer-ownership)
6. [copy — Explicit Copy](#6-copy--explicit-copy)
7. [Ownership dalam Return dan Closures](#7-ownership-dalam-return-dan-closures)
8. [Kapan Harus Digunakan](#8-kapan-harus-digunakan)
9. [Kapan Tidak Digunakan](#9-kapan-tidak-digunakan)
10. [Real Use Cases](#10-real-use-cases)
11. [Ringkasan](#11-ringkasan)

---

## 1. Masalah yang Dipecahkan

### Copy yang Tidak Perlu: Biaya Tersembunyi

Swift secara default meng-copy value type ketika diteruskan ke fungsi. Untuk tipe kecil (Int, Bool, CGPoint), ini murah dan tidak masalah. Tapi untuk tipe besar, copy ini bisa menjadi overhead yang signifikan:

```swift
struct LargeImage {
    var pixels: [UInt8]  // bisa ratusan MB
    var width: Int
    var height: Int
}

// Setiap kali ini dipanggil, Swift COPY seluruh pixels array!
func analyze(_ image: LargeImage) -> AnalysisResult {
    // image adalah SALINAN — copy terjadi di pemanggilan
    return process(image.pixels)
}

// Tiga copy jika dipanggil tiga kali:
let report1 = analyze(myImage)  // copy #1
let report2 = analyze(myImage)  // copy #2
let report3 = analyze(myImage)  // copy #3
```

**Mengapa Swift copy by default?**

Swift menggunakan **Copy-on-Write (CoW)** untuk collection types — copy hanya terjadi saat ada mutation. Tapi copy tetap terjadi jika:
1. Kamu meneruskan ke fungsi yang bisa memutasi (compiler tidak bisa prove tidak akan mutate)
2. Batas antara non-CoW types

**Apa yang developer butuhkan:**
- Cara untuk menyatakan: "fungsi ini HANYA membaca, tidak akan memutasi atau menyimpan"
- Cara untuk menyatakan: "fungsi ini mengambil alih nilai, caller tidak butuh lagi"

Ini adalah inti dari `borrowing` dan `consuming`.

### Masalah Keterbacaan: Intent yang Tidak Jelas

```swift
// Apakah fungsi ini menyimpan reference ke image?
// Apakah aman untuk menggunakan image setelah ini?
// Tidak ada cara tahu dari signature saja:
func upload(_ image: UIImage) async { }
func cache(_ image: UIImage) { }
func process(_ image: UIImage) -> Result { }
```

Dengan `borrowing` dan `consuming`, intent menjadi bagian dari API contract yang di-enforce compiler.

---

## 2. Teori: Ownership dan Copy Semantics

### Tiga Model Melewatkan Value

```
┌─────────────────────────────────────────────────────────┐
│                  Caller memiliki nilai X                  │
└──────────────────────┬──────────────────────────────────┘
                       │ bagaimana X diteruskan ke callee?
           ┌───────────┼───────────────┐
           ▼           ▼               ▼
      borrowing    consuming          inout
   ─────────────  ──────────────   ──────────────
   Caller masih   Ownership         Caller masih
   pegang X       berpindah ke      pegang X, tapi
   Callee: read   callee            callee bisa
   only           Caller: tidak     modifikasi X
                  bisa pakai X lagi langsung
```

### Mengapa Borrowing Lebih Efisien dari Default

Default Swift (tanpa annotation) menggunakan **"let-binding"** — compiler melakukan analisis dan **mungkin** melakukan copy atau tidak. Perilakunya bergantung pada banyak faktor dan tidak bisa diprediksi developer.

`borrowing` memberi **jaminan explicit**: *"tidak ada copy yang dilakukan, nilai hanya dipinjam secara read-only."*

```
Default (tanpa annotation):
Caller → [mungkin copy] → Callee
Compiler memutuskan — tidak bisa diprediksi

borrowing:
Caller → [reference/borrow, tidak ada copy] → Callee
Compile-time guarantee

consuming:
Caller → [move, tidak ada copy] → Callee
Caller tidak bisa akses lagi, tidak ada copy
```

### Hubungan dengan ~Copyable

`borrowing` dan `consuming` adalah konsep yang sudah ada di Swift bahkan sebelum `~Copyable`, tapi menjadi **wajib** untuk parameter `~Copyable` karena compiler harus tahu bagaimana menangani nilai yang tidak bisa di-copy:

```swift
struct NonCopyableHandle: ~Copyable { }

// Tanpa annotation, compiler tidak tahu harus borrow atau consume
// Dengan ~Copyable, kamu HARUS tentukan:
func inspect(handle: borrowing NonCopyableHandle) { }    // wajib explicit
func transfer(handle: consuming NonCopyableHandle) { }   // wajib explicit
```

---

## 3. Tiga Mode Parameter Ownership

### Ringkasan Cepat

| Mode | Keyword | Caller setelah call | Copy terjadi? | Mutasi? |
|---|---|---|---|---|
| Shared (read-only) | `borrowing` | Masih bisa pakai | Tidak | Tidak |
| Owned (take over) | `consuming` | Tidak bisa pakai | Tidak (move) | Ya (callee) |
| Mutating (modify in place) | `inout` | Masih bisa pakai (modified) | Tidak (pointer) | Ya (in-place) |
| Default (copy) | (tidak ada) | Masih bisa pakai | **Ya** | Ya (pada copy) |

---

## 4. borrowing — Read-Only Access

### Konsep

`borrowing` menyatakan: *"Saya hanya membaca nilai ini. Saya tidak akan menyimpannya, mengubahnya, atau mengambil kepemilikannya. Caller tetap memiliki nilai setelah fungsi ini selesai."*

```swift
struct DocumentData {
    var pages: [[String]]  // bisa ribuan halaman
    var metadata: [String: String]
}

// 'borrowing': tidak ada copy, tidak ada ownership transfer
func wordCount(_ doc: borrowing DocumentData) -> Int {
    doc.pages.flatMap { $0 }.joined(separator: " ")
        .components(separatedBy: .whitespaces).count
}

// Penggunaan:
let document = DocumentData(pages: largePages, metadata: [:])
let count = wordCount(document)  // tidak ada copy!
print(count)
// document masih bisa dipakai di sini — borrow selesai
let preview = wordCount(document)  // borrow lagi — fine
```

### Aturan borrowing

```swift
func inspect(value: borrowing LargeStruct) {
    print(value.field)         // ✅ baca — OK
    // value.field = "new"     // ❌ mutasi — tidak diizinkan
    // let copy = value        // ❌ copy explicit — tidak diizinkan tanpa 'copy' keyword
    // someVar = value         // ❌ menyimpan — tidak diizinkan
    // asyncFunc(value)        // ❌ escape ke async — tidak diizinkan
}
```

### borrowing dan Protocol Conformance

```swift
protocol Inspectable {
    func inspect() -> String
}

struct LargeData: Inspectable {
    var items: [String]
    
    // borrowing method: tidak ada copy self
    borrowing func inspect() -> String {
        items.prefix(5).joined(separator: ", ")
    }
    
    // Ini berbeda dari 'nonisolated' di actor — ini tentang copy
    // mutating func: caller mendapat versi yang sudah dimodifikasi
    mutating func append(_ item: String) {
        items.append(item)
    }
}
```

---

## 5. consuming — Transfer Ownership

### Konsep

`consuming` menyatakan: *"Saya mengambil alih nilai ini. Caller tidak boleh menggunakannya lagi setelah memanggil fungsi ini."*

```swift
struct EncryptedData {
    var ciphertext: [UInt8]
    var nonce: [UInt8]
}

// 'consuming': mengambil alih EncryptedData
// Setelah ini, caller tidak bisa pakai variabelnya
func decrypt(_ data: consuming EncryptedData) -> Data {
    // data adalah milik kita sekarang
    let decrypted = performDecryption(data.ciphertext, nonce: data.nonce)
    // data.ciphertext di-zero otomatis saat fungsi ini selesai (jika ~Copyable)
    return Data(decrypted)
}

// Penggunaan:
var encrypted = EncryptedData(ciphertext: bytes, nonce: nonceBytes)
let plaintext = decrypt(encrypted)  // encrypted di-consume di sini
// print(encrypted.ciphertext)       // ❌ ERROR: 'encrypted' sudah di-consume
```

### consuming Method

```swift
struct RequestBuilder {
    private var url: URL
    private var headers: [String: String] = [:]
    private var body: Data?
    
    init(url: URL) { self.url = url }
    
    // consuming method: return self yang sudah dimodifikasi
    // Ini adalah Builder Pattern yang type-safe!
    consuming func header(_ name: String, value: String) -> RequestBuilder {
        var builder = self  // consume self, buat salinan baru
        builder.headers[name] = value
        return builder
    }
    
    consuming func body(_ data: Data) -> RequestBuilder {
        var builder = self
        builder.body = data
        return builder
    }
    
    // consuming: build menghabiskan builder
    consuming func build() -> URLRequest {
        var request = URLRequest(url: url)
        headers.forEach { request.setValue($1, forHTTPHeaderField: $0) }
        request.httpBody = body
        return request
    }
}

// Penggunaan — Builder Pattern yang ergonomis
let request = RequestBuilder(url: apiURL)
    .header("Authorization", value: "Bearer \(token)")
    .header("Content-Type", value: "application/json")
    .body(requestBody)
    .build()
// Setiap tahap consume builder sebelumnya dan buat yang baru
```

### Consuming dalam Pipeline

```swift
// Pipeline processing — setiap tahap mengambil alih dan mengembalikan nilai baru
struct ImageProcessor {
    var pixelData: [UInt8]
    var width: Int
    var height: Int
    
    consuming func resize(to size: CGSize) -> ImageProcessor {
        var new = self
        new.pixelData = performResize(new.pixelData, size: size)
        new.width = Int(size.width)
        new.height = Int(size.height)
        return new
    }
    
    consuming func applyFilter(_ filter: ImageFilter) -> ImageProcessor {
        var new = self
        new.pixelData = filter.apply(to: new.pixelData)
        return new
    }
    
    consuming func compress(quality: Double) -> Data {
        return compressToJPEG(pixelData, quality: quality)
    }
}

// Pipeline yang jelas — setiap tahap mengambil alih
let compressed = ImageProcessor(pixelData: rawPixels, width: 4000, height: 3000)
    .resize(to: CGSize(width: 800, height: 600))   // consume original, hasilkan resized
    .applyFilter(.sharpen)                          // consume resized, hasilkan filtered
    .compress(quality: 0.85)                        // consume filtered, hasilkan Data
```

---

## 6. copy — Explicit Copy

Ketika kamu perlu copy nilai yang sedang di-borrow (atau dalam konteks lain yang tidak otomatis copy):

```swift
func processWithCopy(_ value: borrowing LargeData) -> LargeData {
    // Perlu salinan untuk modifikasi — copy secara eksplisit
    var workingCopy = copy value  // explicit copy keyword
    workingCopy.items.append("new item")
    return workingCopy
}
```

Ini berbeda dari assignment biasa dalam konteks tertentu karena membuat intent jelas: *"Saya tahu ini mahal, dan saya secara sadar melakukan copy di sini."*

---

## 7. Ownership dalam Return dan Closures

### Return Values

```swift
// Nilai yang di-return selalu "consuming" — ownership berpindah ke caller
func createLargeData() -> LargeData {
    var data = LargeData()
    // ... populate data
    return data  // data di-"consume" ke return value — tidak ada copy
}

let result = createLargeData()  // result sekarang memiliki LargeData
```

### Closures dan Capturing

```swift
// Closure yang capture nilai borrowing — tidak bisa escape
func process(data: borrowing LargeData, sync handler: () -> Void) {
    // handler bisa capture data karena tidak escape
    handler()
}

// Closure yang escape — capture harus consume atau copy
func processAsync(data: consuming LargeData, async handler: @escaping () async -> Void) {
    Task {
        await handler()
        // data tidak bisa diakses di sini karena sudah di-consume
    }
}
```

---

## 8. Kapan Harus Digunakan

### Gunakan `borrowing` ketika:

**1. Fungsi hanya membaca nilai (pure read)**
```swift
// Apapun yang hanya "inspect" atau "compute from" tanpa menyimpan
func serialize(data: borrowing UserProfile) -> Data { ... }
func validate(form: borrowing RegistrationForm) throws { ... }
func checksum(buffer: borrowing [UInt8]) -> UInt32 { ... }
func render(scene: borrowing Scene3D) -> UIImage { ... }
```

**2. Fungsi yang sering dipanggil dengan nilai yang sama**
```swift
// Dipanggil berulang — borrowing mencegah copy berulang
func render(frame: borrowing AnimationFrame) {
    // dipanggil 60 kali per detik — tanpa borrowing: 60 copies per detik
}
```

**3. Large value types yang mahal untuk di-copy**
```swift
struct MLModel {
    var weights: [[Float]]  // bisa ratusan MB
}

func inference(model: borrowing MLModel, input: Tensor) -> Tensor {
    // model tidak di-copy — sangat penting untuk performa
}
```

### Gunakan `consuming` ketika:

**1. Builder pattern**
```swift
// Setiap step menciptakan nilai baru
consuming func withOption(_ option: Option) -> Self { ... }
consuming func build() -> Product { ... }
```

**2. Resource transfer yang jelas**
```swift
// Memperjelas bahwa setelah ini, caller tidak boleh pakai lagi
consuming func submit(_ job: Job) async { ... }
consuming func send(_ message: Message) async throws { ... }
```

**3. Pipeline processing**
```swift
// Setiap tahap mengambil alih dan menghasilkan nilai baru
consuming func filtered(by predicate: Predicate) -> DataSet { ... }
consuming func sorted(by comparator: Comparator) -> DataSet { ... }
consuming func limited(to count: Int) -> DataSet { ... }
```

**4. Noncopyable types — WAJIB**
```swift
struct FileHandle: ~Copyable { }
func process(handle: consuming FileHandle) { }  // wajib karena ~Copyable
```

---

## 9. Kapan Tidak Digunakan

### Hindari `borrowing` ketika:

**1. Tipe kecil (Int, Bool, struct dengan sedikit field primitif)**
```swift
// Tidak perlu — copy Int sangat murah, tidak ada manfaat borrowing
func double(value: borrowing Int) -> Int { value * 2 }  // overkill

// Cukup:
func double(_ value: Int) -> Int { value * 2 }
```

**2. Fungsi perlu menyimpan referensi atau mengirim ke async context**
```swift
// borrowing tidak bisa escape ke async atau stored reference
func cache(data: borrowing LargeData) {
    // ❌ tidak bisa: self.cachedData = data  (escape)
    // ❌ tidak bisa: Task { process(data) }  (async escape)
    
    // Kalau butuh ini, gunakan default (copy) atau consuming
}
```

### Hindari `consuming` ketika:

**1. Caller masih butuh nilai setelah call**
```swift
// Jika caller butuh nilai setelah call, jangan consuming
func logAndProcess(data: consuming UserData) { }

let userData = UserData(...)
logAndProcess(data: userData)
// print(userData.name)  // ❌ ERROR jika consuming — tidak bisa
```

**2. Tipe yang perlu di-share ke banyak tempat**
```swift
// Config yang dibaca dari banyak service — jangan consuming
func configure(service: Service, with config: consuming Configuration) {
    // service mengambil alih config — tidak bisa dipakai untuk service lain!
}

// Lebih baik: borrowing atau default copy
func configure(service: Service, with config: borrowing Configuration) { }
```

---

## 10. Real Use Cases

### Use Case 1: Image Processing Pipeline

```swift
import Foundation
import CoreImage

// Large image processing dengan ownership yang jelas
struct ProcessableImage {
    var cgImage: CGImage
    var metadata: [String: Any]
    
    var sizeInBytes: Int {
        cgImage.width * cgImage.height * cgImage.bitsPerPixel / 8
    }
}

struct ImagePipeline {
    
    // Hanya inspect — borrowing
    static func analyze(image: borrowing ProcessableImage) -> ImageAnalysis {
        let brightness = computeBrightness(image.cgImage)
        let sharpness = computeSharpness(image.cgImage)
        return ImageAnalysis(brightness: brightness, sharpness: sharpness, size: image.sizeInBytes)
    }
    
    // Transform — consuming + hasilkan baru
    static func resize(
        _ image: consuming ProcessableImage,
        to size: CGSize
    ) -> ProcessableImage {
        let resizedCG = resizeCGImage(image.cgImage, to: size)
        return ProcessableImage(
            cgImage: resizedCG,
            metadata: image.metadata  // metadata di-copy (kecil, OK)
        )
    }
    
    static func applyFilter(
        _ image: consuming ProcessableImage,
        filter: CIFilter
    ) -> ProcessableImage {
        let ciImage = CIImage(cgImage: image.cgImage)
        filter.setValue(ciImage, forKey: kCIInputImageKey)
        let outputCI = filter.outputImage!
        let outputCG = CIContext().createCGImage(outputCI, from: outputCI.extent)!
        return ProcessableImage(cgImage: outputCG, metadata: image.metadata)
    }
    
    // Consuming final step
    static func export(
        _ image: consuming ProcessableImage,
        format: ExportFormat,
        quality: Double
    ) -> Data {
        switch format {
        case .jpeg: return encodeJPEG(image.cgImage, quality: quality)
        case .png: return encodePNG(image.cgImage)
        case .webp: return encodeWebP(image.cgImage, quality: quality)
        }
    }
}

// Penggunaan:
func processUserAvatar(original: ProcessableImage) async -> Data {
    // Analyze dulu (borrowing — tidak berubah)
    let analysis = ImagePipeline.analyze(image: original)
    
    // Proses (consuming pipeline)
    let processed = ImagePipeline
        .resize(original, to: CGSize(width: 256, height: 256))  // original di-consume
        // original tidak bisa dipakai lagi setelah ini ↑
        
    let filtered: ProcessableImage
    if analysis.sharpness < 0.5 {
        filtered = ImagePipeline.applyFilter(processed, filter: sharpenFilter)
    } else {
        filtered = processed  // move — tidak ada copy
    }
    
    return ImagePipeline.export(filtered, format: .webp, quality: 0.85)
}

struct ImageAnalysis { var brightness, sharpness: Double; var size: Int }
enum ExportFormat { case jpeg, png, webp }

func computeBrightness(_ image: CGImage) -> Double { 0.5 }
func computeSharpness(_ image: CGImage) -> Double { 0.7 }
func resizeCGImage(_ image: CGImage, to size: CGSize) -> CGImage { image }
func encodeJPEG(_ image: CGImage, quality: Double) -> Data { Data() }
func encodePNG(_ image: CGImage) -> Data { Data() }
func encodeWebP(_ image: CGImage, quality: Double) -> Data { Data() }
let sharpenFilter = CIFilter()
```

---

### Use Case 2: Query Builder dengan consuming

```swift
// SQL/NoSQL query builder yang type-safe dan zero-copy
struct QueryBuilder {
    private var table: String
    private var conditions: [String] = []
    private var orderByClause: String?
    private var limitCount: Int?
    private var selectColumns: [String] = ["*"]
    
    init(table: String) {
        self.table = table
    }
    
    // consuming: setiap step menghasilkan builder baru
    consuming func select(_ columns: String...) -> QueryBuilder {
        var builder = self
        builder.selectColumns = columns
        return builder
    }
    
    consuming func `where`(_ condition: String) -> QueryBuilder {
        var builder = self
        builder.conditions.append(condition)
        return builder
    }
    
    consuming func orderBy(_ column: String, ascending: Bool = true) -> QueryBuilder {
        var builder = self
        builder.orderByClause = "\(column) \(ascending ? "ASC" : "DESC")"
        return builder
    }
    
    consuming func limit(_ count: Int) -> QueryBuilder {
        var builder = self
        builder.limitCount = count
        return builder
    }
    
    // Terminal operation — consuming builder, menghasilkan SQL string
    consuming func toSQL() -> String {
        var sql = "SELECT \(selectColumns.joined(separator: ", ")) FROM \(table)"
        if !conditions.isEmpty {
            sql += " WHERE " + conditions.joined(separator: " AND ")
        }
        if let order = orderByClause {
            sql += " ORDER BY \(order)"
        }
        if let limit = limitCount {
            sql += " LIMIT \(limit)"
        }
        return sql
    }
}

// Penggunaan — ergonomis dan type-safe
let query = QueryBuilder(table: "users")
    .select("id", "name", "email")
    .where("active = true")
    .where("age >= 18")
    .orderBy("created_at", ascending: false)
    .limit(10)
    .toSQL()

// query = "SELECT id, name, email FROM users WHERE active = true AND age >= 18 ORDER BY created_at DESC LIMIT 10"
```

---

### Use Case 3: Data Serializer dengan borrowing

```swift
// High-performance serializer — borrowing untuk semua input
struct BinarySerializer {
    private var buffer: [UInt8] = []
    
    // Semua encode methods: borrowing — tidak perlu copy input
    mutating func encode(string: borrowing String) {
        let bytes = Array(string.utf8)
        encode(uint32: UInt32(bytes.count))
        buffer.append(contentsOf: bytes)
    }
    
    mutating func encode(uint32 value: borrowing UInt32) {
        withUnsafeBytes(of: value.bigEndian) { buffer.append(contentsOf: $0) }
    }
    
    mutating func encode(data: borrowing Data) {
        encode(uint32: UInt32(data.count))
        buffer.append(contentsOf: data)
    }
    
    mutating func encode(array: borrowing [String]) {
        encode(uint32: UInt32(array.count))
        for item in array {
            encode(string: item)  // borrow setiap item
        }
    }
    
    // consuming: hasilkan final Data
    consuming func finalize() -> Data {
        Data(buffer)
    }
}

struct UserMessage {
    var id: String
    var content: String
    var attachments: [String]
    var timestamp: UInt32
}

// Serialize tanpa copy UserMessage
func serialize(message: borrowing UserMessage) -> Data {
    var serializer = BinarySerializer()
    serializer.encode(string: message.id)
    serializer.encode(string: message.content)
    serializer.encode(array: message.attachments)
    serializer.encode(uint32: message.timestamp)
    return serializer.finalize()
}

// Penggunaan — message tidak di-copy
let message = UserMessage(id: "msg_123", content: "Hello!", attachments: [], timestamp: 1234567890)
let serialized = serialize(message: message)  // borrowing — tidak ada copy
// message masih valid di sini
print(message.id)  // ✅ OK
```

---

## 11. Ringkasan

### Decision Guide

```
Fungsi ini perlu COPY input?
├── TIDAK perlu (hanya baca) → borrowing
│   Contoh: serialize, analyze, validate, render
│
└── YA perlu modifikasi/ownership
    │
    Caller butuh nilai setelah call?
    ├── TIDAK → consuming (move semantics)
    │   Contoh: submit, publish, build(), export()
    │
    └── YA → Default copy (tidak ada annotation)
        atau inout jika butuh modifikasi in-place
```

### Performance Impact

| Scenario | Tanpa Annotation | Dengan borrowing | Saving |
|---|---|---|---|
| Serialize 10MB struct | Copy 10MB | 0 copy | 100% |
| Render frame 60fps | Copy per frame | 0 copy | 100% |
| ML inference 100MB model | Copy 100MB | 0 copy | 100% |
| Small Int/Bool | Copy 8 bytes | Copy 8 bytes | ~0% |

### Aturan Praktis

```swift
// Tipe kecil (primitif, small struct) → tidak perlu annotation
func isValid(_ count: Int) -> Bool { count > 0 }

// Tipe besar yang hanya dibaca → borrowing
func analyze(_ data: borrowing LargeMatrix) -> Analysis { }

// Builder/pipeline → consuming
consuming func filtered(_ predicate: Predicate) -> QueryResult { }

// Noncopyable → harus borrowing atau consuming
func use(_ handle: borrowing FileHandle) { }
func close(_ handle: consuming FileHandle) { }
```

---

*Dibuat: 2026-05-08 | Swift 6.0*
