---
title: Abstract your Android Navigation for Compose, part 2
date: 2023-05-17 17:01:00 +0200
categories: [Hilt, Compose, Android, Navigation]
tags: [Hilt, Kotlin, Compose, Android, Navigation]
---
<img src="/assets/img/compose/compose_logo.png" alt ="" class="center" >

## Intro
---
Welcome to the second part of the navigation abstraction in Compose using Google's navigation component, in this blog post we'll see the abstraction in code.

## Abstraction
---
In order to understand this code, it's highly recommended to read the [Part #1](/posts/nav-abstraction-part-1/).

We have to create an abstraction for:
- Graph (with starting destination and unique route/ID)
- Destination (with animations, arguments, deep links)
- How to connect destinations to a graph
- Navigation graph entries connected with destinations that have logic and Composable UI render ability
- How to connect navigation entries to a graph
- Arguments (to and from)

### Destination
---

We will start with our basic setup, every destination, doesn't matter if it's a `dialog`, `bottom sheet` or a `screen` would need one of the following:
- a required destination string a.k.a route
- list of arguments (optional)
- list of deep links (optional)

```kotlin
@Immutable
interface NavigationDestination {

    fun destination(): String

    val arguments: List<NamedNavArgument>
        get() = emptyList()

    val deepLinks: List<NavDeepLink>
        get() = emptyList()
}
```

### Animated destination
---
We also would want to create one common place for our animatable destinations (at this time only screens support animations, but hopefully in the future dialogs and bottom sheets).

On Android via the navigation component we have 4 types of transitions:
- *enterTransition*
- *exitTransition*
- *popEnterTransition*
- *popExitTransition*

Translated into the Compose world, we have access to the `AnimatedContentTransitionScope`, for example:
You have access to the `initial` and `target` transitionary state, additionaly you can even utilize `NavigationDestination`'s `List<NamedNavArgument>` **arguments** to drive the screen's transitions.

```kotlin
@Stable
interface AnimatedDestination {

    val enterTransition: (AnimatedContentTransitionScope<NavBackStackEntry>.() -> EnterTransition?)?

    val exitTransition: (AnimatedContentTransitionScope<NavBackStackEntry>.() -> ExitTransition?)?

    val popEnterTransition: (AnimatedContentTransitionScope<NavBackStackEntry>.() -> EnterTransition?)?
        get() = enterTransition

    val popExitTransition: (AnimatedContentTransitionScope<NavBackStackEntry>.() -> ExitTransition?)?
        get() = exitTransition
}
```

For example
```kotlin
 override val enterTransition: (AnimatedContentTransitionScope<NavBackStackEntry>.() -> EnterTransition?) = {
        when (initialState.arguments?.getInt(ON_OPEN_DIRECTIONS)) {
            FROM_WALK_THROUGH -> slideInHorizontally { it }
            FROM_SETTINGS -> fadeIn()
            else -> slideIntoContainer(AnimatedContentTransitionScope.SlideDirection.Up)
        }
    }
```

### Screen destination
---
This type of destination supports Animations, hence we'll just implement `AnimatedDestination`.
```kotlin
@Stable
interface ScreenDestination : NavigationDestination, AnimatedDestination {

    override val enterTransition: (AnimatedContentTransitionScope<NavBackStackEntry>.() -> EnterTransition?)?
        get() = null

    override val exitTransition: (AnimatedContentTransitionScope<NavBackStackEntry>.() -> ExitTransition?)?
        get() = null
}
```

