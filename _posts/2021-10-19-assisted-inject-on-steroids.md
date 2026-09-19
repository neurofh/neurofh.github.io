---
title: Assisted Inject for less boilerplate?
date: 2021-10-19 10:10:00 +0100
categories: [Dagger, Hilt]
tags: [Android, Kotlin, Hilt]
---

<img src="/assets/img/assisted_steroids/1.jpg" alt ="" class="center" alt ="" >

Lives of so many developers were easy before `@AssistedInject` came to `Dagger2`, they still are, but as usual `Dagger2` is explained with so many complications that even an experienced developer can go bonkers, when in fact it's dead simple.

Assisted injection is a dependency injection (DI) pattern that is used to construct an object where some parameters may be provided by the DI framework and others must be passed in at creation time (a.k.a “assisted”) by the developer.

A factory is typically responsible for combining all of the parameters and creating the object, as you would generally do with a factory if you were building a component where you **bind** (using `@BindsInstance`) the dependencies that are used (passed down) in that component, you can take a look at [Dagger Part #5](/posts/dagger-part-5/) where we created the **SingletonComponent** and other *components*.

The building blocks of a dependency that you can inject with an assistance from the DI library (Dagger), consists of:
1. The dependency itself, annotated with `@AssistedInject constructor()`
2. The factory (interface), annoated with `@AssistedFactory`
3. Create (or better named method by yourself) within the factory that returns the assisted preference that also accepts the same function parameters as the ones you'll need to assist your DI library with, in order to create your assisted dependency.

<img src="/assets/img/assisted_steroids/2.jpeg" alt ="" class="center">

Today you'll build something you've used for sure, one time shared preference manager, on steroids.

*This article was written to use shared preference instead of Data store for the sake of brevity.*

---

We create a contract for our one time preference manager, that way we can just swap out the functionality later down the road if we decide to migrate to "Data store". 

```kotlin
interface OneTimePrefContract {
    fun setOneTimeShown()
    val oneTimePrefs: SharedPreferences
    val isOneTimeShown: Boolean
    val isOneTimeNotShown : Boolean
}
```

Our manager would be something simple
```kotlin
class OneTimePreferenceManager @AssistedInject constructor(
    @ApplicationContext private val context: Context,
    @Assisted private val prefsTag: String,
    @Assisted private val prefsBooleanKey: String
)
```
The context is provided by the DI, the tag and the key used for the shared preferences would be assisted by you.

This sample is using two `@Asssited` parameters of the same type, in this case `Dagger2` doesn't know which is which, for this case we have to name them, similarly to `@Named` but just passing a name parameter at `@Assisted`.

```kotlin
class OneTimePreferenceManager @AssistedInject constructor(
    @ApplicationContext private val context: Context,
    @Assisted(PREFS_TAG_KEY) private val prefsTag: String,
    @Assisted(PREFS_BOOLEAN_KEY) private val prefsBooleanKey: String
) {
    
    private companion object {
        private const val PREFS_TAG_KEY = "prefsTag"
        private const val PREFS_BOOLEAN_KEY = "prefsBoolean"
    }
}
```

Now `Dagger2` is satisfied with your sacrifice.


In order to have our `OneTimePreferenceManager` provided everywhere else, we need a factory, go ahead and implement the contract and create a factory as follows
```kotlin
class OneTimePreferenceManager @AssistedInject constructor(
    @ApplicationContext private val context: Context,
    @Assisted(PREFS_TAG_KEY) private val prefsTag: String,
    @Assisted(PREFS_BOOLEAN_KEY) private val prefsBooleanKey: String
) : OneTimePrefContract {
    
    private companion object {
        private const val PREFS_TAG_KEY = "prefsTag"
        private const val PREFS_BOOLEAN_KEY = "prefsBoolean"
    }
    
    @AssistedFactory
    interface OneTimePrefFactory {
        fun create(
            @Assisted(PREFS_TAG_KEY) prefsTag: String,
            @Assisted(PREFS_BOOLEAN_KEY) prefsBooleanKey: String
        ): OneTimePref
    }
    
    override val oneTimePrefs: SharedPreferences
        get() = context.getSharedPreferences(
            prefsTag,
            Context.MODE_PRIVATE
        )
    override val isOneTimeShown get() = oneTimePrefs.getBoolean(prefsBooleanKey, false)
    override val isOneTimeNotShown get() = !isOneTimeShown
    override fun setOneTimeShown() = oneTimePrefs.edit { putBoolean(prefsBooleanKey, true) }
}
```

Inside let's say your `DashboardFragment` when your user has opened the dashboard for the first time and you want to show him how to use it through some steps.
```kotlin
@AndroidEntryPoint
class DashboardFragment : Fragment() {
    
    @Inject
    lateinit var oneTimePrefFactory: OneTimePreferenceManager.OneTimePrefFactory
    
    private val walkThroughPrefs by lazy { oneTimePrefFactory.create("walk-through", "isWalkThroughShown") }

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        if (walkThroughPrefs.isOneTimeNotShown){
            showWalkThrough()
        }
    }
}
```
<img src="/assets/img/assisted_steroids/3.jpeg" alt ="" class="center">

As you no doubt have guessed by now, there are limitations that you have to deal with when using `@AssistedInject`
1. `@AssistedInject` types(dependencies) cannot be injected directly, only the `@AssistedFactory` can be injected
2. `@AssistedInject` types(dependencies) cannot be scoped

Let's say we want to scope our *"WalkThroug One time preference"* to a `Fragment`, we can't.

This code 
```kotlin
    @Inject
    lateinit var oneTimePrefFactory: OneTimePreferenceManager.OneTimePrefFactory
    
    private val walkThroughPrefs by lazy { oneTimePrefFactory.create("walk-through", "isWalkThroughShown") }

```    
is boilerplate-ish to be written, it hurts the eyes *(inserts burning eyes meme)*.

---
For this purpose we'll work with what we have been blessed by `Hilt` and `Kotlin`.

```kotlin
@FragmentScoped
class WalkThroughPrefsProvider @Inject constructor(
    private val oneTimePrefFactory: OneTimePreferenceManager.OneTimePrefFactory
) : OneTimePrefContract by oneTimePrefFactory.create(
    WALK_THROUGH_PREFS,
    WALK_THROUGH_PREFS_SHOWN_KEY
) {
    private companion object {
        private const val WALK_THROUGH_PREFS = "walkThrough"
        private const val WALK_THROUGH_PREFS_SHOWN_KEY = "walkThroughKey"
    }
}
```

```kotlin
@AndroidEntryPoint
class DashboardFragment : Fragment() {
    
    @Inject
    lateinit var walkThroughPrefsProvider: WalkThroughPrefsProvider

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        if (walkThroughPrefsProvider.isOneTimeNotShown){
            showWalkThrough()
        }
    }
}
```

<img src="/assets/img/assisted_steroids/cool.gif" alt ="" class="center">

That's all for this blog post, stay tuned for more.