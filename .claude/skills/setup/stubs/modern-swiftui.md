## Modern SwiftUI Patterns (iOS 17+)

### The Foundational Principle

SwiftUI views are already the ViewModel. A `View` struct holds `@State`, computes derived values as properties, and declares layout — that is the ViewModel job. Do not create a separate class to mirror this. Every `*ViewModel.swift` file you reach for is a sign the logic should either live directly in the View, in a pure function on a Service struct, or in a shared `@Observable` store.

---

### Property Wrapper Selection

| Situation | Correct wrapper | Wrong wrappers |
|---|---|---|
| View-private, transient UI state (sheet flag, search text, form field, toggle) | `@State` | `@StateObject`, `@ObservedObject` |
| Pass mutable access to a child view | `@Binding` | Passing `Array` snapshots |
| Create a binding to a property on an `@Observable` class | `@Bindable` | `@ObservedObject` |
| Inject a shared app/feature-level service | `@Environment(MyService.self)` | `@EnvironmentObject` |
| Own Core Data query results in the rendering view | `@FetchRequest` | Passing `FetchedResults` as Array to children |
| Persistent, cross-launch UI state (tab selection, scroll position) | `@SceneStorage` / `@AppStorage` | `@State` with manual UserDefaults |

**Never use:** `ObservableObject`, `@Published`, `@StateObject`, `@ObservedObject`, `@EnvironmentObject`. These are iOS 14–16 patterns superseded by `@Observable` + `@Environment`.

---

### When to Use `@Observable`

`@Observable` is for **shared, reference-type stores and services** — objects whose state must outlive a single view or be shared across the view tree.

**Correct uses:** App-level services (`SubscriptionService`, `AuthService`), feature-level stores shared by multiple sibling views, long-lived async engines (StoreKit, CloudKit, network monitors).

**Incorrect uses:** A class that exists solely to serve one `*View.swift` file; a class whose properties mirror `@FetchRequest` results; a class that holds form field strings.

**Injection pattern — always `@Environment`, never init parameters:**
```swift
// ✅ App root injects once
@State private var subscriptionService = SubscriptionService()
ContentView().environment(subscriptionService)

// ✅ Descendant pulls from environment
@Environment(SubscriptionService.self) private var subscriptionService

// ❌ Never prop-drill via init
let subscriptionService: SubscriptionService
```

---

### Keep Views as Lightweight State Declarations

A View struct should hold `@State` for local UI, declare `@FetchRequest` for Core Data, and express business logic as **computed properties** — not methods called in `body`, and not separate ViewModel files.

```swift
// ✅ Filtering logic as a computed property in the View
private var filteredCustomers: [Customer] {
    let base = customers.filter { matchesFilter($0) }
    guard !searchText.isEmpty else { return base }
    return base.filter { $0.name.localizedCaseInsensitiveContains(searchText) }
}

// ❌ Wrong — separate @Observable class duplicates FetchRequest state
@Observable final class CustomerListViewModel {
    var customers: [Customer] = []   // manual copy of @FetchRequest
    var searchText: String = ""      // should be @State in the View
}
```

---

### Async: Use `.task`, Not `Task { }` in `.onAppear`

`.task` starts when the view appears and **automatically cancels when the view disappears**.

```swift
// ✅ View-lifecycle async — auto-cancelled on disappear
.task { await subscriptionService.refreshEntitlement(context: moc) }

// ✅ Re-triggerable on value change — replaces debounce patterns
.task(id: address) {
    try? await Task.sleep(for: .milliseconds(600))
    await validateAddress(address)
}

// ✅ Fire-and-forget after user action where cancellation is wrong
Button("Save") { Task { try? await save() } }

// ❌ Wrong — leaks if view disappears
.onAppear { Task { await loadData() } }

// ❌ Wrong — Combine for something async/await handles natively
cancellable = publisher.sink { ... }
```

---

### Concurrency Isolation

- `@Observable` classes that mutate UI-driving state must annotate mutations with `@MainActor`.
- Use `actor` for service types with state accessed from multiple concurrency contexts. A `DispatchQueue` is the pre-Swift-Concurrency equivalent — replace it.
- Use structured concurrency (async functions called from `.task`) over unstructured `Task { }` wherever possible.
- Annotate test methods touching `@Observable`/`@MainActor` types with `@MainActor`.

```swift
// ✅ actor with @MainActor output property
actor NetworkMonitor {
    @MainActor private(set) var isConnected = true
}

// ❌ Raw DispatchQueue — no Swift concurrency isolation
private var networkQueue = DispatchQueue(label: "net", qos: .utility)
```

---

### Testing: Use Swift Testing, Not XCTest

- `import Testing`, use `@Test` and `#expect`, group with `@Suite`.
- Use `@Test(arguments:)` for parameterized cases.
- Always `PersistenceController(inMemory: true)` — never mock Core Data.

```swift
// ✅ Modern
import Testing
@Suite("CustomersService") struct CustomersServiceTests {
    @Test func createCustomer_storesAllFields() throws { #expect(customer.name == "Alice") }
}

// ❌ Legacy
import XCTest
class CustomersServiceTests: XCTestCase {
    func testCreate() { XCTAssertEqual(customer.name, "Alice") }
}
```

---

### Modern SwiftUI Code Review Checklist

Before marking any story done:
- [ ] No new `*ViewModel.swift` files created for a single view
- [ ] No `ObservableObject`, `@Published`, `@StateObject`, `@ObservedObject`, `@EnvironmentObject`
- [ ] No `Array(fetchResults)` passed from parent to child view
- [ ] All shared services injected via `@Environment(MyService.self)`, not init parameters
- [ ] Async work tied to view appearance uses `.task` or `.task(id:)`, not `.onAppear { Task { } }`
- [ ] No `import Combine` in new files
- [ ] No raw `DispatchQueue` in new service code — use `actor` or `async` functions
- [ ] New tests use `import Testing`, not `import XCTest`
- [ ] Sheet/cover presentation state latched in `@State` on `.onAppear`, never derived from `@FetchRequest`
