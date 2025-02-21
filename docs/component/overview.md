
# Component Overview

Every component represents a piece of logic with lifecycle and optional pluggable UI. 

## Examples

### Simplest Component Example

Here is an example of simple Counter component:

```kotlin
class Counter {
    private val _value = MutableValue(State())
    val state: Value<State> = _value

    fun increment() {
        _value.reduce { it.copy(count = it.count + 1) }
    }

    data class State(val count: Int = 0)
}
```

### Jetpack/JetBrains Compose UI Example

```kotlin
@Composable
fun CounterUi(counter: Counter) {
    val state by counter.state.asState()

    Column {
        Text(text = state.count.toString())

        Button(onClick = counter::increment) {
            Text("Increment")
        }
    }
}
```

If you are using only Jetpack/JetBrains Compose UI, then most likely you can use its `State` and `MutableState` directly, without intermediate `Value`/`MutableValue` from Decompose.

### SwiftUI Example

```swift
struct CounterView: View {
    private let counter: Counter
    @ObservedObject
    private var state: ObservableValue<CounterState>

    init(_ counter: Counter) {
        self.counter = counter
        self.state = ObservableValue(counter.state)
    }

    var body: some View {
        VStack(spacing: 8) {
            Text(self.state.value.text)
            Button(action: self.counter.increment, label: { Text("Increment") })
        }
    }
}
```

## `ComponentContext`

Each component has an associated [`ComponentContext`](https://github.com/arkivanov/Decompose/blob/master/decompose/src/commonMain/kotlin/com/arkivanov/decompose/ComponentContext.kt) which implements the following interfaces:

* [RouterFactory](https://github.com/arkivanov/Decompose/blob/master/decompose/src/commonMain/kotlin/com/arkivanov/decompose/RouterFactory.kt), so you can create nested `Routers` in your `Componenets`
* [StateKeeperOwner](https://github.com/arkivanov/Decompose/blob/master/decompose/src/commonMain/kotlin/com/arkivanov/decompose/statekeeper/StateKeeperOwner.kt), so you can preserve any state during configuration changes and/or process death
* [InstanceKeeperOwner](https://github.com/arkivanov/Decompose/blob/master/decompose/src/commonMain/kotlin/com/arkivanov/decompose/instancekeeper/InstanceKeeperOwner.kt), so you can retain instances in your components (like with AndroidX ViewModels)
* [LifecycleOwner](https://github.com/arkivanov/Decompose/blob/master/decompose/src/commonMain/kotlin/com/arkivanov/decompose/lifecycle/LifecycleOwner.kt), so each component has its own lifecycle
* [BackPressedDispatcherOwner](https://github.com/arkivanov/Decompose/blob/master/decompose/src/commonMain/kotlin/com/arkivanov/decompose/backpressed/BackPressedDispatcherOwner.kt), so each component can handle back button events

So if a component requires any of the above features, just pass the `ComponentContext` via the component's constructor. You can use the delegation pattern to add the `ComponentContext` to `this` scope:

```kotlin
class Counter(
    componentContext: ComponentContext
) : ComponentContext by componentContext {

    // The rest of the code
}
```

When instantiating a root component we have to create `ComponentContext` manually. There is [DefaultComponentContext](https://github.com/arkivanov/Decompose/blob/master/decompose/src/commonMain/kotlin/com/arkivanov/decompose/DefaultComponentContext.kt) which is the default implementation class of the `ComponentContext`. There are also handy helper functions provided by Jetpack/JetBrains Compose extension modules.

## Child components

Decompose provides ability to organize components into trees, so each parent component is only aware of its immediate children. Hence the name of the library - "Decompose". You decompose your project by multiple independent reusable components. When adding a sub-tree into another place (reusing), you only need to satisfy its top component's dependencies.

There are two common ways to add a child component:

- Using the `Router` - prefer this option when a navigation between components is required. Please head to the [Router documentation page](https://arkivanov.github.io/Decompose/router/overview/) for more information.
- Adding a component as a permanent child - prefer this option if the child component should exist as long as the parent component.

### Adding a permanent child component

To create the `ComponentContext` for a permanent child component use `ComponentContext.childContext(...)` extension function:

```kotlin
class SomeParent(
    componentContext: ComponentContext
) : ComponentContext by componentContext {

    private val counter = Counter(childContext(key = "Counter"))
}
```

> ⚠️ Never pass parent's `ComponentContext` to children, always use either the `Router` or the `childContext(...)` function.

