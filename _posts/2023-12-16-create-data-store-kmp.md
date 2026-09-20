---
title: Jetpack Preferences DataStore in Kotlin Multiplatform (KMP)
date: 2023-12-16 16:15:00 +0200
categories: [ KMP, Kotlin, Multiplatform, AndroidX ]
tags: [ Kotlin, KMP, Android, Desktop, iOS, Multiplatform, AndroidX ]
---

<img src="/assets/img/kmp/kmp.png" class="center" alt ="">

I decided to delve into Kotlin Multiplatform (KMP), and I quickly became hooked. 
That was a year ago, and I was initially hesitant due to the numerous `build.gradle` configurations that made my head spin. 
However, one year later, things are looking promising for the ecosystem, especially now that it's stable.

I love the technology and how it provides a way to craft a multi-platform target solution.

I must admit that, among all the awesome libraries JetBrains has created, Ktor for the backend was my favorite (sorry Laravel, you'll remain as the project with the best developer documentation I've ever encountered).

As Android developers, we are presented with a new way to store user preferences, called [DataStore](https://developer.android.com/jetpack/androidx/releases/datastore).
Fortunately for us, it comes with KMP support (currently iOS, Android, Desktop, [WASM not yet](https://issuetracker.google.com/issues/316376114)).

I won't bore you with the details of what DataStore is.

tl:dr*; type safe, async, cross platform, easily configurable, performant
key-value storage*.

## Project and library setup
To create your KMP project, it's best to use the [KMP Wizard](https://kmp.jetbrains.com/).

Once you import the project in Android Studio/IntelliJ Idea or Fleet, you will need to add the DataStore Preferences dependency.

```toml
[versions]
androidx-data-store = "1.1.0-alpha07" #at the time of this article

[libraries]
androidx-data-store-core = { module = "androidx.datastore:datastore-preferences-core", version.ref = "androidx-data-store" }
```

Inside your commonMain, add the library dependency.

```kotlin
  commonMain {
    dependencies {
      ...
      implementation(libs.androidx.data.store.core)
    }
  }
```
## Library setup

In KMP, we have `expect` and `actual` for providing platform-specific implementations.

Inside our commonMain, we'll create a new file DataStorePreferences.kt, where we'll provide an expected implementation for other platforms to implement.

`commonMain -> DataStorePreferences.kt`
```kotlin
expect fun dataStorePreferences(
    corruptionHandler: ReplaceFileCorruptionHandler<Preferences>?,
    coroutineScope: CoroutineScope,
    migrations: List<DataMigration<Preferences>>,
): DataStore<Preferences>

internal const val SETTINGS_PREFERENCES = "settings_preferences.preferences_pb"

internal fun createDataStoreWithDefaults(
    corruptionHandler: ReplaceFileCorruptionHandler<Preferences>? = null,
    coroutineScope: CoroutineScope = CoroutineScope(Dispatchers.IO + SupervisorJob()),
    migrations: List<DataMigration<Preferences>> = emptyList(),
    path: () -> String,
) = PreferenceDataStoreFactory
    .createWithPath(
        corruptionHandler = corruptionHandler,
        scope = coroutineScope,
        migrations = migrations,
        produceFile = {
            path().toPath()
        }
    )
```
The code is self-explanatory. 

I used a `createDataStoreWithDefaults` helper function to avoid importing the okio path, this function is used by the library to create the data store file.

`androidMain -> DataStorePreferences.android.kt`
```kotlin
actual fun dataStorePreferences(
    corruptionHandler: ReplaceFileCorruptionHandler<Preferences>?,
    coroutineScope: CoroutineScope,
    migrations: List<DataMigration<Preferences>>,
): DataStore<Preferences> = createDataStoreWithDefaults(
    corruptionHandler = corruptionHandler,
    migrations = migrations,
    coroutineScope = coroutineScope,
    path = {
        File(applicationContext.filesDir, "datastore/$SETTINGS_PREFERENCES").path
    }
)
```

Now, you might wonder where we got the applicationContext as it's not a parameter.

Well, there's a trick you can use that [Firebase has been using](https://firebase.blog/posts/2016/12/how-does-firebase-initialize-on-android) since a long time ago, but now we have the AndroidX Startup library to make it even easier.

```toml
[versions]
androidx-startup = "1.1.1"
[libraries]
androidx-startup = { module = "androidx.startup:startup-runtime", version.ref = "androidx-startup" }
```

Inside the build.gradle file, add the Android library.

```kotlin
androidMain {
  dependencies {
    api(libs.androidx.startup)
  }
}
```

Then create the initializer.

`androidMain -> ContextProvider.kt`
```kotlin
internal lateinit var applicationContext: Context
    private set

data object ContextProviderInitializer

class ContextProvider: Initializer<ContextProviderInitializer> {
    override fun create(context: Context): ContextProviderInitializer {
        applicationContext = context.applicationContext
        return ContextProviderInitializer
    }

    override fun dependencies(): List<Class<out Initializer<*>>> = emptyList()
}
```
Don't worry about memory leaks; we're holding the Application context, which is safe throughout your app's lifecycle.

The last step is inside your AndroidManifest.xml; make sure to add the provider before the closing tag `</application>`.

```xml
        <provider
            android:name="androidx.startup.InitializationProvider"
            android:authorities="${applicationId}.androidx-startup"
            android:exported="false"
            tools:node="merge">

            <meta-data
                android:name="context.ContextProvider"
                android:value="androidx.startup" />
        </provider>
```

`iosMain -> DataStorePreferences.ios.kt`

```kotlin
@OptIn(ExperimentalForeignApi::class)
actual fun dataStorePreferences(
    corruptionHandler: ReplaceFileCorruptionHandler<Preferences>?,
    coroutineScope: CoroutineScope,
    migrations: List<DataMigration<Preferences>>,
): DataStore<Preferences> = createDataStoreWithDefaults(
    corruptionHandler = corruptionHandler,
    migrations = migrations,
    coroutineScope = coroutineScope,
    path = {
        val documentDirectory: NSURL? = NSFileManager.defaultManager.URLForDirectory(
            directory = NSDocumentDirectory,
            inDomain = NSUserDomainMask,
            appropriateForURL = null,
            create = false,
            error = null,
        )
        (requireNotNull(documentDirectory).path + "/$SETTINGS_PREFERENCES")
    }
)
```

`desktopMain -> DataStorePreferences.jvm.kt`
```kotlin
actual fun dataStorePreferences(
    corruptionHandler: ReplaceFileCorruptionHandler<Preferences>?,
    coroutineScope: CoroutineScope,
    migrations: List<DataMigration<Preferences>>,
): DataStore<Preferences> =
    createDataStoreWithDefaults(
        corruptionHandler = corruptionHandler,
        migrations = migrations,
        coroutineScope = coroutineScope,
        path = { SETTINGS_PREFERENCES }
    )
```

Keep in mind that the actual path for Desktop is just for demonstration purposes. Make sure to discuss with your team what's best.

## Sample implementation

There are a few more things left to do, so bear with me for a moment. We are not done just yet.

We will create a simple class for demonstration purposes, nothing fancy.

```kotlin
interface AppPreferences {
    suspend fun isDarkModeEnabled(): Boolean
    suspend fun changeDarkMode(isEnabled: Boolean): Preferences    
}

internal class AppPreferencesImpl(
    private val dataStore: DataStore<Preferences>
) : AppPreferences {

    private companion object {
        private const val PREFS_TAG_KEY = "AppPreferences"
        private const val IS_DARK_MODE_ENABLED = "prefsBoolean"
    }

    private val darkModeKey = booleanPreferencesKey("$PREFS_TAG_KEY$IS_DARK_MODE_ENABLED")

    override suspend fun isDarkModeEnabled() = dataStore.data.map { preferences ->
        preferences[darkModeKey] ?: false
    }.first()

    override suspend fun changeDarkMode(isEnabled : Boolean) = dataStore.edit { preferences ->
        preferences[darkModeKey] = isEnabled
    }
}
```

Since DataStore needs to be instantiated only once, we need to make sure we do that in a somewhat fashionable way.

Let's create a CoroutinesComponent and CoreComponent which will help us in this journey.

```kotlin
interface CoroutinesComponent {
    val mainImmediateDispatcher: CoroutineDispatcher
    val applicationScope: CoroutineScope
}

internal class CoroutinesComponentImpl private constructor() : CoroutinesComponent {

    companion object {
        fun create(): CoroutinesComponent = CoroutinesComponentImpl()
    }

    override val mainImmediateDispatcher: CoroutineDispatcher = Dispatchers.Main.immediate
    override val applicationScope: CoroutineScope
        get() = CoroutineScope(
            SupervisorJob() + mainImmediateDispatcher,
        )
}
```

```kotlin
interface CoreComponent : CoroutinesComponent {
    val appPreferences: AppPreferences
}

internal class CoreComponentImpl internal constructor() : CoreComponent,
    CoroutinesComponent by CoroutinesComponentImpl.create() {

    private val dataStore: DataStore<Preferences> = dataStorePreferences(
        corruptionHandler = null,
        coroutineScope = applicationScope + Dispatchers.IO,
        migrations = emptyList()
    )

    override val appPreferences : AppPreferences = AppPreferencesImpl(dataStore)
}
```

What we did was provide an application scope so that the data store would use. Now let's create a way to initialize all these components.

```kotlin
object ApplicationComponent {
    private var _coreComponent: CoreComponent? = null
    val coreComponent
        get() = _coreComponent
            ?: throw IllegalStateException("Make sure to call ApplicationComponent.init()")

    fun init() {
        _coreComponent = CoreComponentImpl()
    }
}

val coreComponent get() = ApplicationComponent.coreComponent
```

## Initialization

On `Android` we'll have the init happening inside our `:Application`
```kotlin
class MyAwesomeKMPApp : Application() {
    override fun onCreate() {
        super.onCreate()
        ApplicationComponent.init()
    }
}
```

On `iOS` we'll have to create an init function, inside the `MainViewController.kt` let's create it.

```kotlin
fun initialize() {
    ApplicationComponent.init()
}
```

then in the `iOSApp.swift`'s App init block, initialize it.

```swift
@main
struct iOSApp: App {

	init() { MainViewControllerKt.initialize() }

	var body: some Scene {
		WindowGroup {
			ContentView()
		}
	}
}
```

On `Desktop` it's pretty straight forward.

```kotlin
fun main() = application {
    ApplicationComponent.init()

    Window(
        onCloseRequest = ::exitApplication,
    ) {
        App()
    }
}
```
and now with the helper property, you can just use `coreComponent.appPreferences` whenever you need it.

<img src="/assets/img/kmp/kmp_meme.gif" alt ="" class="center" >

## That's all folks

This articale is also partially presenting how to set up manual DI, if enough interest is gathered, I will create an article about it in the new year, as this will probably be my last article for this one.

Happy early New Year, everyone.
