# Swift Concurrency — Panduan Lengkap dari 5.5 sampai 6.2
### Teori Mendalam, Real Use Case, Kapan Digunakan, Kapan Tidak

> **Target pembaca:** iOS/macOS developer yang ingin memahami keseluruhan evolusi Swift Concurrency secara mendalam — bukan sekadar sintaks, tapi filosofi desain dan trade-off di balik setiap fitur.

---

## Daftar Isi

**Swift 5.5 — Fondasi**
1. [async/await](#1-asyncawait)
2. [Structured Concurrency — async let & TaskGroup](#2-structured-concurrency--async-let--taskgroup)
3. [Actors](#3-actors)
4. [MainActor](#4-mainactor)
5. [AsyncSequence & AsyncStream](#5-asyncsequence--asyncstream)

**Swift 6.0 — Strict Concurrency**
6. [Complete Concurrency Checking](#6-complete-concurrency-checking)
7. [Sendable Enforcement](#7-sendable-enforcement)
8. [sending Modifier](#8-sending-modifier)
9. [nonisolated(unsafe)](#9-nonisoulatedunsafe)
10. [@preconcurrency](#10-preconcurrency)
11. [Global Actor Isolation — Default Lebih Ketat](#11-global-actor-isolation--default-lebih-ketat)

**Swift 6.1 — Refinements**
12. [nonisolated pada Stored Property](#12-nonisolated-pada-stored-property)
13. [Improved Actor Isolation Inference](#13-improved-actor-isolation-inference)
14. [@isolated(any)](#14-isolatedany)

**Swift 6.2 — Ergonomics**
15. [Default Actor Isolation per Module](#15-default-actor-isolation-per-module)
16. [Task Naming](#16-task-naming)
17. [Improved async let Cancellation Semantics](#17-improved-async-let-cancellation-semantics)
18. [nonisolated on init](#18-nonisolated-on-init)

**Penutup**
19. [Evolusi Timeline & Keputusan Desain](#19-evolusi-timeline--keputusan-desain)
20. [Decision Guide — Memilih Tool yang Tepat](#20-decision-guide--memilih-tool-yang-tepat)

---

# SWIFT 5.5 — FONDASI (2021)

---

## 1. async/await

### Teori: Masalah Callback Hell dan Cooperative Multitasking

Sebelum `async/await`, concurrency di Swift (dan platform Apple) dibangun di atas **completion handlers** — closure yang dipanggil ketika operasi async selesai. Ini menciptakan beberapa masalah fundamental:

**1. Callback Hell — Nesting yang Tidak Terkendali**
```swift
// Completion handler: setiap level async = satu level nesting
func loadUserProfile(userID: String) {
    fetchUser(userID: userID) { user in
        guard let user = user else { return }
        fetchAvatar(url: user.avatarURL) { image in
            guard let image = image else { return }
            fetchPosts(userID: user.id) { posts in
                guard let posts = posts else { return }
                // Logika sesungguhnya berada di kedalaman 3 closure
                // Error handling di setiap level
                // Sulit dibaca, sulit di-maintain
                DispatchQueue.main.async {
                    self.display(user: user, image: image, posts: posts)
                }
            }
        }
    }
}
```

**2. Error Handling yang Tersebar**
```swift
// Setiap callback harus handle error sendiri — tidak ada propagation
fetchUser(id: id) { result in
    switch result {
    case .failure(let error):
        // handle error 1
        return
    case .success(let user):
        fetchAvatar(url: user.avatarURL) { result in
            switch result {
            case .failure(let error):
                // handle error 2 — duplikasi
                return
            case .success(let image):
                // ... terus
            }
        }
    }
}
```

**3. Kehilangan Thread Context**
```swift
// Callback bisa dipanggil di thread mana saja
// Tidak ada jaminan thread-safety
fetchData { result in
    // Thread apa ini? Main? Background? URLSession queue?
    self.updateUI(result)  // UNSAFE jika bukan main thread
}
```

### Mengapa async/await Lebih Baik

`async/await` bukan hanya syntactic sugar. Di baliknya, Swift menggunakan **cooperative multitasking** — model yang berbeda fundamental dari thread-based concurrency:

```
Thread-based (GCD):          Cooperative (async/await):
─────────────────────        ─────────────────────────────
Thread 1: Task A runs        Thread 1: Task A runs
Thread 1: Task A blocks      Task A hits 'await' → SUSPENDS
  (thread idle, wasted)      Thread 1: picks up Task B
Thread 2: needed for         Task A resumes when ready
  continuation               Thread 1: Task A continues
  (context switch overhead)  (tidak ada wasted threads)
```

Ketika kamu menulis `await`, fungsi di-**suspend** — bukan di-block. Thread yang digunakan dibebaskan untuk mengerjakan task lain. Ini memungkinkan ribuan concurrent operations dengan hanya beberapa thread (sesuai jumlah CPU core).

### Sintaks dan Cara Kerja

```swift
// Setelah: linear, mudah dibaca
func loadUserProfile(userID: String) async throws -> UserProfile {
    let user = try await fetchUser(userID: userID)          // suspend 1
    let image = try await fetchAvatar(url: user.avatarURL)  // suspend 2
    let posts = try await fetchPosts(userID: user.id)       // suspend 3
    return UserProfile(user: user, image: image, posts: posts)
}

// Error propagates dengan throws — tidak perlu handle di setiap level
// Thread safety dijamin oleh Swift runtime (kembali ke caller's executor)
```

**Suspension Point — Apa yang Terjadi di Balik `await`:**
```
1. Kode sebelum await berjalan
2. await: fungsi di-suspend, eksekusi dikembalikan ke Swift runtime
3. Runtime: pilih task lain yang siap berjalan
4. Ketika I/O selesai: runtime jadwalkan ulang suspended function
5. Kode setelah await berjalan (mungkin di thread berbeda, tapi isolation terjaga)
```

### Memanggil async dari Synchronous Context

```swift
// async function tidak bisa dipanggil langsung dari sync — harus dalam Task
class ViewController: UIViewController {
    override func viewDidLoad() {
        super.viewDidLoad()
        
        // Task {} adalah "jembatan" dari sync ke async
        Task {
            do {
                let profile = try await loadUserProfile(userID: "123")
                await MainActor.run { updateUI(with: profile) }
            } catch {
                await MainActor.run { showError(error) }
            }
        }
    }
}
```

### Kapan Menggunakan async/await

**Gunakan ketika:**
- Operasi yang melibatkan I/O: network, disk, database
- Operasi yang membutuhkan waktu: compute-intensive background tasks
- Rangkaian operasi yang saling bergantung (sequential async)
- Mengganti completion handler yang sudah ada

**Jangan gunakan ketika:**
- Operasi synchronous yang cepat (< 1ms) — overhead task tidak sebanding
- Hot path / tight loop numerik — gunakan synchronous code biasa
- Kode yang harus berjalan di context tanpa Swift runtime (misalnya C interop layer)

### Real Use Case: Network Request Chain

```swift
// Layanan authentication dengan token refresh otomatis
actor AuthenticatedAPIClient {
    private var accessToken: String?
    private var refreshToken: String?
    private let baseURL: URL

    init(baseURL: URL) { self.baseURL = baseURL }

    func authenticatedRequest<T: Decodable>(
        endpoint: String,
        type: T.Type
    ) async throws -> T {
        // 1. Pastikan punya token valid
        let token = try await ensureValidToken()

        // 2. Buat request
        var request = URLRequest(url: baseURL.appendingPathComponent(endpoint))
        request.setValue("Bearer \(token)", forHTTPHeaderField: "Authorization")

        // 3. Kirim request
        let (data, response) = try await URLSession.shared.data(for: request)

        // 4. Handle 401 — refresh token dan coba lagi
        if let http = response as? HTTPURLResponse, http.statusCode == 401 {
            try await refreshAccessToken()
            return try await authenticatedRequest(endpoint: endpoint, type: type)
        }

        return try JSONDecoder().decode(T.self, from: data)
    }

    private func ensureValidToken() async throws -> String {
        if let token = accessToken { return token }
        try await refreshAccessToken()
        return accessToken!
    }

    private func refreshAccessToken() async throws {
        guard let refresh = refreshToken else { throw AuthError.notLoggedIn }
        var request = URLRequest(url: baseURL.appendingPathComponent("/auth/refresh"))
        request.httpMethod = "POST"
        request.httpBody = try JSONEncoder().encode(["refresh_token": refresh])
        let (data, _) = try await URLSession.shared.data(for: request)
        let tokens = try JSONDecoder().decode(TokenResponse.self, from: data)
        accessToken = tokens.accessToken
        refreshToken = tokens.refreshToken
    }
}

struct TokenResponse: Decodable { let accessToken, refreshToken: String }
enum AuthError: Error { case notLoggedIn }
```

---

## 2. Structured Concurrency — async let & TaskGroup

### Teori: Mengapa "Structured"?

**Unstructured concurrency** (GCD, Thread) memiliki problem: task yang dibuat bisa hidup lebih lama dari parent-nya, menyebabkan:
- Resource leak (task berjalan meski tidak ada yang butuh hasilnya)
- Sulit di-cancel
- Error tidak bisa propagate ke caller

**Structured concurrency** meminjam konsep dari structured programming (if/for/while) — setiap task punya **lifetime yang dibatasi oleh scope-nya**:

```
Tanpa struktur:                Dengan struktur:
──────────────                 ──────────────────
Parent selesai                 Parent tidak selesai
    │                          sampai SEMUA child selesai
    ↓                              │
Task anak masih berjalan!          ├── Task A
(memory leak, use-after-free)      ├── Task B
                                   └── Task C
                                   ↓
                                   Parent selesai
                                   (semua child sudah selesai)
```

Properti kunci Structured Concurrency:
1. **Lifetime guarantee:** Child task tidak bisa hidup lebih lama dari parent
2. **Automatic cancellation:** Jika parent di-cancel, semua child ikut di-cancel
3. **Error propagation:** Error dari child bisa propagate ke parent
4. **Resource safety:** Tidak ada orphaned tasks

### async let — Concurrent Bindings

`async let` memulai task secara concurrent tapi **di dalam scope yang sama**:

```swift
// Tanpa async let: sequential — total waktu = A + B + C
func loadDashboard() async throws -> Dashboard {
    let user = try await fetchUser()      // tunggu selesai
    let news = try await fetchNews()      // baru mulai
    let weather = try await fetchWeather() // baru mulai
    return Dashboard(user: user, news: news, weather: weather)
    // Total: ~3 detik jika masing-masing 1 detik
}

// Dengan async let: concurrent — total waktu = max(A, B, C)
func loadDashboard() async throws -> Dashboard {
    async let user = fetchUser()          // mulai concurrent
    async let news = fetchNews()          // mulai concurrent
    async let weather = fetchWeather()    // mulai concurrent
    
    // Semua mulai bersamaan, tunggu semua selesai di sini
    return try await Dashboard(
        user: user,
        news: news,
        weather: weather
    )
    // Total: ~1 detik (berjalan paralel)
}
```

**Cancellation otomatis dengan async let:**
```swift
func loadWithTimeout() async throws -> Data {
    async let result = fetchLargeFile()         // mulai
    async let _ = Task.sleep(nanoseconds: 5_000_000_000)  // 5 detik timeout
    
    // Jika fungsi ini di-cancel dari luar:
    // Kedua task di atas otomatis di-cancel
    return try await result
}
```

### TaskGroup — Concurrency Dinamis

Ketika jumlah task tidak diketahui saat compile time:

```swift
// async let: jumlah task FIXED saat compile time
// TaskGroup: jumlah task DINAMIS saat runtime

func downloadAllImages(urls: [URL]) async throws -> [UIImage] {
    try await withThrowingTaskGroup(of: (Int, UIImage).self) { group in
        // Tambah task untuk setiap URL
        for (index, url) in urls.enumerated() {
            group.addTask {
                let (data, _) = try await URLSession.shared.data(from: url)
                guard let image = UIImage(data: data) else {
                    throw ImageError.invalidData
                }
                return (index, image)
            }
        }

        // Kumpulkan hasil — urutan tidak dijamin (selesai duluan = datang duluan)
        var results = [(Int, UIImage)]()
        for try await result in group {
            results.append(result)
        }

        // Sort berdasarkan index untuk menjaga urutan asli
        return results.sorted { $0.0 < $1.0 }.map { $0.1 }
    }
}

enum ImageError: Error { case invalidData }
```

### Membatasi Concurrency

Tanpa batasan, TaskGroup bisa membuat terlalu banyak task concurrent — membebani server atau memori:

```swift
func downloadWithConcurrencyLimit(
    urls: [URL],
    maxConcurrent: Int = 5
) async throws -> [Data] {
    var results = [Data?](repeating: nil, count: urls.count)

    try await withThrowingTaskGroup(of: (Int, Data).self) { group in
        var activeCount = 0
        var nextIndex = 0

        // Seed awal: isi hingga batas concurrency
        while nextIndex < urls.count && activeCount < maxConcurrent {
            let index = nextIndex
            group.addTask {
                let (data, _) = try await URLSession.shared.data(from: urls[index])
                return (index, data)
            }
            activeCount += 1
            nextIndex += 1
        }

        // Proses: setiap kali satu selesai, tambah satu baru
        for try await (index, data) in group {
            results[index] = data
            activeCount -= 1

            if nextIndex < urls.count {
                let idx = nextIndex
                group.addTask {
                    let (data, _) = try await URLSession.shared.data(from: urls[idx])
                    return (idx, data)
                }
                activeCount += 1
                nextIndex += 1
            }
        }
    }

    return results.compactMap { $0 }
}
```

### Kapan Menggunakan

**async let — gunakan ketika:**
- Jumlah operasi concurrent sudah diketahui saat compile time
- Operasi saling independen (tidak butuh hasil satu untuk memulai lain)
- 2–10 operasi concurrent (lebih dari itu, pertimbangkan TaskGroup)

**TaskGroup — gunakan ketika:**
- Jumlah operasi bergantung pada data runtime (array, collection)
- Perlu limit concurrency
- Ingin collect results secara incremental
- Error dari satu task tidak boleh cancel semua (gunakan `withTaskGroup` tanpa throws)

**Jangan gunakan ketika:**
- Operasi sequential (gunakan biasa, tidak perlu group)
- Hasil masing-masing task tidak diperlukan (gunakan `Task {}` biasa)
- Kamu hanya ingin "fire and forget" — gunakan unstructured `Task {}`

### Real Use Case: Batch API Processing

```swift
actor BatchProcessor {
    private let apiClient: AuthenticatedAPIClient

    init(client: AuthenticatedAPIClient) { self.apiClient = client }

    // Proses banyak item dengan concurrency terkontrol
    func processItems<T: Sendable & Decodable>(
        ids: [String],
        endpoint: String,
        maxConcurrent: Int = 10
    ) async throws -> [String: T] {
        var results = [String: T]()
        var errors = [String: Error]()

        await withTaskGroup(of: (String, Result<T, Error>).self) { group in
            var running = 0
            var iterator = ids.makeIterator()

            // Seed
            while running < maxConcurrent, let id = iterator.next() {
                let capturedID = id
                group.addTask {
                    do {
                        let value: T = try await self.apiClient.authenticatedRequest(
                            endpoint: "\(endpoint)/\(capturedID)",
                            type: T.self
                        )
                        return (capturedID, .success(value))
                    } catch {
                        return (capturedID, .failure(error))
                    }
                }
                running += 1
            }

            for await (id, result) in group {
                running -= 1
                switch result {
                case .success(let value): results[id] = value
                case .failure(let error): errors[id] = error
                }

                if let nextID = iterator.next() {
                    let capturedID = nextID
                    group.addTask {
                        do {
                            let value: T = try await self.apiClient.authenticatedRequest(
                                endpoint: "\(endpoint)/\(capturedID)",
                                type: T.self
                            )
                            return (capturedID, .success(value))
                        } catch {
                            return (capturedID, .failure(error))
                        }
                    }
                    running += 1
                }
            }
        }

        if !errors.isEmpty {
            print("⚠️ \(errors.count) item gagal diproses")
        }
        return results
    }
}
```

---

## 3. Actors

### Teori: Model Actor dan Serial Execution

Actor didasarkan pada **Actor Model** yang dikembangkan Carl Hewitt (1973) — model komputasi di mana "actors" adalah unit dasar yang:
1. Memiliki state privat yang tidak bisa diakses dari luar
2. Berkomunikasi hanya melalui message passing
3. Memproses satu message pada satu waktu (serial)

Swift actors mengimplementasikan ini dengan **executor** — sebuah serializer yang memastikan hanya satu piece of code yang mengakses actor state pada satu waktu. Ini berbeda dari mutex/lock:

```
Lock/Mutex:                      Actor:
──────────────────               ────────────────────
Thread A: lock()                 Task A: await actor.method()
Thread A: critical section       Task A: suspended (tapi TIDAK block thread)
Thread B: lock() → BLOCKED       Thread: free, bisa kerjakan Task B
  (thread terbuang menunggu)     Task A: resumed saat actor tersedia
Thread A: unlock()               Task A: runs in actor's executor
Thread B: runs                   (tidak ada thread yang terbuang)
```

Keunggulan actor vs lock:
- Tidak ada thread blocking → lebih efisien
- Tidak ada deadlock dari nested locks
- Compiler enforcement → tidak bisa "lupa" lock
- Lebih composable

### Cara Kerja Internal

```swift
actor Counter {
    var value = 0
    func increment() { value += 1 }
}
```

Di balik layar, compiler men-generate sesuatu setara dengan:
```swift
// Konseptual — bukan kode aktual Swift
class Counter {
    private var value = 0
    private let executor = SerialExecutor()  // antrian serial

    func increment() {
        executor.enqueue {
            self.value += 1  // hanya berjalan dalam executor
        }
    }
}
```

### Reentrancy — Desain yang Disengaja

Actor bersifat **reentrant**: ketika actor sedang menunggu `await` di dalam method-nya, actor bisa menerima dan memproses request lain. Ini bukan bug — ini desain untuk mencegah deadlock:

```
Tanpa reentrancy (deadlock risk):
Actor A menunggu Actor B
Actor B menunggu Actor A
→ DEADLOCK: keduanya tunggu selamanya

Dengan reentrancy:
Actor A menunggu Actor B (via await)
Actor A: "saya suspend, silakan proses request lain"
Actor B: selesai, kirim hasil ke Actor A
Actor A: resume → selesai
→ tidak ada deadlock
```

**Implikasi praktis reentrancy:**
```swift
actor Inventory {
    var stock = 10

    func purchaseItem() async -> Bool {
        guard stock > 0 else { return false }  // check: stock = 10 ✓

        await processPayment()  // SUSPEND — actor bisa terima request lain!
        // Saat resume: stock mungkin sudah berubah!

        stock -= 1  // mungkin stock sudah 0 dari request lain → negative!
        return true
    }
}

// FIX: modifikasi state SEBELUM suspension point
actor Inventory {
    var stock = 10

    func purchaseItem() async -> Bool {
        guard stock > 0 else { return false }
        stock -= 1  // kurangi SEBELUM await → aman dari reentrancy

        await processPayment()  // suspend boleh setelah state di-update
        return true
    }
}
```

### Kapan Menggunakan Actor

**Gunakan actor ketika:**
- Ada shared mutable state yang diakses dari multiple concurrent contexts
- State terdiri dari beberapa field yang harus konsisten satu sama lain
- Operasi bersifat stateful (counter, cache, queue)
- Perlu koordinasi antar-tasks

**Jangan gunakan actor ketika:**
- State adalah value type (struct/enum) — tidak perlu protection
- Semua akses sudah di satu thread (gunakan @MainActor saja)
- Butuh inheritance — actor tidak bisa di-subclass
- Operasi sangat simple dan lightweight — overhead actor hop mungkin signifikan

### Real Use Case: Session Manager

```swift
// Session management yang thread-safe tanpa lock manual
actor SessionManager {
    private var sessions: [String: UserSession] = [:]
    private var cleanupTask: Task<Void, Never>?
    private let sessionTimeout: Duration

    init(sessionTimeout: Duration = .seconds(3600)) {
        self.sessionTimeout = sessionTimeout
    }

    func createSession(for userID: String) -> UserSession {
        let session = UserSession(userID: userID, expiresAt: .now + sessionTimeout)
        sessions[session.id] = session
        startCleanupIfNeeded()
        return session
    }

    func validate(sessionID: String) -> UserSession? {
        guard let session = sessions[sessionID] else { return nil }
        if session.expiresAt < .now {
            sessions.removeValue(forKey: sessionID)
            return nil
        }
        return session
    }

    func invalidate(sessionID: String) {
        sessions.removeValue(forKey: sessionID)
    }

    func invalidateAll(for userID: String) {
        sessions = sessions.filter { $0.value.userID != userID }
    }

    private func startCleanupIfNeeded() {
        guard cleanupTask == nil else { return }
        cleanupTask = Task { [weak self] in
            while !Task.isCancelled {
                try? await Task.sleep(for: .seconds(300))
                await self?.cleanExpiredSessions()
            }
        }
    }

    private func cleanExpiredSessions() {
        let now = ContinuousClock.now
        sessions = sessions.filter { $0.value.expiresAt >= now }
    }
}

struct UserSession: Sendable {
    let id: String = UUID().uuidString
    let userID: String
    var expiresAt: ContinuousClock.Instant
}
```

---

## 4. @MainActor

### Teori: Global Actor dan Main Thread Contract

`@MainActor` adalah **global actor** — sebuah singleton actor yang merepresentasikan main thread. Ini bukan sekadar "jalankan di main thread" — ini adalah **isolation domain** yang di-enforce compiler.

**Mengapa main thread istimewa?**

UIKit dan AppKit tidak thread-safe. Mereka berkomunikasi dengan **window server** (daemon OS yang mengatur tampilan visual) melalui main run loop. Jika kamu update UI dari background thread, dua hal bisa terjadi:
1. Corrupted UI state (karena concurrent writes ke UIKit internal state)
2. Crash dengan "UI API called from background thread" di debug

`@MainActor` mengkodekan invariant ini ke dalam type system: *"kode ini dijamin berjalan di main thread, compiler memverifikasinya."*

### Cara Kerja

```swift
// @MainActor class: semua property dan method diisolasi ke main thread
@MainActor
class UserListViewModel: ObservableObject {
    @Published var users: [User] = []      // dijamin di main thread
    @Published var isLoading = false       // dijamin di main thread

    func loadUsers() async {
        isLoading = true  // di main thread ✓

        // await: hop ke background untuk fetch
        let fetched = await userRepository.fetchAll()

        // Kembali ke main thread setelah await
        users = fetched        // di main thread ✓
        isLoading = false      // di main thread ✓
    }
}
```

### Memanggil @MainActor dari Background

```swift
actor DataProcessor {
    weak var delegate: (any DataProcessorDelegate)?

    func process(data: Data) async {
        let result = await heavyComputation(data)

        // Crossing boundary: actor → @MainActor
        // Butuh await karena berpindah isolation domain
        await delegate?.processingCompleted(result: result)
    }
}

@MainActor
protocol DataProcessorDelegate: AnyObject {
    func processingCompleted(result: ProcessingResult)
}
```

### MainActor.run — One-off Hop

```swift
// Untuk kode yang sebagian besar di background tapi sesekali butuh main thread
actor BackgroundWorker {
    func performWork() async {
        let data = await fetchFromNetwork()
        let processed = processLocally(data)

        // One-off update ke UI tanpa harus mark seluruh fungsi @MainActor
        await MainActor.run {
            NotificationCenter.default.post(
                name: .dataProcessed,
                object: processed
            )
        }
    }
}
```

### Kapan Menggunakan @MainActor

**Gunakan @MainActor ketika:**
- ViewController, View, UIView subclass
- ViewModel yang mengelola @Published properties
- Delegate methods yang update UI
- Timer atau notification handler yang update UI
- AppDelegate, SceneDelegate

**Jangan gunakan @MainActor ketika:**
- Business logic yang tidak menyentuh UI — ini akan menyebabkan UI stutter jika logic berat
- Network layer, database layer, heavy computation
- Fungsi yang sering dipanggil — setiap call membutuhkan hop ke main thread

### Real Use Case: ViewModel dengan Loading State

```swift
@MainActor
final class ArticleListViewModel: ObservableObject {
    enum ViewState {
        case idle
        case loading
        case loaded([Article])
        case error(String)
    }

    @Published private(set) var state: ViewState = .idle
    @Published private(set) var searchQuery = ""

    private let repository: ArticleRepository
    private var searchTask: Task<Void, Never>?

    init(repository: ArticleRepository) {
        self.repository = repository
    }

    func load() {
        guard case .idle = state else { return }
        fetchArticles()
    }

    func search(query: String) {
        searchQuery = query
        searchTask?.cancel()

        guard !query.isEmpty else {
            fetchArticles()
            return
        }

        // Debounce: tunggu 300ms setelah user berhenti ketik
        searchTask = Task {
            try? await Task.sleep(for: .milliseconds(300))
            guard !Task.isCancelled else { return }
            await fetchArticles(query: query)
        }
    }

    private func fetchArticles(query: String? = nil) {
        state = .loading

        Task {
            do {
                let articles: [Article]
                if let q = query {
                    articles = try await repository.search(query: q)
                } else {
                    articles = try await repository.fetchAll()
                }
                // Kembali ke @MainActor setelah await — otomatis
                state = .loaded(articles)
            } catch is CancellationError {
                state = .idle
            } catch {
                state = .error(error.localizedDescription)
            }
        }
    }
}

struct Article: Sendable, Identifiable {
    let id: String
    let title: String
    let content: String
}
```

---

## 5. AsyncSequence & AsyncStream

### Teori: Push vs Pull dan Backpressure

Ada dua model fundamental untuk streaming data:

**Push-based (Combine, NotificationCenter, delegates):**
```
Producer ──push──▶ Consumer
"Data tersedia? Saya kirim sekarang"
```
Masalah: Producer bisa lebih cepat dari Consumer → Consumer kewalahan (perlu backpressure)

**Pull-based (AsyncSequence):**
```
Consumer ──request──▶ Producer ──yield──▶ Consumer
"Saya siap menerima data berikutnya"
```
`AsyncSequence` menggunakan model pull: consumer meminta elemen berikutnya, dan hanya menerimanya ketika siap. Ini menciptakan **backpressure natural** — producer tidak bisa mengirim lebih cepat dari consumer bisa proses.

### AsyncSequence — Protocol

```swift
// Definisi minimal AsyncSequence:
protocol AsyncSequence {
    associatedtype Element
    associatedtype AsyncIterator: AsyncIteratorProtocol

    func makeAsyncIterator() -> AsyncIterator
}

protocol AsyncIteratorProtocol {
    associatedtype Element
    mutating func next() async throws -> Element?
}
```

**Penggunaan dengan `for await`:**
```swift
// for await adalah sintaks khusus untuk AsyncSequence
for await notification in NotificationCenter.default.notifications(named: .UIKeyboardWillShow) {
    let frame = notification.userInfo?[UIKeyboardFrameEndUserInfoKey] as? CGRect
    adjustForKeyboard(frame: frame)
}

// Dengan filter, map, dll — AsyncSequence punya lazy operators
let recentErrors = logStream
    .filter { $0.level == .error }
    .prefix(10)  // hanya 10 pertama
```

### AsyncStream — Bridge dari Callback ke Async

`AsyncStream` adalah cara membuat `AsyncSequence` dari sumber data push-based:

```swift
// Contoh: wrapping NotificationCenter ke AsyncStream
func keyboardNotifications() -> AsyncStream<KeyboardInfo> {
    AsyncStream { continuation in
        let showObserver = NotificationCenter.default.addObserver(
            forName: UIResponder.keyboardWillShowNotification,
            object: nil, queue: nil
        ) { notification in
            let info = KeyboardInfo(from: notification, isShowing: true)
            continuation.yield(info)  // push elemen ke stream
        }

        let hideObserver = NotificationCenter.default.addObserver(
            forName: UIResponder.keyboardWillHideNotification,
            object: nil, queue: nil
        ) { notification in
            let info = KeyboardInfo(from: notification, isShowing: false)
            continuation.yield(info)
        }

        // Cleanup saat stream selesai/di-cancel
        continuation.onTermination = { _ in
            NotificationCenter.default.removeObserver(showObserver)
            NotificationCenter.default.removeObserver(hideObserver)
        }
    }
}

// Penggunaan:
Task { @MainActor in
    for await keyboard in keyboardNotifications() {
        adjustLayout(for: keyboard)
    }
}
```

### AsyncThrowingStream — Dengan Error Handling

```swift
func liveUpdates(for documentID: String) -> AsyncThrowingStream<Document, Error> {
    AsyncThrowingStream { continuation in
        let listener = FirebaseDB.collection("docs").document(documentID)
            .addSnapshotListener { snapshot, error in
                if let error = error {
                    continuation.finish(throwing: error)  // stream selesai dengan error
                    return
                }
                guard let doc = try? snapshot?.data(as: Document.self) else {
                    return  // skip invalid snapshot
                }
                continuation.yield(doc)  // push document update
            }

        continuation.onTermination = { _ in
            listener.remove()  // cleanup Firebase listener
        }
    }
}

// Penggunaan:
do {
    for try await document in liveUpdates(for: "doc_123") {
        updateUI(with: document)
    }
} catch {
    showError(error)
}
```

### Kapan Menggunakan

**AsyncSequence — gunakan untuk:**
- Streaming data: WebSocket messages, server-sent events
- Real-time updates: database listeners, file watchers
- Infinite sequences: timer ticks, sensor data
- Mengganti delegation pattern yang berulang

**AsyncStream — gunakan ketika:**
- Menjembatani callback/delegate API ke async context
- Wrapping notification-based API
- Membuat custom event emitter

**Jangan gunakan ketika:**
- Hanya butuh satu nilai async (gunakan `async/await` biasa)
- Kamu butuh backpressure yang sangat presisi (gunakan custom AsyncIterator)
- Data sudah tersedia sepenuhnya (gunakan Array biasa)

### Real Use Case: WebSocket Live Chat

```swift
actor WebSocketClient {
    enum Message: Sendable {
        case text(String)
        case data(Data)
        case connected
        case disconnected(Error?)
    }

    private var webSocketTask: URLSessionWebSocketTask?
    private var continuation: AsyncStream<Message>.Continuation?

    var messages: AsyncStream<Message> {
        AsyncStream { [weak self] continuation in
            self?.continuation = continuation
            continuation.onTermination = { [weak self] _ in
                Task { await self?.disconnect() }
            }
        }
    }

    func connect(to url: URL) async throws {
        let task = URLSession.shared.webSocketTask(with: url)
        webSocketTask = task
        task.resume()
        continuation?.yield(.connected)
        await receiveLoop()
    }

    private func receiveLoop() async {
        while let task = webSocketTask {
            do {
                let message = try await task.receive()
                switch message {
                case .string(let text): continuation?.yield(.text(text))
                case .data(let data): continuation?.yield(.data(data))
                @unknown default: break
                }
            } catch {
                continuation?.yield(.disconnected(error))
                continuation?.finish()
                break
            }
        }
    }

    func send(_ text: String) async throws {
        try await webSocketTask?.send(.string(text))
    }

    func disconnect() {
        webSocketTask?.cancel(with: .normalClosure, reason: nil)
        webSocketTask = nil
        continuation?.finish()
        continuation = nil
    }
}

// Penggunaan di ViewModel:
@MainActor
class ChatViewModel: ObservableObject {
    @Published var messages: [ChatMessage] = []
    private let client = WebSocketClient()
    private var receiveTask: Task<Void, Never>?

    func connect(to url: URL) async throws {
        try await client.connect(to: url)

        receiveTask = Task {
            for await message in await client.messages {
                switch message {
                case .text(let text):
                    messages.append(ChatMessage(content: text, isMine: false))
                case .disconnected:
                    showReconnectPrompt()
                default: break
                }
            }
        }
    }

    func send(text: String) async throws {
        messages.append(ChatMessage(content: text, isMine: true))
        try await client.send(text)
    }
}

struct ChatMessage: Identifiable {
    let id = UUID()
    let content: String
    let isMine: Bool
}
```

---

# SWIFT 6.0 — STRICT CONCURRENCY (2024)

---

## 6. Complete Concurrency Checking

### Teori: Dari Warning ke Error — Pergeseran Filosofi

Swift 5.x memperkenalkan concurrency tools tapi tidak meng-enforce keamanannya. Developer bisa mengabaikan warning. Swift 6 mengubah ini fundamental: **semua data race check menjadi compile-time error.**

**Mengapa ini perubahan besar?**

Data race adalah salah satu bug paling berbahaya di software:
- Non-deterministic: hanya muncul dalam kondisi tertentu
- Hard to reproduce: timing-dependent
- Hard to debug: bisa muncul jauh dari source bug
- Bisa corrupt memory, data, atau crash di production

Dengan Complete Concurrency Checking, Swift menjadi salah satu dari sedikit bahasa mainstream dengan **compile-time data race safety** — bergabung dengan Rust sebagai trailblazer di area ini.

### Apa yang Di-check

```swift
// SEBELUM Swift 6 (warning):
class AppState {
    var currentUser: User?  // mutable reference type
}

let state = AppState()
Task { state.currentUser = user1 }  // warning: concurrent access
Task { state.currentUser = user2 }  // warning: concurrent access

// SWIFT 6 (error):
// error: reference to var 'currentUser' in concurrently-executing code
// Tidak bisa dikompilasi — harus fix
```

**Kategori error yang dideteksi:**
1. Akses ke non-Sendable type lintas isolation boundary
2. Mutation dari state yang tidak dilindungi dari concurrent access
3. Closure yang capture mutable state tanpa isolation
4. Global variable yang mutable tanpa protection

### Mengaktifkan Bertahap

```swift
// Package.swift — aktifkan complete checking
.target(
    name: "MyTarget",
    swiftSettings: [
        // Level 1: minimal — hampir tidak ada warning
        // Level 2: targeted — warning pada kode yang jelas concurrent
        // Level 3: complete — semua warning Swift 6
        .enableUpcomingFeature("StrictConcurrency"),  // targeted di Swift 5
        // atau:
        .swiftLanguageVersion(.v6)  // complete, semua jadi error
    ]
)
```

### Kapan Menggunakan

Complete Concurrency Checking selalu aktif di Swift 6 — bukan pilihan. Tapi kamu bisa mengontrol kapan bermigrasi:
- **Proyek baru:** Langsung Swift 6 — mulai benar dari awal
- **Proyek existing:** Aktifkan warning dulu (Targeted), fix bertahap, baru pindah ke Swift 6

---

## 7. Sendable Enforcement

### Teori: Type System sebagai Safety Boundary

Sendable enforcement adalah konsekuensi dari Complete Concurrency Checking. Setiap nilai yang melintas isolation boundary **harus bisa dibuktikan aman** oleh compiler.

Lihat file `swift-isolation-swift6.md` untuk teori mendalam tentang Sendable. Di sini fokus pada **enforcement behavior** di Swift 6:

```swift
// Swift 5: warning (bisa diabaikan)
class NonSendableConfig {
    var timeout = 30
}

actor Service {
    func configure(with config: NonSendableConfig) { }  // warning saja
}

// Swift 6: ERROR (tidak bisa dikompilasi)
// error: sending 'config' risks causing data races
// Non-Sendable type cannot cross isolation boundary
```

### Pattern Fix yang Umum

```swift
// Pattern 1: struct (paling sederhana)
struct Config: Sendable {
    var timeout: Int = 30
    var retries: Int = 3
}

// Pattern 2: final class dengan immutable properties
final class Config: Sendable {
    let timeout: Int
    let retries: Int
    init(timeout: Int = 30, retries: Int = 3) {
        self.timeout = timeout
        self.retries = retries
    }
}

// Pattern 3: jadikan bagian dari actor
actor Service {
    var timeout: Int = 30
    var retries: Int = 3
    // Config tidak perlu dikirim — sudah ada di dalam actor
}

// Pattern 4: @unchecked Sendable (last resort)
final class Config: @unchecked Sendable {
    private let lock = NSLock()
    private var _timeout = 30
    var timeout: Int {
        get { lock.withLock { _timeout } }
        set { lock.withLock { _timeout = newValue } }
    }
}
```

---

## 8. sending Modifier

### Teori: Region-Based Isolation

`sending` adalah bagian dari **region-based isolation** — sistem analisis yang lebih halus dari sekadar "apakah tipe ini Sendable?".

Sebelum `sending`, aturannya binary: tipe harus Sendable untuk bisa dikirim lintas isolation. Ini terlalu ketat — banyak nilai yang aman untuk dikirim jika caller tidak mengaksesnya lagi setelah pengiriman.

`sending` mengkodekan **transfer of ownership** ke type system: nilai ini dipindahkan ke isolation domain lain, dan caller tidak boleh mengaksesnya lagi.

```swift
// Tanpa sending: URLRequest tidak Sendable di Swift 5.x
// → tidak bisa dikirim ke async context

// Dengan sending: bisa, asalkan caller tidak akses lagi
func processRequest(_ request: sending URLRequest) async throws -> Data {
    // request sekarang milik fungsi ini
    var mutableRequest = request  // OK: kita memilikinya
    mutableRequest.setValue("v2", forHTTPHeaderField: "API-Version")
    let (data, _) = try await URLSession.shared.data(for: mutableRequest)
    return data
}

// Penggunaan:
var request = URLRequest(url: someURL)
request.httpMethod = "POST"
let data = try await processRequest(request)  // request di-"kirim"
// print(request.httpMethod)  // ❌ ERROR: request sudah di-transfer
```

### sending pada Return Type

```swift
// Fungsi yang menghasilkan nilai untuk dikirim ke isolation lain
actor DataStore {
    func getItemForProcessing(id: String) -> sending ProcessingJob {
        // ProcessingJob tidak harus Sendable
        // tapi karena di-return dengan 'sending', caller bisa kirim ke isolation lain
        return ProcessingJob(id: id, data: store[id])
    }
}
```

### Kapan Menggunakan

**Gunakan `sending` ketika:**
- Fungsi mengkonsumsi nilai dan mengirimkannya ke async context lain
- Tipe tidak Sendable tapi safe untuk di-transfer (ownership jelas)
- Builder pattern yang menghasilkan objek untuk digunakan di isolation berbeda

**Jangan gunakan `sending` ketika:**
- Caller masih butuh nilai setelah call
- Tipe bisa dibuat Sendable (struct dengan value types)
- Ada concurrent access yang mungkin terjadi

---

## 9. nonisolated(unsafe)

### Teori: Escape Hatch dengan Tanggung Jawab

`nonisolated(unsafe)` adalah cara untuk menyatakan ke compiler: *"percayai saya, properti ini aman diakses dari mana saja meskipun tidak ter-isolasi."*

Ini adalah **opt-out dari isolation checking** — compiler tidak akan memperingatkan tentang concurrent access pada property ini. Programmer mengambil seluruh tanggung jawab keamanannya.

```swift
// Tanpa nonisolated(unsafe): actor property hanya bisa diakses dengan await
actor Configuration {
    var timeout: Int = 30  // hanya bisa diakses dengan await
}

// Dengan nonisolated(unsafe): bisa diakses tanpa await
// Tapi compiler tidak memberikan jaminan thread-safety!
actor Configuration {
    nonisolated(unsafe) var debugMode: Bool = false
    // Kamu bertanggung jawab memastikan ini aman diakses concurrent
}

let config = Configuration()
print(config.debugMode)  // OK: tidak perlu await (tapi tidak thread-safe!)
```

### Kapan BOLEH Digunakan (dengan Hati-hati)

```swift
actor AppState {
    // KASUS 1: Nilai yang hanya ditulis sekali saat init, lalu read-only
    // (Write-once, read-many pattern)
    nonisolated(unsafe) let featureFlags: [String: Bool]

    init(flags: [String: Bool]) {
        self.featureFlags = flags  // ditulis sekali
        // Setelah ini: read-only — concurrent read aman
    }
}

// KASUS 2: Wrapping @unchecked Sendable yang sudah memiliki synchronization sendiri
actor MetricsCollector {
    nonisolated(unsafe) var hitCounter: Int32 = 0
    // Jika akses selalu via OSAtomic atau Atomic — aman secara manual

    nonisolated func recordHit() {
        // Asumsi: hanya diakses via atomic ops
        hitCounter += 1  // ⚠️ tidak atomic — ini contoh buruk!
        // Gunakan: _ = OSAtomicIncrement32(&hitCounter)
    }
}
```

### Kapan TIDAK Boleh Digunakan

```swift
// JANGAN: untuk menghindari compiler error tanpa alasan yang valid
actor UserCache {
    nonisolated(unsafe) var users: [User] = []  // ❌ BERBAHAYA

    // Ini seperti menonaktifkan airbag karena mengganggu
    // Concurrent mutation users[] → data corruption, crash
}

// Solusi yang benar:
actor UserCache {
    var users: [User] = []  // protected by actor
}
```

**Aturan praktis:** Jika kamu tidak bisa menjelaskan dengan tepat **mengapa** concurrent access ke property ini aman, jangan gunakan `nonisolated(unsafe)`.

---

## 10. @preconcurrency

### Teori: Jembatan ke Kode Lama

`@preconcurrency` adalah annotation untuk menangani **backward compatibility** — ketika kamu menggunakan library atau framework yang ditulis sebelum Swift concurrency ada, dan mereka belum di-update.

Dua penggunaan utama:

**Penggunaan 1: Import module lama**
```swift
// Library lama yang belum punya Sendable annotation
@preconcurrency import LegacyNetworkLibrary

// Dengan @preconcurrency import:
// - Compiler mengurangi strictness untuk tipe dari LegacyNetworkLibrary
// - Warning yang muncul menjadi lebih ringan (bukan error)
// - Kamu bisa bermigrasi bertahap
```

**Penggunaan 2: Protocol conformance lama**
```swift
// Protocol yang ditulis sebelum concurrency — tidak ada Sendable requirement
protocol LegacyDelegate {
    func didComplete(result: SomeResult)
}

// Class yang conform dengan @preconcurrency — relaxed checking
class MyHandler: @preconcurrency LegacyDelegate {
    @MainActor
    func didComplete(result: SomeResult) {
        // Compiler tidak akan error meskipun ada isolation mismatch
        // dengan protocol yang tidak mengenal isolation
    }
}
```

### Kapan Menggunakan

**Gunakan @preconcurrency ketika:**
- Mengimport third-party library yang belum di-update ke Swift Concurrency
- Conform ke protocol dari Objective-C atau pre-concurrency framework
- Dalam proses migrasi bertahap — sementara, bukan solusi permanen

**Jangan gunakan ketika:**
- Kode yang kamu kontrol — update saja dengan proper isolation
- Sebagai cara menghindari fix yang seharusnya dilakukan
- Dalam kode baru — ini adalah tool migrasi, bukan desain normal

```swift
// Contoh: UIKit delegate yang belum di-annotate @MainActor oleh Apple
class ViewController: UIViewController, @preconcurrency UITableViewDelegate {
    // UITableViewDelegate belum punya @MainActor di semua versi SDK
    // @preconcurrency: "saya tahu ini akan jalan di main thread"

    func tableView(_ tableView: UITableView, didSelectRowAt indexPath: IndexPath) {
        // Dijamin main thread karena UITableViewDelegate
    }
}
```

---

## 11. Global Actor Isolation — Default Lebih Ketat

### Teori: Perubahan Default Inference di Swift 6

Di Swift 5, jika sebuah fungsi dipanggil dari `@MainActor` context, compiler tidak otomatis meng-infer bahwa fungsi tersebut harus berjalan di main actor. Di Swift 6, inference ini lebih **ketat dan eksplisit**.

**Perubahan utama:**

```swift
// Swift 5: actor isolation sering di-infer secara implisit dan tersembunyi
class MyVC: UIViewController {
    func setup() {
        // Swift 5: mungkin di-infer sebagai @MainActor karena UIViewController
        // Swift 6: harus eksplisit atau ada error
        doAsyncWork()
    }

    func doAsyncWork() {
        Task {
            await someActorMethod()  // isolation tidak jelas
        }
    }
}

// Swift 6: harus eksplisit
@MainActor
class MyVC: UIViewController {
    func setup() {  // jelas: @MainActor
        Task {
            await someActorMethod()  // jelas: hop dari @MainActor ke actor
        }
    }
}
```

### Global Actor pada Stored Property

```swift
// Swift 6: global actor bisa di-apply ke stored property individual
class AppConfiguration {
    @MainActor var theme: AppTheme = .default     // hanya di main thread
    @MainActor var fontSize: CGFloat = 16          // hanya di main thread

    // Non-isolated: bisa diakses dari mana saja
    let version: String = "1.0"
    let buildNumber: Int = 42
}

// Penggunaan:
let config = AppConfiguration()
await MainActor.run { config.theme = .dark }  // harus await
print(config.version)  // tidak perlu await — nonisolated
```

---

# SWIFT 6.1 — REFINEMENTS (2025)

---

## 12. nonisolated pada Stored Property

### Teori: Granularitas Lebih Tinggi

Di Swift 6.0, `nonisolated` hanya bisa diterapkan pada **computed properties** dan **methods** dari actor. Swift 6.1 memperluas ini ke **stored properties** — memberikan kontrol yang lebih granular.

**Mengapa ini berguna?**

Sebelumnya, jika kamu punya actor dengan beberapa property, semuanya harus diakses dengan `await`. Bahkan property yang immutable (thread-safe secara alami) pun harus await — overhead yang tidak perlu.

```swift
// Swift 6.0: semua property butuh await, termasuk yang immutable
actor UserProfile {
    let id: UUID        // immutable — tidak perlu protection
    let username: String // immutable — tidak perlu protection
    var bio: String      // mutable — butuh protection

    // Workaround di Swift 6.0: computed property
    nonisolated var displayID: String { id.uuidString }  // OK tapi verbose
}

// Penggunaan yang verbose:
let profile = UserProfile(...)
let id = await profile.id        // overhead untuk hal yang immutable
let username = await profile.username  // overhead tidak perlu
```

```swift
// Swift 6.1: nonisolated pada stored property langsung
actor UserProfile {
    nonisolated let id: UUID         // akses tanpa await ✓
    nonisolated let username: String // akses tanpa await ✓
    var bio: String                  // masih protected ✓
}

// Penggunaan yang lebih natural:
let profile = UserProfile(...)
print(profile.id)       // tanpa await — property immutable
print(profile.username) // tanpa await — property immutable
let bio = await profile.bio  // dengan await — mutable, butuh protection
```

### Aturan `nonisolated` pada Stored Property

```swift
actor MyActor {
    // ✅ Diizinkan: let (immutable) — naturally thread-safe
    nonisolated let constantValue: Int = 42
    nonisolated let createdAt: Date

    // ✅ Diizinkan: var dengan Sendable type + synchronization manual
    nonisolated var atomicCounter: Int = 0  // ⚠️ harus handle thread-safety sendiri

    // ❌ Tidak disarankan: var tanpa synchronization
    // nonisolated var mutableState: String  // bisa jadi data race
}
```

### Kapan Menggunakan

**Gunakan `nonisolated` pada stored property ketika:**
- Property adalah `let` (immutable) — definitif aman
- Property represent identity atau metadata yang tidak berubah
- Mengurangi overhead `await` untuk data yang sering diakses dan tidak berubah

**Jangan gunakan ketika:**
- Property adalah `var` tanpa explicit synchronization
- Tidak yakin apakah concurrent access aman

---

## 13. Improved Actor Isolation Inference

### Teori: Lebih Sedikit Annotation Manual

Swift 6.1 meningkatkan kemampuan compiler untuk **meng-infer isolation secara otomatis** dalam lebih banyak kasus — mengurangi kebutuhan annotation eksplisit.

**Improvement utama:**

```swift
// Swift 6.0: perlu annotation eksplisit di banyak tempat
@MainActor
class ViewController: UIViewController {
    @MainActor  // perlu explicit di Swift 6.0
    func handleTap() { }

    @MainActor  // perlu explicit di Swift 6.0
    var displayText: String = ""
}

// Swift 6.1: isolation diinfer dari context class
@MainActor
class ViewController: UIViewController {
    // Tidak perlu annotation ulang — sudah diinfer dari class
    func handleTap() { }       // diinfer @MainActor ✓
    var displayText: String = "" // diinfer @MainActor ✓
}
```

**Protocol conformance inference:**
```swift
@MainActor
protocol MainActorProtocol {
    func update()
}

// Swift 6.1: conformance meng-infer isolation ke semua methods
class MyClass: MainActorProtocol {
    func update() {
        // Diinfer @MainActor karena protocol isolation
        // Tidak perlu @MainActor annotation eksplisit di method ini
    }
}
```

### Dampak Praktis

```swift
// Sebelum Swift 6.1: banyak annotation repetitif
@MainActor class HomeViewController: UIViewController {
    @MainActor private var viewModel: HomeViewModel
    @MainActor override func viewDidLoad() { super.viewDidLoad() }
    @MainActor override func viewWillAppear(_ animated: Bool) { }
    @MainActor func updateUI() { }
    @MainActor func handleError(_ error: Error) { }
}

// Swift 6.1: annotation di class sudah cukup
@MainActor class HomeViewController: UIViewController {
    private var viewModel: HomeViewModel       // diinfer @MainActor
    override func viewDidLoad() {              // diinfer @MainActor
        super.viewDidLoad()
    }
    override func viewWillAppear(_ animated: Bool) { }  // diinfer
    func updateUI() { }                        // diinfer @MainActor
    func handleError(_ error: Error) { }       // diinfer @MainActor
}
```

---

## 14. @isolated(any)

### Teori: Function Type yang Isolation-Agnostic

`@isolated(any)` adalah annotation untuk **function types** yang memungkinkan fungsi tersebut membawa isolation context-nya sendiri. Ini berbeda dari fungsi biasa yang isolation-nya fixed saat definition.

**Problem yang dipecahkan:**
```swift
// Swift 6.0: function type dengan isolation fixed
typealias MainActorAction = @MainActor () -> Void
typealias ActorAction = () -> Void  // tidak ada isolation info

// Bagaimana membuat function type yang bisa isolated ke actor APAPUN?
// Tidak bisa di Swift 6.0 — harus pilih satu
```

```swift
// Swift 6.1: @isolated(any) — function type yang membawa isolation-nya sendiri
typealias IsolatedAction = @isolated(any) () -> Void

// IsolatedAction bisa di-create dari @MainActor context:
@MainActor func createMainActorAction() -> IsolatedAction {
    return { /* berjalan di @MainActor */ }
}

// Atau dari arbitrary actor:
actor MyActor {
    func createActorAction() -> IsolatedAction {
        return { /* berjalan di MyActor */ }
    }
}

// Caller bisa execute tanpa tahu isolation-nya:
func execute(_ action: @isolated(any) () async -> Void) async {
    await action()  // berjalan di isolation yang dibawa action
}
```

### Kapan Menggunakan

**Gunakan `@isolated(any)` ketika:**
- Membuat callback system yang harus menghormati isolation caller
- Framework code yang perlu menjalankan user-provided closures di isolation yang tepat
- Generic higher-order functions yang isolation-agnostic

**Contoh nyata:**
```swift
// Event handler system yang respects isolation
struct EventHandler {
    let action: @isolated(any) (Event) async -> Void

    func handle(_ event: Event) async {
        await action(event)  // berjalan di isolation yang benar
    }
}

// Penggunaan dari @MainActor context:
@MainActor
class UIController {
    let handler = EventHandler { [weak self] event in
        // Closure ini dijalankan di @MainActor — aman untuk update UI
        self?.updateDisplay(for: event)
    }
}

// Penggunaan dari actor context:
actor BackgroundProcessor {
    let handler = EventHandler { [weak self] event in
        // Closure ini dijalankan di BackgroundProcessor actor
        await self?.processEvent(event)
    }
}
```

---

# SWIFT 6.2 — ERGONOMICS (2025)

---

## 15. Default Actor Isolation per Module

### Teori: Module-Wide Default Isolation

Salah satu friction terbesar saat bermigrasi ke Swift 6 adalah keharusan menambahkan `@MainActor` ke **setiap** ViewController, ViewModel, dan View. Di aplikasi besar dengan ratusan file, ini adalah pekerjaan yang sangat repetitif.

Swift 6.2 memperkenalkan kemampuan untuk **set default isolation seluruh module**:

```swift
// Package.swift — set default isolation untuk target
.target(
    name: "UILayer",
    swiftSettings: [
        .swiftLanguageVersion(.v6),
        .defaultIsolation(MainActor.self)  // semua kode di module ini: @MainActor by default
    ]
)
```

```swift
// UILayer/HomeViewController.swift
// TANPA annotation eksplisit — sudah @MainActor karena module default

class HomeViewController: UIViewController {
    var viewModel: HomeViewModel  // @MainActor — inherited from module default
    var isLoading = false          // @MainActor

    func loadData() async {
        isLoading = true
        let data = await viewModel.fetchData()  // hop ke viewModel isolation
        displayData(data)
        isLoading = false
    }
}

// Untuk fungsi yang memang butuh background — eksplisit opt-out:
nonisolated func heavyComputation(data: Data) -> Result {
    // berjalan tanpa isolation — background thread
}
```

### Dampak dan Trade-off

```
Module UILayer (default: @MainActor)
├── HomeViewController  → @MainActor (automatic)
├── HomeViewModel       → @MainActor (automatic)
├── UserCell            → @MainActor (automatic)
└── heavyComputation()  → nonisolated (explicit opt-out)

Module Core (no default isolation)
├── UserRepository      → actor (explicit)
├── NetworkClient       → actor (explicit)
└── DataParser          → nonisolated (default, no annotation)
```

**Kapan gunakan default isolation:**
- UI layer (ViewController, View, ViewModel)
- Module yang predominantly main-thread
- Migrasi: mengurangi jumlah annotation yang perlu ditambahkan

**Jangan gunakan untuk:**
- Module bisnis logic atau data layer — akan "accidentally" main thread
- Library dengan mixed isolation needs
- Module yang di-share antara UI dan background processing

---

## 16. Task Naming

### Teori: Debuggability sebagai First-Class Concern

Sebelum Swift 6.2, ketika terjadi crash atau hang di dalam Task, stack trace hanya menunjukkan alamat memori atau nama fungsi generik — sangat sulit di-debug:

```
Thread 7 Queue: com.apple.root.default-qos (concurrent)
  #0  0x... in Task.runOnCurrentThread()
  #1  0x... in closure #1 in ViewController.viewDidLoad()
  → Tidak tahu Task ini untuk apa!
```

Swift 6.2 menambahkan **Task naming** — identifier human-readable yang muncul di debugger, Instruments, dan crash reports:

```swift
// Swift 6.2: beri nama pada Task untuk debugging
Task(name: "Load User Profile") {
    try await loadUserProfile()
}

Task(name: "Sync Background Data") {
    await syncData()
}

// Di Instruments / debugger:
// Thread: "Load User Profile" — jauh lebih jelas!
```

### Penggunaan Lanjutan

```swift
// Nama dinamis berdasarkan context
func processOrders(_ orders: [Order]) async {
    await withTaskGroup(of: Void.self) { group in
        for order in orders {
            group.addTask(name: "Process Order \(order.id)") {
                await self.processOrder(order)
            }
        }
    }
}

// Untuk monitoring dan logging
actor TaskMonitor {
    var activeTasks: Set<String> = []

    func trackTask(name: String, operation: @Sendable () async throws -> Void) async rethrows {
        activeTasks.insert(name)
        defer { activeTasks.remove(name) }
        try await operation()
    }
}

// Penggunaan:
let monitor = TaskMonitor()
Task(name: "Critical Payment Processing") {
    await monitor.trackTask(name: "Payment") {
        try await processPayment(order)
    }
}
```

### Kapan Menggunakan

**Gunakan Task naming ketika:**
- Task melakukan operasi yang distinctif dan perlu diidentifikasi
- Debugging concurrent code yang kompleks
- Profiling dengan Instruments
- Long-running background tasks yang perlu dimonitor

**Tidak perlu ketika:**
- Short-lived task yang trivial
- Task yang sudah jelas dari context-nya

---

## 17. Improved async let Cancellation Semantics

### Teori: Masalah Cancellation di Swift 6.0

Di Swift 6.0, ketika `async let` binding keluar scope, child tasks mungkin tidak segera di-cancel dalam semua skenario. Ini bisa menyebabkan:
- Pekerjaan yang tidak diperlukan terus berjalan
- Resource yang tidak di-release tepat waktu
- Perilaku yang sulit diprediksi

Swift 6.2 memperketat semantik ini:

```swift
// Swift 6.2: cancellation lebih deterministic dan immediate

func loadWithFallback() async throws -> Data {
    async let primary = fetchFromPrimaryServer()    // mulai
    async let backup = fetchFromBackupServer()      // mulai concurrent

    do {
        let data = try await primary  // tunggu primary
        // Swift 6.2: 'backup' task SEGERA di-cancel saat primary berhasil
        // Swift 6.0: 'backup' mungkin masih berjalan sebentar
        return data
    } catch {
        // Primary gagal — gunakan backup (sudah berjalan concurrent)
        return try await backup
    }
    // Setelah scope selesai: semua async let yang belum di-await otomatis di-cancel
}
```

### Cancellation Propagation yang Lebih Baik

```swift
// Pattern: race between multiple sources, cancel loser
func fetchWithRace(timeout: Duration) async throws -> Data {
    async let result = fetchData()
    async let timeoutSignal: Void = Task.sleep(for: timeout)

    // Whichever finishes first wins
    do {
        return try await result  // kalau ini selesai duluan...
        // 'timeoutSignal' task di-cancel segera di Swift 6.2
    } catch {
        // Timeout terjadi — result task di-cancel
        throw FetchError.timeout
    }
}
```

### Kapan Ini Penting

**Ini penting ketika:**
- Kamu menggunakan async let untuk "racing" beberapa operasi
- Resource yang digunakan oleh async let mahal (network connections, file handles)
- Perlu behavior yang predictable dan deterministik

---

## 18. nonisolated on init

### Teori: Fleksibilitas Inisialisasi

Sebelum Swift 6.2, `init` dari `@MainActor` class atau actor selalu diisolasi ke actor/MainActor tersebut. Ini menyebabkan semua caller harus `await`:

```swift
// Swift 6.0: init adalah @MainActor — semua caller harus await
@MainActor
class ViewModel {
    var data: [Item] = []

    init() {  // implicitly @MainActor
        // setup
    }
}

// Caller:
Task {
    let vm = await ViewModel()  // harus await meski init trivial!
}
```

Swift 6.2 memungkinkan `nonisolated init` — init yang bisa dipanggil dari context mana saja tanpa `await`:

```swift
// Swift 6.2: nonisolated init
@MainActor
class ViewModel {
    var data: [Item] = []
    private let config: Configuration

    // Init tidak butuh @MainActor — hanya set constant properties
    nonisolated init(config: Configuration) {
        self.config = config
        // Tidak boleh akses @MainActor properties di sini
        // self.data = []  // ❌ ERROR: data adalah @MainActor
    }
}

// Caller — tidak perlu await:
let vm = ViewModel(config: .default)  // synchronous!
// vm sudah bisa digunakan, tapi akses property harus tetap await
await vm.loadInitialData()
```

### Aturan nonisolated init

```swift
@MainActor
class Example {
    let constant: String
    var mutableState: [Int] = []

    nonisolated init(constant: String) {
        self.constant = constant  // ✅ set let property — OK
        // self.mutableState = [1,2,3]  // ❌ mutableState adalah @MainActor
    }
}

actor MyActor {
    nonisolated let id: UUID
    var data: [String] = []

    nonisolated init() {
        self.id = UUID()  // ✅ nonisolated let — OK
        // self.data = ["initial"]  // ❌ data adalah actor-isolated
    }
}
```

### Kapan Menggunakan

**Gunakan `nonisolated init` ketika:**
- Init hanya men-setup immutable (`let`) properties
- Ingin membuat object tanpa butuh async context
- Dependency injection di synchronous context (AppDelegate, SceneDelegate setup)

**Jangan gunakan ketika:**
- Init perlu setup mutable state — harus dalam isolation
- Init melakukan async work (gunakan factory method async sebagai gantinya)

---

# PENUTUP

---

## 19. Evolusi Timeline & Keputusan Desain

```
Swift 5.5 (2021) — FONDASI
├── async/await: ganti callback → linear code
├── Structured Concurrency: lifetime & cancellation guarantee
├── actor: thread-safe mutable state
├── @MainActor: UI thread safety
└── AsyncSequence/Stream: pull-based streaming

Swift 6.0 (2024) — ENFORCEMENT
├── Complete Concurrency Checking: warning → error
├── Sendable enforcement: type system boundary
├── sending: transfer ownership explicitly
├── nonisolated(unsafe): controlled escape hatch
├── @preconcurrency: backward compat bridge
└── Global actor: stricter inference

Swift 6.1 (2025) — GRANULARITY
├── nonisolated stored property: per-property control
├── Improved inference: less annotation noise
└── @isolated(any): isolation-carrying function types

Swift 6.2 (2025) — ERGONOMICS
├── Default module isolation: less boilerplate
├── Task naming: better debuggability
├── async let cancellation: more deterministic
└── nonisolated init: sync object creation
```

**Filosofi evolusi:**
1. **5.5**: Berikan tools yang benar
2. **6.0**: Enforce penggunaan yang benar
3. **6.1**: Kurangi friction di edge cases
4. **6.2**: Kurangi boilerplate, tingkatkan ergonomics

---

## 20. Decision Guide — Memilih Tool yang Tepat

### Untuk Shared Mutable State

```
State diakses concurrent dari banyak context?
├── YA → actor (paling natural untuk mutable state)
│         atau Mutex<Value> (jika sync context, tidak bisa await)
└── TIDAK → tidak perlu isolation khusus
```

### Untuk UI Layer

```
Kode menyentuh UIKit/AppKit?
├── YA → @MainActor
│         Swift 6.2: set default isolation @MainActor per module
└── TIDAK → actor atau nonisolated
```

### Untuk Operasi Concurrent

```
Jumlah operasi diketahui saat compile time?
├── YA (2-5 operasi) → async let
└── TIDAK (array, dinamis) → withTaskGroup / withThrowingTaskGroup
                              pertimbangkan limit concurrency
```

### Untuk Streaming Data

```
Jumlah data: satu nilai atau banyak?
├── Satu nilai → async func dengan return
├── Banyak nilai, finite → Array + async func
└── Banyak nilai, ongoing → AsyncStream / AsyncThrowingStream
```

### Untuk Tipe yang Melintas Isolation

```
Tipe perlu dikirim antar isolation domain?
├── Value type (struct/enum) dengan all-Sendable properties → Sendable otomatis
├── Reference type (class) → jadikan actor, atau final+immutable+Sendable
├── Non-Sendable yang tidak diakses lagi setelah dikirim → sending modifier
└── Legacy type yang tidak bisa diubah → @unchecked Sendable (hati-hati)
```

### Quick Reference Semua Fitur

| Fitur | Swift | Problem | Gunakan |
|---|---|---|---|
| `async/await` | 5.5 | Callback hell | Semua I/O dan async work |
| `async let` | 5.5 | Sequential async yang bisa parallel | 2-5 op concurrent |
| `TaskGroup` | 5.5 | N concurrent tasks dari collection | Batch processing |
| `actor` | 5.5 | Shared mutable state thread-safety | Data store, cache, service |
| `@MainActor` | 5.5 | UI update thread safety | VC, VM, View |
| `AsyncStream` | 5.5 | Bridge callback ke async | Notification, WebSocket |
| Complete Checking | 6.0 | Data race deteksi | Selalu aktif di Swift 6 |
| `Sendable` | 6.0 | Tipe safe lintas isolation | Semua cross-boundary types |
| `sending` | 6.0 | Transfer non-Sendable type | Ownership transfer |
| `nonisolated(unsafe)` | 6.0 | Opt-out isolation (dangerous) | Hanya dengan justifikasi kuat |
| `@preconcurrency` | 6.0 | Backward compat library | Migrasi saja |
| `nonisolated` stored prop | 6.1 | Overhead await untuk let | Immutable actor properties |
| Improved inference | 6.1 | Terlalu banyak annotation | Kurang annotation manual |
| `@isolated(any)` | 6.1 | Isolation-agnostic callbacks | Framework/library code |
| Default module isolation | 6.2 | Boilerplate @MainActor | UI layer module |
| Task naming | 6.2 | Debug concurrent code | Semua long-running Task |
| async let cancellation | 6.2 | Indeterminate cancellation | Selalu (automatic improvement) |
| `nonisolated init` | 6.2 | Sync object creation | Init dengan hanya let properties |

---

*Dibuat: 2026-05-09 | Swift 5.5 → Swift 6.2 | Comprehensive Concurrency Guide*
