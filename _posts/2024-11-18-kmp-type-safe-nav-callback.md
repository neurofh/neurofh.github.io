---
title: Achieving Type-safe Navigation Results in AndroidX Compose for KMP
date: 2024-11-18 07:10:00 +0200
categories: [Kotlin, KMP, Compose Multiplatform, Navigation]
tags: [Kotlin, KMP, Compose Multiplatform, Navigation, Type Safety]
---

## The Quest for Perfect Navigation in Compose Multiplatform

![Compose Logo](/assets/img/compose/compose_logo.png)

Navigation in Kotlin Multiplatform (KMP) applications, especially those using Compose Multiplatform, presents an interesting challenge. While the ecosystem offers several excellent libraries (such as Voyager, Decompose, and Precompose), I consistently find myself returning to Google's AndroidX Navigation. Its elegant API design and flexibility make it a compelling choice, particularly because it doesn't impose architectural decisions that might become restrictive later.

The recent collaboration between Google and JetBrains to bring AndroidX Navigation into the KMP/Compose Multiplatform world has been transformative, especially with its enhanced type safety features. However, one crucial piece remains missing from the type safety puzzle: **callback results between screens**. Let's explore how to implement this vital feature while maintaining complete type safety.

## Understanding the Navigation Architecture

AndroidX Navigation's core functionality revolves around `BackStackEntry` entities in a parent-child relationship:
- The `currentEntry` represents your active screen
- The `previousEntry` acts as its parent
- These entries can communicate and share state bidirectionally

This hierarchical relationship creates an ideal foundation for implementing type-safe navigation results, enabling child screens to pass data back to their parents in a structured and reliable way.

## Implementing Type-safe Navigation Results

### Step 1: Setting Up the Result Passing Mechanism

First, we'll create a mechanism to pass results from the current screen back to its parent:

```kotlin
private const val RESULT_NAME_CONST = "_result" 

@OptIn(InternalSerializationApi::class)
@ExperimentalSerializationApi
inline fun <reified T : Any> NavController.setResultToPreviousEntry(
    data: T,
) {
    val serializer = data::class.serializer() as KSerializer<T>
    val result = Json.encodeToString(serializer, data)
    val resultKey = serializer.descriptor.serialName + RESULT_NAME_CONST
    previousBackStackEntry?.savedStateHandle?.set(key = resultKey, value = result)
}
```

This extension function handles the serialization of our result data and stores it in the parent entry's saved state handle. Using serialization ensures both type safety and graceful handling of process death scenarios.

### Step 2: Consuming the Result

For the parent screen, we need a reliable way to observe and consume these results:

```kotlin
@OptIn(InternalSerializationApi::class)
@Composable
inline fun <reified T : Any> NavBackStackEntry.callbackArgument(
    noinline onValue: (T) -> Unit
) {
    val resultSerializer = T::class.serializer()
    val key = resultSerializer.resultName()
    
    DisposableEffect(this) {
        val observer = LifecycleEventObserver { _, event ->
            if (event == Lifecycle.Event.ON_RESUME && savedStateHandle.contains(key)) {
                val result = savedStateHandle.consumeArgument<String>(key)!!
                val decoded = Json.decodeFromString(resultSerializer, result)
                onValue(decoded)
            }
        }
        lifecycle.addObserver(observer)
        onDispose {
            lifecycle.removeObserver(observer)
        }
    }
}

@PublishedApi
internal fun <T> SavedStateHandle.consumeArgument(key: String) = remove<T>(key)

@OptIn(ExperimentalSerializationApi::class)
@PublishedApi
internal fun <T> KSerializer<T>.resultName() = descriptor.serialName + RESULT_NAME_CONST
```

The lifecycle observer plays a crucial role here, especially in handling dialog scenarios where the parent might not be in the resumed state when receiving the callback.

### Understanding the Lifecycle Observer's Importance

Think of your Android app as a stack of cards. When everything is normal, your main screen (the parent screen) is on top and fully "awake" (in the RESUMED state). It's like having the spotlight on that screen - it can see and react to everything happening.

When you open a dialog, it's like placing a new card on top of your parent screen. The parent screen enters a kind of "standby mode" (the PAUSED state) - imagine the lights dimming while the dialog takes the spotlight.

This is where things become interesting: When your dialog needs to send information back to the parent screen (like when selecting a filter option), we need to be careful. Without proper lifecycle handling, it's like trying to pass a note to someone who's half asleep - they might miss it entirely!

