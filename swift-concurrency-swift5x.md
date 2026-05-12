# Swift Concurrency di Swift 5.x — Adopsi Bertahap Tanpa Swift 6 Mode

### Panduan Lengkap: API, Real Use Case, VIP UIKit, Migration Guide, dan Best Practices

> **Target pembaca:** iOS/macOS developer yang project-nya masih berjalan di Swift 5.x dan ingin mulai
> mengadopsi Swift Concurrency (async/await, actor, Sendable) secara aman dan bertahap,
> tanpa harus langsung melompat ke Swift 6 strict concurrency mode.

---

## Daftar Isi

1. [Landscape: Apa yang Tersedia di Swift 5.x](#1-landscape)
2. [async/await — Fondasi Concurrency](#2-asyncawait)
3. [Structured Concurrency: Task, async let, TaskGroup](#3-structured-concurrency)
4. [Actor & @MainActor — Isolation Berbasis Tipe](#4-actor--mainactor)
5. [Sendable — Keamanan Lintas Batas Concurrency](#5-sendable)
6. [Continuation — Jembatan dari Callback ke Async](#6-continuation)
7. [AsyncSequence & AsyncStream — Streaming Data](#7-asyncsequence--asyncstream)
8. [@preconcurrency — Integrasi Library Lama](#8-preconcurrency)
9. [VIP Pattern + UIKit + Swift Concurrency](#9-vip-pattern--uikit)
10. [Migration Guide: GCD/Callback → Swift Concurrency](#10-migration-guide)
11. [Tips & Best Practices](#11-tips--best-practices)
12. [Ringkasan & Quick Reference](#12-ringkasan--quick-reference)

---

## 1. Landscape

### Apa yang Tersedia di Swift 5.x (Tanpa Swift 6 Mode)

Swift mulai memperkenalkan concurrency secara bertahap sejak Swift 5.5. Semua fitur di bawah
bisa digunakan **tanpa mengaktifkan Swift 6 language mode** — artinya pelanggaran isolation
menghasilkan *warning*, bukan *error*.

| Fitur | Versi Minimum | iOS Minimum | Keterangan |
|---|---|---|---|
| `async`/`await` | Swift 5.5 | iOS 15 | Fondasi utama |
| `actor`, `@MainActor` | Swift 5.5 | iOS 15 | Isolation berbasis tipe |
| `Sendable`, `@Sendable` | Swift 5.5 | iOS 15 | Data race prevention |
| `async let`, `TaskGroup` | Swift 5.5 | iOS 15 | Structured concurrency |
| `Task {}`, `Task.detached {}` | Swift 5.5 | iOS 15 | Unstructured concurrency |
| `AsyncSequence`, `AsyncStream` | Swift 5.5 | iOS 15 | Streaming data async |
| `withCheckedContinuation` | Swift 5.5 | iOS 15 | Bridge callback → async |
| `@preconcurrency` | Swift 5.6 | iOS 15 | Suppress warning library lama |
| `distributed actor` | Swift 5.7 | iOS 16 | Distributed computing (advanced) |
| `@Observable` | Swift 5.9 | iOS 17 | Observation framework baru |
| **Strict concurrency (error)** | **Swift 6 only** | — | Pelanggaran = compile error |
| **`sending` parameter** | **Swift 6 only** | — | Region-based isolation |
| **`~Copyable` noncopyable** | **Swift 6 only** | — | Ownership ketat |

> **Catatan penting:** Di Swift 5.x, compiler memeriksa concurrency tetapi menghasilkan *warning*
> bukan *error*. Ini memberi ruang untuk migrasi bertahap. Namun, tanpa disiplin,
> warning yang diabaikan bisa menjadi data race di runtime.

---

## 2. async/await

### Teori: Masalah yang Dipecahkan

Sebelum async/await, kode asinkron di Swift menggunakan completion handler:

```swift
// SEBELUM — Completion handler: callback pyramid, error prone
func loadUserProfile(id: String,
                     completion: @escaping (Result<User, Error>) -> Void) {
    URLSession.shared.dataTask(with: makeRequest(id: id)) { data, response, error in
        if let error = error {
            completion(.failure(error))
            return
        }
        guard let data = data else {
            completion(.failure(AppError.noData))
            return
        }
        do {
            let user = try JSONDecoder().decode(User.self, from: data)
            completion(.success(user))
        } catch {
            completion(.failure(error))
        }
    }.resume()
}

// Pemanggil harus ingat memanggil completion di semua code path
loadUserProfile(id: "123") { result in
    switch result {
    case .success(let user):
        DispatchQueue.main.async { self.updateUI(user: user) }
    case .failure(let error):
        DispatchQueue.main.async { self.showError(error) }
    }
}
```

Masalah utama:
- Mudah lupa memanggil `completion` → silent hang
- Harus manual dispatch ke main thread untuk UI
- Sulit dibaca karena alur tidak linear
- Error handling terpencar

### Sintaks async/await

```swift
// SESUDAH — async/await: linear, mudah dibaca
func loadUserProfile(id: String) async throws -> User {
    let request = makeRequest(id: id)
    // await menangguhkan fungsi ini, thread tidak diblokir
    let (data, _) = try await URLSession.shared.data(for: request)
    return try JSONDecoder().decode(User.self, from: data)
}

// Cara memanggil dari konteks non-async (misal: UIKit action)
Task {
    do {
        let user = try await loadUserProfile(id: "123")
        await MainActor.run { self.updateUI(user: user) }
    } catch {
        await MainActor.run { self.showError(error) }
    }
}
```

**Keyword penting:**
- `async` — tandai fungsi sebagai asinkron, bisa di-suspend
- `await` — titik di mana fungsi boleh di-suspend, menunggu hasil
- `async throws` — fungsi asinkron yang bisa gagal

### Kapan Menggunakan async/await

**Gunakan:**
- Operasi I/O: network request, baca/tulis file, database query
- Fungsi yang memanggil fungsi async lain
- Mengganti completion handler yang sudah ada

**Jangan gunakan:**
- Komputasi CPU-intensive murni (gunakan `Task.detached` atau `DispatchQueue` global)
- Kode yang harus backward-compatible sampai iOS 14 atau lebih lama

### Real Use Case: NetworkService dengan URLSession

```swift
// Models — Sendable agar aman dikirim lintas thread
struct User: Decodable, Sendable {
    let id: String
    let name: String
    let email: String
}

struct Post: Decodable, Sendable {
    let id: Int
    let title: String
    let body: String
}

// NetworkService — kumpulan pure async function
struct NetworkService {
    private let baseURL = URL(string: "https://jsonplaceholder.typicode.com")!
    private let decoder = JSONDecoder()

    func fetchUser(id: String) async throws -> User {
        let url = baseURL.appendingPathComponent("users/\(id)")
        let (data, response) = try await URLSession.shared.data(from: url)

        guard let http = response as? HTTPURLResponse,
              (200..<300).contains(http.statusCode) else {
            throw NetworkError.badResponse
        }

        return try decoder.decode(User.self, from: data)
    }

    func fetchPosts(userId: String) async throws -> [Post] {
        var components = URLComponents(url: baseURL.appendingPathComponent("posts"),
                                       resolvingAgainstBaseURL: false)!
        components.queryItems = [URLQueryItem(name: "userId", value: userId)]
        let (data, _) = try await URLSession.shared.data(from: components.url!)
        return try decoder.decode([Post].self, from: data)
    }
}

enum NetworkError: Error {
    case badResponse
}
```

---

## 3. Structured Concurrency

### Teori: Task Tree dan Structured Concurrency

Swift Concurrency menggunakan model **structured concurrency** — setiap Task memiliki parent,
dan lifetime-nya terikat pada scope yang membuatnya. Ini mencegah "fire and forget" yang
tidak terdeteksi.

```
Task (root)
├── async let userTask
├── async let postsTask
└── withTaskGroup
    ├── subtask A
    ├── subtask B
    └── subtask C
```

Jika parent di-cancel atau selesai, semua child otomatis di-cancel.

### Task {} vs Task.detached {}

```swift
class ProfileViewController: UIViewController {

    // Task {} — mewarisi actor isolation dari konteks pemanggil
    // Di sini mewarisi @MainActor dari ViewController
    func loadProfile() {
        Task {
            // Berjalan di @MainActor context (inherited)
            let user = try await service.fetchUser(id: "1")
            // Langsung update UI, tidak perlu dispatch ke main
            self.nameLabel.text = user.name
        }
    }

    // Task.detached {} — TIDAK mewarisi isolation, berjalan di background
    // Gunakan hanya jika perlu explicitly lepas dari actor context
    func processInBackground() {
        Task.detached(priority: .background) {
            let result = await heavyComputation()
            // Harus kembali ke MainActor secara manual untuk update UI
            await MainActor.run {
                self.resultLabel.text = result
            }
        }
    }
}
```

> **Rekomendasi:** Gunakan `Task {}` (bukan `Task.detached`) sebagai default.
> `Task.detached` hanya untuk kasus yang benar-benar perlu lepas dari isolation parent.

### async let — Concurrent Binding

`async let` menjalankan beberapa operasi secara paralel tanpa TaskGroup:

```swift
func loadDashboard() async throws -> Dashboard {
    // Ketiga fetch ini berjalan BERSAMAAN, bukan berurutan
    async let user = service.fetchUser(id: currentUserId)
    async let posts = service.fetchPosts(userId: currentUserId)
    async let config = service.fetchAppConfig()

    // await semua sekaligus — total waktu = waktu terlama dari ketiganya
    let (u, p, c) = try await (user, posts, config)

    return Dashboard(user: u, posts: p, config: c)
}

// Bandingkan dengan sequential — LEBIH LAMBAT
func loadDashboardSlow() async throws -> Dashboard {
    let user = try await service.fetchUser(id: currentUserId)   // tunggu selesai
    let posts = try await service.fetchPosts(userId: currentUserId) // baru mulai
    let config = try await service.fetchAppConfig()              // baru mulai
    return Dashboard(user: user, posts: posts, config: config)
}
```

### withTaskGroup — Concurrency Dinamis

Ketika jumlah task tidak diketahui saat compile time:

```swift
func fetchAllUsers(ids: [String]) async throws -> [User] {
    try await withThrowingTaskGroup(of: User.self) { group in
        // Tambahkan task secara dinamis
        for id in ids {
            group.addTask {
                try await service.fetchUser(id: id)
            }
        }

        // Kumpulkan semua hasil
        var users: [User] = []
        for try await user in group {
            users.append(user)
        }
        return users
    }
}
```

### Cancellation

```swift
func longRunningOperation() async throws -> [Result] {
    var results: [Result] = []

    for item in largeDataset {
        // Cek cancellation di setiap iterasi
        try Task.checkCancellation()  // throws CancellationError jika cancelled

        // Atau cek secara manual
        if Task.isCancelled {
            return results  // return partial result
        }

        let result = try await processItem(item)
        results.append(result)
    }

    return results
}

// Simpan referensi Task untuk bisa cancel nanti
class SearchViewController: UIViewController {
    private var searchTask: Task<Void, Never>?

    func search(query: String) {
        // Cancel search sebelumnya jika masih berjalan
        searchTask?.cancel()

        searchTask = Task {
            do {
                let results = try await searchService.search(query: query)
                self.displayResults(results)
            } catch is CancellationError {
                // Task di-cancel, abaikan — tidak perlu show error ke user
                return
            } catch {
                self.showError(error)
            }
        }
    }
}
```

---

## 4. Actor & @MainActor

### Teori: Isolation Berbasis Tipe

Actor memproteksi state-nya dengan memastikan hanya satu Task yang mengaksesnya pada satu waktu.
Tidak perlu lock manual — compiler dan runtime yang menjamin ini.

```
Thread A ──► await cache.value ──► (antri)
                                         │
Thread B ──► await cache.value ──► (antri)
                                         │
                                   Actor (serial access)
                                         │
                               Task A berjalan dulu
                               Task B berjalan setelah A selesai
```

### actor — Custom Actor

```swift
// Tanpa actor — DATA RACE! Bisa crash atau corrupt data
class UnsafeImageCache {
    private var store: [String: UIImage] = [:]  // ← bisa diakses dari mana saja!

    func store(_ image: UIImage, for key: String) {
        store[key] = image  // ← tidak thread-safe
    }

    func image(for key: String) -> UIImage? {
        return store[key]
    }
}

// DENGAN actor — compiler menjamin thread safety
actor ImageCache {
    private var store: [String: UIImage] = [:]

    // Akses ke state hanya bisa lewat await dari luar actor
    func store(_ image: UIImage, for key: String) {
        store[key] = image
    }

    func image(for key: String) -> UIImage? {
        return store[key]
    }

    // nonisolated — tidak butuh await saat dipanggil, tidak akses state actor
    nonisolated var description: String {
        return "ImageCache"
    }
}

// Cara menggunakan
let cache = ImageCache()

Task {
    // await wajib karena ImageCache adalah actor
    await cache.store(image, for: "avatar_123")
    let img = await cache.image(for: "avatar_123")
}
```

### @MainActor — Isolation ke Main Thread

`@MainActor` adalah global actor bawaan Swift yang mewakili main thread.
Tandai class/property/function dengan `@MainActor` untuk memastikan dijalankan di main thread.

```swift
// @MainActor pada class — semua properti dan method berjalan di main thread
@MainActor
class ProfileViewModel {
    // Properti ini otomatis di-isolate ke main thread
    var userName: String = ""
    var isLoading: Bool = false
    var errorMessage: String? = nil

    private let service: NetworkService

    init(service: NetworkService) {
        self.service = service
    }

    // Method ini berjalan di main thread — aman update UI langsung
    func fetchProfile(userId: String) async {
        isLoading = true
        errorMessage = nil

        do {
            // service.fetchUser adalah async func — suspension point
            // setelah await kembali, kita sudah di main thread lagi
            let user = try await service.fetchUser(id: userId)
            userName = user.name
        } catch {
            errorMessage = error.localizedDescription
        }

        isLoading = false
    }

    // nonisolated untuk pure computation yang tidak sentuh state @MainActor
    nonisolated func formatDate(_ date: Date) -> String {
        let formatter = DateFormatter()
        formatter.dateStyle = .medium
        return formatter.string(from: date)
    }
}
```

### isolated Parameter — Inject Isolation

```swift
// Fungsi yang menerima actor sebagai parameter terisolasi
func updateUI(on viewController: isolated UIViewController & AnyObject,
              with data: UserData) {
    // Di sini kita isolated ke viewController
    // Bisa akses properti viewController tanpa await
}

// Lebih umum: inject @MainActor
@MainActor
func updateLabels(name: String, email: String) {
    nameLabel.text = name
    emailLabel.text = email
}
```

### Perbedaan Swift 5.x vs Swift 6

```swift
// Di Swift 5.x: ini menghasilkan WARNING (tidak error)
class MyViewController: UIViewController {
    var items: [String] = []  // Tidak ada isolation

    func loadFromBackground() {
        Task.detached {
            // ⚠️ Warning: crossing actor boundary tanpa await
            // Di Swift 6 ini adalah ERROR
            self.items = ["A", "B"]  // Akses main thread property dari background
        }
    }
}

// Solusi yang benar di Swift 5.x (dan Swift 6):
@MainActor
class MyViewController: UIViewController {
    var items: [String] = []

    func loadFromBackground() {
        Task.detached {
            let newItems = await fetchItems()
            // Kembali ke MainActor untuk update state
            await MainActor.run {
                self.items = newItems
            }
        }
    }
}
```

---

## 5. Sendable

### Teori: Keamanan Data Lintas Batas Actor

`Sendable` adalah protokol marker yang menandai bahwa sebuah tipe aman untuk dikirim
lintas batas concurrency (dari satu actor ke actor lain, atau ke Task baru).

```
Actor A ──► [UserDTO: Sendable] ──► Actor B     ✅ Aman
Actor A ──► [UserObject: class] ──► Actor B     ⚠️  Berbahaya — bisa data race
```

### Sendable Otomatis

```swift
// Struct dengan semua properti Sendable → otomatis Sendable
struct UserDTO: Sendable {
    let id: UUID       // UUID: Sendable ✅
    let name: String   // String: Sendable ✅
    let age: Int       // Int: Sendable ✅
}

// Enum dengan associated values yang Sendable → otomatis Sendable
enum LoadingState: Sendable {
    case idle
    case loading
    case success(UserDTO)  // UserDTO: Sendable ✅
    case failure(String)   // String: Sendable ✅
}

// Value type sederhana tidak perlu deklarasi eksplisit (compiler inferkan)
```

### @unchecked Sendable — Opt-out Safety Check

Untuk class yang thread-safe secara manual tapi tidak bisa diverifikasi compiler:

```swift
// NSCache tidak conform Sendable, tapi thread-safe
final class ImageCacheWrapper: @unchecked Sendable {
    // NSCache sudah thread-safe secara internal
    private let cache = NSCache<NSString, UIImage>()

    func set(_ image: UIImage, for key: String) {
        cache.setObject(image, forKey: key as NSString)
    }

    func get(_ key: String) -> UIImage? {
        return cache.object(forKey: key as NSString)
    }
}

// Class dengan lock manual
final class ThreadSafeCounter: @unchecked Sendable {
    private var _value: Int = 0
    private let lock = NSLock()

    var value: Int {
        lock.withLock { _value }
    }

    func increment() {
        lock.withLock { _value += 1 }
    }
}
```

> **Perhatian:** `@unchecked Sendable` artinya kamu yang bertanggung jawab membuktikan
> thread safety-nya. Jangan gunakan sebagai jalan pintas untuk menyembunyikan bug.

### @Sendable Closure

Closure yang akan dikirim ke Task atau actor lain harus ditandai `@Sendable`:

```swift
// @Sendable closure — tidak boleh capture mutable non-Sendable state
func performAsync(work: @Sendable @escaping () async -> Void) {
    Task {
        await work()
    }
}

// Penggunaan
performAsync {
    // Boleh: immutable capture
    let localValue = someImmutableValue
    await process(localValue)
}

// Salah — compiler warning di Swift 5.x (error di Swift 6)
var mutableState = [String]()
performAsync {
    mutableState.append("item")  // ⚠️ Mutation dari @Sendable closure
}
```

---

## 6. Continuation

### Teori: Jembatan dari Callback ke Async World

Banyak API Apple dan library pihak ketiga masih menggunakan completion handler.
`Continuation` memungkinkan kita membungkus API tersebut menjadi `async throws` function
tanpa mengubah library aslinya.

```
Kode async (await) ←──── Continuation ────→ Callback API
```

### withCheckedContinuation — Aman dengan Runtime Check

```swift
// Membungkus URLSession dataTask (cara lama) menjadi async
func fetchDataLegacy(from url: URL) async throws -> Data {
    try await withCheckedThrowingContinuation { continuation in
        URLSession.shared.dataTask(with: url) { data, response, error in
            // Resume HARUS dipanggil tepat satu kali
            if let error = error {
                continuation.resume(throwing: error)
                return
            }

            guard let data = data else {
                continuation.resume(throwing: NetworkError.noData)
                return
            }

            continuation.resume(returning: data)
            // Setelah ini jangan panggil continuation lagi!
        }.resume()
    }
}

// Membungkus operasi yang tidak bisa gagal
func requestPermission() async -> Bool {
    await withCheckedContinuation { continuation in
        PHPhotoLibrary.requestAuthorization { status in
            continuation.resume(returning: status == .authorized)
        }
    }
}
```

**Aturan Continuation:**
- `resume` HARUS dipanggil **tepat 1 kali**
- Memanggil 0 kali → task leak (hang selamanya)
- Memanggil 2+ kali → crash (`withCheckedContinuation` akan crash, `withUnsafeContinuation` undefined behavior)

### AsyncStream dari Delegate

Untuk API yang memanggil callback berulang kali (delegate, NotificationCenter):

```swift
// CLLocationManager → AsyncStream
class LocationService {
    private let locationManager = CLLocationManager()

    // Expose lokasi sebagai AsyncStream
    var locationStream: AsyncStream<CLLocation> {
        AsyncStream { continuation in
            let delegate = LocationDelegate { location in
                continuation.yield(location)
            }

            continuation.onTermination = { _ in
                // Cleanup saat stream dihentikan
                delegate.stopUpdates()
            }

            locationManager.delegate = delegate
            locationManager.startUpdatingLocation()
        }
    }
}

// Penggunaan
let locationService = LocationService()

Task {
    for await location in locationService.locationStream {
        print("Lokasi baru: \(location.coordinate)")

        // Stream otomatis berhenti saat Task di-cancel
        if shouldStop {
            break
        }
    }
}
```

```swift
// NotificationCenter → AsyncStream
extension NotificationCenter {
    func notifications(named name: Notification.Name,
                       object: AnyObject? = nil) -> AsyncStream<Notification> {
        AsyncStream { continuation in
            let observer = addObserver(forName: name, object: object, queue: nil) { notification in
                continuation.yield(notification)
            }

            continuation.onTermination = { _ in
                self.removeObserver(observer)
            }
        }
    }
}

// Penggunaan
Task {
    for await _ in NotificationCenter.default.notifications(named: UIApplication.didBecomeActiveNotification) {
        await refreshData()
    }
}
```

---

## 7. AsyncSequence & AsyncStream

### Teori: Data yang Datang Seiring Waktu

`AsyncSequence` adalah versi asinkron dari `Sequence` — cocok untuk data yang tidak tersedia
sekaligus, tapi datang secara bertahap.

```
Sequence (sync):  [1, 2, 3, 4, 5]          → tersedia langsung
AsyncSequence:     1 ... (delay) ... 2 ... (delay) ... 3    → tersedia bertahap
```

### AsyncStream — Custom Sequence Sederhana

```swift
// Timer sebagai AsyncStream
func timerStream(interval: TimeInterval) -> AsyncStream<Date> {
    AsyncStream { continuation in
        let timer = Timer.scheduledTimer(withTimeInterval: interval, repeats: true) { _ in
            continuation.yield(Date())
        }

        continuation.onTermination = { _ in
            timer.invalidate()
        }
    }
}

// Penggunaan
Task {
    for await tick in timerStream(interval: 1.0) {
        print("Tick: \(tick)")

        if someCondition { break }
    }
}
```

### AsyncThrowingStream — Stream yang Bisa Error

```swift
// WebSocket messages sebagai AsyncThrowingStream
func webSocketMessages(url: URL) -> AsyncThrowingStream<String, Error> {
    AsyncThrowingStream { continuation in
        let task = URLSession.shared.webSocketTask(with: url)

        func receiveNext() {
            task.receive { result in
                switch result {
                case .success(let message):
                    switch message {
                    case .string(let text):
                        continuation.yield(text)
                    case .data(let data):
                        continuation.yield(String(data: data, encoding: .utf8) ?? "")
                    @unknown default:
                        break
                    }
                    receiveNext()  // terus mendengarkan pesan berikutnya

                case .failure(let error):
                    continuation.finish(throwing: error)
                }
            }
        }

        task.resume()
        receiveNext()

        continuation.onTermination = { _ in
            task.cancel(with: .goingAway, reason: nil)
        }
    }
}

// Penggunaan
Task {
    do {
        for try await message in webSocketMessages(url: wsURL) {
            handleMessage(message)
        }
    } catch {
        handleDisconnection(error)
    }
}
```

### Transformasi AsyncSequence

```swift
// filter, map, dll. tersedia di AsyncSequence
let highValueEvents = eventStream
    .filter { $0.value > 100 }
    .map { EventViewModel(event: $0) }

Task {
    for await event in highValueEvents {
        displayEvent(event)
    }
}
```

---

## 8. @preconcurrency

### Teori: Library Lama Belum Adopt Sendable

Banyak library populer (Firebase, Realm, Alamofire versi lama) belum menandai tipe mereka
sebagai `Sendable`. Saat kamu menggunakannya di kode async, compiler memunculkan warning.

### @preconcurrency import

```swift
// Suppress semua warning Sendable dari module ini
@preconcurrency import RealmSwift
@preconcurrency import FirebaseFirestore

// Sekarang bisa gunakan tipe dari library tanpa warning
actor DataRepository {
    func fetchFromFirestore() async throws -> [QueryDocumentSnapshot] {
        // QueryDocumentSnapshot bukan Sendable, tapi warning di-suppress
        let snapshot = try await db.collection("users").getDocuments()
        return snapshot.documents
    }
}
```

### @preconcurrency pada Protocol Conformance

```swift
// Protocol dari library lama yang belum adopt Sendable
@preconcurrency
protocol LegacyDataSource: AnyObject {
    func fetchData(completion: @escaping ([String]) -> Void)
}

// Implementasi tanpa warning
class ModernDataSource: LegacyDataSource, @preconcurrency Sendable {
    func fetchData(completion: @escaping ([String]) -> Void) {
        Task {
            let data = await loadAsync()
            completion(data)
        }
    }
}
```

> **Catatan:** `@preconcurrency` bersifat sementara — idealnya dihapus setelah library
> bersangkutan sudah update dan conform `Sendable` dengan benar.

---

## 9. VIP Pattern + UIKit

### Arsitektur

VIP (View-Interactor-Presenter) + Swift Concurrency di Swift 5.x dengan prinsip isolation:

```
ProfileScene/
├── ProfileViewController.swift    — @MainActor, UI + trigger action
├── ProfileInteractor.swift        — actor, business logic + orchestration
├── ProfilePresenter.swift         — @MainActor, format data → ViewModel
├── ProfileWorker.swift            — pure async func, network/cache
├── ProfileModels.swift            — Sendable struct: Request, Response, ViewModel
└── ProfileRouter.swift            — @MainActor, navigasi
```

**Prinsip isolation per layer:**
- `ProfileViewController` → `@MainActor` (UI, main thread)
- `ProfileInteractor` → `actor` (business logic, serial access)
- `ProfilePresenter` → `@MainActor` (format data, dipanggil dari main thread)
- `ProfileWorker` → `async func` (stateless, tidak perlu actor)
- `ProfileModels` → semua `Sendable` (lintas batas actor)

### ProfileModels.swift

```swift
// Semua tipe Sendable agar bisa dikirim lintas actor

struct ProfileRequest: Sendable {
    let userId: String
}

struct ProfileResponse: Sendable {
    let user: User
    let posts: [Post]
}

struct ProfileViewModel: Sendable {
    let displayName: String
    let email: String
    let postCount: String
    let posts: [PostCellViewModel]
}

struct PostCellViewModel: Sendable {
    let id: Int
    let title: String
    let preview: String
}

struct User: Decodable, Sendable {
    let id: Int
    let name: String
    let email: String
    let username: String
}

struct Post: Decodable, Sendable {
    let id: Int
    let userId: Int
    let title: String
    let body: String
}
```

### ProfileWorker.swift

```swift
// Worker: stateless, pure async function
// Tidak perlu actor karena tidak punya mutable state
struct ProfileWorker {
    private let session: URLSession
    private let baseURL = URL(string: "https://jsonplaceholder.typicode.com")!
    private let decoder = JSONDecoder()

    init(session: URLSession = .shared) {
        self.session = session
    }

    func fetchUser(id: String) async throws -> User {
        let url = baseURL.appendingPathComponent("users/\(id)")
        let (data, response) = try await session.data(from: url)

        guard let http = response as? HTTPURLResponse,
              (200..<300).contains(http.statusCode) else {
            throw ProfileError.networkError
        }

        return try decoder.decode(User.self, from: data)
    }

    func fetchPosts(userId: Int) async throws -> [Post] {
        let url = baseURL.appendingPathComponent("posts")
        var components = URLComponents(url: url, resolvingAgainstBaseURL: false)!
        components.queryItems = [URLQueryItem(name: "userId", value: "\(userId)")]

        let (data, _) = try await session.data(from: components.url!)
        return try decoder.decode([Post].self, from: data)
    }
}

enum ProfileError: Error, LocalizedError {
    case networkError
    case notFound

    var errorDescription: String? {
        switch self {
        case .networkError: return "Koneksi bermasalah, coba lagi."
        case .notFound: return "Profil tidak ditemukan."
        }
    }
}
```

### ProfileInteractor.swift

```swift
// Interactor sebagai actor — melindungi business logic state
// Semua akses dari luar harus lewat await
actor ProfileInteractor {
    // Protocol untuk komunikasi ke presenter
    // Tidak perlu @MainActor — Swift otomatis hop ke MainActor
    // saat memanggil method @MainActor pada protocol
    weak var presenter: ProfilePresentationLogic?
    private let worker: ProfileWorker

    // State internal interactor — dilindungi actor isolation
    private var currentRequest: ProfileRequest?
    private var loadingTask: Task<Void, Never>?

    init(worker: ProfileWorker = ProfileWorker()) {
        self.worker = worker
    }

    // Dipanggil dari ViewController via Task { await interactor.fetchProfile(request:) }
    func fetchProfile(request: ProfileRequest) async {
        // Cancel request sebelumnya jika masih berjalan
        loadingTask?.cancel()
        currentRequest = request

        loadingTask = Task {
            do {
                // Fetch user dan posts secara paralel
                async let user = worker.fetchUser(id: request.userId)
                async let posts = worker.fetchPosts(userId: Int(request.userId) ?? 0)

                let (fetchedUser, fetchedPosts) = try await (user, posts)

                // Cek apakah sudah di-cancel sebelum meneruskan ke presenter
                guard !Task.isCancelled else { return }

                let response = ProfileResponse(user: fetchedUser, posts: fetchedPosts)

                // Swift otomatis hop ke @MainActor karena presentProfile di protocol adalah @MainActor
                await presenter?.presentProfile(response: response)
            } catch is CancellationError {
                // Request di-cancel, abaikan
            } catch {
                await presenter?.presentError(error: error)
            }
        }
    }
}

// Protocol untuk ViewController memanggil Interactor
protocol ProfileBusinessLogic: AnyObject {
    func fetchProfile(request: ProfileRequest) async
}

extension ProfileInteractor: ProfileBusinessLogic {}
```

### ProfilePresenter.swift

```swift
// Presenter berjalan di @MainActor — aman update UI langsung
@MainActor
class ProfilePresenter {
    weak var viewController: ProfileDisplayLogic?

    func presentProfile(response: ProfileResponse) {
        // Format data mentah menjadi ViewModel yang siap display
        let postViewModels = response.posts.map { post in
            PostCellViewModel(
                id: post.id,
                title: post.title,
                preview: String(post.body.prefix(80)).appending("...")
            )
        }

        let viewModel = ProfileViewModel(
            displayName: response.user.name,
            email: response.user.email,
            postCount: "\(response.posts.count) postingan",
            posts: postViewModels
        )

        viewController?.displayProfile(viewModel: viewModel)
    }

    func presentError(error: Error) {
        let message = (error as? LocalizedError)?.errorDescription
            ?? "Terjadi kesalahan. Silakan coba lagi."
        viewController?.displayError(message: message)
    }
}

// Protocol untuk Interactor memanggil Presenter
@MainActor
protocol ProfilePresentationLogic: AnyObject {
    func presentProfile(response: ProfileResponse)
    func presentError(error: Error)
}

extension ProfilePresenter: ProfilePresentationLogic {}
```

### ProfileViewController.swift

```swift
// ViewController di @MainActor — semua UI update aman di main thread
@MainActor
class ProfileViewController: UIViewController {

    // MARK: - UI Components
    private let nameLabel = UILabel()
    private let emailLabel = UILabel()
    private let postsCountLabel = UILabel()
    private let tableView = UITableView()
    private let activityIndicator = UIActivityIndicatorView(style: .medium)

    // MARK: - VIP
    var interactor: ProfileBusinessLogic?
    var router: ProfileRoutingLogic?

    // MARK: - State (dilindungi @MainActor)
    private var posts: [PostCellViewModel] = []

    // MARK: - Lifecycle
    override func viewDidLoad() {
        super.viewDidLoad()
        setupVIP()
        setupUI()
        fetchProfile()
    }

    private func setupVIP() {
        let interactor = ProfileInteractor()
        let presenter = ProfilePresenter()
        let router = ProfileRouter()

        presenter.viewController = self
        interactor.presenter = presenter
        router.viewController = self

        self.interactor = interactor
        self.router = router
    }

    // MARK: - Actions
    private func fetchProfile() {
        activityIndicator.startAnimating()
        let request = ProfileRequest(userId: "1")

        // Task {} mewarisi @MainActor dari ViewController
        // Setelah await, kembali ke main thread otomatis
        Task {
            await interactor?.fetchProfile(request: request)
        }
    }

    // MARK: - Display Logic
    func displayProfile(viewModel: ProfileViewModel) {
        // Sudah di @MainActor — langsung update UI
        activityIndicator.stopAnimating()
        nameLabel.text = viewModel.displayName
        emailLabel.text = viewModel.email
        postsCountLabel.text = viewModel.postCount
        posts = viewModel.posts
        tableView.reloadData()
    }

    func displayError(message: String) {
        activityIndicator.stopAnimating()
        let alert = UIAlertController(title: "Error",
                                      message: message,
                                      preferredStyle: .alert)
        alert.addAction(UIAlertAction(title: "OK", style: .default))
        present(alert, animated: true)
    }

    // MARK: - UI Setup (omitted for brevity)
    private func setupUI() { /* ... */ }
}

// Protocol untuk Presenter memanggil ViewController
@MainActor
protocol ProfileDisplayLogic: AnyObject {
    func displayProfile(viewModel: ProfileViewModel)
    func displayError(message: String)
}

extension ProfileViewController: ProfileDisplayLogic {}
```

### ProfileRouter.swift

```swift
// Router di @MainActor — navigasi selalu di main thread
@MainActor
class ProfileRouter {
    weak var viewController: UIViewController?

    func routeToPostDetail(post: PostCellViewModel) {
        let detailVC = PostDetailViewController(post: post)
        viewController?.navigationController?.pushViewController(detailVC, animated: true)
    }
}

@MainActor
protocol ProfileRoutingLogic: AnyObject {
    func routeToPostDetail(post: PostCellViewModel)
}

extension ProfileRouter: ProfileRoutingLogic {}
```

### Alur Data VIP + Concurrency

```
User Tap                    MainActor (ViewController)
    │                               │
    └─ Task { await interactor.fetchProfile() }
                                    │
                            actor (Interactor)
                                    │
                         async let user = worker.fetch()
                         async let posts = worker.fetch()
                                    │  ← keduanya paralel
                            await (user, posts)
                                    │
                      await presenter?.presentProfile(response) // auto-hop ke @MainActor
                                    │
                            @MainActor (Presenter)
                                    │
                     viewController?.displayProfile()
                                    │
                            @MainActor (ViewController)
                                    │
                              Update UI ✅
```

### Trade-off di Swift 5.x

| Aspek | Swift 5.x | Swift 6 |
|---|---|---|
| Crossing actor boundary tanpa await | Warning ⚠️ | Error ❌ |
| @MainActor class dengan non-isolated method | Warning ⚠️ | Error ❌ |
| Mutable capture di @Sendable closure | Warning ⚠️ | Error ❌ |
| Disiplin yang dibutuhkan | Tinggi (manual) | Dijamin compiler |

---

## 10. Migration Guide

### Strategi: Layer by Layer, Bukan Big Bang

Jangan coba migrasi seluruh project sekaligus. Migrasi per layer dimulai dari yang paling
"dalam" (Worker/Service), karena layer tersebut paling sedikit dependency-nya.

```
Layer luar (ViewController)  ←── migrasi terakhir
          ↑
Layer tengah (Interactor/Presenter) ←── setelah Worker selesai
          ↑
Layer dalam (Worker/Service)  ←── MULAI DI SINI
```

---

### Phase 1 — Identifikasi & Persiapan

**Aktifkan concurrency warning di Xcode:**

```
Build Settings → Other Swift Flags → tambahkan:
-warn-concurrency
-Xfrontend -strict-concurrency=targeted
```

**Atau di Package.swift:**

```swift
.target(
    name: "MyApp",
    swiftSettings: [
        .unsafeFlags(["-warn-concurrency",
                      "-Xfrontend", "-strict-concurrency=targeted"])
    ]
)
```

**Checklist Phase 1:**
- [ ] Audit kode: cari semua `DispatchQueue`, `DispatchGroup`, completion handler
- [ ] Catat semua class yang diakses dari multiple thread
- [ ] Identifikasi data model yang perlu `Sendable`
- [ ] Tentukan target iOS minimum (iOS 15 diperlukan untuk async/await)
- [ ] Aktifkan concurrency warning di Xcode/SPM

---

### Phase 2 — Migrasi Layer Worker/Service

Layer ini paling mudah karena biasanya stateless.

**Sebelum:**

```swift
class UserService {
    func fetchUser(id: String,
                   completion: @escaping (Result<User, Error>) -> Void) {
        URLSession.shared.dataTask(with: makeURL(id: id)) { data, _, error in
            if let error = error { completion(.failure(error)); return }
            guard let data = data else { completion(.failure(AppError.noData)); return }
            do {
                let user = try JSONDecoder().decode(User.self, from: data)
                completion(.success(user))
            } catch {
                completion(.failure(error))
            }
        }.resume()
    }
}
```

**Sesudah:**

```swift
struct UserService {  // struct, bukan class — stateless
    func fetchUser(id: String) async throws -> User {
        let (data, _) = try await URLSession.shared.data(from: makeURL(id: id))
        return try JSONDecoder().decode(User.self, from: data)
    }
}
```

**Checklist Phase 2:**
- [ ] Ganti `URLSession.dataTask` → `URLSession.data(for:)` (iOS 15+)
- [ ] Ubah signature: hapus `completion`, tambahkan `async throws`
- [ ] Tandai model dengan `: Sendable`
- [ ] Wrap callback pihak ketiga dengan `withCheckedContinuation`
- [ ] Ubah `class` menjadi `struct` jika tidak punya mutable state

---

### Phase 3 — Migrasi Layer Interactor/UseCase

**Sebelum:**

```swift
class ProfileInteractor {
    var presenter: ProfilePresentationLogic?

    func fetchProfile(request: ProfileRequest) {
        let group = DispatchGroup()
        var fetchedUser: User?
        var fetchedPosts: [Post] = []

        group.enter()
        UserService().fetchUser(id: request.userId) { result in
            fetchedUser = try? result.get()
            group.leave()
        }

        group.enter()
        PostService().fetchPosts { result in
            fetchedPosts = (try? result.get()) ?? []
            group.leave()
        }

        group.notify(queue: .main) { [weak self] in
            guard let user = fetchedUser else { return }
            let response = ProfileResponse(user: user, posts: fetchedPosts)
            self?.presenter?.presentProfile(response: response)
        }
    }
}
```

**Sesudah:**

```swift
actor ProfileInteractor {
    weak var presenter: ProfilePresentationLogic?
    private let userService = UserService()
    private let postService = PostService()

    func fetchProfile(request: ProfileRequest) async {
        do {
            async let user = userService.fetchUser(id: request.userId)
            async let posts = postService.fetchPosts(userId: request.userId)
            let response = ProfileResponse(user: try await user,
                                           posts: try await posts)
            // Swift otomatis hop ke @MainActor karena method di protocol adalah @MainActor
            await presenter?.presentProfile(response: response)
        } catch {
            await presenter?.presentError(error: error)
        }
    }
}
```

**Checklist Phase 3:**
- [ ] Ubah `class` menjadi `actor` untuk interactor yang punya state
- [ ] Ganti `DispatchGroup` dengan `async let` atau `withTaskGroup`
- [ ] Ganti `[weak self]` capture list dengan `Task` scoping
- [ ] Gunakan `await presenter?.method()` langsung — Swift auto-hop ke `@MainActor` via protocol

---

### Phase 4 — Migrasi Layer ViewController

**Sebelum:**

```swift
class ProfileViewController: UIViewController {
    func fetchProfile() {
        showLoading()
        interactor.fetchProfile(request: request) // completion-based
    }

    // Dipanggil dari interactor via completion
    func displayProfile(viewModel: ProfileViewModel) {
        DispatchQueue.main.async {  // harus manual dispatch ke main
            self.hideLoading()
            self.nameLabel.text = viewModel.displayName
        }
    }
}
```

**Sesudah:**

```swift
@MainActor  // Seluruh class di main thread
class ProfileViewController: UIViewController {
    func fetchProfile() {
        showLoading()
        Task {
            await interactor.fetchProfile(request: request)
        }
    }

    // Dipanggil dari presenter yang sudah @MainActor
    func displayProfile(viewModel: ProfileViewModel) {
        // Tidak perlu DispatchQueue.main.async — sudah di @MainActor
        hideLoading()
        nameLabel.text = viewModel.displayName
    }
}
```

**Checklist Phase 4:**
- [ ] Tambahkan `@MainActor` di class declaration ViewController
- [ ] Hapus semua `DispatchQueue.main.async {}` (tidak diperlukan lagi)
- [ ] Ganti panggilan completion-based ke `Task { await ... }`
- [ ] Simpan `Task` reference untuk cancel saat `viewDidDisappear`

---

### Phase 5 — Persiapan Swift 6 (Opsional, Per-Target)

Aktifkan Swift 6 mode per-target atau per-file untuk validasi lebih ketat:

**Per-target di Xcode:**
```
Build Settings → Swift Language Version → Swift 6
```

**Per-file (eksperimental):**
```swift
// Tambahkan di atas file
// swift-language-version: 6
```

**Checklist Phase 5:**
- [ ] Enable Swift 6 mode di satu target non-produksi dulu
- [ ] Fix semua warning yang kini menjadi error
- [ ] Ganti `@preconcurrency` yang tidak dibutuhkan lagi
- [ ] Pastikan semua `@unchecked Sendable` memang benar thread-safe
- [ ] Update dependency pihak ketiga ke versi yang sudah Swift 6 compatible

---

## 11. Tips & Best Practices

### 1. Jangan Campur GCD dan Swift Concurrency di Layer yang Sama

```swift
// BURUK — campur GCD dan Swift Concurrency
func badMix() async {
    let result = await fetch()
    DispatchQueue.main.async {  // tidak perlu! sudah di actor context
        self.label.text = result
    }
}

// BAIK — konsisten dengan Swift Concurrency
@MainActor
func goodApproach() async {
    let result = await fetch()
    label.text = result  // sudah di main thread lewat @MainActor
}
```

### 2. Selalu Simpan Task Reference Jika Perlu Cancel

```swift
@MainActor
class SearchViewController: UIViewController {
    private var activeTask: Task<Void, Never>?

    func search(query: String) {
        activeTask?.cancel()  // cancel search sebelumnya
        activeTask = Task {
            let results = try? await searchService.search(query: query)
            guard !Task.isCancelled else { return }
            displayResults(results ?? [])
        }
    }

    override func viewDidDisappear(_ animated: Bool) {
        super.viewDidDisappear(animated)
        activeTask?.cancel()  // bersihkan saat view hilang
    }
}
```

### 3. @MainActor ≠ DispatchQueue.main

```swift
// @MainActor — COMPILE TIME guarantee, lebih kuat
@MainActor func updateUI() {
    label.text = "Updated"
}

// DispatchQueue.main.async — RUNTIME dispatch, bisa nested
DispatchQueue.main.async {
    DispatchQueue.main.async {  // tidak deadlock, tapi tidak efisien
        label.text = "Updated"
    }
}

// @MainActor tidak bisa digunakan di callback non-async
URLSession.shared.dataTask(with: url) { data, _, _ in
    // Di sini bukan @MainActor context meskipun class-nya @MainActor
    // Harus gunakan Task { @MainActor in ... } atau DispatchQueue.main.async
    Task { @MainActor in
        self.label.text = "Done"
    }
}
```

### 4. Hindari @unchecked Sendable Berlebihan

```swift
// JANGAN — @unchecked Sendable sebagai jalan pintas
final class DataHolder: @unchecked Sendable {
    var items: [String] = []  // ← Tidak ada proteksi! Tetap data race!
}

// LEBIH BAIK — gunakan actor
actor DataHolder {
    var items: [String] = []
}

// ATAU — buat immutable
struct DataHolder: Sendable {
    let items: [String]  // let, bukan var
}
```

### 5. nonisolated untuk Pure Functions

```swift
actor UserManager {
    var users: [User] = []

    // Harus await dari luar — mengakses state actor
    func addUser(_ user: User) {
        users.append(user)
    }

    // nonisolated — tidak akses state, tidak perlu await
    nonisolated func validateEmail(_ email: String) -> Bool {
        email.contains("@") && email.contains(".")
    }

    // nonisolated computed property — bebas dari isolation
    nonisolated var description: String {
        "UserManager"
    }
}

let manager = UserManager()
// Tidak perlu await!
let isValid = manager.validateEmail("user@example.com")
```

### 6. Aktifkan Warning Sebelum Swift 6 Mode

Lakukan secara bertahap:
```
-warn-concurrency                          → warning ringan
-Xfrontend -strict-concurrency=targeted   → warning menengah
-Xfrontend -strict-concurrency=complete   → hampir setara Swift 6
Swift 6 Language Mode                      → semua jadi error
```

### 7. Actor Tidak Menjamin Urutan Eksekusi

```swift
actor Counter {
    var value = 0

    func increment() {
        value += 1
    }
}

let counter = Counter()

// PERHATIAN: Tidak ada jaminan urutan eksekusi di antara Task berbeda
Task { await counter.increment() }
Task { await counter.increment() }
Task { await counter.increment() }
// Hasil akhir selalu 3, tapi urutan eksekusinya tidak terdefinisi
```

### 8. Prefer AsyncStream daripada Combine untuk Kode Baru

```swift
// Combine — masih valid, tapi lebih kompleks untuk use case sederhana
let publisher = PassthroughSubject<Event, Never>()
var cancellables = Set<AnyCancellable>()

publisher
    .sink { event in handleEvent(event) }
    .store(in: &cancellables)

// AsyncStream — lebih simpel, terintegrasi dengan Swift Concurrency
let (stream, continuation) = AsyncStream.makeStream(of: Event.self)

Task {
    for await event in stream {
        handleEvent(event)
    }
}
```

### 9. Jangan Block Main Thread

```swift
// SALAH — blocking main thread!
func loadData() {
    let result = Task { await service.fetch() }.value  // ← DEADLOCK jika di @MainActor
}

// BENAR — propagate async ke atas
func loadData() async {
    let result = await service.fetch()
    displayResult(result)
}

// ATAU dari non-async context
func viewDidLoad() {
    Task {
        await loadData()
    }
}
```

### 10. Structured Concurrency untuk Scope Terbatas

```swift
// Gunakan withTaskGroup untuk operasi yang scope-nya jelas
func processImages(_ urls: [URL]) async throws -> [UIImage] {
    try await withThrowingTaskGroup(of: UIImage.self) { group in
        for url in urls {
            group.addTask { try await downloadImage(from: url) }
        }

        var images: [UIImage] = []
        for try await image in group {
            images.append(image)
        }
        return images
    }
    // Semua task selesai saat keluar dari scope ini
}

// Hindari Task.detached untuk operasi yang hasilnya masih dibutuhkan caller
```

---

## 12. Ringkasan & Quick Reference

### API Quick Reference

| API | Kegunaan | Min. Swift | Min. iOS |
|---|---|---|---|
| `async`/`await` | Fungsi asinkron linear | 5.5 | 15 |
| `Task {}` | Jalankan async dari sync context | 5.5 | 15 |
| `Task.detached {}` | Async tanpa mewarisi isolation | 5.5 | 15 |
| `async let` | Concurrent binding, paralel | 5.5 | 15 |
| `withTaskGroup` | Concurrency dinamis | 5.5 | 15 |
| `actor` | Isolation state custom | 5.5 | 15 |
| `@MainActor` | Isolation ke main thread | 5.5 | 15 |
| `nonisolated` | Bebas dari actor isolation | 5.5 | 15 |
| `Sendable` | Marker keamanan lintas actor | 5.5 | 15 |
| `@unchecked Sendable` | Opt-out safety check | 5.5 | 15 |
| `withCheckedContinuation` | Bridge callback → async (safe) | 5.5 | 15 |
| `AsyncStream` | Custom async sequence | 5.5 | 15 |
| `@preconcurrency` | Suppress warning library lama | 5.6 | 15 |
| `@Observable` | Observation framework | 5.9 | 17 |

---

### Decision Guide: Pilih yang Mana?

**Butuh menjalankan async dari UIKit action / lifecycle?**
→ `Task { await ... }` (mewarisi @MainActor dari ViewController)

**Butuh beberapa async operasi berjalan paralel?**
→ `async let` (jika jumlah fix) atau `withTaskGroup` (jika dinamis)

**Butuh melindungi state dari multiple thread?**
→ `actor` (state yang perlu sequential access)
→ `@MainActor class` (state yang harus di main thread)

**Butuh bridge dari callback/delegate ke async?**
→ `withCheckedContinuation` (single callback)
→ `AsyncStream` (callback berulang / delegate)

**Butuh data model yang aman dikirim lintas actor?**
→ `struct: Sendable` (preferred, immutable)
→ `@unchecked Sendable` (class dengan manual locking, as last resort)

**Library pihak ketiga menghasilkan Sendable warning?**
→ `@preconcurrency import LibraryName`

---

### Checklist Migrasi Bertahap

**Persiapan:**
- [ ] Minimal iOS 15 (atau gunakan `@available(iOS 15, *)`)
- [ ] Aktifkan `-warn-concurrency` di build flags
- [ ] Identifikasi semua GCD, DispatchGroup, completion handler

**Layer Worker/Service:**
- [ ] Ganti callback menjadi `async throws`
- [ ] Model data tambahkan `: Sendable`
- [ ] Wrap 3rd party callback dengan `withCheckedContinuation`

**Layer Interactor/UseCase:**
- [ ] Ubah ke `actor` (jika ada state) atau keep sebagai `struct` (jika stateless)
- [ ] Ganti `DispatchGroup` dengan `async let` / `withTaskGroup`
- [ ] Gunakan `await presenter?.method()` langsung — Swift auto-hop ke `@MainActor` via protocol

**Layer ViewController:**
- [ ] Tambahkan `@MainActor` di class declaration
- [ ] Hapus `DispatchQueue.main.async {}`
- [ ] Ganti completion handler dengan `Task { await ... }`
- [ ] Simpan `Task` reference untuk cancellation

**Menuju Swift 6:**
- [ ] Upgrade `-strict-concurrency=complete`
- [ ] Fix semua warning yang tersisa
- [ ] Enable Swift 6 mode per-target
- [ ] Hapus `@preconcurrency` yang sudah tidak diperlukan

---

*Dibuat: 2026-05-12 | Swift 5.5–5.10 | iOS 15+ | Panduan Adopsi Bertahap Swift Concurrency*

---

## 13. Summary

### Gambaran Besar

Swift Concurrency di Swift 5.x bukan fitur tunggal — melainkan sekumpulan tool yang bekerja bersama untuk menggantikan pola lama (GCD, callback, delegate) dengan model yang lebih aman dan mudah dibaca. Kuncinya adalah memahami **siapa yang memiliki data** dan **di thread mana sesuatu boleh berjalan**.

```
Masalah Lama                    Solusi Swift Concurrency
─────────────────────────────────────────────────────────
Callback hell                 → async/await
DispatchGroup                 → async let / withTaskGroup
DispatchQueue.main.async      → @MainActor
Manual lock (NSLock, dll.)    → actor
Delegate berulang             → AsyncStream
Callback satu kali            → withCheckedContinuation
Data race pada class          → Sendable / actor
Library lama → warning        → @preconcurrency
```

---

### Hierarki Isolation di VIP + UIKit

```
┌─────────────────────────────────────────────────────┐
│  @MainActor                                         │
│  ┌─────────────────┐   ┌──────────────────────────┐ │
│  │ViewController   │   │ Presenter                │ │
│  │ - trigger Task  │   │ - format → ViewModel     │ │
│  │ - update UI     │   │ - panggil ViewController │ │
│  └────────┬────────┘   └──────────────────────────┘ │
└───────────┼─────────────────────────────────────────┘
            │ await interactor.fetch()
┌───────────▼─────────────────────────────────────────┐
│  actor (ProfileInteractor)                          │
│  - orchestrate business logic                       │
│  - await worker (async)                             │
│  - await presenter (auto-hop ke @MainActor)         │
└───────────┬─────────────────────────────────────────┘
            │ try await worker.fetch()
┌───────────▼─────────────────────────────────────────┐
│  async func (ProfileWorker)                         │
│  - stateless, pure async                            │
│  - network / cache / database                       │
└─────────────────────────────────────────────────────┘
            │
┌───────────▼─────────────────────────────────────────┐
│  Sendable Models                                    │
│  struct Request / Response / ViewModel              │
│  - value type, aman lintas batas actor              │
└─────────────────────────────────────────────────────┘
```

---

### Aturan Sederhana Per Layer

| Layer | Tipe | Isolation | Alasan |
|---|---|---|---|
| ViewController | `class` | `@MainActor` | UIKit hanya boleh disentuh di main thread |
| Presenter | `class` | `@MainActor` | Memanggil ViewController yang @MainActor |
| Router | `class` | `@MainActor` | Navigasi UIKit butuh main thread |
| Interactor | `actor` | actor isolation | Menjaga state business logic dari data race |
| Worker | `struct` | none (async) | Stateless, tidak perlu isolation |
| Models | `struct` | `Sendable` | Dikirim lintas actor, harus value type |

---

### Konsekuensi Tidak Conform Sendable

| Tipe | Tanpa Sendable di Swift 5.x | Tanpa Sendable di Swift 6 |
|---|---|---|
| `struct` | ⚠️ Warning, tetap jalan | ❌ Compile error |
| `class` | ⚠️ Warning + **risiko data race di runtime** | ❌ Compile error |
| `enum` | ⚠️ Warning, tetap jalan | ❌ Compile error |

> **Prinsip:** Model yang melewati batas actor **harus** `Sendable`. Gunakan `struct` + `let`
> sebagai default — otomatis `Sendable` tanpa deklarasi eksplisit.

---

### Pola Pemanggilan yang Benar

```swift
// ✅ ViewController (@MainActor) → Interactor (actor)
Task {
    await interactor.fetchProfile(request: request)
}

// ✅ Interactor (actor) → Worker (async func)
let user = try await worker.fetchUser(id: id)

// ✅ Interactor (actor) → Presenter (@MainActor)
// Swift otomatis hop ke MainActor karena protocol adalah @MainActor
await presenter?.presentProfile(response: response)

// ✅ Protocol dengan @MainActor di level protocol — bukan per method
@MainActor
protocol ProfilePresentationLogic: AnyObject {
    func presentProfile(response: ProfileResponse)
    func presentError(error: Error)
}

// ✅ Model yang dikirim lintas actor
struct ProfileResponse: Sendable {
    let user: User      // User: Sendable
    let posts: [Post]   // Post: Sendable
}
```

---

### Pola yang Harus Dihindari

```swift
// ❌ Campur GCD dan Swift Concurrency di layer yang sama
func load() async {
    let data = await fetch()
    DispatchQueue.main.async { self.label.text = data }  // tidak perlu, sudah @MainActor
}

// ❌ Task.detached tanpa alasan jelas
Task.detached {
    await self.update()  // kehilangan isolation context
}

// ❌ @unchecked Sendable sebagai jalan pintas
final class MyModel: @unchecked Sendable {
    var items: [String] = []  // tidak ada proteksi → data race
}

// ❌ @MainActor per-method di protocol — redundan
protocol MyProtocol: AnyObject {
    @MainActor func methodA()  // cukup taruh @MainActor di protocol-nya
    @MainActor func methodB()
}

// ❌ Block main thread
let result = Task { await service.fetch() }.value  // deadlock jika dipanggil dari @MainActor
```

---

### Jalur Migrasi Bertahap

```
Swift 5.x (sekarang)
        │
        ▼
1. Aktifkan -warn-concurrency
        │
        ▼
2. Migrasi Worker/Service → async throws + Sendable model
        │
        ▼
3. Migrasi Interactor → actor
        │
        ▼
4. Migrasi ViewController/Presenter → @MainActor
        │
        ▼
5. -strict-concurrency=complete (semua warning tampil)
        │
        ▼
6. Swift 6 Mode (warning → error, compiler menjamin segalanya)
```

Tidak ada kewajiban sampai ke Step 6 — project yang berhenti di Step 4 sudah jauh lebih aman
dari kode GCD tradisional.
