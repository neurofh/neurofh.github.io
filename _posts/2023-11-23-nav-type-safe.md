---
title: Oh no, another type safe Compose Navigation library for Android
date: 2023-11-23 01:20:00 +0200
categories: [ Compose, Android, Navigation ]
tags: [ Kotlin, Compose, Android, Navigation ]
---

<img src="/assets/img/compose/compose_logo.png" alt ="" class="center" >

## Intro

We live in a society where another day brings out the most necessary thing for an Android developer,
another Navigation library.

But this isn't one, or it kinda is, but with a twist, a type safe abstraction over the Android Navigation Component for
Compose.

The opinions in this article are mine and reflect my experience, as every article is opinionated, so is this one.

## Why?

I was searching for something that will suit my needs and scale based on the size of the projects I do, which sometimes
consist
of more than 50 screens with lots of dialogs, bottom sheet elements that consume all kind of arguments, none of the
solutions really scaled well without forcing
me to do weird things to just connect my routes, except
for the official Android Navigation library for Compose by Google, no matter how many i've tried
like [Appyx](https://github.com/bumble-tech/appyx), [Decompose](https://github.com/arkivanov/Decompose) the only one
that came close to the
scalability was [Voyager](https://github.com/adrielcafe/voyager) which I use for multi platform projects, really nice job guys.

I hope the Navigation controller by Google gets KMP support so that I can also adapt this library.

I came back to Google's library, but it lacked type safety, so instead of complaining how incomplete it is, as a
developer I found a "solution" to a problem I had, maybe you have it too.

Special thanks to [compose-destinations](https://github.com/raamcosta/compose-destinations) as I borrowed the code for
parsing the arguments from there.

## Creating a type safe navigation

This isn't just a stolen solution, it started from the
bottom [Part #1](/posts/nav-abstraction-part-1/), [Part #2](/posts/nav-abstraction-part-2/), [Part #3](/posts/nav-abstraction-part-3/)
and now we're here.

To understand the structure and the idea, you can read **Part #1**.

## Intro again

Shortly, every NavHost has a root **Graph** and additional **Graphs**, every **Graph** has a *name/route*, also it has a
starting **Destination** and additional
*destinations* where a **Destination** can be one either a: *screen*, *dialog* or *bottom sheet*, every **Destination**
has a *name/route*, *arguments*, *deep links* and *callback arguments* from that destination.

1. **screen** - can
   have [animations](https://developer.android.com/reference/kotlin/androidx/navigation/compose/package-summary#(androidx.navigation.NavGraphBuilder).composable(kotlin.String,kotlin.collections.List,kotlin.collections.List,kotlin.Function1,kotlin.Function1,kotlin.Function1,kotlin.Function1,kotlin.Function2)):
   enter/exit, pop-(enter/exit) transitions, has
   an [AnimatedContentScope](https://developer.android.com/reference/kotlin/androidx/compose/animation/AnimatedContentScope)
   when added to the `NavHost`.
2. **dialog** -
   has [properties](https://developer.android.com/reference/kotlin/androidx/compose/ui/window/DialogProperties#DialogProperties(kotlin.Boolean,kotlin.Boolean,androidx.compose.ui.window.SecureFlagPolicy,kotlin.Boolean,kotlin.Boolean))
   when added to the `NavHost`.
3. **bottom sheet** - has
   a [ColumnScope](https://developer.android.com/reference/kotlin/androidx/compose/foundation/layout/ColumnScope) when
   added to the `NavHost`.

And each **Destination** renders UI, but we'll decouple that from the navigation code for a good reason,
it's important to note that the part that's gonna be responsbile for rendering the UI will be called `Content`,
you can host your UI logic inside a `Content` object that will implement the generated destination.

<img src="/assets/img/nav_type_safe/joke_1.jpg" alt ="" class="center" >

# Setup

Let me shamelessly introduce you my library for type safe navigation, [foSho](https://github.com/FunkyMuse/foSho).

It's called **foSho** which translates **for sure**, anyways, let's write some code.

Just know that every part of the Navigation code we'll write will have `internal` visiblity modifier as a requirement,
no matter if it's *single module* or *multi module*.

It's based on KSP's plugin to generate code with the help of KotlinPoet by Square.

<img src="/assets/img/nav_type_safe/joke_2.jpg" alt ="" class="center" >

The library is available through [JitPack](https://jitpack.io/#FunkyMuse/foSho).

- Add JitPack to your project's settings.gradle

```kotlin
dependencyResolutionManagement {
  ..
  repositories {
    ..
    maven {
      setUrl("https://jitpack.io")
    }
  }
```

- Add the dependency in the build.gradle

```toml
[versions]
foSho = <version>
ksp = <version>

[libraries]
foSho-android = { module = "com.github.FunkyMuse.foSho:navigator-android", version.ref = "foSho" }
foSho-codegen = { module = "com.github.FunkyMuse.foSho:navigator-codegen", version.ref = "foSho" }

[plugins]
ksp = { id = "com.google.devtools.ksp", version.ref = "ksp" }
```

Inside the project build.gradle make sure to add the [KSP](https://github.com/google/ksp) plugin

```kotlin
plugins {
  ..
  alias(libs.plugins.ksp).apply(false)
}
```

Inside the `:app` module

```kotlin
plugins {
  ..
  alias(libs.plugins.ksp)
}
```

```kotlin
dependencies {
  implementation(libs.foSho.android)
  ksp(libs.foSho.codegen)
}
```

If you don't have "feature" modules and you don't plan to have your "screens"/"navigation" outside the `:app` module,
inside the `:app` module make sure to add the KSP argument, **the library is multi module by default**.

```kotlin
ksp {
  arg("foSho.singleModule", "true")
}
```

If you intend to use `Hilt/Dagger/Anvil` with `foSho` and want your generated ViewModel arguments injectable,
inside `:app` module set `foSho.injectViewModelArguments` to "true", default is "false"

```kotlin
ksp {
  arg("foSho.injectViewModelArguments", "true")
}
```

this will generate inectable `ViewModel` arguments by adding the `@Inject` to the constructor, the argument is **false**
by default

```kotlin

class MyViewModelArguments @Inject constructor(
  override val savedStateHandle: SavedStateHandle
)
```

that you can easily inject them ex. using `Hilt`

```kotlin
@HiltViewModel
class MyViewModel @Inject constructor(
  private val myViewModelArguments: MyViewModelArguments
) : ViewModel()
```

## Single module

For single module everything that's navigation logic and UI is going to be in the `:app` module and by now you have
added the required argument

```kotlin
ksp {
  arg("foSho.singleModule", "true")
}
```

Let's explore a scenario for a `Home` graph that is a top level destination and also a starting graph.

In code this will look like this

`HomeGraph.kt`

```kotlin
@Graph(
  startingDestination = Home::class,
  destinations = [HomeDetails::class],
  rootGraph = true
)
internal object HomeGraph

@Destination
@Argument(
  name = "hideBottomNav",
  argumentType = ArgumentType.BOOLEAN,
  defaultValue = DefaultBooleanValueFalse::class
)
internal object Home : Screen

@Destination(generateViewModelArguments = true, generateScreenEntryArguments = false)
@Argument(name = "cardId", argumentType = ArgumentType.INT)
@CallbackArgument(name = "clickedShare", argumentType = ArgumentType.BOOLEAN)
internal object HomeDetails : Screen {
  override val deepLinksList: List<DeepLinkContract>
    get() = listOf(
      DeepLinkContract(
        action = Intent.ACTION_VIEW,
        uriPattern = "custom:url/{cardId}"
      )
    )
}
  ```

`HomeContent.kt`

```kotlin
@Content
internal object HomeContent : HomeDestination {
  @Composable
  override fun AnimatedContentScope.Content() {
    val argumentsFromHomeDetailsDestination = HomeDetailsCallbackArguments.rememberHomeDetailsCallbackArguments()
    argumentsFromHomeDetailsDestination.OnSingleBooleanCallbackArgument(
      key = HomeDetailsCallbackArguments.CLICKED_SHARE,
      onValue = {
        if (it == true) {
          //user has clicked share
        }
      }
    )
    HomeScreen(onClick = {
      Navigator.navigateSingleTop(HomeDetailsDestination.openHomeDetails(cardId = 42))
    })
  }
}
```

`HomeDetailsContent.kt`

```kotlin
@Content
internal object HomeDetailsContent : HomeDetailsDestination {
  @Composable
  override fun AnimatedContentScope.Content() {

  }
}
```

The library is constructed to have a nested navigation by default, meaning that even for one destination, you will
need to create a **Graph**, which is a good practice IMO.

When you annotate your graph with `@Graph` you must provide

- startingDestination: that is your `@Destination` with one of the interfaces `Screen`, `Dialog`, `BottomSheet`
- destinations: just as your `startingDestination`, these are the other ones that are part of the graph
- rootGraph: is `false` by default, but we must have one `rootGraph`

You must note that it's not a good practice to replace the `rootGraph`, because by doing so when you pop the backstack
the navigation component won't know how to do that because it has been changed, this is a common scenario with `Login`
situations,
your **Home** screen should be the starting point, you will then check if the user is logged in or not, you will render
the UI on the `Home` screen only
if the user is logged in, otherwise you will redirect to the `Login` screen and start the flow.

Every `@Destination` that you have created with one of the interfaces `Screen`, `Dialog`, `BottomSheet` can additionally
be annotated with an
`@Argument` which are the type safe arguments generated for that destination, as well as `@CallbackArgument` which are
the type safe callback arguments generated from that destination.

Every `@Destination` doesn't generate type safe arguments by default but you can control that i.e `@Destination(generateViewModelArguments = true, generateScreenEntryArguments = true)`.

The `@Argument` and `@CallbackArgument` have: name, type, whether the argument is nullable and a default value for
which you must implement the `ArgumentValue` interface, there are some pre-defined commonly
used ones as `DefaultBooleanValueFalse` etc...

Whem you click build, the type safe arguments will be generated:
1. `HomeViewModelArguments`, if `"foSho.injectViewModelArguments"` *ksp arg* is set to true these will be injectable otherwise you will have to provide the `SavedStateHandle` manually as a parameter
2. `HomeNavArguments` and a helper function `rememberHomeNavArguments` that you can use to obtain the type safe arguments in the `Content`.

When you add arguments to a destination for example `HomeDetails`, it generates an **open** function, indicating that
you can open that destination with the argument, for example in our case `HomeDetailsDestination.openHomeDetails(cardId = //provide int value)` can be
used to navigate to our details screen by calling `Navigator.navigateSingleTop(HomeDetailsDestination.openHomeDetails(cardId = 42))`.

When you add callback arguments to a destination for example `HomeDetails` it generates `HomeDetailsCallbackArguments` by appending `CallbackArguments` to the
class you annotated, also there's a composable function created to obtain these callback arguments
`HomeDetailsCallbackArguments.rememberHomeDetailsCallbackArguments()` it follows the convention of
`rememberHomeDetailsCallbackArguments` it appends the generated class name for the callback arguments, do note that
each callback argument must be consumed for example you can obtain the argument `clickedShare` it's generated for you
but also you have a function called `consumeClickedShare` generated for you as well, after you use the argument, it's better to consume it otherwise it's gonna be there on config change and process death but if that's your
use case then don't consume it.

Or you can use helpers functions which do that for you like `HomeDetailsCallbackArguments.rememberHomeDetailsCallbackArguments().OnSingleCallbackArgument<String>` that consumes the argument
  when its value is not null, some commonly used are there as well like [OnSingleBooleanCallbackArgument](https://funkymuse.dev/foSho/navigator-android/dev.funkymuse.fosho.navigator.android/-on-single-boolean-callback-argument.html), [OnSingleStringCallbackArgument](https://funkymuse.dev/foSho/navigator-android/dev.funkymuse.fosho.navigator.android/-on-single-string-callback-argument.html) etc..

You can also have the deep links by implementing the list, I decided that it's just a plain `listOf` so that we don't
clutter the annotations, but it may be just an annotation in the future as well?

```kotlin
override val deepLinksList: List<DeepLinkContract>
  get() = listOf(
    DeepLinkContract(
      action = Intent.ACTION_VIEW,
      uriPattern = "custom:url/{cardId}"
    )
  )
  ```

Every building block follows the same structure as the Android Navigation Component for Compose by Google.

When you build the project or (`./gradlew kspDebugKotlin`) for the defined destination `Home` you get a generated `HomeDestination`,
meaning `Destination` suffix
is added to your code and you will implement that destination for your `@Content`.

There's only one function you will need to override and there you can host your logic and the UI for it.

For the `HomeGraph` the suffix `NavigationGraph` is added, meaning you get `HomeNavigationGraph` generated for you,
all of that is aggregated in the `GraphFactory`.

After that you can hook up everything using the extension function [addGraphs](https://funkymuse.dev/foSho/navigator-android/dev.funkymuse.fosho.navigator.android/add-graphs.html?query=fun%20NavGraphBuilder.addGraphs(navigationGraphs:%20ImmutableList%3CAndroidGraph%3E,%20showAnimations:%20Boolean%20=%20true,%20onGraph:%20(androidGraph:%20AndroidGraph)%20-%3E%20Unit?%20=%20null)) inside your `NavHost`.

If you decide to use the arguments inside the `Content` you must add `CompositionLocalProvider(LocalNavHostController provides navController)` once your `rememberNavController` is available, you can check the single module sample.

Finally you can add the [NavHostControllerEvents](https://funkymuse.dev/foSho/navigator-android/dev.funkymuse.fosho.navigator.android/-nav-host-controller-events.html?query=fun%20NavHostControllerEvents(navigator:%20NavigatorDirections,%20navController:%20NavHostController,%20onEvent:%20(navigatorEvent:%20NavigatorIntent)%20-%3E%20Unit?%20=%20null)) to finish hooking up everything
```kotlin
NavHostControllerEvents(
  navigator = Navigator,
  navController = navController,
)
```

You use the [Navigator](https://funkymuse.dev/foSho/navigator-android/dev.funkymuse.fosho.navigator.android.navigator.impl/-navigator/index.html) to dispatch commands as well as consume those commands.

You can check out the [sample](https://github.com/FunkyMuse/foSho/tree/main/app) for a single module integration.

## Multi module

The multi module code follows the same principle, you don't need to add any KSP arguments but there's a way to structure
your
navigation code so that it's available globally.

To achieve a global navigation you have to create an umbrella module, you can call it `navigator` or however you see fit
where you would write the code
for the Graphs.

- `:navigator` module; `UserAccountGraph.kt`

```kotlin
@Graph(
startingDestination = UserAccountDetails::class,
destinations = [
    EditAccountDetails::class,
    DeleteAccount::class,
    ChangePassword::class,
  ]
)
internal object UserAccountGraph

@Destination
internal object UserAccountDetails : Screen

@Destination
internal object ChangePassword : BottomSheet

@Destination
internal object DeleteAccount : Dialog

@Destination(generateScreenEntryArguments = true)
@Argument(name = "email", argumentType = ArgumentType.STRING)
@CallbackArgument(name = "isAccountUpdated", argumentType = ArgumentType.BOOLEAN)
internal object EditAccountDetails : Screen

```

- `:navigator` module; `HomeGraph.kt`

```kotlin
@Graph(startingDestination = Home::class, rootGraph = true)
internal object HomeGraph

@Destination
@Argument(name = "hideBottomNav", argumentType = ArgumentType.BOOLEAN, defaultValue = DefaultBooleanValueFalse::class)
internal object Home : Screen
  ```

Your `:navigator` module acts as the only point where you have the navigation code and nothing else, here you
can control the `arg` whether to generate injectable view model arguments through

  ```kotlin
ksp {
  arg("foSho.injectViewModelArguments", "true")
}
  ```

Then inside your feature modules you will add the `:navigator` module

- `:feature:user_details`: `UserAccountDetailsContent.kt`

```kotlin
@Content
internal object UserAccountDetailsContent : UserAccountDetailsDestination {
  @Composable
  override fun AnimatedContentScope.Content() {

}
}
```

and also make sure to add the `:feature:user_details` and `:navigator` in your `:app` module so that it will aggregate everything for you into
the `GraphFactory`.

## Until next time

At the moment of writing of this article I don't have a multi module sample project, but I am in the process of making one very soon and will update the Github repo with a link to it and this article.
I hope the aforementioned code is enough for you to use the library in a multi module environment.

Thank you for your wholehearted attention, I hope that this library will help you scale your projects like it did help me,
I decided to share this library as I had it created for my personal projects.

