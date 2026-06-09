## Swift/SwiftUI Guardrails (iOS 18+)

Full reference patterns live in `docs/setup/swift/`. This section contains only the always-active hard rules.

### Hard Rejections — Never Use in New Code

| Banned | Replacement |
|---|---|
| `ObservableObject`, `@Published` | `@Observable` |
| `@StateObject`, `@ObservedObject` | `@State` (owned) / `@Bindable` (child) |
| `@EnvironmentObject` | `@Environment(MyService.self)` |
| `import Combine` in new files | `async/await`, `.task`, `AsyncStream` |
| `DispatchQueue` in new service code | `actor`, `async func`, `@MainActor` |
| `.onAppear { Task { } }` | `.task { }` |
| `*ViewModel.swift` for a single view | Logic as computed properties in the View |

### Property Wrapper Quick Reference

| Situation | Use |
|---|---|
| View-private transient state (sheet flag, search text, form field) | `@State` |
| Pass mutable access to a child view | `@Binding` |
| Bind to a property on an `@Observable` object in a child view | `@Bindable` |
| Shared app/feature-level service | `@Environment(MyService.self)` |
| Core Data query results owned by the rendering view | `@FetchRequest` |
| Persistent cross-launch UI state | `@SceneStorage` / `@AppStorage` |

### Pre-Implementation Checklist

Before marking any story done, verify:
- [ ] No `ObservableObject`, `@Published`, `@StateObject`, `@ObservedObject`, `@EnvironmentObject`
- [ ] No new `*ViewModel.swift` file created to serve a single view
- [ ] Shared services injected via `@Environment(MyService.self)`, not init parameters
- [ ] View-lifecycle async uses `.task` or `.task(id:)`, not `.onAppear { Task { } }`
- [ ] New tests use `import Testing`, not `import XCTest`

> Full patterns, code examples, and architecture guidance: `docs/setup/swift/`
