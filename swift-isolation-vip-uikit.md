# Swift Isolation + UIKit: Pattern VIP (View–Interactor–Presenter)
### Panduan Lengkap dengan Penjelasan Setiap Baris Kode

> **Studi kasus:** Fitur "User List" — menampilkan daftar user dari API, dengan loading state, error handling, pull-to-refresh, dan navigasi ke detail.

---

## Daftar Isi

1. [Gambaran Arsitektur VIP + Isolation](#1-gambaran-arsitektur-vip--isolation)
2. [Models & Namespace](#2-models--namespace)
3. [Protocols](#3-protocols)
4. [Worker — actor](#4-worker--actor)
5. [Interactor — actor](#5-interactor--actor)
6. [Presenter — @MainActor](#6-presenter--mainactor)
7. [ViewController — @MainActor](#7-viewcontroller--mainactor)
8. [Router — @MainActor](#8-router--mainactor)
9. [Configurator — Dependency Injection](#9-configurator--dependency-injection)
10. [Alur Data Lengkap](#10-alur-data-lengkap)
11. [Swift Concurrency: Task Cancellation](#11-swift-concurrency-task-cancellation)
12. [Swift Concurrency: async let — Parallel Fetch](#12-swift-concurrency-async-let--parallel-fetch)
13. [Swift Concurrency: TaskGroup — Bulk Avatar Prefetch](#13-swift-concurrency-taskgroup--bulk-avatar-prefetch)
14. [Swift Concurrency: AsyncStream — Live Polling](#14-swift-concurrency-asyncstream--live-polling)
15. [Swift Concurrency: withCheckedContinuation — Bridge Delegate](#15-swift-concurrency-withcheckedcontinuation--bridge-delegate)
16. [Observable dalam VIP UIKit — Motivasi & Arsitektur](#16-observable-dalam-vip-uikit--motivasi--arsitektur)
17. [Implementasi Observable VIP — DisplayState Pattern](#17-implementasi-observable-vip--displaystate-pattern)
18. [Trade-offs: Pure VIP vs Observable VIP](#18-trade-offs-pure-vip-vs-observable-vip)

---

## 1. Gambaran Arsitektur VIP + Isolation

### Diagram Alur

```
┌─────────────────────────────────────────────────────────────────┐
│                        @MainActor                               │
│  ┌────────────────────┐        ┌───────────────────────────┐    │
│  │   ViewController   │───────▶│        Router             │    │
│  │  (UIKit / UI only) │        │  (push/present/dismiss)   │    │
│  └────────┬───────────┘        └───────────────────────────┘    │
│           │ displayUsers()                ▲                     │
│           │ displayError()                │ routeToDetail()     │
└───────────┼───────────────────────────────┼─────────────────────┘
            │ await fetchUsers()            │
            ▼                              │
┌─────────────────────────────────────────┐│
│              actor                      ││
│         Interactor                      ││
│   (business logic, state)               ││
│                                         ││
│   uses: Worker (actor)                  ││
└─────────────────────────────────────────┘│
            │ await presenter.presentUsers(response)
            ▼
┌─────────────────────────────────────────┐
│              @MainActor                 │
│          Presenter                      │
│   (format data, tidak ada UIKit)        │
│                                         │
│   calls: viewController.displayUsers()  │
└─────────────────────────────────────────┘
```

### Pembagian Isolation

| Komponen | Isolation | Alasan |
|---|---|---|
| `ViewController` | `@MainActor` | Semua UIKit harus di main thread |
| `Presenter` | `@MainActor` | Memanggil ViewController yang @MainActor |
| `Interactor` | `actor` | Punya mutable state (loading flag, cache) |
| `Worker` | `actor` | Network/data access yang thread-safe |
| `Router` | `@MainActor` | Navigation pakai UIKit |
| Models (Request/Response/ViewModel) | `Sendable struct` | Harus melewati isolation boundary |

### Mengapa Bukan Semua @MainActor?

Menaruh semua layer di `@MainActor` memang mudah, tapi **memblokir main thread** untuk pekerjaan berat seperti parsing JSON, query database, atau transformasi data. Dengan `actor` di Interactor dan Worker, pekerjaan tersebut berjalan di background — main thread tetap responsif untuk animasi dan gesture.

---

## 2. Models & Namespace

```swift
import Foundation

// MARK: - Domain Model

// Sendable wajib karena User akan melewati batas isolation:
// dari Worker (actor) → Interactor (actor) → Presenter (@MainActor)
struct User: Sendable, Equatable {
    let id: String
    let name: String
    let email: String
    let avatarURL: URL?
}

// MARK: - VIP Namespace

// Enum kosong sebagai namespace — pola Clean Swift untuk menghindari
// nama bertabrakan dan membuat struktur data per use-case lebih jelas
enum UserList {

    // Setiap use-case punya tiga lapisan data:
    // Request  → dari View ke Interactor
    // Response → dari Interactor ke Presenter (data mentah)
    // ViewModel → dari Presenter ke View (sudah diformat, siap tampil)

    enum FetchUsers {

        // Request: data apa yang View kirim ke Interactor.
        // Kosong di sini karena fetch tidak butuh parameter.
        // Tetap dibuat struct untuk konsistensi dan kemudahan ekstensi di masa depan.
        struct Request: Sendable {}

        // Response: data mentah dari business logic.
        // Sendable karena dikirim dari Interactor (actor) ke Presenter (@MainActor).
        struct Response: Sendable {
            let users: [User]
        }

        // ViewModel: data yang sudah siap ditampilkan.
        // Hanya berisi String/primitif — Presenter yang bertanggung jawab format.
        struct ViewModel: Sendable {
            struct DisplayedUser: Sendable {
                let id: String
                let fullName: String        // sudah di-capitalize
                let emailLabel: String      // sudah diformat "Email: ..."
                let avatarURL: URL?
            }
            let displayedUsers: [DisplayedUser]
        }
    }

    enum SelectUser {

        // Request berisi index yang di-tap user di tableView
        struct Request: Sendable {
            let index: Int
        }

        // Response berisi User yang dipilih untuk diteruskan ke Router
        struct Response: Sendable {
            let selectedUser: User
        }

        struct ViewModel: Sendable {
            let userId: String
            let userName: String
        }
    }
}
```

**Penjelasan poin penting:**

- **`Sendable`** di setiap model adalah **wajib** di Swift 6. Tanpa ini, compiler akan error ketika model dikirim melewati batas isolation (misalnya dari `actor` Interactor ke `@MainActor` Presenter).
- **Namespace `enum UserList`** adalah konvensi Clean Swift (VIP). Setiap fitur punya namespacenya sendiri sehingga nama `Request`, `Response`, `ViewModel` tidak bertabrakan di seluruh project.
- **`Request` kosong** tetap dibuat — agar jika suatu saat butuh parameter (filter, pagination), tidak perlu mengubah signature seluruh chain.

---

## 3. Protocols

```swift
import UIKit

// MARK: - Display Logic (View ← Presenter)

// Protocol ini dimiliki oleh View.
// Semua method wajib @MainActor karena memanggil UIKit.
// Presenter akan await method ini ketika memanggilnya dari @MainActor-nya sendiri.
@MainActor
protocol UserListDisplayLogic: AnyObject {
    func displayUsers(_ viewModel: UserList.FetchUsers.ViewModel)
    func displayLoading(_ isLoading: Bool)
    func displayError(message: String)
    func displaySelectedUser(_ viewModel: UserList.SelectUser.ViewModel)
}

// MARK: - Business Logic (Interactor ← View)

// Protocol ini dimiliki oleh Interactor.
// Method dibuat async karena Interactor adalah actor —
// memanggilnya dari @MainActor ViewController butuh await.
protocol UserListBusinessLogic: AnyObject, Sendable {
    func fetchUsers(request: UserList.FetchUsers.Request) async
    func selectUser(request: UserList.SelectUser.Request) async
}

// MARK: - Presentation Logic (Presenter ← Interactor)

// Protocol ini dimiliki oleh Presenter.
// @MainActor di sini karena Presenter harus memformat dan langsung
// memanggil View yang juga @MainActor.
// Ketika Interactor (actor) memanggil method ini, Swift otomatis
// menjadwalkan eksekusi ke main thread lewat await.
@MainActor
protocol UserListPresentationLogic: AnyObject {
    func presentUsers(_ response: UserList.FetchUsers.Response)
    func presentLoading(_ isLoading: Bool)
    func presentError(_ error: Error)
    func presentSelectedUser(_ response: UserList.SelectUser.Response)
}

// MARK: - Routing Logic (Router ← View)

// @MainActor karena navigasi selalu butuh UIKit (push, present, dll)
@MainActor
protocol UserListRoutingLogic: AnyObject {
    func routeToUserDetail(userId: String, userName: String)
}

// MARK: - Data Passing (Router ← View, shared dengan Interactor)

// DataStore adalah jembatan data antara View dan Router.
// Disimpan di Interactor dan dibaca oleh Router saat navigasi.
// Sendable karena bisa diakses dari beberapa isolation context.
protocol UserListDataStore: AnyObject {
    var selectedUser: User? { get }
}
```

**Penjelasan poin penting:**

- **`@MainActor` di protocol** berarti semua conforming type harus memenuhi kontrak tersebut di main thread. Ketika Interactor (`actor`) memanggil `presenter.presentUsers(...)`, Swift akan otomatis menjadwalkan ke main thread — tanpa kita perlu `DispatchQueue.main.async`.
- **`Sendable` di `UserListBusinessLogic`** memungkinkan protocol ini disimpan di dalam `actor` sebagai property. Tanpa ini, menyimpan referensi ke protocol di dalam actor akan menyebabkan compile error di Swift 6.
- **`async` method di `UserListBusinessLogic`** merefleksikan kenyataan bahwa Interactor adalah `actor` — memanggilnya dari konteks berbeda memerlukan `await`.

---

## 4. Worker — actor

```swift
import Foundation

// Worker bertanggung jawab atas satu hal saja: ambil data dari sumber eksternal.
// Dijadikan actor untuk memastikan tidak ada concurrent access ke state internalnya
// (misalnya URLSession, cache, atau connection pool).
actor UserWorker {

    // URLSession sudah thread-safe (Sendable), tapi kita tetap simpan di dalam
    // actor agar konfigurasi session tidak bisa diubah dari luar secara bersamaan.
    private let session: URLSession
    private let baseURL: URL

    init(
        session: URLSession = .shared,
        baseURL: URL = URL(string: "https://jsonplaceholder.typicode.com")!
    ) {
        self.session = session
        self.baseURL = baseURL
    }

    // Fungsi ini berjalan di dalam isolation domain actor ini (background thread).
    // Return type [User] adalah Sendable sehingga aman dikirim ke caller (Interactor).
    func fetchUsers() async throws -> [User] {
        let url = baseURL.appendingPathComponent("/users")
        let request = URLRequest(url: url, cachePolicy: .reloadIgnoringLocalCacheData)

        // await di sini melepas thread actor sementara — actor bisa melayani
        // permintaan lain selagi menunggu network response.
        let (data, response) = try await session.data(for: request)

        // Validasi HTTP response — tidak butuh await, ini synchronous check.
        guard let httpResponse = response as? HTTPURLResponse,
              (200...299).contains(httpResponse.statusCode) else {
            throw WorkerError.invalidResponse
        }

        // JSONDecoder.decode adalah operasi CPU-bound.
        // Karena kita berada di actor (bukan main thread), ini aman dan tidak
        // memblokir UI.
        let dtos = try JSONDecoder().decode([UserDTO].self, from: data)

        // Transformasi dari DTO (raw API model) ke Domain Model.
        // Transformasi ini juga terjadi di background — efisien.
        return dtos.map { dto in
            User(
                id: String(dto.id),
                name: dto.name,
                email: dto.email,
                avatarURL: URL(string: "https://i.pravatar.cc/150?u=\(dto.id)")
            )
        }
    }

    enum WorkerError: Error, LocalizedError {
        case invalidResponse

        var errorDescription: String? {
            switch self {
            case .invalidResponse: return "Server mengembalikan respons tidak valid."
            }
        }
    }
}

// MARK: - DTO (Data Transfer Object)

// DTO hanya digunakan di Worker — tidak perlu Sendable karena
// tidak pernah keluar dari isolation domain Worker.
private struct UserDTO: Decodable {
    let id: Int
    let name: String
    let email: String
}
```

**Penjelasan poin penting:**

- **`actor UserWorker`** menjamin bahwa `session` dan `baseURL` tidak diakses secara concurrent dari beberapa thread. Meskipun `URLSession` sudah Sendable, membungkusnya dalam actor adalah praktik yang baik untuk skalabilitas (misalnya saat nanti menambah retry logic atau request queuing).
- **`UserDTO` adalah `private`** — detail implementasi network tidak bocor ke layer lain. Hanya `User` (domain model) yang keluar dari Worker.
- **`await session.data(for:)`** melepas isolation domain Actor sementara — inilah kekuatan Actor: bisa melayani permintaan lain selagi menunggu I/O.

---

## 5. Interactor — actor

```swift
import Foundation

// Interactor adalah "otak" dari fitur — semua business logic ada di sini.
// Dijadikan actor karena:
// 1. Punya mutable state (isLoading, cachedUsers, selectedUser)
// 2. Dipanggil dari beberapa tempat secara concurrent (pull-to-refresh, auto-refresh)
// 3. Isolation memastikan state tidak corrupt saat ada concurrent calls
actor UserListInteractor: UserListBusinessLogic, UserListDataStore {

    // Presenter disimpan sebagai weak reference untuk mencegah retain cycle.
    // 'any' keyword karena ini existential — kita tidak tahu concrete type-nya.
    // Tidak perlu Sendable karena diakses hanya dari isolation domain actor ini.
    weak var presenter: (any UserListPresentationLogic)?

    private let worker: UserWorker

    // State internal — karena ini actor, semua akses ke property ini
    // otomatis thread-safe. Tidak butuh lock atau DispatchQueue.
    private var isLoading = false
    private var cachedUsers: [User] = []

    // UserListDataStore conformance — digunakan Router untuk mengambil
    // data user yang dipilih saat navigasi.
    // 'nonisolated' karena Router (@MainActor) membacanya tanpa await.
    // TAPI: nonisolated + var mutable adalah berbahaya.
    // Kita gunakan nonisolated(unsafe) karena Router hanya membaca
    // setelah Interactor sudah selesai menulis (terurut oleh VIP flow).
    nonisolated(unsafe) private(set) var selectedUser: User?

    init(worker: UserWorker = UserWorker()) {
        self.worker = worker
    }

    // MARK: - UserListBusinessLogic

    func fetchUsers(request: UserList.FetchUsers.Request) async {
        // Guard: hindari multiple concurrent fetch.
        // Karena ini actor, pengecekan dan assignment ini atomic.
        guard !isLoading else { return }
        isLoading = true

        // Panggil Presenter untuk tampilkan loading indicator.
        // 'await' diperlukan karena presenter adalah @MainActor —
        // kita menyeberangi isolation boundary dari actor ke @MainActor.
        await presenter?.presentLoading(true)

        do {
            // Panggil Worker — ini juga actor, butuh await.
            // Thread actor ini dilepas selagi menunggu network.
            let users = try await worker.fetchUsers()

            // Setelah await, kita kembali ke isolation domain Interactor.
            // Update cache — aman karena kita satu-satunya yang bisa akses ini.
            cachedUsers = users

            // Kirim Response ke Presenter.
            // Response adalah Sendable struct — aman melewati isolation boundary.
            let response = UserList.FetchUsers.Response(users: users)

            // 'await' lagi karena presenter adalah @MainActor.
            await presenter?.presentUsers(response)

        } catch {
            await presenter?.presentError(error)
        }

        // Selalu matikan loading, baik sukses maupun error.
        isLoading = false
        await presenter?.presentLoading(false)
    }

    func selectUser(request: UserList.SelectUser.Request) async {
        // Validasi index — cachedUsers aman diakses di sini (dalam actor).
        guard request.index < cachedUsers.count else { return }

        let user = cachedUsers[request.index]

        // Set selectedUser untuk dibaca Router nanti.
        // nonisolated(unsafe) memungkinkan ini, tapi kita tahu Router
        // hanya akan membacanya setelah routing dipanggil (setelah ini).
        selectedUser = user

        let response = UserList.SelectUser.Response(selectedUser: user)
        await presenter?.presentSelectedUser(response)
    }
}
```

**Penjelasan poin penting:**

- **`guard !isLoading else { return }`** — karena ini `actor`, pengecekan `isLoading` dan assignment-nya adalah atomik. Tidak ada race condition. Dua `Task` yang memanggil `fetchUsers()` bersamaan dijamin salah satunya akan `return` di baris ini.
- **`await presenter?.presentLoading(true)`** — ini adalah momen penting: kita menyeberangi isolation boundary dari `actor` Interactor ke `@MainActor` Presenter. Swift menjadwalkan ini ke main thread secara otomatis.
- **`nonisolated(unsafe) var selectedUser`** — trade-off yang disengaja. Karena `UserListDataStore` perlu diakses secara `nonisolated` oleh Router, kita perlu `nonisolated(unsafe)`. Ini aman **hanya karena** kita tahu Router membaca nilai ini selalu setelah Interactor selesai menulisnya (urutan dijamin oleh VIP flow).

---

## 6. Presenter — @MainActor

```swift
import UIKit

// Presenter bertanggung jawab satu hal: format data untuk tampil di View.
// @MainActor karena:
// 1. Memanggil ViewController yang @MainActor
// 2. Method-nya dipanggil dari Interactor (actor) dengan await — Swift
//    otomatis menjadwalkan eksekusi ke main thread
// Tidak ada business logic di sini. Tidak ada network call. Hanya format string.
@MainActor
final class UserListPresenter: UserListPresentationLogic {

    // weak untuk mencegah retain cycle (Presenter → ViewController → Presenter).
    weak var viewController: (any UserListDisplayLogic)?

    // MARK: - UserListPresentationLogic

    // Method ini berjalan di @MainActor (main thread).
    // Ketika Interactor memanggil 'await presenter?.presentUsers(response)',
    // Swift menjadwalkan eksekusi method ini ke main thread secara otomatis.
    func presentUsers(_ response: UserList.FetchUsers.Response) {
        // map di sini adalah pure transformation — tidak ada side effect.
        // Ini adalah CPU work ringan, aman dilakukan di main thread.
        let displayedUsers = response.users.map { user in
            UserList.FetchUsers.ViewModel.DisplayedUser(
                id: user.id,
                // Presenter yang memutuskan format tampilan — bukan View, bukan Interactor.
                fullName: user.name
                    .split(separator: " ")
                    .map { $0.capitalized }
                    .joined(separator: " "),
                emailLabel: user.email.lowercased(),
                avatarURL: user.avatarURL
            )
        }

        let viewModel = UserList.FetchUsers.ViewModel(displayedUsers: displayedUsers)

        // Langsung panggil View — kita sudah di @MainActor, tidak butuh await.
        viewController?.displayUsers(viewModel)
    }

    func presentLoading(_ isLoading: Bool) {
        viewController?.displayLoading(isLoading)
    }

    func presentError(_ error: Error) {
        // Presenter memutuskan pesan error yang user-friendly.
        // LocalizedError sudah ditangani Worker — kita tinggal ambil deskripsinya.
        let message = error.localizedDescription
        viewController?.displayError(message: message)
    }

    func presentSelectedUser(_ response: UserList.SelectUser.Response) {
        let viewModel = UserList.SelectUser.ViewModel(
            userId: response.selectedUser.id,
            userName: response.selectedUser.name
        )
        viewController?.displaySelectedUser(viewModel)
    }
}
```

**Penjelasan poin penting:**

- **`@MainActor final class`** — `final` penting di sini karena `@MainActor` class yang di-subclass bisa menimbulkan masalah isolation inheritance. Selalu `final` untuk Presenter dan ViewController dalam VIP.
- **Format string ada di Presenter, bukan di View.** View hanya membaca `viewModel.fullName` — tidak perlu tahu cara formatnya. Ini memudahkan testing: kita bisa unit test Presenter tanpa UIKit sama sekali.
- **Tidak ada `DispatchQueue.main.async`** — `@MainActor` menggantikannya sepenuhnya dengan cara yang type-safe dan terverifikasi compiler.

---

## 7. ViewController — @MainActor

```swift
import UIKit

// ViewController hanya bertanggung jawab untuk:
// 1. Menampilkan data yang sudah diformat oleh Presenter
// 2. Mengirim user action ke Interactor
// Tidak ada business logic. Tidak ada format string. Tidak ada network call.
@MainActor
final class UserListViewController: UIViewController {

    // MARK: - VIP References

    // Interactor disimpan sebagai protocol — loose coupling.
    // 'any' keyword karena ini existential type.
    var interactor: (any UserListBusinessLogic)?
    var router: (any UserListRoutingLogic)?

    // MARK: - UI Components

    private lazy var tableView: UITableView = {
        let table = UITableView(frame: .zero, style: .plain)
        table.register(UserCell.self, forCellReuseIdentifier: UserCell.reuseID)
        table.translatesAutoresizingMaskIntoConstraints = false
        table.delegate = self
        table.dataSource = self
        return table
    }()

    private lazy var activityIndicator: UIActivityIndicatorView = {
        let indicator = UIActivityIndicatorView(style: .large)
        indicator.hidesWhenStopped = true
        indicator.translatesAutoresizingMaskIntoConstraints = false
        return indicator
    }()

    private lazy var refreshControl: UIRefreshControl = {
        let control = UIRefreshControl()
        // addTarget bisa pakai selector atau closure action
        control.addAction(
            UIAction { [weak self] _ in self?.handleRefresh() },
            for: .valueChanged
        )
        return control
    }()

    // State View — hanya diakses di @MainActor, selalu thread-safe.
    private var displayedUsers: [UserList.FetchUsers.ViewModel.DisplayedUser] = []

    // MARK: - Lifecycle

    override func viewDidLoad() {
        super.viewDidLoad()
        setupUI()
        fetchUsers()
    }

    // MARK: - Setup

    private func setupUI() {
        title = "Users"
        view.backgroundColor = .systemBackground
        tableView.refreshControl = refreshControl

        view.addSubview(tableView)
        view.addSubview(activityIndicator)

        NSLayoutConstraint.activate([
            tableView.topAnchor.constraint(equalTo: view.safeAreaLayoutGuide.topAnchor),
            tableView.leadingAnchor.constraint(equalTo: view.leadingAnchor),
            tableView.trailingAnchor.constraint(equalTo: view.trailingAnchor),
            tableView.bottomAnchor.constraint(equalTo: view.bottomAnchor),
            activityIndicator.centerXAnchor.constraint(equalTo: view.centerXAnchor),
            activityIndicator.centerYAnchor.constraint(equalTo: view.centerYAnchor),
        ])
    }

    // MARK: - Actions

    private func fetchUsers() {
        // Task mewarisi isolation dari konteks pembuatannya (@MainActor).
        // Tapi setelah 'await interactor?.fetchUsers', eksekusi berpindah ke Interactor (actor).
        // Setelah Interactor selesai dan memanggil Presenter, Presenter langsung
        // memanggil displayUsers() di sini — kembali ke @MainActor.
        Task {
            await interactor?.fetchUsers(request: UserList.FetchUsers.Request())
        }
    }

    private func handleRefresh() {
        Task {
            await interactor?.fetchUsers(request: UserList.FetchUsers.Request())
        }
    }
}

// MARK: - UserListDisplayLogic

// Implementasi protocol di extension untuk keterbacaan kode.
// Semua method ini @MainActor (diwarisi dari class annotation).
extension UserListViewController: UserListDisplayLogic {

    func displayUsers(_ viewModel: UserList.FetchUsers.ViewModel) {
        displayedUsers = viewModel.displayedUsers
        // tableView.reloadData() harus di main thread — terjamin karena @MainActor.
        tableView.reloadData()
        // Hentikan refresh control jika aktif (pull-to-refresh).
        refreshControl.endRefreshing()
    }

    func displayLoading(_ isLoading: Bool) {
        if isLoading {
            activityIndicator.startAnimating()
        } else {
            activityIndicator.stopAnimating()
        }
    }

    func displayError(message: String) {
        refreshControl.endRefreshing()

        let alert = UIAlertController(
            title: "Oops!",
            message: message,
            preferredStyle: .alert
        )
        alert.addAction(UIAlertAction(title: "Coba Lagi", style: .default) { [weak self] _ in
            self?.fetchUsers()
        })
        alert.addAction(UIAlertAction(title: "Tutup", style: .cancel))
        present(alert, animated: true)
    }

    func displaySelectedUser(_ viewModel: UserList.SelectUser.ViewModel) {
        // Delegasikan navigasi ke Router — View tidak tahu cara navigate.
        router?.routeToUserDetail(userId: viewModel.userId, userName: viewModel.userName)
    }
}

// MARK: - UITableViewDataSource

extension UserListViewController: UITableViewDataSource {

    func tableView(_ tableView: UITableView, numberOfRowsInSection section: Int) -> Int {
        displayedUsers.count
    }

    func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
        let cell = tableView.dequeueReusableCell(
            withIdentifier: UserCell.reuseID,
            for: indexPath
        ) as! UserCell
        cell.configure(with: displayedUsers[indexPath.row])
        return cell
    }
}

// MARK: - UITableViewDelegate

extension UserListViewController: UITableViewDelegate {

    func tableView(_ tableView: UITableView, didSelectRowAt indexPath: IndexPath) {
        tableView.deselectRow(at: indexPath, animated: true)

        // Kirim aksi user ke Interactor.
        // Task mewarisi @MainActor isolation, lalu berpindah ke Interactor (actor)
        // ketika await dipanggil.
        Task {
            await interactor?.selectUser(
                request: UserList.SelectUser.Request(index: indexPath.row)
            )
        }
    }
}

// MARK: - UserCell

final class UserCell: UITableViewCell {

    static let reuseID = "UserCell"

    private let nameLabel = UILabel()
    private let emailLabel = UILabel()
    private let avatarView = UIImageView()

    override init(style: UITableViewCell.CellStyle, reuseIdentifier: String?) {
        super.init(style: style, reuseIdentifier: reuseIdentifier)
        setupCell()
    }

    required init?(coder: NSCoder) { fatalError() }

    private func setupCell() {
        nameLabel.font = .systemFont(ofSize: 16, weight: .semibold)
        emailLabel.font = .systemFont(ofSize: 13)
        emailLabel.textColor = .secondaryLabel

        avatarView.layer.cornerRadius = 20
        avatarView.clipsToBounds = true
        avatarView.backgroundColor = .systemGray5
        avatarView.translatesAutoresizingMaskIntoConstraints = false

        let textStack = UIStackView(arrangedSubviews: [nameLabel, emailLabel])
        textStack.axis = .vertical
        textStack.spacing = 2
        textStack.translatesAutoresizingMaskIntoConstraints = false

        contentView.addSubview(avatarView)
        contentView.addSubview(textStack)

        NSLayoutConstraint.activate([
            avatarView.leadingAnchor.constraint(equalTo: contentView.leadingAnchor, constant: 16),
            avatarView.centerYAnchor.constraint(equalTo: contentView.centerYAnchor),
            avatarView.widthAnchor.constraint(equalToConstant: 40),
            avatarView.heightAnchor.constraint(equalToConstant: 40),

            textStack.leadingAnchor.constraint(equalTo: avatarView.trailingAnchor, constant: 12),
            textStack.trailingAnchor.constraint(equalTo: contentView.trailingAnchor, constant: -16),
            textStack.centerYAnchor.constraint(equalTo: contentView.centerYAnchor),
        ])
    }

    func configure(with user: UserList.FetchUsers.ViewModel.DisplayedUser) {
        nameLabel.text = user.fullName
        emailLabel.text = user.emailLabel
        // Load avatar dengan Task — aman karena cell sudah @MainActor context
        if let url = user.avatarURL {
            Task {
                await loadAvatar(from: url)
            }
        }
    }

    // Menggunakan nonisolated sendable func untuk load image di background
    private func loadAvatar(from url: URL) async {
        guard let (data, _) = try? await URLSession.shared.data(from: url),
              let image = UIImage(data: data) else { return }
        // Kembali ke @MainActor (context dari caller) untuk update UI
        avatarView.image = image
    }
}
```

**Penjelasan poin penting:**

- **`Task { await interactor?.... }`** — pattern ini adalah cara standar memanggil async method dari synchronous UIKit callback (`viewDidLoad`, `didSelectRowAt`). Task mewarisi `@MainActor` isolation dari ViewController, tapi setelah `await` melewati ke Interactor.
- **`displayedUsers` diakses tanpa lock** — karena ViewController adalah `@MainActor`, semua akses ke `displayedUsers` dijamin di satu thread. Tidak perlu `DispatchQueue` atau lock.
- **`loadAvatar` di dalam cell** — menggunakan `async` dengan `await URLSession` sehingga image download tidak memblokir UI. Swift menangani threading secara otomatis.

---

## 8. Router — @MainActor

```swift
import UIKit

// Router bertanggung jawab untuk navigasi — push, present, dismiss.
// @MainActor karena semua navigasi UIKit harus di main thread.
@MainActor
final class UserListRouter: UserListRoutingLogic {

    // weak reference ke ViewController sebagai source navigasi.
    weak var viewController: UserListViewController?

    // DataStore adalah Interactor itu sendiri — Router membaca data
    // dari sini untuk diteruskan ke destination ViewController.
    weak var dataStore: (any UserListDataStore)?

    // MARK: - UserListRoutingLogic

    func routeToUserDetail(userId: String, userName: String) {
        // Buat destination ViewController.
        // Di aplikasi nyata, ini mungkin diambil dari Storyboard atau factory.
        let detailVC = UserDetailViewController(userId: userId, userName: userName)

        // Navigasi UIKit — aman karena kita di @MainActor.
        viewController?.navigationController?.pushViewController(detailVC, animated: true)
    }
}

// MARK: - Placeholder destination

// Placeholder — di project nyata ini adalah scene VIP terpisah
final class UserDetailViewController: UIViewController {
    private let userId: String
    private let userName: String

    init(userId: String, userName: String) {
        self.userId = userId
        self.userName = userName
        super.init(nibName: nil, bundle: nil)
    }

    required init?(coder: NSCoder) { fatalError() }

    override func viewDidLoad() {
        super.viewDidLoad()
        title = userName
        view.backgroundColor = .systemBackground
    }
}
```

**Penjelasan poin penting:**

- **Router tidak tahu cara fetch data** — ia hanya mengambil `selectedUser` dari `dataStore` (Interactor) yang sudah diisi sebelumnya oleh Interactor di langkah `selectUser`.
- **`@MainActor` di Router** adalah keharusan — `pushViewController`, `present`, `dismiss` semuanya harus di main thread.

---

## 9. Configurator — Dependency Injection

```swift
import UIKit

// Configurator bertugas merakit semua komponen VIP dan menghubungkannya.
// Ini adalah satu-satunya tempat di mana semua dependensi dibuat dan disambungkan.
// Di aplikasi nyata, ini bisa digantikan oleh DI container.
@MainActor
enum UserListConfigurator {

    // Factory method yang mengembalikan ViewController yang sudah ter-wire penuh.
    static func makeViewController() -> UserListViewController {

        // 1. Buat semua komponen
        let viewController = UserListViewController()
        let interactor = UserListInteractor()
        let presenter = UserListPresenter()
        let router = UserListRouter()

        // 2. Sambungkan View → Interactor
        viewController.interactor = interactor

        // 3. Sambungkan View → Router
        viewController.router = router

        // 4. Sambungkan Interactor → Presenter
        // 'await' tidak diperlukan di sini — kita sedang setup, belum ada concurrent access.
        // Presenter assignment ke actor property dilakukan melalui isolasi actor Interactor.
        // Ini terjadi sebelum ViewController ditampilkan, jadi aman.
        Task {
            // Kita perlu masuk ke isolation domain Interactor untuk assign presenter-nya.
            await setPresenter(presenter, on: interactor)
        }

        // 5. Sambungkan Presenter → View
        presenter.viewController = viewController

        // 6. Sambungkan Router → View & DataStore
        router.viewController = viewController
        router.dataStore = interactor

        return viewController
    }

    // Helper untuk assign presenter ke Interactor dari luar actor.
    // Ini adalah cara yang benar untuk mengakses isolated property actor dari luar.
    private static func setPresenter(
        _ presenter: any UserListPresentationLogic,
        on interactor: UserListInteractor
    ) async {
        await interactor.setPresenter(presenter)
    }
}
```

Tambahkan method `setPresenter` ke `UserListInteractor`:

```swift
// Tambahkan ke extension UserListInteractor
extension UserListInteractor {
    // Method ini berjalan di dalam isolation domain actor — aman untuk assign.
    func setPresenter(_ presenter: any UserListPresentationLogic) {
        self.presenter = presenter
    }
}
```

**Penggunaan di AppDelegate / SceneDelegate:**

```swift
@MainActor
class SceneDelegate: UIResponder, UIWindowSceneDelegate {
    var window: UIWindow?

    func scene(
        _ scene: UIScene,
        willConnectTo session: UISceneSession,
        options connectionOptions: UIScene.ConnectionOptions
    ) {
        guard let windowScene = (scene as? UIWindowScene) else { return }

        // Buat ViewController melalui Configurator — semua wiring terjadi di sini.
        let userListVC = UserListConfigurator.makeViewController()
        let navController = UINavigationController(rootViewController: userListVC)

        window = UIWindow(windowScene: windowScene)
        window?.rootViewController = navController
        window?.makeKeyAndVisible()
    }
}
```

**Penjelasan poin penting:**

- **Configurator adalah satu-satunya tempat yang tahu tentang concrete type** — semua komponen lain hanya tahu tentang protocol. Ini memudahkan testing (tinggal ganti concrete type dengan mock).
- **`Task { await setPresenter(...) }`** — karena `presenter` adalah property actor (isolated), kita harus mengaksesnya dari dalam isolation domain actor. Cara yang bersih adalah via `async` method.

---

## 10. Alur Data Lengkap

### Skenario: User membuka app, fetch users berhasil

```
1. SceneDelegate.scene(_:willConnectTo:)
   └─ UserListConfigurator.makeViewController()
       └─ Semua komponen dibuat dan diwire

2. UserListViewController.viewDidLoad() [@MainActor]
   └─ fetchUsers()
       └─ Task { await interactor?.fetchUsers(request:) }
                                    │
              Isolation boundary ───┼──────────────────────────
              @MainActor → actor    │
                                    ▼
3. UserListInteractor.fetchUsers(request:) [actor]
   ├─ guard !isLoading           → cek atomic, aman
   ├─ isLoading = true           → set atomic, aman
   ├─ await presenter?.presentLoading(true)
   │            │
   │   boundary ┼── actor → @MainActor
   │            ▼
   │   UserListPresenter.presentLoading(true) [@MainActor]
   │   └─ viewController?.displayLoading(true)
   │       └─ activityIndicator.startAnimating()  ✅ main thread
   │
   ├─ let users = try await worker.fetchUsers()
   │            │
   │   boundary ┼── actor(Interactor) → actor(Worker)
   │            ▼
   │   UserWorker.fetchUsers() [actor]
   │   ├─ URLSession.data(for:)   → network I/O, thread dilepas
   │   ├─ JSONDecoder.decode(...)  → CPU work di background
   │   └─ return [User]           → [User] adalah Sendable ✅
   │            │
   │   kembali ke Interactor ─────┘
   │
   ├─ cachedUsers = users         → update state, aman (dalam actor)
   └─ await presenter?.presentUsers(response)
                │
       boundary ┼── actor → @MainActor
                ▼
4. UserListPresenter.presentUsers(_:) [@MainActor]
   ├─ map users → [DisplayedUser]  → format string, di main thread
   └─ viewController?.displayUsers(viewModel)
                │
                ▼
5. UserListViewController.displayUsers(_:) [@MainActor]
   ├─ displayedUsers = viewModel.displayedUsers
   └─ tableView.reloadData()        ✅ UIKit di main thread
```

### Skenario: User tap sebuah cell

```
1. UITableViewDelegate.didSelectRowAt [:] [@MainActor]
   └─ Task { await interactor?.selectUser(request:) }
                                │
              boundary ─────────┼── @MainActor → actor
                                ▼
2. UserListInteractor.selectUser(request:) [actor]
   ├─ let user = cachedUsers[index]  → aman (dalam actor)
   ├─ selectedUser = user            → simpan untuk Router
   └─ await presenter?.presentSelectedUser(response)
                │
       boundary ┼── actor → @MainActor
                ▼
3. UserListPresenter.presentSelectedUser(_:) [@MainActor]
   └─ viewController?.displaySelectedUser(viewModel)
                │
                ▼
4. UserListViewController.displaySelectedUser(_:) [@MainActor]
   └─ router?.routeToUserDetail(userId:, userName:)
                │
                ▼
5. UserListRouter.routeToUserDetail(:) [@MainActor]
   ├─ buat UserDetailViewController
   └─ navigationController?.pushViewController(...)  ✅ UIKit di main thread
```

---

---

## 11. Swift Concurrency: Task Cancellation

### Problem

Ketika user membuka layar lalu langsung menutupnya sebelum fetch selesai, request network tetap berjalan di background — membuang bandwidth dan CPU. Di UIKit dengan delegate/completion handler, kita harus menyimpan referensi ke request dan membatalkannya secara manual. Dengan Swift Concurrency, cancellation bersifat **cooperative** dan **terstruktur**.

### Konsep

- `Task` adalah unit kerja yang bisa di-cancel via `task.cancel()`
- Cancellation bersifat **cooperative**: fungsi async harus secara aktif mengecek pembatalan dengan `try Task.checkCancellation()` atau menunggu fungsi yang already-cancellable seperti `URLSession.data`
- `URLSession.data(for:)` sudah mendukung cancellation otomatis — ketika Task di-cancel, ia melempar `CancellationError`
- **Penting**: cancel tidak langsung menghentikan kode — ia hanya menandai Task sebagai "cancelled", dan kode harus kooperatif memeriksa tanda ini

### Implementasi di Worker

```swift
actor UserWorker {
    // ...existing code...

    func fetchUsers() async throws -> [User] {
        // Periksa sebelum mulai — jika Task sudah di-cancel bahkan sebelum request,
        // lempar CancellationError sekarang daripada memulai network request yang sia-sia.
        try Task.checkCancellation()

        let url = baseURL.appendingPathComponent("/users")
        let request = URLRequest(url: url)

        // URLSession.data sudah mendukung cancellation: jika Task di-cancel saat
        // menunggu response, ini otomatis melempar CancellationError.
        let (data, response) = try await session.data(for: request)

        // Setelah operasi berat (network), cek lagi — siapa tahu Task di-cancel
        // persis ketika response baru tiba. Tidak perlu memparse data yang tidak akan dipakai.
        try Task.checkCancellation()

        guard let httpResponse = response as? HTTPURLResponse,
              (200...299).contains(httpResponse.statusCode) else {
            throw WorkerError.invalidResponse
        }

        let dtos = try JSONDecoder().decode([UserDTO].self, from: data)
        return dtos.map { dto in
            User(
                id: String(dto.id),
                name: dto.name,
                email: dto.email,
                avatarURL: URL(string: "https://i.pravatar.cc/150?u=\(dto.id)")
            )
        }
    }
}
```

### Implementasi di Interactor

```swift
actor UserListInteractor: UserListBusinessLogic, UserListDataStore {
    // ...existing code...

    func fetchUsers(request: UserList.FetchUsers.Request) async {
        guard !isLoading else { return }
        isLoading = true
        await presenter?.presentLoading(true)

        do {
            let users = try await worker.fetchUsers()
            // Jika Task di-cancel selama worker.fetchUsers(), CancellationError
            // akan di-catch di sini — kita bisa handle dengan bersih.
            cachedUsers = users
            await presenter?.presentUsers(UserList.FetchUsers.Response(users: users))
        } catch is CancellationError {
            // Cancellation bukan error yang perlu ditampilkan ke user.
            // Cukup matikan loading — view sudah tidak relevan.
            // Tidak perlu log atau alert.
        } catch {
            await presenter?.presentError(error)
        }

        isLoading = false
        await presenter?.presentLoading(false)
    }
}
```

### Implementasi di ViewController

```swift
@MainActor
final class UserListViewController: UIViewController {
    var interactor: (any UserListBusinessLogic)?
    var router: (any UserListRoutingLogic)?

    // Simpan Task reference untuk bisa di-cancel sewaktu-waktu.
    // 'Task<Void, Never>' karena kita handle error di dalam Task body.
    private var fetchTask: Task<Void, Never>?

    override func viewDidLoad() {
        super.viewDidLoad()
        setupUI()
        fetchUsers()
    }

    override func viewWillDisappear(_ animated: Bool) {
        super.viewWillDisappear(animated)
        // Cancel fetch jika view sedang menghilang — misalnya user tap back.
        // Task yang sudah cancel akan membersihkan dirinya sendiri.
        fetchTask?.cancel()
    }

    private func fetchUsers() {
        // Cancel request sebelumnya sebelum membuat yang baru.
        // Ini penting untuk pull-to-refresh: mencegah dua fetch berjalan bersamaan
        // dan response yang lebih lama menimpa response yang lebih baru.
        fetchTask?.cancel()

        fetchTask = Task {
            await interactor?.fetchUsers(request: UserList.FetchUsers.Request())
        }
    }

    private func handleRefresh() {
        fetchTask?.cancel()
        fetchTask = Task {
            await interactor?.fetchUsers(request: UserList.FetchUsers.Request())
        }
    }
}
```

### Menggunakan `withTaskCancellationHandler` untuk Cleanup

Ketika ada resource eksternal (timer, koneksi) yang harus dibersihkan saat cancel:

```swift
actor UserWorker {

    // Contoh: fetch dengan explicit cleanup saat cancel.
    // withTaskCancellationHandler memungkinkan kita mendaftarkan callback
    // yang dipanggil SEGERA saat Task di-cancel — bahkan jika Task sedang suspended.
    func fetchUsersWithCleanup() async throws -> [User] {
        var requestReference: URLSessionDataTask? = nil

        return try await withTaskCancellationHandler {
            // Body: kode yang dijalankan normal
            try await withCheckedThrowingContinuation { continuation in
                let task = session.dataTask(with: URLRequest(url: baseURL)) { data, _, error in
                    if let error {
                        continuation.resume(throwing: error)
                    } else if let data {
                        // parse...
                        continuation.resume(returning: []) // simplified
                    }
                }
                requestReference = task
                task.resume()
            }
        } onCancel: {
            // Handler ini dipanggil SEGERA saat Task.cancel() dipanggil,
            // bahkan jika body sedang suspended menunggu network.
            // Gunakan ini untuk membatalkan resource yang tidak bisa di-cancel secara async.
            requestReference?.cancel()
        }
    }
}
```

**Kapan Pakai `withTaskCancellationHandler`:**
- Wrapping API lama yang punya cancel method sendiri (`URLSessionDataTask.cancel()`, `AVAudioPlayer.stop()`)
- Membersihkan subscription atau listener saat Task di-cancel
- Menutup koneksi database atau socket

---

## 12. Swift Concurrency: async let — Parallel Fetch

### Problem

Banyak layar butuh data dari beberapa sumber sekaligus. Cara naif adalah fetch satu per satu secara sequential — lambat dan tidak efisien padahal tidak ada dependensi antar request.

### Konsep

`async let` menjalankan beberapa operasi async **secara bersamaan** tanpa harus menunggu satu selesai sebelum memulai yang lain. Hasil baru diambil saat semua sudah selesai dengan `await`. Ini adalah **structured concurrency** — semua child task otomatis di-cancel jika salah satu throw.

### Scenario: Fetch Users + Stats Bersamaan

Tambahkan method baru ke Worker:

```swift
actor UserWorker {
    // ...existing methods...

    // Fetch statistik user (endpoint berbeda dari /users)
    func fetchUserStats() async throws -> UserStats {
        let url = baseURL.appendingPathComponent("/posts/count") // contoh endpoint
        let (data, _) = try await session.data(from: url)
        return try JSONDecoder().decode(UserStats.self, from: data)
    }

    // Fetch satu featured user (misalnya dari endpoint /users/featured)
    func fetchFeaturedUser() async throws -> User? {
        let url = baseURL.appendingPathComponent("/users/1") // simplified
        let (data, _) = try await session.data(from: url)
        let dto = try JSONDecoder().decode(UserDTO.self, from: data)
        return User(id: String(dto.id), name: dto.name, email: dto.email, avatarURL: nil)
    }
}

struct UserStats: Sendable, Decodable {
    let totalUsers: Int
    let activeUsers: Int
}
```

Update Response model untuk menyertakan stats:

```swift
enum UserList {
    enum FetchUsers {
        struct Request: Sendable {}

        struct Response: Sendable {
            let users: [User]
            let stats: UserStats?      // tambahan
            let featuredUser: User?    // tambahan
        }

        struct ViewModel: Sendable {
            struct DisplayedUser: Sendable {
                let id: String
                let fullName: String
                let emailLabel: String
                let avatarURL: URL?
            }
            let displayedUsers: [DisplayedUser]
            let statsLabel: String?    // "10 users aktif dari 50 total"
            let featuredUser: DisplayedUser?
        }
    }
}
```

Implementasi `async let` di Interactor:

```swift
actor UserListInteractor: UserListBusinessLogic, UserListDataStore {

    func fetchUsers(request: UserList.FetchUsers.Request) async {
        guard !isLoading else { return }
        isLoading = true
        await presenter?.presentLoading(true)

        do {
            // TANPA async let (sequential — lambat):
            // let users = try await worker.fetchUsers()        // tunggu selesai
            // let stats = try await worker.fetchUserStats()    // baru mulai ini
            // let featured = try await worker.fetchFeaturedUser() // baru mulai ini
            // Total waktu = A + B + C

            // DENGAN async let (parallel — cepat):
            // Ketiga request dimulai BERSAMAAN.
            // 'async let' adalah deklarasi: "mulai hitung ini sekarang,
            //  aku akan ambil hasilnya nanti saat butuh."
            async let users = worker.fetchUsers()
            async let stats = worker.fetchUserStats()
            async let featured = worker.fetchFeaturedUser()

            // 'await' baru terjadi di sini — menunggu SEMUA selesai.
            // Jika salah satu throw, yang lain otomatis di-cancel (structured concurrency).
            // Total waktu ≈ max(A, B, C) — biasanya jauh lebih cepat.
            let (fetchedUsers, fetchedStats, featuredUser) = try await (users, stats, featured)

            cachedUsers = fetchedUsers

            let response = UserList.FetchUsers.Response(
                users: fetchedUsers,
                stats: fetchedStats,
                featuredUser: featuredUser
            )
            await presenter?.presentUsers(response)

        } catch {
            await presenter?.presentError(error)
        }

        isLoading = false
        await presenter?.presentLoading(false)
    }
}
```

Update Presenter untuk format stats:

```swift
@MainActor
final class UserListPresenter: UserListPresentationLogic {

    func presentUsers(_ response: UserList.FetchUsers.Response) {
        let displayedUsers = response.users.map { user in
            UserList.FetchUsers.ViewModel.DisplayedUser(
                id: user.id,
                fullName: user.name.split(separator: " ").map { $0.capitalized }.joined(separator: " "),
                emailLabel: user.email.lowercased(),
                avatarURL: user.avatarURL
            )
        }

        // Format stats — logic ini ada di Presenter, bukan di View.
        let statsLabel: String? = response.stats.map { stats in
            "\(stats.activeUsers) user aktif dari \(stats.totalUsers) total"
        }

        let featuredUser = response.featuredUser.map { user in
            UserList.FetchUsers.ViewModel.DisplayedUser(
                id: user.id,
                fullName: user.name,
                emailLabel: user.email,
                avatarURL: user.avatarURL
            )
        }

        let viewModel = UserList.FetchUsers.ViewModel(
            displayedUsers: displayedUsers,
            statsLabel: statsLabel,
            featuredUser: featuredUser
        )
        viewController?.displayUsers(viewModel)
    }
    // ...other methods...
}
```

**Kapan Pakai `async let`:**
- Fetch 2–5 sumber data bersamaan yang hasilnya semua dibutuhkan
- Jumlah task sudah diketahui saat compile-time
- Jika salah satu gagal, tidak perlu melanjutkan yang lain

**Kapan Tidak Pakai `async let`:**
- Jumlah task dinamis (tidak diketahui saat compile-time) → gunakan `TaskGroup`
- Ingin partial result (tampilkan data yang sudah selesai tanpa menunggu semua) → gunakan `TaskGroup` dengan `for await`

---

## 13. Swift Concurrency: TaskGroup — Bulk Avatar Prefetch

### Problem

Setelah user list tampil, kita ingin pre-load semua avatar agar cell langsung menampilkan gambar tanpa delay. Jumlah user tidak diketahui saat compile-time — bisa 5, bisa 100. `async let` tidak bisa menangani jumlah dinamis.

### Konsep

`withTaskGroup` / `withThrowingTaskGroup` menjalankan sejumlah task secara concurrent dengan jumlah yang ditentukan saat runtime. Task ditambahkan dengan `group.addTask {}` dan hasilnya dikumpulkan dengan `for await`. Ada juga **concurrency limiting** untuk mencegah terlalu banyak request sekaligus.

### Implementasi AvatarCache

```swift
// Cache untuk menyimpan data avatar agar tidak di-download ulang.
// Dijadikan actor untuk thread-safe read/write dari banyak concurrent task.
actor AvatarCache {
    static let shared = AvatarCache()
    private init() {}

    private var cache: [String: Data] = [:]

    func data(for userID: String) -> Data? {
        cache[userID]
    }

    func store(_ data: Data, for userID: String) {
        cache[userID] = data
    }

    func clear() {
        cache.removeAll()
    }
}
```

### Implementasi TaskGroup di Worker

```swift
actor UserWorker {
    // ...existing methods...

    // Prefetch avatar untuk semua user secara concurrent tapi terbatas.
    // 'maxConcurrent' mencegah kita membuka terlalu banyak koneksi sekaligus
    // yang bisa menyebabkan throttling dari server atau exhausting thread pool.
    func prefetchAvatars(
        for users: [User],
        maxConcurrent: Int = 5
    ) async -> [String: Data] {

        // Kita pakai withTaskGroup (bukan throwing) karena avatar adalah
        // operasi best-effort — jika satu gagal, lanjutkan yang lain.
        await withTaskGroup(of: (String, Data?).self) { group in
            var results: [String: Data] = [:]

            // Chunking manual untuk membatasi concurrency.
            // Tanpa ini, jika ada 100 user, kita membuka 100 koneksi sekaligus.
            let chunks = users.chunked(into: maxConcurrent)

            for chunk in chunks {
                // Mulai fetch untuk satu chunk — max 'maxConcurrent' task bersamaan.
                for user in chunk {
                    guard let url = user.avatarURL else { continue }
                    group.addTask {
                        // Cek cache dulu sebelum download.
                        if let cached = await AvatarCache.shared.data(for: user.id) {
                            return (user.id, cached)
                        }
                        let data = try? await URLSession.shared.data(from: url).0
                        return (user.id, data)
                    }
                }

                // Kumpulkan hasil chunk ini sebelum memulai chunk berikutnya.
                // Ini adalah cara manual membatasi concurrency — tunggu satu batch
                // selesai sebelum mulai batch berikutnya.
                for await (id, data) in group {
                    if let data {
                        results[id] = data
                        // Simpan ke cache untuk request berikutnya.
                        await AvatarCache.shared.store(data, for: id)
                    }
                }
            }

            return results
        }
    }
}

// Helper extension untuk chunking array
private extension Array {
    func chunked(into size: Int) -> [[Element]] {
        stride(from: 0, to: count, by: size).map {
            Array(self[$0..<Swift.min($0 + size, count)])
        }
    }
}
```

### Alternatif: TaskGroup dengan Partial Results

Scenario: tampilkan user segera, lalu update cell satu per satu ketika avatar-nya siap.

```swift
actor UserListInteractor: UserListBusinessLogic, UserListDataStore {
    private let avatarCache = AvatarCache.shared

    // Trigger prefetch di background — tidak blocking, tidak await.
    // Interactor mendelegasikan ke worker dan hasilnya di-update ke presenter
    // satu per satu ketika tiap avatar selesai diload.
    func prefetchAvatars(for users: [User]) {
        // Task.detached karena kita tidak ingin propagate cancellation dari caller.
        // Prefetch adalah operasi opsional — jika view hilang, tetap simpan ke cache
        // agar berguna jika user buka layar ini lagi.
        Task.detached(priority: .utility) { [weak self] in
            guard let self else { return }

            await withTaskGroup(of: (String, Data?).self) { group in
                for user in users {
                    guard let url = user.avatarURL else { continue }
                    group.addTask {
                        let data = try? await URLSession.shared.data(from: url).0
                        return (user.id, data)
                    }
                }

                // for await di sini memberikan partial results —
                // setiap avatar yang selesai langsung dikirim ke presenter.
                for await (id, data) in group {
                    guard let data else { continue }
                    await AvatarCache.shared.store(data, for: id)
                    // Notify presenter bahwa avatar untuk user ini sudah siap.
                    // Presenter bisa update cell individual tanpa reload seluruh table.
                    await self.presenter?.presentAvatarReady(userID: id, data: data)
                }
            }
        }
    }
}
```

**Kapan Pakai `TaskGroup`:**
- Jumlah task dinamis (tidak diketahui compile-time)
- Ingin partial results (tampilkan hasil begitu tersedia, tidak perlu semua selesai)
- Operasi bulk yang perlu dibatasi concurrency-nya

**Kapan Tidak Pakai `TaskGroup`:**
- Jumlah task kecil dan tetap (2-5) → `async let` lebih ringkas
- Semua task harus sukses agar hasilnya berguna → `async let` dengan throwing semantics lebih jelas

---

## 14. Swift Concurrency: AsyncStream — Live Polling

### Problem

Beberapa layar membutuhkan data yang terus-menerus diperbarui — polling setiap N detik, atau menerima push dari server. Dengan callback/timer, kode menjadi tersebar dan sulit di-cancel dengan bersih. `AsyncStream` memberikan model **push-based** yang bisa di-`for await`.

### Konsep

`AsyncStream<Element>` adalah sequence async yang menghasilkan nilai satu per satu via `continuation.yield()`. Producer (yang menghasilkan data) dan consumer (yang mengkonsumsi) berjalan secara independen — producer mengirim kapan pun siap, consumer menerima saat ada nilai baru.

### Implementasi Live Polling di Worker

```swift
actor UserWorker {
    // ...existing methods...

    // Menghasilkan [User] secara berkala — setiap 'interval' sekali.
    // Caller cukup `for await users in worker.liveUserStream()` — polling
    // ditangani sepenuhnya di sini, termasuk cleanup saat Task di-cancel.
    func liveUserStream(interval: Duration = .seconds(30)) -> AsyncStream<[User]> {
        AsyncStream { continuation in
            let task = Task {
                // Fetch pertama segera — jangan tunggu interval pertama.
                if let users = try? await fetchUsers() {
                    continuation.yield(users)
                }

                // Loop sampai Task di-cancel.
                while !Task.isCancelled {
                    // Tidur dulu sebelum fetch berikutnya.
                    // Task.sleep throws CancellationError jika Task di-cancel saat sleeping.
                    try? await Task.sleep(for: interval)

                    // Cek lagi setelah sleep — siapa tahu di-cancel persis saat tidur.
                    guard !Task.isCancelled else { break }

                    if let users = try? await fetchUsers() {
                        continuation.yield(users)
                    }
                }

                // Sinyal bahwa stream sudah selesai.
                continuation.finish()
            }

            // Ketika consumer tidak lagi mengkonsumsi stream (mis. view dismissed),
            // onTermination dipanggil otomatis — kita cancel polling task.
            continuation.onTermination = { _ in
                task.cancel()
            }
        }
    }
}
```

### Implementasi di Interactor

```swift
actor UserListInteractor: UserListBusinessLogic, UserListDataStore {
    // ...existing properties...

    // Simpan task polling agar bisa di-cancel dari luar.
    private var liveUpdateTask: Task<Void, Never>?

    func startLiveUpdates() {
        // Cegah multiple polling task berjalan bersamaan.
        liveUpdateTask?.cancel()

        liveUpdateTask = Task {
            // AsyncStream dari Worker — setiap 30 detik ada nilai baru.
            // 'for await' di sini adalah loop yang terus berjalan
            // sampai stream selesai atau Task di-cancel.
            for await users in worker.liveUserStream(interval: .seconds(30)) {
                // Guard penting: cek cancellation di setiap iterasi.
                // Task.isCancelled bisa true di antara dua iterasi.
                guard !Task.isCancelled else { break }

                cachedUsers = users
                let response = UserList.FetchUsers.Response(users: users, stats: nil, featuredUser: nil)
                await presenter?.presentUsers(response)
            }
        }
    }

    func stopLiveUpdates() {
        // Cancel task polling → onTermination dipanggil → inner Task di Worker di-cancel
        // → loop di Worker berhenti → stream selesai → for await di Interactor berhenti.
        // Ini adalah chain cancellation yang bersih dan terstruktur.
        liveUpdateTask?.cancel()
        liveUpdateTask = nil
    }
}
```

### Implementasi di ViewController

```swift
@MainActor
final class UserListViewController: UIViewController {
    var interactor: (any UserListBusinessLogic)?
    var router: (any UserListRoutingLogic)?

    override func viewDidAppear(_ animated: Bool) {
        super.viewDidAppear(animated)
        // Mulai live polling saat layar terlihat.
        Task { await interactor?.startLiveUpdates() }
    }

    override func viewDidDisappear(_ animated: Bool) {
        super.viewDidDisappear(animated)
        // Hentikan polling saat layar tidak terlihat — hemat baterai dan bandwidth.
        Task { await interactor?.stopLiveUpdates() }
    }
}
```

### Tambahkan ke Protocol

```swift
protocol UserListBusinessLogic: AnyObject, Sendable {
    func fetchUsers(request: UserList.FetchUsers.Request) async
    func selectUser(request: UserList.SelectUser.Request) async
    func startLiveUpdates() async    // tambahan
    func stopLiveUpdates() async     // tambahan
}
```

### Variasi: AsyncStream dari Notification

Scenario: refresh list ketika app kembali dari background.

```swift
extension NotificationCenter {
    // Wrap NotificationCenter menjadi AsyncStream — tidak perlu addObserver/removeObserver manual.
    func notifications(named name: Notification.Name) -> AsyncStream<Notification> {
        AsyncStream { continuation in
            let observer = addObserver(forName: name, object: nil, queue: nil) { notification in
                continuation.yield(notification)
            }
            continuation.onTermination = { [weak self] _ in
                self?.removeObserver(observer)
            }
        }
    }
}

// Penggunaan di Interactor:
func observeAppForeground() {
    Task {
        for await _ in NotificationCenter.default.notifications(
            named: UIApplication.willEnterForegroundNotification
        ) {
            // Refresh saat app kembali foreground
            guard !Task.isCancelled else { break }
            await fetchUsers(request: UserList.FetchUsers.Request())
        }
    }
}
```

**Kapan Pakai `AsyncStream`:**
- Polling berkala yang harus bisa di-cancel dengan bersih
- Mengubah event-based API (timer, NotificationCenter, delegate) menjadi async sequence
- Real-time updates yang datang secara push (WebSocket, server-sent events)

**Kapan Tidak Pakai `AsyncStream`:**
- Operasi satu kali (sekali fetch, satu response) → gunakan `async/await` biasa
- Sudah punya `Combine` publisher yang reliable di codebase → pertimbangkan konsistensi

---

## 15. Swift Concurrency: withCheckedContinuation — Bridge Delegate

### Problem

UIKit sangat bergantung pada pola delegate dan completion handler. Tidak semua API iOS sudah mendukung `async/await`. Kita perlu cara untuk menggunakan API lama ini dari dalam `async` function tanpa `DispatchQueue.main.async` atau nested callback.

### Konsep

`withCheckedContinuation` dan `withCheckedThrowingContinuation` mengubah **callback-based API** menjadi **async function** yang bisa di-`await`. Execution dihentikan sementara di suspension point, lalu dilanjutkan ketika `continuation.resume(...)` dipanggil di callback. "Checked" berarti Swift melakukan runtime check — pastikan `resume` dipanggil tepat satu kali.

### Scenario A: UIImagePickerController menjadi async

```swift
// Simpan continuation sebagai property — harus @unchecked Sendable karena
// UIImagePickerControllerDelegate callback datang dari main thread,
// dan continuation sendiri adalah Sendable.
@MainActor
final class UserListViewController: UIViewController {
    // ...existing code...

    // Menyimpan continuation di antara method pickImage() dan delegate callback.
    // CheckedContinuation tidak boleh diakses dari banyak thread, tapi karena
    // kita di @MainActor, ini aman.
    private var imageContinuation: CheckedContinuation<UIImage?, Never>?

    // API lama (UIImagePickerController) dibungkus menjadi async.
    // Caller cukup: let image = await pickAvatarImage()
    func pickAvatarImage() async -> UIImage? {
        // withCheckedContinuation menangguhkan eksekusi di sini.
        // Eksekusi dilanjutkan ketika imageContinuation.resume() dipanggil.
        await withCheckedContinuation { continuation in
            self.imageContinuation = continuation

            let picker = UIImagePickerController()
            picker.sourceType = .photoLibrary
            picker.delegate = self
            present(picker, animated: true)
        }
    }
}

// MARK: - UIImagePickerControllerDelegate

extension UserListViewController: UIImagePickerControllerDelegate, UINavigationControllerDelegate {

    func imagePickerController(
        _ picker: UIImagePickerController,
        didFinishPickingMediaWithInfo info: [UIImagePickerController.InfoKey: Any]
    ) {
        let image = info[.originalImage] as? UIImage
        dismiss(animated: true)

        // Resume continuation dengan hasil — eksekusi di 'await withCheckedContinuation'
        // dilanjutkan, dan 'image' menjadi return value dari pickAvatarImage().
        imageContinuation?.resume(returning: image)
        imageContinuation = nil    // WAJIB: nil-kan setelah resume untuk mencegah double-resume
    }

    func imagePickerControllerDidCancel(_ picker: UIImagePickerController) {
        dismiss(animated: true)

        // User cancel — resume dengan nil. Continuation HARUS di-resume
        // dalam semua code path, jika tidak caller akan hang selamanya.
        imageContinuation?.resume(returning: nil)
        imageContinuation = nil
    }
}
```

Penggunaan dari ViewController:

```swift
// Dari action button "Ganti Avatar" — ini async yang bersih, tidak ada callback nesting.
@objc private func handleChangeAvatar() {
    Task {
        guard let image = await pickAvatarImage() else { return }
        // Lanjutkan dengan image yang dipilih user
        await interactor?.uploadAvatar(image: image)
    }
}
```

### Scenario B: CLLocationManager menjadi async (throwing)

```swift
import CoreLocation

// Wrapper CLLocationManager yang mengubah delegate pattern menjadi async.
// Menggunakan 'withCheckedThrowingContinuation' karena lokasi bisa gagal (denied, timeout).
final class LocationService: NSObject, CLLocationManagerDelegate {

    private let manager = CLLocationManager()
    private var locationContinuation: CheckedContinuation<CLLocationCoordinate2D, Error>?

    func requestCurrentLocation() async throws -> CLLocationCoordinate2D {
        try await withCheckedThrowingContinuation { continuation in
            self.locationContinuation = continuation
            manager.delegate = self
            manager.requestWhenInUseAuthorization()
            manager.requestLocation()    // Single location request, calls delegate once
        }
    }

    // MARK: - CLLocationManagerDelegate

    func locationManager(_ manager: CLLocationManager, didUpdateLocations locations: [CLLocation]) {
        guard let location = locations.first else { return }
        locationContinuation?.resume(returning: location.coordinate)
        locationContinuation = nil
    }

    func locationManager(_ manager: CLLocationManager, didFailWithError error: Error) {
        // Resume dengan error — caller akan catch ini.
        locationContinuation?.resume(throwing: error)
        locationContinuation = nil
    }
}

// Penggunaan di Interactor:
actor UserListInteractor: UserListBusinessLogic, UserListDataStore {
    private let locationService = LocationService()

    func fetchNearbyUsers() async {
        do {
            let coordinate = try await locationService.requestCurrentLocation()
            let users = try await worker.fetchUsers(near: coordinate)
            cachedUsers = users
            await presenter?.presentUsers(UserList.FetchUsers.Response(users: users, stats: nil, featuredUser: nil))
        } catch {
            await presenter?.presentError(error)
        }
    }
}
```

### Scenario C: URLSession dengan Delegate menjadi async

```swift
// Ketika butuh upload progress — URLSession.upload(for:from:) tidak memberikan progress.
// Kita harus pakai delegate, lalu bridge ke async.
actor UploadService: NSObject, URLSessionTaskDelegate {

    private var progressContinuation: AsyncStream<Double>.Continuation?

    func uploadWithProgress(data: Data, to url: URL) -> (
        progress: AsyncStream<Double>,
        result: Task<Data, Error>
    ) {
        let (stream, continuation) = AsyncStream.makeStream(of: Double.self)
        self.progressContinuation = continuation

        let task = Task {
            try await withCheckedThrowingContinuation { (cont: CheckedContinuation<Data, Error>) in
                var request = URLRequest(url: url)
                request.httpMethod = "POST"
                request.httpBody = data

                let session = URLSession(configuration: .default, delegate: self, delegateQueue: nil)
                session.uploadTask(with: request, from: data) { data, _, error in
                    if let error {
                        cont.resume(throwing: error)
                    } else {
                        cont.resume(returning: data ?? Data())
                    }
                    continuation.finish()
                }.resume()
            }
        }

        return (progress: stream, result: task)
    }

    // Delegate untuk progress — dipanggil oleh URLSession
    nonisolated func urlSession(
        _ session: URLSession,
        task: URLSessionTask,
        didSendBodyData bytesSent: Int64,
        totalBytesSent: Int64,
        totalBytesExpectedToSend: Int64
    ) {
        let progress = Double(totalBytesSent) / Double(totalBytesExpectedToSend)
        Task { await updateProgress(progress) }
    }

    private func updateProgress(_ progress: Double) {
        progressContinuation?.yield(progress)
    }
}

// Penggunaan:
func uploadAvatar() async {
    guard let imageData = UIImage(named: "avatar")?.jpegData(compressionQuality: 0.8) else { return }
    let uploadService = UploadService()
    let url = URL(string: "https://api.example.com/avatar")!

    let (progressStream, resultTask) = uploadService.uploadWithProgress(data: imageData, to: url)

    // Pantau progress di Task terpisah
    Task {
        for await progress in progressStream {
            await MainActor.run {
                print("Upload progress: \(Int(progress * 100))%")
                // Update progress bar di UI
            }
        }
    }

    // Tunggu hasil upload
    do {
        let responseData = try await resultTask.value
        print("Upload berhasil: \(responseData.count) bytes response")
    } catch {
        print("Upload gagal: \(error)")
    }
}
```

**Aturan `withCheckedContinuation`:**

| Aturan | Penjelasan |
|---|---|
| `resume` dipanggil tepat satu kali | Kurang dari sekali → caller hang selamanya. Lebih dari sekali → crash (checked) |
| Nil-kan setelah `resume` | Mencegah accidental double-resume di edge case |
| Gunakan `withCheckedThrowingContinuation` jika bisa error | Memberikan error propagation yang bersih ke caller |
| Jangan simpan continuation terlalu lama | Continuation menyimpan seluruh call stack — memory leak jika tidak pernah di-resume |
| Gunakan `AsyncStream` jika callback bisa dipanggil berkali-kali | Continuation hanya untuk callback yang dipanggil tepat satu kali |

**Kapan Pakai `withCheckedContinuation`:**
- Menggunakan UIKit API yang masih callback-based (`UIImagePickerController`, `PHPickerViewController`)
- Wrapping delegate API yang fire tepat satu kali (CLLocationManager single request, AVAudioPlayer)
- Mengintegrasikan library third-party yang belum mendukung async/await

**Kapan Tidak Pakai:**
- Callback bisa dipanggil berkali-kali → gunakan `AsyncStream`
- API sudah punya versi async (`URLSession.data(for:)`, `PHPickerViewController` dengan async) → gunakan langsung
- Untuk Combine publisher → gunakan `AsyncPublisher` atau `values` property

---

## 16. Observable dalam VIP UIKit — Motivasi & Arsitektur

### Masalah yang Ingin Dipecahkan

Pada VIP murni (section 1–10), Presenter berkomunikasi ke ViewController melalui **protocol delegate**:

```swift
// Pure VIP — Presenter memanggil ViewController secara langsung
@MainActor
final class UserListPresenter: UserListPresentationLogic {
    weak var viewController: (any UserListDisplayLogic)?  // ← weak untuk cegah retain cycle

    func presentUsers(_ response: UserList.FetchUsers.Response) {
        // format...
        viewController?.displayUsers(viewModel)  // ← panggil method VC langsung
    }
}
```

Pendekatan ini solid, tapi ada beberapa friction:

1. **Retain cycle** — Presenter harus menyimpan `weak var viewController`. Jika lupa `weak`, terjadi retain cycle. Kalau `weak`, bisa jadi `nil` di waktu yang tidak terduga.
2. **Coupling implisit** — Presenter harus tahu _kapan_ memanggil method mana. Urutan pemanggilan penting: `presentLoading(true)` harus sebelum `presentUsers`, harus sebelum `presentLoading(false)`. Tidak ada enforcement dari compiler.
3. **Testing Presenter** membutuhkan `MockViewController` — kita perlu membuat spy/mock yang merekam method call, padahal yang sebenarnya ingin kita uji adalah: "apakah data diformat dengan benar?"

### Ide: Ganti Protocol Delegate dengan @Observable DisplayState

Alih-alih Presenter _memanggil_ ViewController, kita ubah menjadi: Presenter _memperbarui state_, ViewController _mengobservasi state_.

```
Pure VIP:
  Interactor ──await──▶ Presenter ──memanggil──▶ ViewController
                        (push model: Presenter aktif memutuskan kapan VC di-update)

Observable VIP:
  Interactor ──await──▶ Presenter ──update──▶ DisplayState ◀──observe── ViewController
                        (state model: VC reaktif, otomatis update saat state berubah)
```

### Arsitektur Baru: DisplayState Pattern

```
┌─────────────────────────────────────────────────────────────────────┐
│                          @MainActor                                 │
│  ┌──────────────────┐          ┌──────────────────────────────────┐ │
│  │  ViewController  │─────────▶│           Router                 │ │
│  │  (UI only)       │          │  (navigasi UIKit)                │ │
│  └────────┬─────────┘          └──────────────────────────────────┘ │
│           │ observes (withObservationTracking)                       │
│           ▼                                                          │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │             UserListDisplayState  (@Observable)              │   │
│  │                                                              │   │
│  │  var displayedUsers: [DisplayedUser]                         │   │
│  │  var isLoading: Bool                                         │   │
│  │  var pendingError: String?       ← one-time event            │   │
│  │  var pendingNavigation: ViewModel? ← one-time event          │   │
│  └───────────────────▲──────────────────────────────────────────┘   │
│                      │ update properties                            │
│  ┌───────────────────┴──────────────────────────────────────────┐   │
│  │             Presenter  (@MainActor)                          │   │
│  │  let displayState = UserListDisplayState()                   │   │
│  │  ← TIDAK ada weak var viewController lagi                    │   │
│  └──────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
            ▲ await presenter.presentUsers(response)
            │
┌───────────┴──────────────────────────────────────────────────────┐
│                    actor  Interactor                              │
│  (business logic, tidak berubah dari pure VIP)                   │
└──────────────────────────────────────────────────────────────────┘
```

### Perubahan Per Komponen

| Komponen | Pure VIP | Observable VIP |
|---|---|---|
| **Presenter** | `weak var viewController?`, panggil method langsung | `let displayState`, update property |
| **ViewController** | Implement `UserListDisplayLogic` protocol | Observe `displayState` via `withObservationTracking` |
| **Protocol** | `UserListDisplayLogic` penuh | `UserListDisplayLogic` bisa dihapus atau disederhanakan |
| **Configurator** | `presenter.viewController = viewController` | `viewController.configure(displayState: presenter.displayState)` |
| **Retain cycle** | Perlu `weak` | Tidak ada — VC tidak di-hold oleh Presenter |
| **iOS minimum** | iOS 15 (async/await) | iOS 17 (`@Observable`) |

---

## 17. Implementasi Observable VIP — DisplayState Pattern

### Step 1: UserListDisplayState — Jembatan antara Presenter dan View

```swift
import Observation

// DisplayState adalah satu-satunya sumber kebenaran untuk tampilan layar ini.
// @Observable: setiap perubahan property otomatis dideteksi oleh observer.
// @MainActor: selalu diakses dari main thread — konsisten dengan UIKit.
@Observable
@MainActor
final class UserListDisplayState {

    // State reguler — berubah berkali-kali sepanjang lifecycle layar
    var displayedUsers: [UserList.FetchUsers.ViewModel.DisplayedUser] = []
    var isLoading: Bool = false

    // "Pending" events — one-time actions yang harus dikonsumsi ViewController
    // dan di-reset segera setelah ditangani.
    //
    // Mengapa "pending" bukan state biasa?
    // Error dan navigasi adalah PERINTAH (commands), bukan state.
    // Jika kita simpan sebagai state permanen, error akan ditampilkan ulang
    // setiap kali applyState() terpanggil akibat perubahan property lain.
    var pendingError: String? = nil
    var pendingNavigation: UserList.SelectUser.ViewModel? = nil
}
```

### Step 2: Protocol (Disederhanakan)

```swift
// UserListDisplayLogic tidak lagi dibutuhkan — Presenter tidak memanggil VC.
// Kita bisa hapus protocol ini, atau pertahankan hanya jika butuh interoperabilitas
// dengan Pure VIP di modul lain.
//
// Yang tetap dibutuhkan:
protocol UserListBusinessLogic: AnyObject, Sendable {
    func fetchUsers(request: UserList.FetchUsers.Request) async
    func selectUser(request: UserList.SelectUser.Request) async
}

protocol UserListPresentationLogic: AnyObject {
    func presentUsers(_ response: UserList.FetchUsers.Response)
    func presentLoading(_ isLoading: Bool)
    func presentError(_ error: Error)
    func presentSelectedUser(_ response: UserList.SelectUser.Response)
}

// UserListDataStore dan UserListRoutingLogic tidak berubah dari Pure VIP.
```

### Step 3: Presenter — Update State, Bukan Panggil VC

```swift
import UIKit

// Presenter tidak lagi tahu keberadaan ViewController.
// Tugasnya sama: format Response → ViewModel.
// Bedanya: hasil format disimpan ke displayState, bukan dikirim ke VC.
@MainActor
final class UserListPresenter: UserListPresentationLogic {

    // DisplayState dimiliki Presenter — Presenter yang menciptakan dan mengupdate-nya.
    // ViewController mendapat referensi ke object ini via Configurator.
    // Tidak ada lagi 'weak var viewController' — tidak ada retain cycle.
    let displayState = UserListDisplayState()

    // MARK: - UserListPresentationLogic

    func presentUsers(_ response: UserList.FetchUsers.Response) {
        // Formatting logic sama persis dengan Pure VIP — tidak ada yang berubah.
        let displayedUsers = response.users.map { user in
            UserList.FetchUsers.ViewModel.DisplayedUser(
                id: user.id,
                fullName: user.name
                    .split(separator: " ")
                    .map { $0.capitalized }
                    .joined(separator: " "),
                emailLabel: user.email.lowercased(),
                avatarURL: user.avatarURL
            )
        }
        // Alih-alih: viewController?.displayUsers(viewModel)
        // Kita update state — observer akan merespons secara otomatis.
        displayState.displayedUsers = displayedUsers
    }

    func presentLoading(_ isLoading: Bool) {
        displayState.isLoading = isLoading
    }

    func presentError(_ error: Error) {
        // Set pending error — ViewController akan menampilkan dan langsung me-reset.
        // Jika sudah ada pendingError yang belum dikonsumsi, timpa saja —
        // error terbaru lebih relevan.
        displayState.pendingError = error.localizedDescription
    }

    func presentSelectedUser(_ response: UserList.SelectUser.Response) {
        displayState.pendingNavigation = UserList.SelectUser.ViewModel(
            userId: response.selectedUser.id,
            userName: response.selectedUser.name
        )
    }
}
```

### Step 4: ViewController — Observe DisplayState

```swift
import UIKit
import Observation

// ViewController tidak lagi mengimplementasikan UserListDisplayLogic.
// Sebagai gantinya, ia mengobservasi displayState dan merespons perubahan.
@MainActor
final class UserListViewController: UIViewController {

    // MARK: - VIP References

    var interactor: (any UserListBusinessLogic)?
    var router: (any UserListRoutingLogic)?

    // DisplayState datang dari Configurator — dimiliki Presenter, dibaca VC.
    // Private(set) agar hanya Configurator yang bisa inject via configure().
    private var displayState: UserListDisplayState?

    // Task untuk observasi — di-cancel saat view hilang.
    private var observationTask: Task<Void, Never>?

    // Cache lokal untuk diffing (mencegah reloadData yang tidak perlu)
    private var currentDisplayedUsers: [UserList.FetchUsers.ViewModel.DisplayedUser] = []

    // MARK: - UI Components (sama seperti Pure VIP)

    private lazy var tableView: UITableView = {
        let table = UITableView(frame: .zero, style: .plain)
        table.register(UserCell.self, forCellReuseIdentifier: UserCell.reuseID)
        table.translatesAutoresizingMaskIntoConstraints = false
        table.delegate = self
        table.dataSource = self
        return table
    }()

    private lazy var activityIndicator: UIActivityIndicatorView = {
        let indicator = UIActivityIndicatorView(style: .large)
        indicator.hidesWhenStopped = true
        indicator.translatesAutoresizingMaskIntoConstraints = false
        return indicator
    }()

    private lazy var refreshControl: UIRefreshControl = {
        let control = UIRefreshControl()
        control.addAction(UIAction { [weak self] _ in self?.handleRefresh() }, for: .valueChanged)
        return control
    }()

    // MARK: - Configuration (dipanggil oleh Configurator)

    // Inject displayState dari luar — VC tidak pernah membuat displayState sendiri.
    func configure(displayState: UserListDisplayState) {
        self.displayState = displayState
    }

    // MARK: - Lifecycle

    override func viewDidLoad() {
        super.viewDidLoad()
        setupUI()
        startObserving()   // mulai observe SEBELUM fetch — jangan sampai ada state update yang terlewat
        fetchUsers()
    }

    override func viewWillDisappear(_ animated: Bool) {
        super.viewWillDisappear(animated)
        observationTask?.cancel()
    }

    override func viewWillAppear(_ animated: Bool) {
        super.viewWillAppear(animated)
        // Restart observasi jika sebelumnya di-cancel saat viewWillDisappear
        if observationTask == nil || observationTask?.isCancelled == true {
            startObserving()
        }
    }

    // MARK: - Observation

    private func startObserving() {
        guard let displayState else { return }

        observationTask = Task { @MainActor [weak self] in
            // Bungkus withObservationTracking dalam AsyncStream untuk continuous tracking.
            // withObservationTracking sendiri adalah "fire-once" — kita perlu re-register
            // setiap kali onChange dipanggil.
            let changes = AsyncStream<Void> { continuation in
                func track() {
                    withObservationTracking {
                        // Akses semua property yang ingin di-track.
                        // SwiftUI melakukan ini secara otomatis di body,
                        // tapi di UIKit kita harus eksplisit.
                        _ = displayState.displayedUsers
                        _ = displayState.isLoading
                        _ = displayState.pendingError
                        _ = displayState.pendingNavigation
                    } onChange: {
                        // Dipanggil satu kali ketika SALAH SATU property berubah.
                        continuation.yield(())
                        // Re-register untuk menangkap perubahan berikutnya.
                        // Task untuk menjamin ini berjalan di @MainActor.
                        Task { @MainActor in track() }
                    }
                }
                track()  // Mulai tracking sekarang

                continuation.onTermination = { _ in }  // cleanup jika stream di-terminate
            }

            for await _ in changes {
                guard !Task.isCancelled else { break }
                self?.applyState()
            }
        }
    }

    // Dipanggil setiap kali ADA property di displayState yang berubah.
    // Fungsi ini idempoten — aman dipanggil berkali-kali dengan state yang sama.
    private func applyState() {
        guard let state = displayState else { return }

        // --- Regular state (idempoten) ---

        // Hanya reload jika ada perubahan nyata — mencegah scroll jump yang tidak perlu.
        if state.displayedUsers != currentDisplayedUsers {
            currentDisplayedUsers = state.displayedUsers
            tableView.reloadData()
        }

        if state.isLoading {
            activityIndicator.startAnimating()
        } else {
            activityIndicator.stopAnimating()
            refreshControl.endRefreshing()
        }

        // --- One-time events (consume and reset) ---
        // PENTING: reset SEBELUM aksi, bukan setelah.
        // Jika reset setelah aksi (misal setelah present alert), ada risiko applyState()
        // dipanggil lagi dari tempat lain sebelum reset selesai — double alert.

        if let message = state.pendingError {
            // Reset segera — mencegah error muncul dua kali jika applyState dipanggil ulang
            displayState?.pendingError = nil
            showErrorAlert(message: message)
        }

        if let nav = state.pendingNavigation {
            // Reset segera — mencegah double navigation
            displayState?.pendingNavigation = nil
            router?.routeToUserDetail(userId: nav.userId, userName: nav.userName)
        }
    }

    // MARK: - Actions

    private func fetchUsers() {
        Task { await interactor?.fetchUsers(request: UserList.FetchUsers.Request()) }
    }

    private func handleRefresh() {
        Task { await interactor?.fetchUsers(request: UserList.FetchUsers.Request()) }
    }

    private func showErrorAlert(message: String) {
        let alert = UIAlertController(title: "Oops!", message: message, preferredStyle: .alert)
        alert.addAction(UIAlertAction(title: "Coba Lagi", style: .default) { [weak self] _ in
            self?.fetchUsers()
        })
        alert.addAction(UIAlertAction(title: "Tutup", style: .cancel))
        present(alert, animated: true)
    }

    // MARK: - Setup

    private func setupUI() {
        title = "Users"
        view.backgroundColor = .systemBackground
        tableView.refreshControl = refreshControl
        view.addSubview(tableView)
        view.addSubview(activityIndicator)
        NSLayoutConstraint.activate([
            tableView.topAnchor.constraint(equalTo: view.safeAreaLayoutGuide.topAnchor),
            tableView.leadingAnchor.constraint(equalTo: view.leadingAnchor),
            tableView.trailingAnchor.constraint(equalTo: view.trailingAnchor),
            tableView.bottomAnchor.constraint(equalTo: view.bottomAnchor),
            activityIndicator.centerXAnchor.constraint(equalTo: view.centerXAnchor),
            activityIndicator.centerYAnchor.constraint(equalTo: view.centerYAnchor),
        ])
    }
}

// MARK: - UITableViewDataSource & Delegate

extension UserListViewController: UITableViewDataSource, UITableViewDelegate {

    func tableView(_ tableView: UITableView, numberOfRowsInSection section: Int) -> Int {
        currentDisplayedUsers.count
    }

    func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
        let cell = tableView.dequeueReusableCell(withIdentifier: UserCell.reuseID, for: indexPath) as! UserCell
        cell.configure(with: currentDisplayedUsers[indexPath.row])
        return cell
    }

    func tableView(_ tableView: UITableView, didSelectRowAt indexPath: IndexPath) {
        tableView.deselectRow(at: indexPath, animated: true)
        Task { await interactor?.selectUser(request: UserList.SelectUser.Request(index: indexPath.row)) }
    }
}
```

> **Mengapa `DisplayedUser` harus `Equatable`?**
> `applyState()` membandingkan `state.displayedUsers != currentDisplayedUsers` untuk menghindari `reloadData()` yang tidak perlu. Tambahkan `Equatable` ke `DisplayedUser`:
>
> ```swift
> struct DisplayedUser: Sendable, Equatable {
>     let id: String
>     let fullName: String
>     let emailLabel: String
>     let avatarURL: URL?
> }
> ```

### Step 5: Configurator — Wiring Tanpa Circular Reference

```swift
@MainActor
enum UserListConfigurator {

    static func makeViewController() -> UserListViewController {

        // 1. Buat semua komponen
        let viewController = UserListViewController()
        let interactor = UserListInteractor()
        let presenter = UserListPresenter()
        let router = UserListRouter()

        // 2. Inject displayState ke ViewController
        //    Presenter yang memiliki displayState — VC hanya membacanya.
        //    Tidak ada lagi: presenter.viewController = viewController
        //    Sehingga tidak ada circular reference sama sekali.
        viewController.configure(displayState: presenter.displayState)

        // 3. Wiring standar VIP
        viewController.interactor = interactor
        viewController.router = router

        // 4. Sambungkan Interactor → Presenter (lewat actor boundary)
        Task {
            await interactor.setPresenter(presenter)
        }

        // 5. Sambungkan Router
        router.viewController = viewController
        router.dataStore = interactor

        return viewController
    }
}
```

### Alur Data Lengkap (Observable VIP)

```
1. viewDidLoad → startObserving() → withObservationTracking terdaftar
   └─ fetchUsers() → Task { await interactor.fetchUsers() }

2. Interactor.fetchUsers() [actor]
   ├─ await presenter.presentLoading(true)
   │   └─ displayState.isLoading = true ←── withObservationTracking fires
   │       └─ applyState(): activityIndicator.startAnimating()
   │
   ├─ let users = try await worker.fetchUsers()
   │
   ├─ await presenter.presentUsers(response)
   │   └─ displayState.displayedUsers = [...] ←── fires
   │       └─ applyState(): tableView.reloadData()
   │
   └─ await presenter.presentLoading(false)
       └─ displayState.isLoading = false ←── fires
           └─ applyState(): activityIndicator.stopAnimating()

3. User tap cell → interactor.selectUser()
   └─ presenter.presentSelectedUser(response)
       └─ displayState.pendingNavigation = ViewModel(...) ←── fires
           └─ applyState():
               displayState.pendingNavigation = nil  (reset)
               router.routeToUserDetail(...)
```

---

## 18. Trade-offs: Pure VIP vs Observable VIP

Ini adalah inti dari pertanyaan yang sering muncul ketika mempertimbangkan migrasi. Tidak ada jawaban mutlak — setiap trade-off bergantung pada konteks project.

---

### Trade-off 1: Commands vs State — Masalah One-time Events

Ini adalah **trade-off terbesar** dan paling sering menjadi sumber bug tersembunyi.

**Pure VIP — Presenter berbicara dalam "commands":**

```swift
// Ini adalah PERINTAH — dipanggil sekali, dieksekusi sekali, selesai.
viewController?.displayError(message: "Koneksi gagal")
// Tidak ada state yang tersimpan — tidak mungkin error ini muncul lagi
// kecuali Presenter memanggil displayError() lagi.
```

**Observable VIP — Presenter berbicara dalam "state":**

```swift
// Ini adalah STATE — disimpan di displayState.pendingError
displayState.pendingError = "Koneksi gagal"

// Masalah: applyState() bisa dipanggil lagi oleh perubahan property LAIN.
// Misal: isLoading berubah → applyState() → pendingError masih ada → double alert!
//
// Solusi wajib: reset SEGERA setelah dikonsumsi.
if let message = state.pendingError {
    displayState?.pendingError = nil   // ← WAJIB, atau bug
    showErrorAlert(message: message)
}
```

**Mengapa ini berbahaya:** Programmer yang tidak familiar dengan pola ini akan lupa me-reset `pendingError`. Hasilnya: alert error muncul setiap kali ada perubahan state lain — bug yang sangat sulit di-reproduce karena hanya terjadi saat ada perubahan bersamaan.

**Solusi lebih defensif — EventQueue:**

```swift
// Alternatif: bungkus one-time event dalam wrapper yang auto-consume
@Observable
@MainActor
final class UserListDisplayState {
    var displayedUsers: [UserList.FetchUsers.ViewModel.DisplayedUser] = []
    var isLoading: Bool = false

    // Gunakan array sebagai queue — multiple events tidak saling menimpa
    private(set) var errorQueue: [String] = []
    private(set) var navigationQueue: [UserList.SelectUser.ViewModel] = []

    // Hanya Presenter yang boleh enqueue
    fileprivate func enqueueError(_ message: String) {
        errorQueue.append(message)
    }

    fileprivate func enqueueNavigation(_ viewModel: UserList.SelectUser.ViewModel) {
        navigationQueue.append(viewModel)
    }

    // ViewController yang dequeue saat mengkonsumsi
    func dequeueError() -> String? {
        guard !errorQueue.isEmpty else { return nil }
        return errorQueue.removeFirst()
    }

    func dequeueNavigation() -> UserList.SelectUser.ViewModel? {
        guard !navigationQueue.isEmpty else { return nil }
        return navigationQueue.removeFirst()
    }
}

// applyState() lebih aman:
private func applyState() {
    guard let state = displayState else { return }
    // ...regular state...

    // Queue — ambil semua yang pending
    while let message = state.dequeueError() {
        showErrorAlert(message: message)
    }
    while let nav = state.dequeueNavigation() {
        router?.routeToUserDetail(userId: nav.userId, userName: nav.userName)
    }
}
```

---

### Trade-off 2: Ownership & Retain Cycle

**Pure VIP:**

```
Configurator: presenter.viewController = viewController  (strong)
                        ↑
Agar tidak retain cycle, harus weak:
Presenter: weak var viewController: (any UserListDisplayLogic)?
                                    ↑
Jika lupa weak → retain cycle → memory leak yang sulit dideteksi
Jika weak → viewController bisa nil saat dipanggil → silent failure
```

**Observable VIP:**

```
Presenter: let displayState = UserListDisplayState()  (owns)
ViewController: var displayState: UserListDisplayState?  (non-owning reference, tidak retain Presenter)

Ownership linear: Presenter → DisplayState ← ViewController
Tidak ada circular reference. Tidak ada yang perlu weak.
Configurator: viewController.configure(displayState: presenter.displayState)
              presenter TIDAK tahu keberadaan ViewController — DI lebih bersih.
```

**Pemenang: Observable VIP** — model ownership lebih jelas, tidak ada footgun `weak` yang bisa dilupakan.

---

### Trade-off 3: Explicitness vs Reactivity — Debugging

**Pure VIP — sangat mudah di-debug:**

```swift
// Stack trace saat error muncul di layar:
// 1. UserListInteractor.fetchUsers() — throw
// 2. UserListPresenter.presentError(_:) ← breakpoint di sini
// 3. UserListViewController.displayError(message:) ← atau di sini
// Jelas, linear, mudah di-trace
```

**Observable VIP — lebih implisit:**

```swift
// Stack trace saat applyState() dipanggil:
// 1. withObservationTracking.onChange ← dipanggil oleh runtime
// 2. continuation.yield(())
// 3. AsyncStream for-await loop
// 4. self?.applyState()  ← breakpoint di sini, tapi TIDAK JELAS property mana yang berubah

// Untuk mengetahui property mana yang trigger, harus tambahkan logging manual:
private func applyState() {
    guard let state = displayState else { return }
    // Debugging: print apa yang berubah
    // (ini tidak bisa otomatis karena withObservationTracking tidak memberitahu PROPERTY mana)
    if state.displayedUsers != currentDisplayedUsers { print("users changed") }
    if state.isLoading { print("loading changed") }
    // ...
}
```

**Pemenang: Pure VIP** — data flow eksplisit dan deterministik lebih mudah di-debug, terutama saat onboarding developer baru ke codebase.

---

### Trade-off 4: Testing Presenter

**Pure VIP — butuh Mock ViewController:**

```swift
// Mock yang harus dibuat hanya untuk test
@MainActor
final class MockUserListViewController: UserListDisplayLogic {
    var displayedUsers: [UserList.FetchUsers.ViewModel.DisplayedUser] = []
    var loadingStates: [Bool] = []
    var errorMessages: [String] = []

    func displayUsers(_ viewModel: UserList.FetchUsers.ViewModel) {
        displayedUsers = viewModel.displayedUsers
    }
    func displayLoading(_ isLoading: Bool) { loadingStates.append(isLoading) }
    func displayError(message: String) { errorMessages.append(message) }
    func displaySelectedUser(_ viewModel: UserList.SelectUser.ViewModel) {}
}

// Test
@Test func testPresentUsers() async {
    let mockVC = MockUserListViewController()
    let presenter = UserListPresenter()
    presenter.viewController = mockVC

    let response = UserList.FetchUsers.Response(users: [User(id: "1", name: "Ari", email: "a@b.com", avatarURL: nil)])
    presenter.presentUsers(response)

    #expect(mockVC.displayedUsers.first?.fullName == "Ari")
}
```

**Observable VIP — test langsung dari displayState:**

```swift
// Tidak perlu mock apapun — cukup cek state
@Test func testPresentUsers() async {
    let presenter = UserListPresenter()

    let response = UserList.FetchUsers.Response(users: [User(id: "1", name: "ari supriatna", email: "a@b.com", avatarURL: nil)])
    await MainActor.run { presenter.presentUsers(response) }

    // Assert langsung ke displayState — lebih intuitif
    let users = await MainActor.run { presenter.displayState.displayedUsers }
    #expect(users.first?.fullName == "Ari Supriatna")  // test format capitalize
    #expect(users.first?.emailLabel == "a@b.com")
}
```

**Pemenang: Observable VIP** — test Presenter jauh lebih simpel, tidak ada mock infrastructure yang perlu di-maintain.

---

### Trade-off 5: Testing ViewController

**Pure VIP — inject viewModel langsung:**

```swift
// Test display logic VC dengan memanggil method langsung
@Test @MainActor func testDisplayUsersUpdatesTable() {
    let vc = UserListViewController()
    vc.loadViewIfNeeded()

    let viewModel = UserList.FetchUsers.ViewModel(displayedUsers: [
        .init(id: "1", fullName: "Ari Supriatna", emailLabel: "ari@test.com", avatarURL: nil)
    ])
    vc.displayUsers(viewModel)  // panggil langsung — synchronous

    #expect(vc.tableView.numberOfRows(inSection: 0) == 1)
}
```

**Observable VIP — harus tunggu observasi async:**

```swift
// Harus setup displayState dan tunggu observation cycle
@Test @MainActor func testDisplayUsersUpdatesTable() async {
    let displayState = UserListDisplayState()
    let vc = UserListViewController()
    vc.configure(displayState: displayState)
    vc.loadViewIfNeeded()

    // Harus tunggu observasi selesai setup
    try? await Task.sleep(for: .milliseconds(50))

    displayState.displayedUsers = [
        .init(id: "1", fullName: "Ari Supriatna", emailLabel: "ari@test.com", avatarURL: nil)
    ]

    // Harus tunggu observation cycle dan applyState() dipanggil
    try? await Task.sleep(for: .milliseconds(50))

    #expect(vc.tableView.numberOfRows(inSection: 0) == 1)
}
```

**Pemenang: Pure VIP** — test ViewController lebih deterministic karena synchronous. Observable VIP membutuhkan `Task.sleep` yang rapuh (flaky test risk).

---

### Trade-off 6: iOS Minimum Version

| Fitur | Minimum |
|---|---|
| `async/await`, `actor`, `@MainActor` | iOS 15 |
| `@Observable`, `withObservationTracking` | **iOS 17** |

Jika project mendukung iOS 15 atau 16, Observable VIP **tidak bisa digunakan** sama sekali. Pure VIP dengan Swift Concurrency tetap berjalan di iOS 15.

---

### Trade-off 7: Jumlah `applyState()` Call

Dalam Pure VIP, setiap `presentLoading`, `presentUsers`, `presentLoading(false)` adalah 3 pemanggilan terpisah dan independen.

Dalam Observable VIP, setiap assignment ke `displayState` memicu `onChange` → yield → `applyState()`. Artinya untuk satu fetch yang berhasil:

```
1. displayState.isLoading = true       → applyState() #1
2. displayState.displayedUsers = [...]  → applyState() #2
3. displayState.isLoading = false      → applyState() #3
```

Tiga kali `applyState()` untuk satu fetch — tapi `tableView.reloadData()` hanya terjadi sekali (karena ada guard `users != currentDisplayedUsers`). Overhead-nya kecil dan tidak terasa di UI, tapi perlu disadari.

---

### Ringkasan Trade-offs

| Aspek | Pure VIP | Observable VIP |
|---|---|---|
| **One-time events** | ✅ Natural (commands) | ⚠️ Butuh reset manual — footgun |
| **Retain cycle** | ⚠️ Perlu `weak` hati-hati | ✅ Tidak ada retain cycle |
| **Debugging** | ✅ Linear, mudah trace | ⚠️ Implicit, butuh logging tambahan |
| **Test Presenter** | ⚠️ Butuh Mock VC | ✅ Langsung cek state |
| **Test ViewController** | ✅ Synchronous, deterministic | ⚠️ Async, risiko flaky test |
| **iOS minimum** | ✅ iOS 15 | ❌ iOS 17 only |
| **Call applyState** | N/A | ⚠️ 3x per fetch (overhead kecil) |
| **Onboarding developer** | ✅ Mudah — flow eksplisit | ⚠️ Perlu paham Observable |
| **DI / coupling** | ⚠️ Presenter tahu VC | ✅ Presenter tidak tahu VC |

### Kapan Pilih Observable VIP?

**Pilih Observable VIP jika:**
- Target minimum iOS 17+
- Tim sudah familiar dengan `@Observable` dan Swift Concurrency
- Test coverage lebih difokuskan ke Presenter (business/formatting logic) daripada VC
- Ingin satu pattern yang sama antara UIKit dan SwiftUI (karena Observable bekerja di keduanya)
- Project baru — tidak ada legacy VIP code yang perlu di-maintain

**Tetap dengan Pure VIP jika:**
- Target iOS 15 atau 16
- Tim lebih besar dan beragam level — explicitness Pure VIP lebih mudah onboarding
- Test suite banyak di ViewController level (integration test UIKit)
- Sudah ada codebase Pure VIP yang besar — ROI migrasi tidak sebanding
- Debugging adalah prioritas — stack trace eksplisit lebih berharga

---

## Struktur File Project

```
Features/
└── UserList/
    ├── UserListModels.swift          ← Namespace, Request/Response/ViewModel, Domain Model
    ├── UserListProtocols.swift       ← Semua protocol (Display, Business, Presentation, Routing)
    ├── UserListWorker.swift          ← actor UserWorker
    ├── UserListInteractor.swift      ← actor UserListInteractor
    ├── UserListPresenter.swift       ← @MainActor UserListPresenter
    ├── UserListViewController.swift  ← @MainActor UserListViewController
    ├── UserListRouter.swift          ← @MainActor UserListRouter
    └── UserListConfigurator.swift    ← Factory / DI
```

---

## Ringkasan Keputusan Isolation

| Keputusan | Alasan |
|---|---|
| Interactor = `actor` bukan `@MainActor` | Parsing JSON dan transformasi data berjalan di background — main thread tetap bebas |
| Worker = `actor` terpisah | Interactor tidak bergantung pada detail networking — mudah diganti mock saat testing |
| Presenter = `@MainActor` | Satu-satunya layer yang boleh menyentuh View API — menjamin semua UI update di main thread |
| Models = `Sendable struct` | Harus melewati minimal 3 isolation boundary (Worker → Interactor → Presenter → View) |
| Protocol method Presenter = `@MainActor` | Compiler memaksa semua caller menggunakan `await` — tidak ada yang bisa "lupa" thread |
| `Task {}` di ViewController, bukan `Task.detached` | Mewarisi `@MainActor` isolation — tidak perlu `MainActor.run {}` saat kembali ke UI |

## Ringkasan Fitur Swift Concurrency yang Digunakan

| Fitur | Digunakan Di | Tujuan |
|---|---|---|
| `actor` | Worker, Interactor | Isolasi state — thread-safe tanpa lock manual |
| `@MainActor` | ViewController, Presenter, Router | Jamin semua UIKit di main thread |
| `async/await` | Semua layer | Alur async yang linear dan mudah dibaca |
| `Task {}` | ViewController | Bridge sync UIKit callback → async method |
| `Task.cancel()` | ViewController | Batalkan fetch saat view hilang — hemat resource |
| `Task.checkCancellation()` | Worker | Cek cancellation sebelum operasi berat |
| `withTaskCancellationHandler` | Worker | Cleanup resource eksternal saat cancel |
| `async let` | Interactor | Fetch beberapa sumber data secara parallel |
| `withTaskGroup` | Worker | Bulk operation dengan jumlah task dinamis |
| `Task.detached` | Interactor | Operasi background yang tidak terikat isolation parent |
| `AsyncStream` | Worker | Live polling — push values ke consumer secara async |
| `for await` | Interactor | Konsumsi AsyncStream secara sequential |
| `withCheckedContinuation` | ViewController | Bridge UIKit delegate → async function |
| `withCheckedThrowingContinuation` | LocationService | Bridge delegate yang bisa throw → async throws |
| `Sendable` | Semua models | Izinkan nilai melewati isolation boundary dengan aman |
| `nonisolated(unsafe)` | Interactor.selectedUser | Trade-off eksplisit untuk DataStore pattern |

---

*Dibuat: 2026-05-08 | Diperbarui: 2026-05-09 | Swift 6.0 | UIKit | Pattern: VIP (Clean Swift)*