That's why we implement the LifecycleEventObserver - think of it as a personal assistant for the parent screen that:
1. Monitors when the parent screen "wakes up" (returns to RESUMED state)
2. Checks for any pending messages (results)
3. Ensures proper message delivery

Without this careful handling:
- Results could be lost during dialog dismissal
- The parent screen might process results prematurely
- Information might be delivered redundantly
- The app could become unstable

This becomes particularly important when considering real-world scenarios:
- Device rotation (configuration changes)
- Incoming phone calls
- App backgrounding
- Nested dialog chains

Our lifecycle observer guarantees that:
1. Results remain safely stored until they can be processed
2. Processing occurs only when the parent screen is fully active
3. Each result is processed exactly once
4. The system remains robust during device state changes

## Real-world Example: Search Features

Let's implement a practical example with a search feature and its filters:

### Defining Our Destinations

```kotlin
@Serializable
data object SearchGraph {
    @Serializable
    data object SearchScreen

    @Serializable
    data class SearchFiltersScreen(
        val orderBy: String,
        val sortBy: String,
        val category: String? = null,
        val query: String? = null,
    )
}
```

### Setting Up the Navigation Graph

```kotlin
fun NavGraphBuilder.searchNavigation(navHostController: NavHostController) {
    navigation<SearchGraph>(startDestination = SearchGraph.SearchScreen()) {
        searchContent(navHostController)
        searchFilterContent(navHostController)
    }
}
```

### Implementing the Search Screen

```kotlin
private fun NavGraphBuilder.searchContent(navHostController: NavHostController) {
    composable<SearchGraph.SearchScreen>() { entry ->
        val viewModel = viewModel<SearchViewModel>()
        val pagingItems = viewModel.pagingData.collectAsLazyPagingItems()
        
        entry.callbackArgument<SearchFiltersScreen> {
            viewModel.updateSearchFilter(it)
        }
        
        SearchScreen(
            pagingItems = pagingItems,
            openSearchFilter = {
                navHostController.navigate(
                    SearchGraph.SearchFiltersScreen(
                        orderBy = "date",
                        sortBy = "ascending"
                    )
                )
            }
        )
    }
}
```

### Implementing the Filter Screen

```kotlin
private fun NavGraphBuilder.searchFilterContent(navHostController: NavHostController) {
    composable<SearchGraph.SearchFiltersScreen>() { entry ->
        val args = entry.toRoute<SearchGraph.SearchFiltersScreen>()
        SearchFilterScreen(
            args = args,
            updateSearchFilterParams = { newOrder, newSort, newCategory, newQuery ->
                navHostController.setResultToPreviousEntry(
                    SearchFiltersScreen(
                        orderBy = newOrder,
                        sortBy = newSort,
                        category = newCategory,
                        query = newQuery
                    )
                )
            }
        )
    }
}
```

## Looking Ahead

While this implementation provides a robust solution for type-safe navigation results in Compose Multiplatform, official support in the AndroidX Navigation library would be welcome. Until then, this approach offers a reliable and type-safe way to handle navigation results while maintaining process death compatibility.

The solution's elegance lies in its simplicity and type safety. While navigation results could potentially be handled at the repository layer, integrating them into the navigation system offers distinct advantages:
- Automatic process death handling
- Predictable data flow
- Complete type safety
- Seamless navigation integration

### Pro Tips
- Always plan for process death in your navigation implementation (at least when targeting mobile)
- Choose meaningful serializable names for navigation arguments
- Maintain a clean, organized navigation graph
- Test your navigation flows thoroughly, especially with lifecycle events

![KMP hehe](/assets/img/kmp/kmp_joke.jpg)

## Wrapping Up

As someone passionate about the Kotlin Multiplatform ecosystem, it's exciting to contribute improvements to our existing tools. This implementation of type-safe navigation results demonstrates how we can build upon the solid foundation provided by AndroidX Navigation and Compose Multiplatform.

While recovering from surgery, I wanted to share this solution with the community. I hope it helps streamline your KMP/CMP development process!

Stay tuned for more articles about Kotlin Multiplatform and Compose Multiplatform. The ecosystem continues to evolve rapidly, bringing new possibilities with each update!

Happy coding! 🚀

---

*P.S. Found this helpful? Share it with your KMP colleagues! Don't forget to explore the official AndroidX Navigation documentation for more navigation patterns and best practices.*
