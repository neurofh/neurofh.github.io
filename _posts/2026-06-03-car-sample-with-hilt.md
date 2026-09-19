---
title: "Hilt in Android Auto: From Manual Factories to a Cleaner Screen Provider"
date: 2026-06-05 11:30:00 +0200
categories: [Android, Android Auto, DI]
---

If you've ever followed the [Android Auto Codelabs](https://developer.android.com/codelabs/car-app-library-fundamentals), you've seen the "sample" way of building a Car App. It works, it's functional, but as soon as you try to scale it beyond a simple demo, you hit a wall: **Dependency Management**.

Keep in mind this is only one way to wire things, I'm pretty sure many others exist, I'm exploring things on Android Auto and Android Wear lately and how to connect and wire dependencies in an easier manner as I'm planning to implement both of these in a toy project.

In the codelab, everything is manually instantiated. You create a `PlacesRepository` in the `MainScreen`, then maybe you pass it to the `DetailScreen`, and before you know it, you're playing \"pass the dependency\" through five different constructors.

It works... but at a mental cost. And let's be honest, manual DI works but up until a certain point (as pointed in my previous articles).

Today, we're going to fix that. We'll start with the Codelab code and move it into a world of compile-time safety.

## The Starting Point (The "Before")

In the base codelab, your `CarAppService` looks something like this:

```kotlin
// before
class PlacesCarAppService : CarAppService() {
    override fun onCreateSession(): Session {
        return PlacesSession()
    }
}
```

And your screens? They're just creating their own repositories.

```kotlin
// before
class MainScreen(carContext: CarContext) : Screen(carContext) {
    override fun onGetTemplate(): Template {
        val repository = PlacesRepository() // Manual instantiation
        // ...
    }
}
```

This makes testing a nightmare.

## Step 1: Introducing Hilt

First, we need to make the Car App aware of Hilt. Since `Session` and `Screen` aren't standard Android entry points, we start at the `CarAppService`.

```kotlin
@AndroidEntryPoint
class PlacesCarAppService : CarAppService() {
    @Inject lateinit var sessionProvider: Provider<PlacesSession>

    override fun onCreateSession(): Session {
        return sessionProvider.get()
    }
}
```

Wait, `sessionProvider.get()`? That's right. Because `PlacesSession` only has static dependencies (like our `ScreenProvider`), we don't even need Assisted Injection here. We can use a standard `@Inject` constructor and tell Hilt to provide it via a `Provider`.

## Step 2: The Assisted Injection Rabbit Hole

The first attempt at DI usually leads here: `@AssistedInject` for every screen.

```kotlin
class MainScreen @AssistedInject constructor(
    @Assisted carContext: CarContext,
    private val repository: PlacesRepository
) : Screen(carContext) {

    @AssistedFactory
    interface Factory {
        fun create(carContext: CarContext): MainScreen
    }
}
```

This is better! Hilt provides the `repository`, and we provide the `carContext`. But here's the catch: now `PlacesSession` has to inject `MainScreen.Factory` if `MainScreen` navigates to `DetailScreen`, it needs `DetailScreen.Factory`.

You end up with constructors that look like a factory for factories. If you have 20 screens, your Session or your \"Navigator\" becomes a dependency nightmare. **The boilerplate is back, just in a different outfit.**

## Step 3: The \"Screen Provider\" (Multi-map Binding)

If you've used Hilt for `ViewModels`, you know you don't inject `ViewModelFactory` manually. Hilt uses multi-binding to give you a map of all ViewModels. We can do the exact same thing for Car App Screens.

First, we define a common interface:

```kotlin
fun interface ScreenFactory {
    fun create(carContext: CarContext, param: Any?): Screen //param Any is for demonstration
}
```

Then, we create a specialized `ScreenProvider` to handle the resolution logic.

```kotlin
@Singleton
class ScreenProvider @Inject constructor(
    private val screenFactories: Map<Class<out Screen>, @JvmSuppressWildcards ScreenFactory>
) {
    fun get(carContext: CarContext, screenClass: Class<out Screen>, param: Any? = null): Screen {
        return screenFactories[screenClass]?.create(carContext, param) ?: error(\"No factory for $screenClass\")
    }
}
```

And bind our screens in a Hilt module. This is where the magic happens: instead of the screen defining its factory, we define how to create the screen in a central module.

