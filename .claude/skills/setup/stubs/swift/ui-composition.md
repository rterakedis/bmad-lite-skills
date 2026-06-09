# UI Composition & Layout Rules

> iOS 18+ | Preventing massive views, subview extraction, and layout decisions.

---

## The 50-Line Rule for View Body

If `var body: some View` exceeds ~50 lines, it must be decomposed. This is not a style preference — large bodies create re-render surface area and make conditional logic harder to audit.

Decomposition priority order:
1. Extract to a private computed property returning `some View` (zero overhead, stays in same file)
2. Extract to a private nested `struct` view (use when the subview has its own `@State`)
3. Extract to a separate file in the same feature folder (use when the component is reused or complex enough to warrant independent testing)

```swift
// ✅ Computed property extraction — stays in same file, no overhead
struct OrderListView: View {
    var body: some View {
        VStack {
            headerSection
            orderList
            footerSection
        }
    }

    private var headerSection: some View {
        HStack { ... }
    }

    private var orderList: some View {
        List(orders) { OrderRow(order: $0) }
    }
}

// ✅ Nested struct — when the subview needs its own @State
private struct ExpandableOrderRow: View {
    let order: Order
    @State private var isExpanded = false

    var body: some View { ... }
}

// ❌ Inline logic that should be a subview
var body: some View {
    VStack {
        if orders.isEmpty {
            VStack(spacing: 16) {
                Image(systemName: "tray")
                    .font(.largeTitle)
                Text("No orders yet")
                    .font(.headline)
                Button("Browse") { ... }
            }  // extract this to EmptyOrdersView
        }
        // ... 80 more lines
    }
}
```

---

## `List` vs `ScrollView` + Lazy Stack

| Scenario | Use |
|---|---|
| Homogeneous rows with swipe actions, reorder, or section headers | `List` |
| Mixed-content feed, card grid, or custom spacing between items | `ScrollView` + `LazyVStack` |
| Fixed small number of items (under ~20, no scroll needed) | `VStack` |
| Grid layout | `ScrollView` + `LazyVGrid` |

`List` provides automatic row recycling, swipe-to-delete, edit mode, and accessibility. Prefer it for tabular data. Do not wrap `List` inside `ScrollView` — they conflict.

```swift
// ✅ List for rows with swipe actions
List(orders) { order in
    OrderRow(order: order)
        .swipeActions { Button("Delete", role: .destructive) { delete(order) } }
}

// ✅ LazyVStack for a feed with mixed content types
ScrollView {
    LazyVStack(spacing: 16) {
        ForEach(feedItems) { item in
            FeedCard(item: item)
        }
    }
    .padding(.horizontal)
}

// ❌ VStack for a potentially unbounded list — loads everything at once
ScrollView {
    VStack {
        ForEach(orders) { OrderRow(order: $0) }
    }
}
```

---

## View Modifier Conventions

- Custom modifiers that apply to many views belong in `Shared/Components/` as `ViewModifier` conformances.
- Do not chain more than 5-6 modifiers on a single view inline — extract to a custom modifier.
- Never use `.appearance()` UIKit proxy calls from SwiftUI — use native SwiftUI styling.

```swift
// ✅ Custom modifier for a repeated pattern
struct CardStyle: ViewModifier {
    func body(content: Content) -> some View {
        content
            .padding()
            .background(.regularMaterial)
            .clipShape(RoundedRectangle(cornerRadius: 12))
            .shadow(radius: 2)
    }
}

extension View {
    func cardStyle() -> some View { modifier(CardStyle()) }
}

// Usage
OrderRow(order: order).cardStyle()
```

---

## Conditional Views

Prefer `if/else` inside `body` over opaque `AnyView` wrapping. `AnyView` defeats SwiftUI's diffing and should only appear when returning from a function that must erase the type.

```swift
// ✅ if/else in body — SwiftUI diffs correctly
var body: some View {
    if isLoading {
        ProgressView()
    } else {
        OrderList()
    }
}

// ❌ AnyView — erases type, breaks diffing
func content() -> AnyView {
    if isLoading { return AnyView(ProgressView()) }
    return AnyView(OrderList())
}
```
