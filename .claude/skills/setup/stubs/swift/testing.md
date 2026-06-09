# Testing Conventions

> iOS 18+ | Swift Testing framework, Core Data test setup, and concurrency-safe tests.

---

## Swift Testing — Required for All New Tests

Use `import Testing` for all new test files. `import XCTest` is legacy — do not add new XCTest classes.

```swift
// ✅ Modern
import Testing

@Suite("OrderService")
struct OrderServiceTests {

    @Test func createOrder_storesAllFields() throws {
        let order = Order(id: "123", total: 49.99, status: .pending)
        #expect(order.id == "123")
        #expect(order.total == 49.99)
        #expect(order.status == .pending)
    }

    @Test(arguments: [OrderStatus.pending, .processing, .shipped])
    func orderStatus_isTransitionable(from status: OrderStatus) {
        #expect(status.canTransition(to: .cancelled))
    }
}

// ❌ Legacy
import XCTest
class OrderServiceTests: XCTestCase {
    func testCreateOrder() {
        XCTAssertEqual(order.id, "123")
    }
}
```

---

## Core Data Tests — Always In-Memory

Never use the default persistent store in tests. Always create a `PersistenceController(inMemory: true)` instance. This is faster, isolated per test, and requires no cleanup.

```swift
// ✅ In-memory controller for every test
@Suite("CustomerService")
struct CustomerServiceTests {
    let controller = PersistenceController(inMemory: true)
    var context: NSManagedObjectContext { controller.container.viewContext }

    @Test func createCustomer_persistsToStore() throws {
        let service = CustomerService(context: context)
        let customer = try service.create(name: "Alice", email: "alice@example.com")
        #expect(customer.name == "Alice")
        #expect(customer.email == "alice@example.com")
    }
}

// ❌ Real persistent store — leaves state between tests, slow
let controller = PersistenceController.shared
```

---

## Testing `@MainActor`-Isolated Types

When testing `@Observable` classes or functions annotated with `@MainActor`, annotate the entire test struct or the individual test method with `@MainActor`.

```swift
// ✅ @MainActor on the suite — all tests run on main actor
@Suite("AuthService")
@MainActor
struct AuthServiceTests {

    @Test func signIn_setsAuthenticated() async throws {
        let service = AuthService()
        try await service.signIn(credentials: .mock)
        #expect(service.isAuthenticated == true)
    }
}
```

---

## What to Test

Test **service and business logic** — not SwiftUI rendering. Views are not unit-testable in isolation; their correctness is verified by running the app.

**Test these:**
- Service methods (create, update, delete, fetch)
- Computed properties with non-trivial logic
- Actor state transitions
- Validation functions

**Do not write unit tests for:**
- View `body` layout
- Navigation stack state
- Core Data `@FetchRequest` results (that's an integration concern; use the simulator)
