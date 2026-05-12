# Sandbooks — Swift 6 Learning Guides

Kumpulan panduan belajar Swift 6 dan Swift Concurrency dalam Bahasa Indonesia. Setiap file mencakup teori mendalam, kapan digunakan, kapan tidak, dan real use cases siap pakai.

---

## Urutan Materi

### Bagian 1 — Swift Concurrency (Fondasi)

| # | File | Topik |
|---|---|---|
| 1 | [swift-concurrency-complete.md](swift-concurrency-complete.md) | Panduan lengkap Swift Concurrency dari Swift 5.5 hingga Swift 6.2 — `async/await`, `Task`, `actor`, `AsyncStream`, dan semua fitur baru di tiap versi |
| 2 | [swift-isolation-swift6.md](swift-isolation-swift6.md) | Deep dive Swift Isolation — isolation domain, `@MainActor`, `Sendable`, `nonisolated`, region-based isolation, dan migration guide Swift 5 → Swift 6 |
| 3 | [swift-isolation-vip-uikit.md](swift-isolation-vip-uikit.md) | Contoh praktis VIP (View–Interactor–Presenter) di UIKit dengan Swift Isolation, dilengkapi Task Cancellation, `async let`, `TaskGroup`, `AsyncStream`, dan `withCheckedContinuation` |
| 4 | [swift-concurrency-swift5x.md](swift-concurrency-swift5x.md) | Adopsi Swift Concurrency di project Swift 5.x — semua API (`async/await`, `actor`, `Sendable`, `AsyncStream`, `Continuation`), implementasi VIP UIKit, migration guide GCD → async/await, tips & best practices |

---

### Bagian 2 — Fitur Bahasa Swift 6

| # | File | Topik |
|---|---|---|
| 5 | [swift6-typed-throws.md](swift6-typed-throws.md) | Typed Throws — `throws(ErrorType)`, generic throws, menghilangkan overhead existential boxing pada error handling |
| 6 | [swift6-noncopyable-types.md](swift6-noncopyable-types.md) | Noncopyable Types (`~Copyable`) — move semantics, RAII pattern, `consuming`/`borrowing`, garansi `deinit` |
| 7 | [swift6-borrowing-consuming.md](swift6-borrowing-consuming.md) | Borrowing & Consuming parameter modifiers — eliminasi copy yang tidak perlu, ownership semantics, performa tanpa `inout` |
| 8 | [swift6-pack-iteration.md](swift6-pack-iteration.md) | Parameter Packs & Pack Iteration — type-safe variadic generics, `for value in repeat each values`, compile-time unrolling |

---

### Bagian 3 — Konkurensi Tingkat Rendah

| # | File | Topik |
|---|---|---|
| 9 | [swift6-synchronization.md](swift6-synchronization.md) | Synchronization — `Mutex<Value>`, `Atomic<Value>`, memory ordering, perbandingan performa vs `actor` |

---

### Bagian 4 — State Management & UI

| # | File | Topik |
|---|---|---|
| 10 | [swift6-observable.md](swift6-observable.md) | Observable — `@Observable` macro, fine-grained dependency tracking, `@Bindable`, `withObservationTracking`, integrasi UIKit, perbandingan dengan `ObservableObject` |

---

### Bagian 5 — Ekosistem & Tooling

| # | File | Topik |
|---|---|---|
| 11 | [swift6-swift-testing.md](swift6-swift-testing.md) | Swift Testing — `@Test`, `@Suite`, `#expect`, `#require`, parameterized tests, tags, traits, dan migrasi dari XCTest |
| 12 | [swift6-package-access-level.md](swift6-package-access-level.md) | Package Access Level — keyword `package`, arsitektur multi-modul SPM, berbagi implementasi antar modul tanpa ekspos ke publik |
| 13 | [swift6-conditional-compilation.md](swift6-conditional-compilation.md) | Conditional Compilation — menggunakan fitur Swift 6 di project Swift 5, strategi migrasi bertahap 5 fase, compatibility layer |

---

## Prasyarat

- Xcode 16+ untuk Swift 6
- Pengetahuan dasar Swift (struct, class, protocol, closure)
- Tidak wajib paham concurrency sebelumnya — Bagian 1 dimulai dari dasar

## Lisensi

Bebas digunakan untuk keperluan belajar.