```kotlin
@Module
@InstallIn(SingletonComponent::class)
object ScreenModule {
    @Provides
    @IntoMap
    @ScreenKey(MainScreen::class)
    fun provideMainScreen(
        repo: PlacesRepository,
        navigatorProvider: Provider<Navigator> // Defer retrieval to break cycles
    ) = ScreenFactory { ctx, _ ->
        MainScreen(ctx, repo, navigatorProvider.get())
    }

    @Provides
    @IntoMap
    @ScreenKey(DetailScreen::class)
    fun provideDetailScreen(repo: PlacesRepository) = ScreenFactory { ctx, param ->
        DetailScreen(ctx, param as Int, repo)
    }
}
```

## Step 4: The Navigator (The \"After\")

Now, we wrap the `ScreenProvider` into a `Navigator` singleton. This provides a clean API for pushing new screens, separating the \"how to create\" from the \"how to navigate\".

```kotlin
@Singleton
class Navigator @Inject constructor(
    private val screenProvider: ScreenProvider
) {
    fun push(manager: ScreenManager, ctx: CarContext, clazz: Class<out Screen>, param: Any? = null) {
        manager.push(screenProvider.get(ctx, clazz, param))
    }
}
```

## Step 5: Breaking the Cycle

You might have noticed something in Step 3: `Provider<Navigator>`.

This is a classic Dagger/Hilt \"gotcha.\" The `Navigator` needs the `ScreenProvider`, which needs the `Map<Class, ScreenFactory>`, but the `ScreenFactory` (specifically the one for `MainScreen`) needs the `Navigator`.

**This is a circular dependency.** By injecting `Provider<Navigator>`, we tell Hilt: \"Don't give me the Navigator now; give it to me only when I actually call `.create()` on the factory.\" This breaks the cycle and keeps the compiler happy.

## The Final Result: Zero-Boilerplate Sessions

With this infrastructure in place, our `PlacesSession` becomes incredibly clean. It only needs the `ScreenProvider` to resolve the initial screen:

```kotlin
// In PlacesSession.kt
class PlacesSession @Inject constructor(
    private val screenProvider: ScreenProvider
) : Session() {

    override fun onCreateScreen(intent: Intent): Screen {
        return screenProvider.get(carContext, MainScreen::class.java)
    }
}
```

And our screens? They use the `Navigator` and stay **completely clean**. No factories, no `@AssistedInject` boilerplate, just standard constructors:

```kotlin
// after
class MainScreen(
    carContext: CarContext,
    private val repository: PlacesRepository,
    private val navigator: Navigator
) : Screen(carContext) {

    override fun onGetTemplate(): Template {
        // ...
        .setOnClickListener {
            navigator.push(screenManager, carContext, DetailScreen::class.java, it.id)
        }
    }
}
```

## Wrapping Up

We've moved from manual instantiation to a scalable, modular architecture.
*   **CarAppService** injects a Provider for the Session.
*   **Session** uses the **ScreenProvider** to resolve the initial screen.
*   **Navigator** uses the **ScreenProvider** to push new screens.
*   **Screens** use the **Navigator** to move around.

This approach can be improved and it's a part of me toying around with Android Auto and Android Wear so keep in mind that this is not to be used in production as the `ScreenFactory`'s `Any` castings can be improved further with custom KSP processor basically what [ViewModelComponentBuilder](https://github.com/google/dagger/blob/master/hilt-android/main/java/dagger/hilt/android/internal/builders/ViewModelComponentBuilder.java) achieves that you can build on top of Hilt.

There's two more approaches that you can use and that's dispatching events to a Channel/SharedFlow that'll break the cycle which is essentially an event bus and a callback mechanism so your previous layer handles it or you push it one more layer up but essentially `Screen` is what should handle nav, this sample is just to highlight some of the painful aspects when you attempt to do the "right thing" without too much abstractions throw around when building a non-parked Car app.

Also you can use `Lazy` instead of `Provider` but that won't bring any significant improvements anyway.

This approach gives you the an idea how to leverage usage of Hilt while respecting the unique lifecycle of the Android Car App Library.

**P.S.** If you think this is cool, wait until you see how I do this in **Metro**. If there's enough interest, I'll write a follow-up on how Metro eliminates some of the boilerplate Hilt/Dagger have.

Until then, stay car ready and hydrated. 🚗💨
