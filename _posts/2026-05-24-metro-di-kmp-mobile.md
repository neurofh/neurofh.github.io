---
title: "Metro DI for KMP mobile: From God Objects and manually checked Singletons to Compile-Time Injection"
date: 2026-05-24 19:30:00 +0200
categories: [Kotlin, KMP]
tags: [Kotlin, KMP, Metro, DI, Compose, Multiplatform]
---

## The Starting Point: God Object, Custom Singletons, and Lazy Maps

In the [previous article](https://funkymuse.dev/posts/metro-di-ktor-server/) i talked about how the Ktor backend replaced a God Object `BackendComponent` with Metro's compile-time graph. The mobile side of [Rudio](https://rudio.app/) had a remarkably similar story, except it was more interesting because there were actually *two* God Objects working together and someone (looks in the code, realizes it was me) had written a custom thread-safe singleton.

The centerpiece was `ApplicationComponent` (this should give you flashbacks if you've ever written manual DI or something similar in a KMP project):

```kotlin
object ApplicationComponent {

    private val appInitializer: MutableSet<AppInitializer> = mutableSetOf(
        NapierInitializer,
        FirebaseInitializer(),
        InAppReviewDataStoreCounterInitializer(),
        RevenueCatInitializer(),
        ObservabilityKeysInitializer(),
    )

    private var _coreComponent: CoreComponentImpl? = null
    val coreComponent: CoreComponent
        get() {
            return _coreComponent ?: error("CoreComponent not initialized")
        }

    fun initializeComponent() {
        _coreComponent = getSingleton { CoreComponentImpl() }
        appInitializer.forEach { it.initialize() }
    }

    @PublishedApi
    internal val singletons = mutableMapOf<KClass<*>, Any>()

    @PublishedApi
    internal val singletonFactories = mutableMapOf<KClass<*>, () -> Any>()

    inline fun <reified T : Any> registerSingleton(noinline factory: () -> T) { ... }
    inline fun <reified T : Any> getSingleton(): T { ... }
}

val coreComponent get() = ApplicationComponent.coreComponent
```

Version 1.0.X shipped with this approach, only from 1.1.X is when Metro landed, no crashes whatsoever regarding any of the DI, it works... but at a mental cost if you ask me.

There was a global `coreComponent` property holding singletons so every ViewModel could do `= coreComponent.navigatorDispatcher` in its constructor defaults. Convenient, until you wonder what `coreComponent` means in a test that never called `initializeComponent()`.

`CoreComponent` was an interface listing everything the app might ever need:

```kotlin
interface CoreComponent : CoroutinesComponent, DatabaseComponent {
    val json: Json
    val dataStore: DataStore<Preferences>
    val httpClient: HttpClient
    val navigatorDispatcher: NavigatorDispatcher
    val navigatorDirections: NavigatorDirections
    val invalidAuthenticationStateStore: InvalidAuthenticationStateStore
    val snackStateStore: SnackStateStore
    val toastManager: ToastManager
    val locationRepository: LocationRepository
    val appleSignIn: AppleSignIn
    val googleSignIn: GoogleSignIn
    val permissionHandler: PermissionHandler
    val analyticsTraces: AnalyticsTraces
    val analyticsRepository: AnalyticsRepository
    val adsRepository: AdsRepository
    val purchasesRepository: PurchasesRepository
    val userInMemoryCache: UserInMemoryCache
}
```

And `CoreComponentImpl` brought that interface to life with 18+ lazy properties:

```kotlin
internal class CoreComponentImpl(
    private val coroutinesComponent: CoroutinesComponent = CoroutinesComponentImpl.create()
) : CoreComponent, CoroutinesComponent by coroutinesComponent {

    init {
        SingletonImageLoader.setSafe { context -> ImageLoader.Builder(context)... }
    }

    override val json: Json by lazy { getSingleton { commonJsonSerialization() } }
    override val toastManager: ToastManager by lazy { getSingleton { ToastManagerImpl(applicationScope) } }
    override val locationRepository: LocationRepository by lazy { getSingleton { createPlatformLocationRepository() } }
    // ... 15 more of these
}
```

Then there was `CoroutinesComponentImpl`:

```kotlin
internal class CoroutinesComponentImpl : CoroutinesComponent {
    companion object {
        fun create(): CoroutinesComponent = getSingleton(::CoroutinesComponentImpl)
    }

    override val mainImmediateDispatcher = Dispatchers.Main.immediate
    override val mainDispatcher = Dispatchers.Main
    override val ioDispatcher = Dispatchers.IO
    override val defaultDispatcher = Dispatchers.Default
    override val applicationScope: CoroutineScope = CoroutineScope(SupervisorJob() + mainImmediateDispatcher)
}
```

And finally, there was `SingletonCheck`, adapted from [Dagger's `SingleCheck.java`](https://github.com/google/dagger/blob/master/dagger-runtime/main/java/dagger/internal/SingleCheck.java) and ported to KMP (my code was probably an older commit in Dagger's code so changes might be possible) with `SynchronizedObject` instead of the Java lock:

```kotlin
class SingletonCheck<T : Any>(private val factory: () -> T) {
    @Volatile private var instance: T? = null
    private val lock = SynchronizedObject()
    private var creating = false

    fun get(): T {
        return instance ?: synchronized(lock) {
            instance ?: run {
                if (creating) error("Reentrant singleton creation detected")
                creating = true
                factory().also {
                    instance = it
                    creating = false
                }
            }
        }
    }
}

fun <T : Any> getSingleton(obj: () -> T) = SingletonCheck(obj).get()
```

It's also completely unnecessary when a compile-time DI graph handles object lifetimes for you. When you end up porting Dagger internals into your own project, well, it's probably time to just use a real DI framework, so i did.

![Choices](/assets/img/metro_mobile_di/kmp_dev_meme.jpg)


## The KMP Dimension

The backend story was simple: one JVM target, one `BackendGraph`. The "mobile" side is three platforms (Android, iOS, and JVM desktop) and each one needs slightly different things at startup.

Android needs an `Application` reference to create a `DataStore`, open the SQLCipher database, and initialize Room. iOS initializes the graph without any platform type; all its platform bindings are contributed via `@BindingContainer` before the graph is created. Desktop is the same shape as iOS minus the mobile-specific SDKs.

This meant `AppGraph` couldn't be a single `interface` with a single factory. It needed three flavors of the same contract.

## AppGraph: Three Flavors of One Contract

The Android graph requires an `Application` to wire platform dependencies. Metro's `@DependencyGraph.Factory` covers that:

```kotlin
// android/app/src/main/kotlin/.../di/AppGraph.kt
@DependencyGraph(AppScope::class)
@SingleIn(AppScope::class)
interface AppGraph : ViewModelGraph, MetroAppComponentProviders {
    override val metroViewModelFactory: MetroViewModelFactory
    val appInitializers: Set<AppInitializer>
    val navGraphRegistrars: Set<NavGraphRegistrar>

    @DependencyGraph.Factory
    fun interface Factory {
        fun create(@Provides application: Application): AppGraph
    }
}
```

`ViewModelGraph` is metrox's marker interface that enables ViewModel multibinding. `MetroAppComponentProviders` is the Android-specific metrox hook that ties `MetroViewModelFactory` into the Activity's `ViewModelProvider`. Metro generates the factory implementation at compile time.

The iOS graph has no factory. All platform bindings arrive via `@ContributesTo @BindingContainer` objects that Metro discovers automatically:

```kotlin
// composeApp/src/iosMain/kotlin/.../di/AppGraph.kt
@DependencyGraph(AppScope::class)
@SingleIn(AppScope::class)
interface AppGraph : ViewModelGraph {
    override val metroViewModelFactory: MetroViewModelFactory
    val navigatorDispatcher: NavigatorDispatcher
    val appInitializers: Set<AppInitializer>
    val navGraphRegistrars: Set<NavGraphRegistrar>
    val updateFcmTokenUseCase: UpdateFcmTokenUseCase
}
```

`navigatorDispatcher` and `updateFcmTokenUseCase` are listed directly because Swift code calls into the graph to wire navigation and push token updates. Android doesn't need these exposed on the graph because it accesses them through the ViewModel layer.

`AppScope` is still just a class, same as the backend:

```kotlin
abstract class AppScope private constructor()
```

Nothing else. The private constructor means no accidental instantiation. Metro uses it as an aggregation marker to collect every `@ContributesTo(AppScope::class)` contribution into a single graph.

The entry points are straightforward. On Android:

```kotlin
class RudioApplication : Application(), MetroApplication {

    val appGraph: AppGraph by lazy {
        createGraphFactory<AppGraph.Factory>().create(this)
    }

    override val appComponentProviders: MetroAppComponentProviders
        get() = appGraph

    override fun onCreate() {
        super.onCreate()
        appGraph.appInitializers
            .sortedBy { it.order.order }
            .forEach { it.initialize() }
    }
}
```

On iOS, `MainViewController.kt` exposes three Kotlin functions that Swift calls directly:

```kotlin
private val graph: AppGraph by lazy { createGraph<AppGraph>() }

fun initializeAppGraph() {
    graph.appInitializers
        .sortedBy { it.order.order }
        .forEach { it.initialize() }
}

fun updateFcmToken(token: String) {
    graph.updateFcmTokenUseCase.execute(token = token)
}

fun MainViewController(onAppInitialized: (() -> Unit)? = null): UIViewController {
    return ComposeUIViewController {
        App(
            viewModelFactory = graph.metroViewModelFactory,
            onAppInitialized = onAppInitialized,
        )
    }
}
```

Swift calls them in `AppDelegate`:

```swift
import RudioKt

class AppDelegate: NSObject, UIApplicationDelegate {

    func application(
        _ application: UIApplication,
        didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]? = nil
    ) -> Bool {
        MainViewControllerKt.initializeAppGraph()
        // ...
        return true
    }

    func messaging(_ messaging: Messaging, didReceiveRegistrationToken fcmToken: String?) {
        guard let token = fcmToken else { return }
        MainViewControllerKt.updateFcmToken(token: token)
    }
}
```

`createGraph<AppGraph>()` vs `createGraphFactory<AppGraph.Factory>().create(this)`: same Metro API, two shapes, each appropriate to its platform.

## @BindingContainer Across Platforms

The old `CoreComponentImpl` mixed everything together in one class: coroutines, serialization, networking, storage, platform services. With Metro, these split naturally by concern and by platform.

**For the sake of simplicity i have another CoroutineBindings that provides the coroutine dependencies but the article is too long already, so i've merged them in the AppBindings**.

`AppBindings` in `commonMain` provides everything that's platform-independent:

```kotlin
@ContributesTo(AppScope::class)
@BindingContainer
object AppBindings {

    @Provides
    @SingleIn(AppScope::class)
    @ApplicationScope
    fun provideApplicationScope(
        @MainImmediateDispatcher mainImmediate: CoroutineDispatcher,
    ): CoroutineScope = CoroutineScope(SupervisorJob() + mainImmediate)

    @Provides
    @IoDispatcher
    fun provideIoDispatcher(): CoroutineDispatcher = Dispatchers.IO

    @Provides
    @DefaultDispatcher
    fun provideDefaultDispatcher(): CoroutineDispatcher = Dispatchers.Default

    @Provides
    @MainDispatcher
    fun provideMainDispatcher(): CoroutineDispatcher = Dispatchers.Main

    @Provides
    @MainImmediateDispatcher
    fun provideMainImmediateDispatcher(): CoroutineDispatcher = Dispatchers.Main.immediate

    @Provides
    @SingleIn(AppScope::class)
    fun provideJson(): Json = commonJsonSerialization() //which is the same one as backend's code

    @Provides
    @SingleIn(AppScope::class)
    fun provideHttpClient(
        json: Json,
        authRepository: Lazy<AuthRepository>,
        tokenRepository: TokenRepository,
    ): HttpClient = createHttpClient(json = json, ...)

    @Provides
    fun provideClock(): Clock = Clock.System
}
```

`applicationScope` is now a `@SingleIn` `val`-backed `CoroutineScope`. One instance, `SupervisorJob` scoped to the application lifetime, dispatcher taken through `@MainImmediateDispatcher`, pretty standard things nothing fancy.

Platform-specific things go in their own `@BindingContainer`. On Android, `AppBindingsAndroid` in `androidMain` provides the `Application` context, `DataStore`, and the SQLCipher-backed Room database [you can check my other blog post on how i implemented sql cipher with some challenges in KMP when you pull in firebase and other dependencies that have lsqlite as transitive one](https://funkymuse.dev/posts/encrypt-kmp-database-with-firebase-in-project/):

```kotlin
@ContributesTo(AppScope::class)
@BindingContainer
object AppBindingsAndroid {

    @Provides
    @SingleIn(AppScope::class)
    @ApplicationContext
    fun provideApplicationContext(application: Application): Context = application.applicationContext

    @Provides
    @SingleIn(AppScope::class)
    fun provideDataStore(
        @ApplicationContext context: Context,
        @IoDispatcher ioDispatcher: CoroutineDispatcher,
        @ApplicationScope scope: CoroutineScope,
    ): DataStore<Preferences> = createDataStoreWithDefaults(context = context, ...)

    @Provides
    @SingleIn(AppScope::class)
    fun provideDatabase(
        @ApplicationContext context: Context,
        @ApplicationScope scope: CoroutineScope,
        @IoDispatcher ioDispatcher: CoroutineDispatcher,
    ): AppDatabase = Room.databaseBuilder(context, AppDatabase::class.java, dbFileName)
        .openHelperFactory(SupportOpenHelperFactory(...))
        .setDefaults()
        .build()
}
```

iOS and JVM have equivalent files with their platform-specific Room/DataStore variants. Each file is annotated `@ContributesTo(AppScope::class)`. Metro merges all contributions automatically: no central list, no explicit `@Module` includes, no registration call.

## Three Graphs, One Underlying Contract

`AppGraph` is declared three times: once in `androidMain`, once in `iosMain`, once in `jvmMain` (desktop). They look different on the surface. Android's has a `Factory` that takes `Application`. iOS's exposes `navigatorDispatcher` and `updateFcmTokenUseCase` because Swift interop needs to call into them directly. JVM's is closer to the iOS shape.

Under the hood, Metro generates one implementation per platform and each one ends up with the exact same set of bindings. The reason: `AppBindings` in `commonMain` contributes dispatchers, `Json`, `HttpClient`, and `Clock`. `AppBindingsAndroid` / `AppBindingsIos` / `AppBindingsJvm` each contribute `DataStore` and `AppDatabase` using their platform's storage API. Because all four files carry `@ContributesTo(AppScope::class)`, Metro collects them all into the graph for that platform's compilation target.

The graph interfaces differ because the entry point requirements differ. Android needs `Application` at graph creation time; iOS and JVM don't. iOS Swift code needs to call a few things directly through the graph boundary; the other platforms don't. But every ViewModel, every Repository, every UseCase compiled into that platform sees identical bindings: the same `@IoDispatcher`, the same `HttpClient`, the same `AppDatabase`. There's no per-platform conditional logic in any of them.

## Do We Even Need expect/actual?

Before the Metro migration, platform-specific behavior was often modeled as top-level `expect fun`. Several utilities in the codebase lived in `shared/src/commonMain/kotlin/app/rudio/wifi/map/platform/` as bare expect functions:

```kotlin
// shared/src/commonMain - the old pattern
expect fun deviceModel(): String
expect fun osVersion(): String
expect fun appVersion(): String
```

Each platform provided the matching `actual fun`. It compiled. It ran. But the moment you tried to test a ViewModel that called `deviceModel()` in `commonTest`, you hit a wall: the function is a concrete expect declaration. There is no interface to swap out. Writing a fake in `commonTest` would require providing an `actual` for every platform target in the test source set, and those actuals would need to call real platform APIs. Unit tests on the JVM can't reach Android or iOS APIs. The fake cannot exist.

So i deleted all of those files and replaced them with interfaces:

```kotlin
// shared/src/commonMain - testable
interface DeviceInfoProvider {
    fun deviceId(): String
    fun deviceModel(): String
    fun osVersion(): String
    fun appVersion(): String
}

// shared/src/androidMain
@Inject
@ContributesBinding(AppScope::class)
class DeviceInfoProviderImpl(
    @ApplicationContext private val context: Context,
) : DeviceInfoProvider {
    // real implementation using PackageManager, Build, etc.
}
```

Now `FakeDeviceInfoProvider` lives in `composeApp/src/commonTest` and every test injects it directly:

```kotlin
class FakeDeviceInfoProvider(
    private val id: String = "test-device-id",
) : DeviceInfoProvider {
    override fun deviceId() = id
    override fun deviceModel() = "test"
    override fun osVersion() = "test"
    override fun appVersion() = "1.0"
}
```

The `@ContributesBinding(AppScope::class)` on the platform class tells Metro to bind `DeviceInfoProviderImpl` as `DeviceInfoProvider`. Every ViewModel receives the interface, never the platform class. The whole point: fakes can now exist in `commonTest`.

That said, `expect`/`actual` is still the right tool for several categories:

**Platform-specific Composable functions:** `@Composable fun GoogleMap(...)`, etc... Composables aren't injected through the DI graph; they're called from the Compose tree. `expect fun` is the only mechanism that lets `commonMain` declare a Composable that has a platform-specific implementation.

**Wrapping a platform SDK class:** `expect class AdmobAdsManager`, `expect class AdsConsentManager`, `expect class DatabaseKeyManager`. When the platform SDK class *is* the implementation (Google's AdMob SDK on Android, a no-op stub on other targets), the `expect`/`actual` wrapping is the right layer. The trick is to then put a testable `interface` (`AdsManager`, `ConsentManager`) in front of it so the `expect class` itself doesn't leak into ViewModel constructors, you can inject these too later on as well.

**Scaffolding required by libraries:** `expect object AppDatabaseConstructor : RoomDatabaseConstructor<AppDatabase>`. Room's KMP code generation requires this pattern. It's not a design choice; it's what the library demands.

**Simple platform values:** `expect val currentPlatform: Platform`. When it's just a value with no methods, no lifecycle, no state, `expect val` is the simplest correct tool.

The rule of thumb: if something needs a `Fake` in `commonTest`, model it as an interface and bind the platform implementation via `@ContributesBinding`. If it wraps a platform SDK that only exists on one target and tests don't need it, `expect`/`actual` is fine.

## Testing With Fakes

The whole point of converting `expect fun` to `interface` + `@ContributesBinding` is this: in tests you never touch the DI graph at all.

Each integration test lives in `composeApp/src/commonTest` and constructs the ViewModel directly through its constructor, passing fakes instead of real implementations. No `ApplicationComponent.initializeComponent()`, no graph creation, no mocking framework:

```kotlin
@OptIn(ExperimentalTestApi::class, ExperimentalCoroutinesApi::class)
class SignUpIntegrationTest {

    @BeforeTest
    fun setUpMainDispatcher() {
        Dispatchers.setMain(UnconfinedTestDispatcher())
    }

    @AfterTest
    fun tearDownMainDispatcher() {
        Dispatchers.resetMain()
    }

    @Test
    fun createAccountButton_isDisabledWithoutInput() = runComposeUiTest {
        val viewModel = signUpViewModel()
        renderScreen(viewModel)
        createAccountButton().assertIsNotEnabled()
    }

    @Test
    fun signUp_navigatesToCheckEmail_onSuccess() = runComposeUiTest {
        val nav = FakeNavigatorDispatcher()
        val authRepository = FakeAuthRepository(
            signUpResult = SignUpResult.Success(user = verifiedUser, tokens = tokens),
        )
        val viewModel = signUpViewModel(nav = nav, authRepository = authRepository)
        renderScreen(viewModel)
        fillForm()
        createAccountButton().performClick()

        assertEquals(1, nav.navigateSingleTopCalls.size)
        assertTrue(nav.navigateSingleTopCalls.first() is CheckYourEmailGraph.CheckYourEmailScreen)
    }

    private fun signUpViewModel(
        nav: FakeNavigatorDispatcher = FakeNavigatorDispatcher(),
        authRepository: FakeAuthRepository = FakeAuthRepository(),
        passwordCredentials: FakePasswordCredentials = FakePasswordCredentials(),
        // ...
    ): SignUpViewModel = SignUpViewModel(
        navigatorDispatcher = nav,
        authRepository = authRepository,
        passwordCredentialManager = passwordCredentials,
        // ...
    )

    private fun ComposeUiTest.createAccountButton(): SemanticsNodeInteraction {
        val createAccount = runBlockingGetString(Res.string.create_account)
        return onNode(hasText(createAccount) and hasClickAction())
    }
}
```

`FakeAuthRepository` has a configurable result and records every call. The default is a failure case so most tests don't need to configure anything:

```kotlin
class FakeAuthRepository(
    private val signUpResult: SignUpResult = SignUpResult.Failure.EmailAlreadyRegistered,
    // ...
) : AuthRepository {

    val signUpCalls = mutableListOf<SignUpInvocation>()

    override suspend fun signUp(
        email: String,
        username: String,
        password: String,
    ): SignUpResult {
        signUpCalls += SignUpInvocation(email, username, password)
        return signUpResult
    }
}
```

A test that needs success overrides only the result it cares about. A test that needs to assert the repository received specific arguments checks `authRepository.signUpCalls`. Nothing else changes.

Shared fakes live in `composeApp/src/commonTest/kotlin/.../fakes/`. One fake per interface, named `Fake<InterfaceName>`. They're shared across test classes: `FakeNavigatorDispatcher`, `FakeAuthRepository`, `FakeDeviceInfoProvider`, `FakeToastManager`, and about 30 others. A new test needs two things: a `private fun viewModel(...)` helper that wires fakes with sensible defaults, and the actual test methods. That's it.

## ViewModels: From companion fun factory() to @Inject

The old ViewModel pattern was constructor defaults pointing at `coreComponent.*` and a companion `factory()` function that callers had to know about:

```kotlin
// before
class SignInViewModel(
    savedStateHandle: SavedStateHandle,
    private val navigatorDispatcher: NavigatorDispatcher = coreComponent.navigatorDispatcher,
    private val signInWithEmailUseCase: SignInWithEmailUseCase = SignInWithEmailUseCase.create(),
    private val passwordCredentialManager: PasswordCredentials = PasswordCredentials.create(),
    private val socialSignInPresenter: SocialSignInPresenter = SocialSignInPresenter(),
    private val toastManager: ToastManager = coreComponent.toastManager,
) : ViewModel() {

    companion object {
        fun factory() = provideSavedStateFactory {
            SignInViewModel(savedStateHandle = it)
        }
    }
}
```

`SignInWithEmailUseCase.create()` might seem magical but `create()` was a companion object that called the constructor of `SignInWithEmailUseCase` which was private with default arguments so that we don't expose dependencies that are with `implementation` in the creator module where they'll need to be changed to `api` in for no reason, shitty guard but it worked.

And the call site:

```kotlin
// before
val viewModel = viewModel<SignInViewModel>(factory = SignInViewModel.factory())
```

Every ViewModel that needed `SavedStateHandle` followed this pattern. Every ViewModel that didn't use `SavedStateHandle` used `provideFactory` instead of `provideSavedStateFactory`. These were custom helpers:

```kotlin
inline fun <reified T : ViewModel> provideFactory(crossinline creator: () -> T) =
    viewModelFactory { initializer { creator() } }

inline fun <reified T : ViewModel> provideSavedStateFactory(crossinline creator: (SavedStateHandle) -> T) =
    viewModelFactory { initializer { val ssh = createSavedStateHandle(); creator(ssh) } }
```

After Metro, the same ViewModel:

```kotlin
// after
@AssistedInject
@ViewModelKey
class SignInViewModel(
    private val navigatorDispatcher: NavigatorDispatcher,
    private val signInWithEmailUseCase: SignInWithEmailUseCase,
    private val passwordCredentialManager: PasswordCredentials,
    private val socialSignInPresenter: SocialSignInPresenter,
    private val toastManager: ToastManager,
    @Assisted private val savedStateHandle: SavedStateHandle,
) : ViewModel() {

    @AssistedFactory
    @ViewModelAssistedFactoryKey(SignInViewModel::class)
    @ContributesIntoMap(AppScope::class)
    fun interface Factory : ViewModelAssistedFactory {
        override fun create(extras: CreationExtras): SignInViewModel =
            create(extras.createSavedStateHandle())
        fun create(@Assisted savedStateHandle: SavedStateHandle): SignInViewModel
    }
}
```

`@AssistedInject` tells Metro that `SavedStateHandle` is provided externally at creation time, not through the graph. The nested `Factory` bridges Metro's `@AssistedFactory` with metrox's `ViewModelAssistedFactory` contract. `@ContributesIntoMap(AppScope::class)` auto-registers this factory into the `Map<KClass<out ViewModel>, ViewModelAssistedFactory>` that `MetroViewModelFactory` consults.

Similarly you can provide direct parameters, i still prefer passing a `SavedStateHandle`, just a preference...

ViewModels that don't need `SavedStateHandle` are simpler, plain `@Inject` with `@ContributesIntoMap`:

```kotlin
@Inject
@ViewModelKey
@ContributesIntoMap(AppScope::class, binding = binding<ViewModel>())
class MainViewModel(
    private val navigatorDispatcher: NavigatorDispatcher,
    // ... other deps, no defaults, no coreComponent
) : ViewModel()
```

The call site becomes:

```kotlin
// after
val viewModel = metroViewModel<MainViewModel>()               // plain inject
val viewModel = assistedMetroViewModel<SignInViewModel>()     // assisted inject
```

`provideFactory`, `provideSavedStateFactory`, and every `companion object { fun factory() }` block across the entire codebase: deleted.

## AppInitializer: Ordered Startup Without a God Object

The old startup was a hardcoded `MutableSet` inside `ApplicationComponent`:

```kotlin
// before - in ApplicationComponent
private val appInitializer: MutableSet<AppInitializer> = mutableSetOf(
    NapierInitializer,
    FirebaseInitializer(),
    InAppReviewDataStoreCounterInitializer(),
    RevenueCatInitializer(),
    ObservabilityKeysInitializer(),
)

fun initializeComponent() {
    _coreComponent = getSingleton { CoreComponentImpl() }
    appInitializer.forEach { it.initialize() }
}
```

Adding an initializer meant editing `ApplicationComponent`. The set is unordered: the iteration order of `mutableSetOf` follows insertion order, which is the declaration order in source code. If logging needs to come before Firebase, you'd better not reorder those lines.

After Metro, each initializer is self-registering and carries its own order:

```kotlin
interface AppInitializer {
    val order: AppInitializerOrder
    fun initialize()
}

enum class AppInitializerOrder(val order: Int) {
    LOGGING(order = 100),
    FIREBASE(order = 200),
    OBSERVABILITY_KEYS(order = 300),
    REVENUE_CAT(order = 400),
    DATA_STORE(order = 500),
}
```

Gaps between values are intentional: insert a new initializer between existing ones without renumbering anything.

`NapierInitializer` as a concrete example:

```kotlin
@Inject
@ContributesIntoSet(AppScope::class)
class NapierInitializer(
    private val crashlyticsAntilog: CrashlyticsAntilog,
) : AppInitializer {
    override val order: AppInitializerOrder = AppInitializerOrder.LOGGING

    override fun initialize() {
        if (BuildKonfig.isDebug) Napier.base(DebugAntilog())
        else Napier.base(crashlyticsAntilog)
    }
}
```

`@ContributesIntoSet(AppScope::class)` auto-registers it into `Set<AppInitializer>`. The startup loop on both Android and iOS:

```kotlin
appGraph.appInitializers
    .sortedBy { it.order.order }
    .forEach { it.initialize() }
```

`FirebaseInitializer` is `expect`/`actual`, platform-specific initialization, but the call site in the startup loop never has to care. Add a new initializer, pick an order value, done. `ApplicationComponent` doesn't exist to be edited.

## Self-Registering Nav Graphs

This is the before/after that the diff makes most viscerally clear. The old `App.kt` contained 35 explicit nav graph calls inside the `NavHost` block:

```kotlin
// before - in App.kt
NavHost(...) {
    onBoardingGraph()
    mapGraph()
    profileGraph()
    leaderBoardGraph()
    signInGraph()
    signUpGraph()
    forgotPasswordGraph()
    resetPasswordGraph()
    verifyEmailGraph()
    checkYourEmailGraph()
    disabledAccountGraph()
    settingsGraph()
    editProfileGraph()
    changePasswordGraph()
    changeEmailGraph()
    redeemPremiumCodeGraph()
    faqGraph()
    publicUserProfileGraph()
    userPoisGraph()
    poiDetailsGraph()
    cityOrCountrySearchGraph()
    reportPoiGraph()
    speedTestGraph()
    speedTestHistoryGraph()
    editHistoryGraph()
    verificationHistoryGraph()
    suggestEditSuccessGraph()
    xpRewardGraph()
    dailyXpClaimGraph()
    achievementsGraph()
    offlineRegionsGraph()
    offlineWifiListGraph()
    offlinePoiDetailsGraph()
    purchaseGraph()
    adNotLoadedGraph()
}
```

Each of these was a plain extension function on `NavGraphBuilder` defined inside its feature package and imported in `App.kt`. Adding a screen meant adding a function, adding an import, and adding a call in this block: three places to touch in a file that already had 35 of them.

After Metro, `NavGraphRegistrar` is a `fun interface`:

```kotlin
fun interface NavGraphRegistrar {
    fun NavGraphBuilder.register()
}
```

Each feature owns its own `@Inject @ContributesIntoSet` implementation. The old `NavGraphBuilder.signInGraph()` extension function becomes:

```kotlin
@Inject
@ContributesIntoSet(AppScope::class)
class SignInNavGraph(
    private val signIn: SignInContent,
) : NavGraphRegistrar {
    override fun NavGraphBuilder.register() {
        navigation<SignInGraph>(startDestination = SignInGraph.SignInScreen::class) {
            with(signIn) { register() }
        }
    }
}
```

```kotlin
fun interface NavContent {
    fun NavGraphBuilder.register()
}
```

```kotlin
@Inject
class SignInContent : NavContent {
    override fun NavGraphBuilder.register() {
        composable<SignInGraph.SignInScreen> { backStackEntry ->
            val viewModel = assistedMetroViewModel<SignInViewModel>()
            ...
            ...
            ...
        }
    }
}
```

`SignInContent` is an `@Inject class` that implements `NavContent` (the composable side). `SignInNavGraph` wires the navigation structure; `SignInContent` provides the composable. Metro constructs and injects both.

`App.kt`'s `NavHost` becomes:

```kotlin
// after
NavHost(...) {
    mainViewModel.navGraphRegistrars.forEach {
        with(it) { register() }
    }
}
```

Add a new screen: create the `NavGraph` class, annotate it, it's registered. Delete a screen: delete its files, it's gone. `App.kt` doesn't change.

One important note: `AppScope` in `@ContributesIntoSet` must always import `app.rudio.wifi.map.di.scope.AppScope`. Accidentally importing `dev.zacsweers.metro.AppScope` (Metro's own default) contributes to a *different* scope and Metro will report a `Metro/MissingBinding` error for `Set<NavGraphRegistrar>` at compile time. The error message is clear once you've seen it once; confusing if you haven't, or you can use that default one as well, up to you.

## ActivityGraph: A Child Scope for a Reason, a wrinkle in our set-up

Android has one more wrinkle: `Activity`-bound state. `ViewModelScope` gives you "lives as long as the ViewModel", `AppScope` gives you "lives as long as the application". But sometimes you need "lives as long as this specific Activity.

`PlatformActivity` is just a type alias to ComponentActivity and on iOS and JVM is just a stub.

That's `ActivityGraph`:

```kotlin
@GraphExtension(ActivityScope::class)
interface ActivityGraph {
    val platformActivity: PlatformActivity

    @ContributesTo(AppScope::class)
    @GraphExtension.Factory
    fun interface Factory {
        fun createActivityGraph(@Provides platformActivity: PlatformActivity): ActivityGraph
    }
}
```

`@GraphExtension(ActivityScope::class)` declares a child graph scoped to `ActivityScope`. The nested `Factory` is annotated `@ContributesTo(AppScope::class)` so Metro auto-merges it into the parent `AppGraph`, no manual wiring in `AppGraph` itself. The Activity creates the child:

```kotlin
@ActivityKey
@ContributesIntoMap(AppScope::class, binding<Activity>())
@Inject
class MainActivity(
    private val metroViewModelFactory: MetroViewModelFactory,
    private val navigatorDispatcher: NavigatorDispatcher,
) : ComponentActivity() {

    lateinit var activityGraph: ActivityGraph
        private set

    override fun onCreate(savedInstanceState: Bundle?) {
        .... code
        super.onCreate(savedInstanceState)
        activityGraph = (application as RudioApplication).appGraph.createActivityGraph(this)
        if (savedInstanceState == null) {
            intent?.data?.toString()?.let { navigatorDispatcher.processDeepLink(deepLink = it) }
        }
        ... rest of the code
    }
}
```

The graph, and everything scoped `@SingleIn(ActivityScope::class)` inside it, is garbage-collected when the Activity instance is dead. ViewModels in `AppScope` cannot depend on `ActivityGraph` bindings, the scope hierarchy enforces this at compile time. A ViewModel that held a `PlatformActivity` reference would survive configuration changes and leak the old Activity. The scoping rules make that a compile error rather than a memory leak caught in a profiler session.

A concrete reason this scope exists: permission handling on Android requires a `ComponentActivity` to register `ActivityResultLauncher`s and check grant state. That's an Activity-bound dependency, not an application-level one. Putting it in `AppScope` would mean holding a live `Activity` reference across configuration changes: instant memory leak. `ActivityGraph` is the right scope. The implementation is created with the Activity and destroyed with it, and Metro's scope rules make it a compile error for any `AppScope` consumer to depend on it.

## @SingleIn: Same Rule, Mobile Edition

The rule is identical to the backend: stateless classes are plain `@Inject class`. `@SingleIn(AppScope::class)` is opt-in, reserved for things that hold shared mutable state:

**Add `@SingleIn`:**
- `NavigatorDispatcher` (shared logic across the nav stack)
- `ToastManager` (shared custom toast logic and handler)
- `UserInMemoryCache` (shared in-memory cache)
- `HttpClient` (connection pool single instance)
- `DataStore<Preferences>` (file-backed, one instance)
- `AppDatabase` / Room (database connection, one instance)
- `CoroutineScope` (single `SupervisorJob` + `MainImmediateDispatcher`)

**Leave as plain `@Inject`:**
- Every `Repository` that only reads from injected `HttpClient` or database DAOs
- Every `UseCase` that only calls repositories
- Every `Mapper` that only transforms data types

The instinct to make everything a singleton to "be safe" is exactly what caused the old `getSingleton { ... }` call scattered across `CoreComponentImpl`. Metro makes the safe default correct: fresh instance unless you explicitly opt into shared lifetime, remember, not everything needs to be a singleton.

## The Migration in One Commit

The mobile Metro adoption landed in a single commit. The diff touched roughly every feature's ViewModel and nav file, removed the four God Object files (`ApplicationComponent`, `CoreComponent`, `CoreComponentImpl`, `SingletonCheck`), and dropped the `ViewModelExtensions` helpers. The net line count was negative, surprisingly.

What stayed exactly the same: every Repository, every UseCase, every Mapper, every domain model. Constructor injection was already the pattern; the only change was removing the default values that silently reached into `coreComponent`. The logic didn't move. The wiring did and i am happy it did.

This is the DI setup powering [Rudio](https://rudio.app/) - find free WiFi hotspots around you.

Stay hydrated and keep your scopes clean.
