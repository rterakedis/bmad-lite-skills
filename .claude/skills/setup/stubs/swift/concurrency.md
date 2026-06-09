# Concurrency & Safe Threading

> iOS 18+ / Swift 6 | Async patterns, actor isolation, and UI thread safety.

---

## `.task` — The Default for View-Lifecycle Async

`.task` starts when the view appears and **automatically cancels when the view disappears**. This is the correct default for any async work tied to a view's presence on screen.

```swift
// ✅ Auto-cancelled on disappear
.task { await orderService.refresh() }

// ✅ Re-triggered on value change — replaces manual debounce
.task(id: searchText) {
    try? await Task.sleep(for: .milliseconds(400))
    await viewModel.search(searchText)
}

// ✅ Fire-and-forget after user action where cancellation is wrong
Button("Save") { Task { try? await save() } }

// ❌ Leaks if view disappears before task completes
.onAppear { Task { await loadData() } }

// ❌ Combine for something async/await handles natively
cancellable = publisher.sink { ... }
```

---

## `@MainActor` — UI Thread Safety

Any `@Observable` class that mutates properties observed by a view must annotate those mutations with `@MainActor`. Apply it at the class level when all methods drive UI state; apply it at the method level when only some mutations are UI-facing.

```swift
// ✅ Class-level @MainActor — all property mutations on main thread
@Observable @MainActor final class AuthService {
    private(set) var isAuthenticated = false

    func signIn(credentials: Credentials) async throws {
        isAuthenticated = try await APIClient.authenticate(credentials)
    }
}

// ✅ Method-level — background work, then main-thread update
@Observable final class SyncEngine {
    @MainActor private(set) var lastSyncDate: Date?

    func sync() async {
        let result = await performBackgroundSync()   // background
        await MainActor.run { lastSyncDate = result } // main thread
    }
}
```

---

## `actor` — Protecting Mutable State Across Contexts

Use `actor` for service types whose state is accessed from multiple concurrency contexts. Replace any `DispatchQueue` used for thread-safety with an `actor`.

```swift
// ✅ actor with @MainActor output property
actor NetworkMonitor {
    @MainActor private(set) var isConnected = true

    func startMonitoring() {
        // actor-isolated internal state mutations are safe
    }
}

// ❌ Raw DispatchQueue — no Swift concurrency isolation
private let networkQueue = DispatchQueue(label: "net", qos: .utility)
```

---

## `TaskGroup` — Structured Parallel Work

```swift
// ✅ Parallel fetches with structured concurrency
func loadDashboard() async throws -> Dashboard {
    try await withThrowingTaskGroup(of: DashboardSection.self) { group in
        group.addTask { try await fetchOrders() }
        group.addTask { try await fetchNotifications() }
        group.addTask { try await fetchProfile() }

        var sections: [DashboardSection] = []
        for try await section in group {
            sections.append(section)
        }
        return Dashboard(sections: sections)
    }
}

// ❌ Unstructured concurrent tasks with no cancellation coordination
async let orders = fetchOrders()    // only fine for 2-3 known tasks
```

---

## Swift 6 Strict Concurrency

With Swift 6 strict concurrency enabled, the compiler enforces actor isolation at compile time. Key implications:

- `@Observable` classes accessed from multiple isolation contexts require explicit `@MainActor` or `nonisolated` annotations.
- Closures capturing `self` in `Task {}` must be `@Sendable` — avoid capturing mutable state without isolation.
- Prefer structured concurrency (`async let`, `TaskGroup`, `.task`) over `Task { }` to get automatic cancellation and isolation inference.
- Test methods that touch `@MainActor`-isolated types must be annotated `@MainActor`.

```swift
// ✅ Swift 6 safe — @Sendable closure, no implicit self capture
.task {
    await orderService.refresh()  // crosses isolation boundary cleanly
}

// ❌ May warn under Swift 6 — captures mutable state without isolation
Task {
    self.localState = await fetchSomething()
}
```
