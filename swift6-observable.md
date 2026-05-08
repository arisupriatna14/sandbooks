# Observable di Swift 6
### Panduan Mendalam: Teori, Cara Kerja, Real Use Cases, dan Trade-offs

---

## Daftar Isi

1. [Masalah yang Dipecahkan](#1-masalah-yang-dipecahkan)
2. [Teori & Cara Kerja Internal](#2-teori--cara-kerja-internal)
3. [Sintaks Dasar](#3-sintaks-dasar)
4. [@Bindable — Dua Arah Binding](#4-bindable--dua-arah-binding)
5. [@ObservationIgnored — Opt-out dari Tracking](#5-observationignored--opt-out-dari-tracking)
6. [withObservationTracking — Observasi Manual](#6-withobservationtracking--observasi-manual)
7. [Thread Safety & Swift 6](#7-thread-safety--swift-6)
8. [Perbandingan: ObservableObject vs @Observable](#8-perbandingan-observableobject-vs-observable)
9. [Real Use Cases](#9-real-use-cases)
10. [Kapan Harus Digunakan](#10-kapan-harus-digunakan)
11. [Kapan Tidak Digunakan](#11-kapan-tidak-digunakan)
12. [Keunggulan & Kekurangan](#12-keunggulan--kekurangan)
13. [Ringkasan](#13-ringkasan)

---

## 1. Masalah yang Dipecahkan

### Era Sebelum Observable: `ObservableObject` + `@Published`

Sebelum `@Observable` hadir di Swift 5.9, SwiftUI menggunakan `ObservableObject` dari framework **Combine**:

```swift
// Swift 5.x — cara lama
import Combine

class UserViewModel: ObservableObject {
    @Published var name: String = ""
    @Published var email: String = ""
    @Published var isLoading: Bool = false
    var internalCache: [String: Any] = [:]  // juga di-observe, padahal tidak mau
}
```

**Masalah besar `ObservableObject`:**

#### Masalah 1: Granularitas Terlalu Kasar (Over-notification)

```swift
class CartViewModel: ObservableObject {
    @Published var items: [CartItem] = []
    @Published var totalPrice: Double = 0
    @Published var couponCode: String = ""
    @Published var isCheckingOut: Bool = false
}

// View A hanya menampilkan totalPrice.
// View B hanya menampilkan items.
// View C hanya menampilkan couponCode.
//
// Ketika 'isCheckingOut' berubah (tidak relevan untuk A, B, C),
// SEMUA VIEW di-invalidate dan di-render ulang.
// ObservableObject menggunakan objectWillChange publisher —
// satu sinyal untuk SEMUA perubahan property.
```

`objectWillChange` adalah **all-or-nothing**: satu property berubah → semua subscriber dinotifikasi → semua view yang mengobservasi object ini di-render ulang, meski properti yang mereka tampilkan tidak berubah.

#### Masalah 2: Ketergantungan pada Combine

`ObservableObject` adalah protocol dari framework **Combine** — bukan bagian dari Swift standard library. Ini berarti:
- Di server-side Swift (Linux), Combine tidak tersedia
- Di UIKit tanpa SwiftUI, harus import Combine hanya untuk state management
- Testing lebih kompleks karena Combine infrastructure ikut terbawa

#### Masalah 3: Boilerplate `@Published`

Setiap property yang ingin di-observe harus diberi anotasi `@Published`. Jika lupa — property itu tidak di-observe dan UI tidak update. Jika tidak sengaja menambahkan `@Published` ke property internal — property itu ikut men-trigger re-render yang tidak perlu.

#### Masalah 4: `@EnvironmentObject` Tidak Type-Safe

```swift
// Lupa inject environmentObject → crash saat runtime, bukan compile error
struct SomeView: View {
    @EnvironmentObject var viewModel: CartViewModel  // crash jika tidak di-inject
}
```

---

## 2. Teori & Cara Kerja Internal

### Observation Framework

`@Observable` adalah **Swift macro** yang diperkenalkan di Swift 5.9 (Xcode 15, iOS 17) sebagai bagian dari framework **Observation** — bukan Combine. Framework ini murni Swift, tidak bergantung pada Objective-C runtime atau Combine.

### Ekspansi Macro

Ketika kita menulis:

```swift
@Observable
class UserViewModel {
    var name: String = ""
    var age: Int = 0
}
```

Swift macro meng-expand kode tersebut menjadi kira-kira:

```swift
class UserViewModel: Observable {
    // Setiap property yang di-observe dibungkus dengan
    // akses ke _$observationRegistrar secara otomatis.

    @ObservationIgnored
    private var _name: String = ""

    var name: String {
        get {
            // Registrar mencatat: "siapa pun yang sedang mengakses,
            // daftarkan dirimu sebagai subscriber untuk property 'name'"
            _$observationRegistrar.access(keyPath: \.name)
            return _name
        }
        set {
            // Registrar memberitahu semua subscriber bahwa 'name' akan berubah
            _$observationRegistrar.withMutation(keyPath: \.name) {
                _name = newValue
            }
        }
    }

    @ObservationIgnored
    private var _age: Int = 0

    var age: Int {
        get {
            _$observationRegistrar.access(keyPath: \.age)
            return _age
        }
        set {
            _$observationRegistrar.withMutation(keyPath: \.age) {
                _age = newValue
            }
        }
    }

    // Registrar ini adalah "pusat kendali" observasi.
    // Ia menyimpan mapping: keyPath → Set<Observer>
    @ObservationIgnored
    private let _$observationRegistrar = ObservationRegistrar()
}
```

### Mekanisme Dependency Tracking Otomatis

Ini adalah perbedaan paling fundamental antara `@Observable` dan `ObservableObject`:

```
ObservableObject:
  View subscribe ke OBJECT → satu property berubah → VIEW DI-RENDER ULANG

@Observable:
  View membaca property X → SwiftUI MENCATAT bahwa View ini bergantung pada X
  Property Y berubah → SwiftUI cek: "apakah ada View yang bergantung pada Y?" → tidak ada → View TIDAK di-render ulang
  Property X berubah → SwiftUI cek: "ada View yang bergantung pada X?" → ada → View DI-RENDER ULANG
```

Dependency tracking ini terjadi secara **otomatis saat runtime** berdasarkan property mana yang dibaca di dalam body View. Tidak perlu anotasi manual.

```swift
@Observable
class DashboardViewModel {
    var userName: String = "Ari"
    var postCount: Int = 42
    var followerCount: Int = 1000
    var isLoading: Bool = false
}

// View ini HANYA membaca userName dan postCount.
// SwiftUI mencatat dependensi: [userName, postCount]
// Ketika isLoading atau followerCount berubah → View ini TIDAK di-render ulang.
struct HeaderView: View {
    var viewModel: DashboardViewModel

    var body: some View {
        VStack {
            Text(viewModel.userName)   // → dependency: userName
            Text("\(viewModel.postCount) posts")  // → dependency: postCount
        }
    }
}
```

### `ObservationRegistrar`

`ObservationRegistrar` adalah struct internal yang menjadi jantung dari Observation framework. Ia:
1. **Merekam akses** (`access`): saat property dibaca, registrar mencatat siapa observer-nya
2. **Memicu mutasi** (`withMutation`): saat property ditulis, registrar memberitahu semua observer yang bergantung pada property tersebut
3. **Menyimpan mapping** per-keyPath: `[PartialKeyPath<T>: Set<AnyObserver>]`

---

## 3. Sintaks Dasar

### Deklarasi

```swift
import Observation

// Cukup tambahkan @Observable — tidak perlu ObservableObject, tidak perlu @Published.
// Berlaku untuk class (reference type) — struct tidak bisa @Observable karena
// struct tidak punya identity yang stable untuk observation.
@Observable
final class ProductViewModel {
    // Semua stored property otomatis di-observe — tidak perlu @Published.
    var products: [Product] = []
    var searchQuery: String = ""
    var isLoading: Bool = false
    var selectedCategory: Category = .all
    var errorMessage: String? = nil

    // Computed property juga otomatis di-track berdasarkan stored property
    // yang dibaca di dalamnya.
    var filteredProducts: [Product] {
        if searchQuery.isEmpty { return products }
        return products.filter { $0.name.localizedCaseInsensitiveContains(searchQuery) }
    }
}
```

### Penggunaan di SwiftUI

```swift
struct ProductListView: View {
    // Tidak perlu @StateObject/@ObservedObject/@EnvironmentObject —
    // cukup simpan sebagai stored property biasa atau gunakan @State untuk ownership.
    @State private var viewModel = ProductViewModel()

    var body: some View {
        NavigationStack {
            List(viewModel.filteredProducts) { product in  // track: filteredProducts
                ProductRow(product: product)
            }
            .searchable(text: $viewModel.searchQuery)      // track: searchQuery
            .overlay {
                if viewModel.isLoading {                   // track: isLoading
                    ProgressView()
                }
            }
            .task {
                await viewModel.loadProducts()
            }
        }
    }
}

// Passing ke child view — tidak perlu @ObservedObject, cukup var biasa.
// SwiftUI otomatis track property mana yang dibaca di body child view.
struct ProductRow: View {
    var product: Product  // nilai yang sudah di-copy — tidak di-observe langsung

    var body: some View {
        Text(product.name)
    }
}

// Passing viewModel ke child — child menerima reference ke object yang sama.
struct ProductDetailView: View {
    var viewModel: ProductViewModel  // tidak perlu @ObservedObject

    var body: some View {
        Text(viewModel.selectedCategory.rawValue)  // track: selectedCategory
    }
}
```

### Dengan `@Environment` (Pengganti @EnvironmentObject)

```swift
// Inject via .environment() — type-safe, tidak perlu @EnvironmentObject
struct MyApp: App {
    @State private var authViewModel = AuthViewModel()

    var body: some Scene {
        WindowGroup {
            ContentView()
                .environment(authViewModel)  // inject sebagai environment value
        }
    }
}

// Konsumsi di child view
struct ProfileView: View {
    @Environment(AuthViewModel.self) private var auth  // type-safe, compile error jika lupa inject

    var body: some View {
        Text("Hello, \(auth.currentUser?.name ?? "Guest")")
    }
}
```

---

## 4. @Bindable — Dua Arah Binding

Ketika kita butuh `$viewModel.propertyName` (binding untuk TextField, Toggle, dll) dari `@Observable` object, gunakan `@Bindable`.

```swift
@Observable
final class SettingsViewModel {
    var notificationsEnabled: Bool = true
    var username: String = ""
    var selectedTheme: Theme = .system
}

struct SettingsView: View {
    // @State untuk ownership (view ini yang memiliki viewModel)
    @State private var viewModel = SettingsViewModel()

    var body: some View {
        Form {
            Toggle("Notifikasi", isOn: $viewModel.notificationsEnabled)
            // $viewModel.notificationsEnabled bekerja langsung karena
            // @State sudah menyediakan Bindable access.
        }
    }
}

// Ketika viewModel datang dari luar (tidak di-own oleh view ini):
struct SettingsSheet: View {
    // @Bindable membuat kita bisa membuat Binding ke property @Observable object
    // yang datang sebagai parameter (bukan @State).
    @Bindable var viewModel: SettingsViewModel

    var body: some View {
        Form {
            TextField("Username", text: $viewModel.username)        // binding ke username
            Picker("Tema", selection: $viewModel.selectedTheme) {   // binding ke selectedTheme
                ForEach(Theme.allCases) { theme in
                    Text(theme.name).tag(theme)
                }
            }
        }
    }
}
```

**Kapan `@Bindable` diperlukan:**

| Skenario | Cara |
|---|---|
| View memiliki viewModel (`@State`) | Gunakan `$viewModel.property` langsung |
| viewModel datang dari parameter | Gunakan `@Bindable var viewModel` lalu `$viewModel.property` |
| viewModel dari `@Environment` | Ambil dengan `@Environment`, lalu buat `@Bindable` lokal |

```swift
// @Environment + @Bindable
struct AccountView: View {
    @Environment(AccountViewModel.self) private var account

    var body: some View {
        // Buat @Bindable lokal dari environment value
        @Bindable var account = account
        TextField("Nama", text: $account.displayName)
    }
}
```

---

## 5. @ObservationIgnored — Opt-out dari Tracking

Property yang diberi `@ObservationIgnored` **tidak akan di-track** — perubahan pada property ini tidak akan men-trigger re-render View manapun.

```swift
@Observable
final class DataViewModel {
    // Di-track — perubahan memicu re-render
    var visibleItems: [Item] = []
    var isLoading: Bool = false

    // Tidak di-track — perubahan tidak memicu re-render apapun.
    // Gunakan untuk: internal cache, logging, metadata, dependencies eksternal.
    @ObservationIgnored
    private var internalCache: [String: Item] = [:]

    @ObservationIgnored
    private var networkService: NetworkService = NetworkService()

    @ObservationIgnored
    var lastRefreshDate: Date? = nil  // metadata yang tidak perlu tampil di UI

    // Computed property yang HANYA mengakses @ObservationIgnored property
    // → tidak men-trigger observation apapun.
    var cacheSize: Int {
        internalCache.count  // mengakses ignored property — tidak di-track
    }

    func refresh() async {
        isLoading = true  // tracked — View akan update
        let items = await networkService.fetchItems()  // ignored property — tidak trigger observation
        internalCache = Dictionary(uniqueKeysWithValues: items.map { ($0.id, $0) })  // ignored
        visibleItems = items  // tracked — View akan update
        lastRefreshDate = Date()  // ignored — View tidak di-render ulang
        isLoading = false  // tracked — View akan update
    }
}
```

---

## 6. withObservationTracking — Observasi Manual

`withObservationTracking` berguna ketika kita butuh observasi **di luar SwiftUI** — misalnya di UIKit, di layer business logic, atau untuk testing.

```swift
import Observation

@Observable
final class CounterModel {
    var count: Int = 0
    var label: String = "Start"
}

// Observasi manual — cocok untuk UIKit atau non-SwiftUI context
func setupManualObservation(model: CounterModel, label: UILabel) {
    func observe() {
        withObservationTracking {
            // Blok ini dijalankan SEGERA dan mencatat dependency.
            // Semua property yang diakses di sini akan di-track.
            label.text = "\(model.count)"  // → track: count
        } onChange: {
            // Blok ini dipanggil SATU KALI ketika dependency berubah.
            // Setelah dipanggil, tracking berhenti — harus daftar ulang.
            // Ini adalah "fire-once" observer — berbeda dengan Combine publisher.
            DispatchQueue.main.async {
                observe()  // Daftar ulang untuk melanjutkan tracking
            }
        }
    }
    observe()
}
```

**Pola rekursif** di atas adalah idiom standar untuk continuous observation dengan `withObservationTracking`. Setiap kali dependency berubah, kita daftar ulang observasi — sehingga tracking terus berjalan.

### Integrasi UIKit dengan withObservationTracking

```swift
// UIKit ViewController yang mengobservasi @Observable model
final class CounterViewController: UIViewController {
    private let model = CounterModel()
    private var observationTask: Task<Void, Never>?

    private let countLabel = UILabel()
    private let incrementButton = UIButton(type: .system)

    override func viewDidLoad() {
        super.viewDidLoad()
        setupUI()
        startObserving()
    }

    override func viewDidDisappear(_ animated: Bool) {
        super.viewDidDisappear(animated)
        observationTask?.cancel()
    }

    private func startObserving() {
        // Bungkus dalam Task agar bisa di-cancel saat view hilang.
        // Gunakan AsyncStream untuk mengubah "fire-once" withObservationTracking
        // menjadi continuous async sequence.
        observationTask = Task { @MainActor in
            let stream = AsyncStream<Void> { continuation in
                func scheduleObservation() {
                    withObservationTracking {
                        // Akses property yang ingin di-track
                        _ = self.model.count
                        _ = self.model.label
                    } onChange: {
                        continuation.yield(())
                        // Jadwal observasi ulang setelah yield
                        DispatchQueue.main.async { scheduleObservation() }
                    }
                }
                scheduleObservation()
                continuation.onTermination = { _ in }
            }

            for await _ in stream {
                guard !Task.isCancelled else { break }
                // Update UI saat ada perubahan
                countLabel.text = "\(model.count)"
                title = model.label
            }
        }
    }

    @objc private func handleIncrement() {
        model.count += 1
    }

    private func setupUI() {
        // ... setup constraints
    }
}
```

---

## 7. Thread Safety & Swift 6

### `@Observable` Tidak Otomatis Thread-Safe

`@Observable` **bukan** actor dan **tidak** memberikan thread safety secara otomatis. `ObservationRegistrar` menjamin keamanan registrasi observer, tapi akses ke property dari multiple thread tetap bisa menyebabkan data race.

```swift
// BERBAHAYA — tidak ada isolation
@Observable
final class CounterViewModel {
    var count: Int = 0  // bisa di-mutasi dari thread mana saja

    func increment() {
        count += 1  // jika dipanggil dari background thread, data race!
    }
}
```

### Solusi 1: `@MainActor` + `@Observable`

Paling umum untuk ViewModel yang digunakan dari SwiftUI:

```swift
// Semua akses ke ViewModel dijamin di main thread.
// Swift 6 compiler akan error jika kita mencoba mengakses
// property ini dari non-MainActor context tanpa await.
@Observable
@MainActor
final class UserViewModel {
    var users: [User] = []
    var isLoading: Bool = false

    private let service: UserService

    init(service: UserService = UserService()) {
        self.service = service
    }

    func loadUsers() async {
        isLoading = true
        do {
            // await melepas @MainActor isolation untuk network call
            users = try await service.fetchUsers()
        } catch {
            // kembali ke @MainActor setelah await
        }
        isLoading = false
    }
}

// Swift 6: tidak ada warning, tidak ada data race
struct UserListView: View {
    @State private var viewModel = UserViewModel()

    var body: some View {
        List(viewModel.users) { user in Text(user.name) }
            .task { await viewModel.loadUsers() }
    }
}
```

### Solusi 2: `actor` + Manual Publish ke `@Observable`

Ketika butuh state yang di-isolasi di non-main thread tapi hasil akhirnya perlu di-observe SwiftUI:

```swift
// Actor untuk business logic — berjalan di background
actor UserService {
    private var cachedUsers: [User] = []

    func fetchAndCache() async throws -> [User] {
        let users = try await APIClient.shared.fetchUsers()
        cachedUsers = users
        return users
    }
}

// @Observable @MainActor sebagai "presentation layer"
@Observable
@MainActor
final class UserViewModel {
    var users: [User] = []
    var isLoading: Bool = false

    private let service = UserService()

    func loadUsers() async {
        isLoading = true
        do {
            // Crossing isolation boundary: @MainActor → actor
            let fetchedUsers = try await service.fetchAndCache()
            // Kembali ke @MainActor — update observable state
            users = fetchedUsers
        } catch { }
        isLoading = false
    }
}
```

### Solusi 3: `nonisolated` untuk Computed Property Berat

```swift
@Observable
@MainActor
final class AnalyticsViewModel {
    var rawData: [DataPoint] = []

    // Komputasi berat di main thread bisa menyebabkan janky UI.
    // Hitung di background, publish hasilnya.
    var processedData: [ProcessedPoint] = []

    func processData() async {
        // Hitung di background dengan Task.detached
        let result = await Task.detached(priority: .userInitiated) {
            // nonisolated context — tidak di @MainActor
            self.rawData.map { ProcessedPoint(from: $0) }  // rawData di-capture
        }.value

        // Update di @MainActor
        processedData = result
    }
}
```

---

## 8. Perbandingan: ObservableObject vs @Observable

| Aspek | `ObservableObject` + `@Published` | `@Observable` |
|---|---|---|
| **Framework** | Combine (tidak tersedia di Linux) | Observation (Swift standard) |
| **Granularitas** | Per-object (all-or-nothing) | Per-property (fine-grained) |
| **Boilerplate** | `@Published` di setiap property | Tidak ada anotasi per-property |
| **Binding** | `@ObservedObject`, `@StateObject`, `@EnvironmentObject` | `@State`, `@Bindable`, `@Environment` |
| **Thread safety** | Tidak otomatis | Tidak otomatis |
| **iOS/macOS minimum** | iOS 13 / macOS 10.15 | iOS 17 / macOS 14 |
| **Performa rendering** | Sering over-render | Minimal re-render |
| **Non-SwiftUI support** | Via Combine publisher | Via `withObservationTracking` |
| **Swift 6 compatibility** | Perlu extra annotation | Lebih natural dengan `@MainActor` |
| **Nested observable** | Tidak otomatis | Otomatis (nested @Observable di-track) |

### Contoh Nested Observable

```swift
// ObservableObject — nested TIDAK otomatis di-observe
class AddressViewModel: ObservableObject {
    @Published var city: String = ""
}

class UserViewModel: ObservableObject {
    // Perubahan di nested.city TIDAK akan men-trigger UserViewModel.objectWillChange
    // Harus subscribe manual ke nested.objectWillChange dan forward ke self.
    var address = AddressViewModel()

    private var cancellable: AnyCancellable?
    init() {
        // Boilerplate manual untuk propagate nested changes
        cancellable = address.objectWillChange
            .sink { [weak self] _ in self?.objectWillChange.send() }
    }
}

// @Observable — nested OTOMATIS di-track
@Observable
final class Address {
    var city: String = ""
}

@Observable
final class UserViewModel {
    // Perubahan di address.city otomatis di-track oleh SwiftUI
    // yang mengakses viewModel.address.city di body-nya.
    var address = Address()
}
```

---

## 9. Real Use Cases

### Use Case 1: MVVM dengan SwiftUI — E-Commerce Cart

```swift
import Observation

struct CartItem: Identifiable, Sendable {
    let id: UUID
    let product: Product
    var quantity: Int
    var totalPrice: Double { product.price * Double(quantity) }
}

@Observable
@MainActor
final class CartViewModel {
    private(set) var items: [CartItem] = []
    var couponCode: String = ""
    var isCheckingOut: Bool = false
    var checkoutError: String? = nil

    // Computed — otomatis di-track, hanya dihitung ulang jika items atau couponCode berubah
    var subtotal: Double {
        items.reduce(0) { $0 + $1.totalPrice }
    }

    var discount: Double {
        couponCode == "SWIFT6" ? subtotal * 0.1 : 0
    }

    var total: Double {
        subtotal - discount
    }

    var itemCount: Int {
        items.reduce(0) { $0 + $1.quantity }
    }

    func addItem(_ product: Product) {
        if let index = items.firstIndex(where: { $0.product.id == product.id }) {
            items[index].quantity += 1
        } else {
            items.append(CartItem(id: UUID(), product: product, quantity: 1))
        }
    }

    func removeItem(at offsets: IndexSet) {
        items.remove(atOffsets: offsets)
    }

    func checkout() async {
        guard !isCheckingOut else { return }
        isCheckingOut = true
        checkoutError = nil

        do {
            try await OrderService.shared.placeOrder(items: items, coupon: couponCode)
            items.removeAll()
            couponCode = ""
        } catch {
            checkoutError = error.localizedDescription
        }
        isCheckingOut = false
    }
}

// Views — masing-masing hanya re-render berdasarkan property yang dibaca

struct CartSummaryView: View {
    var viewModel: CartViewModel  // hanya baca total, subtotal, discount

    var body: some View {
        VStack(alignment: .leading, spacing: 8) {
            HStack { Text("Subtotal"); Spacer(); Text(viewModel.subtotal.currencyFormatted) }
            if viewModel.discount > 0 {
                HStack { Text("Diskon"); Spacer(); Text("-\(viewModel.discount.currencyFormatted)") }
            }
            Divider()
            HStack { Text("Total").bold(); Spacer(); Text(viewModel.total.currencyFormatted).bold() }
        }
        // Re-render hanya ketika subtotal, discount, atau total berubah
        // (yang terjadi ketika items atau couponCode berubah)
    }
}

struct CartBadgeView: View {
    var viewModel: CartViewModel  // hanya baca itemCount

    var body: some View {
        Text("\(viewModel.itemCount)")
            .font(.caption2)
            .padding(4)
            .background(.red)
            .foregroundStyle(.white)
            .clipShape(.circle)
        // Re-render HANYA ketika itemCount berubah
        // Tidak peduli dengan couponCode, isCheckingOut, dll
    }
}

struct CheckoutButton: View {
    @Bindable var viewModel: CartViewModel  // butuh binding untuk isCheckingOut (jika perlu toggle)

    var body: some View {
        Button {
            Task { await viewModel.checkout() }
        } label: {
            if viewModel.isCheckingOut {
                ProgressView()
            } else {
                Text("Checkout — \(viewModel.total.currencyFormatted)")
            }
        }
        .disabled(viewModel.isCheckingOut || viewModel.items.isEmpty)
        // Re-render hanya ketika isCheckingOut atau items.isEmpty berubah
    }
}
```

---

### Use Case 2: Multi-Screen State Sharing via @Environment

```swift
// AuthViewModel di-share ke seluruh app via @Environment
@Observable
@MainActor
final class AuthViewModel {
    private(set) var currentUser: AppUser? = nil
    private(set) var isAuthenticated: Bool = false
    var isLoading: Bool = false

    func login(email: String, password: String) async throws {
        isLoading = true
        defer { isLoading = false }
        let user = try await AuthService.shared.login(email: email, password: password)
        currentUser = user
        isAuthenticated = true
    }

    func logout() async {
        await AuthService.shared.logout()
        currentUser = nil
        isAuthenticated = false
    }
}

@main
struct MyApp: App {
    @State private var auth = AuthViewModel()

    var body: some Scene {
        WindowGroup {
            RootView()
                .environment(auth)
        }
    }
}

struct RootView: View {
    @Environment(AuthViewModel.self) private var auth

    var body: some View {
        if auth.isAuthenticated {
            MainTabView()
        } else {
            LoginView()
        }
        // Re-render hanya ketika isAuthenticated berubah
    }
}

struct ProfileView: View {
    @Environment(AuthViewModel.self) private var auth

    var body: some View {
        VStack {
            Text(auth.currentUser?.displayName ?? "Guest")
            // Re-render hanya ketika currentUser berubah
            Button("Logout") { Task { await auth.logout() } }
        }
    }
}

struct NavigationBarBadge: View {
    @Environment(AuthViewModel.self) private var auth

    var body: some View {
        if auth.isLoading {
            ProgressView().scaleEffect(0.7)
        }
        // Re-render hanya ketika isLoading berubah — tidak peduli currentUser dll
    }
}
```

---

### Use Case 3: Integrasi UIKit dengan withObservationTracking

```swift
// ViewModel yang digunakan bersama SwiftUI dan UIKit
@Observable
@MainActor
final class FeedViewModel {
    var posts: [Post] = []
    var isRefreshing: Bool = false
    var unreadCount: Int = 0

    private let service: FeedService

    init(service: FeedService = .shared) {
        self.service = service
    }

    func refresh() async {
        isRefreshing = true
        posts = (try? await service.fetchPosts()) ?? posts
        isRefreshing = false
    }
}

// UIKit ViewController yang mengobservasi FeedViewModel
@MainActor
final class FeedViewController: UIViewController {
    private let viewModel: FeedViewModel
    private var observationTask: Task<Void, Never>?

    private let tableView = UITableView()
    private let refreshControl = UIRefreshControl()
    private let badgeLabel = UILabel()

    init(viewModel: FeedViewModel) {
        self.viewModel = viewModel
        super.init(nibName: nil, bundle: nil)
    }

    required init?(coder: NSCoder) { fatalError() }

    override func viewDidLoad() {
        super.viewDidLoad()
        setupUI()
        startObserving()
    }

    override func viewDidDisappear(_ animated: Bool) {
        super.viewDidDisappear(animated)
        observationTask?.cancel()
    }

    private func startObserving() {
        observationTask = Task { @MainActor in
            // Bungkus withObservationTracking dalam AsyncStream untuk continuous observation
            let changes = AsyncStream<Void> { continuation in
                func track() {
                    withObservationTracking {
                        // Track property yang kita butuhkan
                        _ = viewModel.posts
                        _ = viewModel.isRefreshing
                        _ = viewModel.unreadCount
                    } onChange: {
                        continuation.yield(())
                        Task { @MainActor in track() }  // daftar ulang
                    }
                }
                track()
            }

            for await _ in changes {
                guard !Task.isCancelled else { break }
                applySnapshot()
            }
        }
    }

    private func applySnapshot() {
        tableView.reloadData()  // atau gunakan diffable data source

        if viewModel.isRefreshing {
            refreshControl.beginRefreshing()
        } else {
            refreshControl.endRefreshing()
        }

        badgeLabel.text = viewModel.unreadCount > 0 ? "\(viewModel.unreadCount)" : nil
        badgeLabel.isHidden = viewModel.unreadCount == 0
    }

    @objc private func handleRefresh() {
        Task { await viewModel.refresh() }
    }

    private func setupUI() {
        refreshControl.addTarget(self, action: #selector(handleRefresh), for: .valueChanged)
        tableView.refreshControl = refreshControl
        // ... layout setup
    }
}
```

---

### Use Case 4: Nested Observable — Master-Detail

```swift
@Observable
final class FilterState {
    var priceRange: ClosedRange<Double> = 0...1000
    var selectedBrands: Set<String> = []
    var sortOrder: SortOrder = .relevance
    var showInStock: Bool = true

    var isDefault: Bool {
        priceRange == 0...1000 && selectedBrands.isEmpty
            && sortOrder == .relevance && showInStock
    }

    func reset() {
        priceRange = 0...1000
        selectedBrands = []
        sortOrder = .relevance
        showInStock = true
    }
}

@Observable
@MainActor
final class ProductListViewModel {
    var products: [Product] = []
    var isLoading: Bool = false

    // Nested @Observable — perubahan di filter.priceRange
    // akan otomatis di-track oleh View yang membaca filter.priceRange
    var filter = FilterState()

    var filteredProducts: [Product] {
        products.filter { product in
            filter.priceRange.contains(product.price)
            && (filter.selectedBrands.isEmpty || filter.selectedBrands.contains(product.brand))
            && (!filter.showInStock || product.isInStock)
        }
    }

    func loadProducts() async {
        isLoading = true
        products = (try? await ProductService.shared.fetchAll()) ?? []
        isLoading = false
    }
}

// View yang membaca filter.priceRange akan re-render jika priceRange berubah,
// tapi TIDAK jika filter.selectedBrands berubah (karena tidak membacanya di body ini)
struct PriceFilterView: View {
    @Bindable var filter: FilterState

    var body: some View {
        RangeSlider(value: $filter.priceRange, in: 0...10000)
        // Re-render hanya ketika filter.priceRange berubah
    }
}
```

---

## 10. Kapan Harus Digunakan

**Gunakan `@Observable` ketika:**

- **Project SwiftUI dengan target iOS 17+ / macOS 14+** — tidak ada alasan untuk tidak menggunakan `@Observable` di project baru
- **ViewModel dengan banyak property** tapi setiap View hanya menggunakan sebagian — manfaat fine-grained tracking sangat terasa
- **Nested state** yang perlu di-observe — `@Observable` secara otomatis men-track nested @Observable tanpa boilerplate manual
- **Butuh `@Environment` yang type-safe** — `@Observable` bekerja langsung dengan `.environment()` tanpa `@EnvironmentObject`
- **Library atau module yang ingin zero Combine dependency** — `@Observable` tidak membutuhkan Combine sama sekali
- **UIKit + SwiftUI hybrid** — `withObservationTracking` memungkinkan UIKit komponen mengobservasi state yang sama dengan SwiftUI
- **Performa tinggi** — jika profiling menunjukkan over-rendering dari `ObservableObject`, migrasi ke `@Observable` sering kali langsung menyelesaikan masalah

---

## 11. Kapan Tidak Digunakan

**Jangan gunakan `@Observable` ketika:**

- **Target minimum iOS 16 ke bawah** — `@Observable` tidak tersedia, wajib pakai `ObservableObject`. Tidak ada workaround.
- **Project sudah heavily invested di Combine pipeline** — jika seluruh data layer sudah menggunakan `AnyPublisher`, `PassthroughSubject`, dll, refactor ke `@Observable` membutuhkan perubahan besar tanpa manfaat langsung yang sebanding
- **Butuh operator Combine** (`debounce`, `throttle`, `combineLatest`, `flatMap`) untuk transformasi reactive yang kompleks — `@Observable` tidak memiliki ekuivalen ini. Pertimbangkan kombinasi `@Observable` + Combine pipeline terpisah, atau AsyncSequence
- **Butuh fine-grained Combine subscription** dengan backpressure management — AsyncStream lebih cocok, bukan `@Observable`
- **Value type (struct) yang ingin di-observe** — `@Observable` hanya bekerja untuk `class`. Untuk struct, gunakan `@State` langsung di SwiftUI
- **Server-side Swift** tanpa SwiftUI — `withObservationTracking` bisa dipakai, tapi overhead framework kurang sepadan untuk server use case. Pertimbangkan Observer pattern sederhana atau Combine (jika Linux-compatible Combine tersedia)
- **Synchronous notification** yang tidak boleh melewati run loop — `@Observable` menjadwalkan notifikasi perubahan secara asynchronous melalui SwiftUI update cycle

---

## 12. Keunggulan & Kekurangan

### Keunggulan

| Keunggulan | Detail |
|---|---|
| **Fine-grained rendering** | View hanya re-render ketika property yang benar-benar dibaca berubah — performa optimal |
| **Zero boilerplate** | Tidak ada `@Published`, tidak ada `objectWillChange.send()` manual |
| **Nested tracking otomatis** | Perubahan di nested @Observable langsung di-track tanpa subscription manual |
| **Tidak butuh Combine** | Zero dependency pada Combine — bisa digunakan di mana saja Swift berjalan |
| **Type-safe environment** | `.environment(viewModel)` + `@Environment(ViewModel.self)` — compile error jika lupa inject |
| **Macro transparency** | Bisa di-inspect via "Expand Macro" di Xcode — tidak ada magic tersembunyi |
| **Lebih natural dengan Swift 6** | Bekerja seamless dengan `@MainActor`, `Sendable`, dan strict concurrency |
| **Sederhana untuk testing** | Tidak perlu setup Combine subscribers atau manage AnyCancellable |

### Kekurangan

| Kekurangan | Detail |
|---|---|
| **iOS 17+ only** | Tidak bisa digunakan di project dengan minimum deployment target iOS 16 ke bawah |
| **Hanya class** | Struct tidak bisa `@Observable` — membatasi beberapa pola desain yang value-type oriented |
| **Tidak ada built-in operator** | Tidak ada `debounce`, `throttle`, `merge` seperti di Combine — harus implementasi manual atau kombinasi dengan AsyncSequence |
| **withObservationTracking adalah fire-once** | Harus daftar ulang setelah setiap onChange — pola rekursif bisa membingungkan jika tidak familiar |
| **Tidak ada replay/backpressure** | Tidak seperti Combine publisher yang bisa `share()`, `multicast()`, atau buffer — setiap observer mendapat state saat ini, bukan historical values |
| **Debugging lebih sulit** | Macro expansion membuat stack trace kurang jelas dibanding kode eksplisit |
| **Tidak thread-safe otomatis** | Harus kombinasikan dengan `@MainActor` atau `actor` secara eksplisit — berbeda dari anggapan awal banyak developer |

---

## 13. Ringkasan

```
@Observable adalah pengganti modern ObservableObject untuk iOS 17+.
Ia memecahkan masalah over-rendering dengan fine-grained dependency tracking per-property,
menghilangkan kebutuhan Combine, dan memangkas boilerplate @Published.

Kunci penggunaan yang benar:
  ┌─────────────────────────────────────────────────────┐
  │  @Observable + @MainActor  → ViewModel standar      │
  │  @State                    → ownership di SwiftUI   │
  │  @Bindable                 → two-way binding        │
  │  @Environment              → shared state cross-view│
  │  @ObservationIgnored       → opt-out dari tracking  │
  │  withObservationTracking   → UIKit / manual use     │
  └─────────────────────────────────────────────────────┘

Gunakan ketika: iOS 17+, SwiftUI, butuh performa rendering optimal.
Jangan gunakan ketika: iOS 16-, heavily Combine, butuh operator reactive.
```

### Decision Tree

```
Butuh state management?
  │
  ├─ iOS 16 ke bawah? ──────────────────────→ ObservableObject + @Published
  │
  ├─ iOS 17+, SwiftUI? ─────────────────────→ @Observable + @MainActor ✅
  │
  ├─ UIKit saja? ───────────────────────────→ @Observable + withObservationTracking
  │
  ├─ Butuh debounce/throttle/combine? ──────→ @Observable + AsyncSequence/Combine pipeline
  │
  └─ Server-side / non-UI? ────────────────→ actor atau simple Observer pattern
```

---

*Dibuat: 2026-05-09 | Swift 6.0 | Observation Framework | iOS 17+ / macOS 14+*