### Dialog destination
---
This type of destination requires of us to provide [DialogProperties](https://developer.android.com/reference/kotlin/androidx/compose/ui/window/DialogProperties) which can modify the appearance of a dialog, **do keep in mind that this type of a destination doesn't have a surface** (i.e like a rounded corner background, you have to create one common modifier or a wrapped composable around your UI, otherwise you'll be placing your UI elements on a transparent background).

```kotlin
@Stable
interface DialogDestination : NavigationDestination {
    val dialogProperties: DialogProperties
}
```

### Bottom sheet destination
---
This type of destination is for our Bottom Sheets that will live in a modal bottom sheet layout.
```kotlin
/**
 * Keep in mind that the parent of this type is a
 * [androidx.compose.foundation.layout.Column]
 */
@Stable
interface BottomSheetDestination : NavigationDestination
```
### Graph
---
Each graph has a starting destination and a unique route/id, we can utilize the commonly used `NavigationDestination` which every type implements already.

```kotlin
@Immutable
interface NavigationGraph {
    val startingDestination: NavigationDestination
    val route: String
}
```
### Graph entry
---
Each navigation graph entry is added to a parent (Graph), to provide a better developer experience, each navigation graph entry should have the `NavigationDestination` it belongs to, where with just one click you can see the arguments, deep links, animations etc...

The `Render` function is where you'll host your logic, it's stability is kinda hacked so that it doesn't recompose unnecessarily, only when the `StableHolder` notifies that the `NavHostController` changed.

```kotlin
@Immutable
interface NavigationGraphEntry {
    val navigationDestination: NavigationDestination
    @Composable
    fun Render(controller: StableHolder<NavHostController>)
}
```

### Navigation graph entry arguments
---
Each navigation graph entry can receive arguments, for our arguments we'll use this brilliant [repo](https://github.com/raamcosta/compose-destinations/tree/main/compose-destinations/src/main/java/com/ramcosta/composedestinations/navargs) and won't try to reinvent the wheel, but for our type safety we would need the following.

```kotlin
@Immutable
interface NavigationEntryArguments {
    val currentNavBackStackEntry: NavBackStackEntry
    val arguments: Bundle? get() = currentNavBackStackEntry.arguments
}
```
For obtaining the arguments in an easier way, we'd use context receivers, take a look at [this class](https://github.com/FunkyMuse/Composed/blob/main/navigation/src/main/java/com/funkymuse/composed/navigation/arguments/nav_entry/NavEntryArgumentsExtensions.kt).

This implementation would help you obtain the arguments you've sent to the screen inside the `@Composable fun Render()`.

### Navigation view model arguments
---
Each navigation entry if accompanied by a view model can receive a type safe arguments with a simple implementation of
```kotlin
interface ViewModelNavigationArguments {
    val savedStateHandle: SavedStateHandle
}
```
For obtaining the arguments in an easier way, we'd use context receivers again, take a look at [this class](https://github.com/FunkyMuse/Composed/blob/main/navigation/src/main/java/com/funkymuse/composed/navigation/arguments/viewmodel/ViewModelArgumentsExtensions.kt).

### Arguments to previous destination
---
We want to have callbacks too, usually ones we send to the previous screen.

To support them one can approach this in the following way (due note that you might change the naming to your own suitable way).

```kotlin
/**
 * Name should start ArgumentsFromXDestination where X destination is the one you're sending arguments from
 */
@Immutable
interface AddArgumentsToPreviousDestination {
    val navHostController: StableHolder<NavHostController>
    fun consumeArgumentsAtReceivingDestination()
    private val currentBackStackEntrySavedStateHandle: SavedStateHandle?
        get() = navHostController.item.currentBackStackEntry?.savedStateHandle
    private val previousBackStackEntrySavedStateHandle: SavedStateHandle?
        get() = navHostController.item.previousBackStackEntry?.savedStateHandle

    fun <T> addArgumentToNavEntry(route: String, key: String, value: T?) {
        navHostController.item.getBackStackEntry(route).savedStateHandle[key] = value
    }

    fun consumeArgument(key: String) {
        currentBackStackEntrySavedStateHandle?.set(key, null)
    }

    fun <T> set(key: String, value: T?) {
        previousBackStackEntrySavedStateHandle?.set(key, value)
    }

    fun consumeArguments(vararg key: String) {
        key.forEach(::consumeArgument)
    }
}
```

The required `navHostController` takes care for adding the arguments to and from, this class supports everything that can be stored in a `Bundle`/`SavedStateHandle`, it's use case is single shot event which you must `consumeArgumentsAtReceivingDestination`.

In order for us to send arguments to the previous destination we have to set those and consume them, we set them *FROM* the current entry to the *PREVIOUS* entry.

We can also build a helper function to observe the callback, with more at this [link](https://github.com/FunkyMuse/Composed/blob/main/navigation/src/main/java/com/funkymuse/composed/navigation/arguments/previous_destination/AddArgumentsToPreviousDestinationExtensions.kt)
```kotlin

@Composable
fun <T> AddArgumentsToPreviousDestination.OnSingleCallbackArgument(
    key: String,
    initialValue: T? = null,
    consumeWhen: (T?) -> Boolean = { it != null },
    onValue: (T?) -> Unit
) {
    val currentEntryHandle = navHostController.currentEntry.savedStateHandle
    val value by currentEntryHandle.getStateFlow(key, initialValue).collectAsState(initial = initialValue)
    val currentOnValue by rememberUpdatedState(newValue = onValue)
    LaunchedEffect(value) {
        currentOnValue(value)
        if (consumeWhen(value)) {
            consumeArgument(key)
        }
    }
}
```
The idea is whenever the key changes and if we provide an initial value, we react upon the value and then we consume it if it's been received as a non null or a custom predicate.

Inside the `Render` function
```kotlin
val argumentsFromConfirmationDialog = remember { ArgumentsFromConfirmationDialog(controller) }
    ArgumentsFromConfirmationDialog.OnSingleBooleanCallbackArgument(
            key = ArgumentsFromConfirmationDialog.REQUEST_RESULT_KEY_CONFIRMATION,
            onValue = { refreshSettingsClick ->
                if (refreshSettingsClick == true) {
                    settingsViewModel.fetchSettings()
                }
            }
        )
```

### Navigator intent
---
We would need to send commands to navigate, we can model this pretty easily.

We have:
- Navigating away from a screen (you might know it as a navigate up)
- Popping the current backstack (additionally with a custom logic)
- Top level destination (usually known from bottom navigation in case you're not familiar with the term)
- Directions (that cover everything else)

All of that in code
```kotlin
@Stable
sealed interface NavigatorIntent {

    object NavigateUp : NavigatorIntent

    object PopCurrentBackStack : NavigatorIntent

    data class PopBackStack(
        val route: String,
        val inclusive: Boolean,
        val saveState: Boolean = false,
    ) : NavigatorIntent

    class Directions(
        val destination: String,
        val builder: NavOptionsBuilder.() -> Unit = {},
    ) : NavigatorIntent {
        override fun toString(): String = "destination=$destination"
    }

    data class NavigateTopLevel(val route: String) : NavigatorIntent
}
```

### Navigator
---
We would want to transform this navigator intent into directions so we would split this into two interfaces, one that's only the commands and the other one that's having these commands transformed into a consumable `Flow`.

```kotlin
@Immutable
interface NavigatorDirections {
    val directions: Flow<NavigatorIntent>
}
```

```kotlin
@Immutable
interface Navigator {

    fun navigateUp()

    fun navigateSingleTop(
        destination: String,
        builder: NavOptionsBuilder.() -> Unit = { launchSingleTop = true },
    )

    fun navigate(
        destination: String,
    )

    fun navigateTopLevel(
        destination: String,
    )

    fun navigate(
        destination: String,
        builder: NavOptionsBuilder.() -> Unit,
    )

    fun popBackStack(
        destination: String,
        inclusive: Boolean,
        saveState: Boolean = false,
    )

    fun popCurrentBackStack()

    fun navigate(navigatorIntent: NavigatorIntent)
}
```
We include most of the helper functions you'll need in case you use this `Navigator` directly or if accessed from a deepend layer there's the `fun navigate(navigatorIntent: NavigatorIntent)`.

## Closing notes
---
This will be the end of this blog post, thanks for your attention, continue to [Part #3](/posts/nav-abstraction-part-3/) where we'll wrap up the implementation.
