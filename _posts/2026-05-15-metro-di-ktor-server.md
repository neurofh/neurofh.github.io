---
title: "Metro DI for Ktor Backend: From a God Object through manual DI to Constructor Injection"
date: 2026-05-15 17:35:00 +0200
categories: [Kotlin, Ktor, Backend]
tags: [Kotlin, Ktor, Metro, DI, Backend, Testing, KMP]
---

## The Starting Point: a God Object

When [Rudio](https://rudio.app/) was a created a few months ago it used the simplest possible DI: one big `object BackendComponent`
with everything wired manually via `by lazy` and `lateinit var`. No framework, no annotations, no ceremony, then i started adding things and manual DI became a tedious and tiring task because everywhere i used default parameters to achieve default constructor injection, one could argue that things could've been broken into multiple components like we have the database and dispatcher to make it less messy and indeed that's the thing, but after some time, you just get tired and few layers down you forget where things come from and go to.

It looked like this:

```kotlin
object BackendComponent : DatabaseComponent,
    DispatcherComponent by DispatcherComponentImpl() {

    val json by lazy { commonJsonSerialization() }

    val httpClient: HttpClient by lazy {
        HttpClient(OkHttp) {
            install(ContentNegotiation) { json(json) }
            // ...
        }
    }

    val authService by lazy { AuthService() }
    val bCryptVerifyer: BCrypt.Verifyer by lazy { BCrypt.verifyer() }
    val redisService: RedisService by lazy {
        if (appConfig.redis.useInMemory) InMemoryRedisService() else JedisRedisService()
    }

    lateinit var applicationScope: CoroutineScope
    lateinit var appConfig: ApplicationConfiguration
    lateinit var dataSource: DataSource
    lateinit var dslContext: DSLContext
}
```

Routes were plain extension functions with default parameter injection where controllers got instantiated with their own `= ClassName()` defaults and you get the gist by now (as shown in the example below):

```kotlin
fun Route.pingHealthRoute(
    controller: HealthController = HealthController(),
) {
    head(HealthRoutes.PING) { controller.ping() }
    get(HealthRoutes.PING) { controller.ping() }
}
```

`HealthController` has no dependencies where `HealthController()` is a perfectly fine default.
The technique looks clean here. Then a controller with real dependencies shows up:

```kotlin
class PoiController(
    private val poiRepository: PoiRepository = PoiRepository(),
    private val encryptionService: PasswordEncryptionService = PasswordEncryptionService(),
    private val redisService: RedisService = BackendComponent.redisService,
    private val xpController: XpController = XpController(),
    private val xpRepository: XpRepository = XpRepository(),
    private val editHistoryController: EditHistoryController = EditHistoryController(),
    private val passwordViewLogRepository: PasswordViewLogRepository = PasswordViewLogRepository(),
    private val poiResponseMapper: PoiWithWifiToPoiResponseMapper = PoiWithWifiToPoiResponseMapper(),
    // ... 6 more
)
```

When the dependencies are all stateless value objects, `= PoiRepository()` is fine where each call site
gets a fresh instance. But as soon as you bring something like `BackendComponent.redisService` in the constructor,
you've coupled the entire default parameter chain to the God Object (the `BackendComponent`). 

The controller can still be
instantiated with fakes in tests, the constructor parameters are right there. 

What breaks down is the accidental path: any test that doesn't explicitly override every default silently pulls from `BackendComponent`,
which means `BackendComponent` has to be initialized, which means the test is no longer isolated.
Forget one dependency and you're hitting the real Redis in a unit test, so you have to be really careful how you construct it in the test, i did it, it's doable but in reality it's a tedious task considering that this was only one backend module with my database being another (Gradle) module but this is even more complex if you leverage Ktor's server modules in which way i'll plan to proceed further and manual DI loses it's power.

After Metro, the exact same constructor without the defaults:

```kotlin
@Inject
class PoiController(
    private val poiRepository: PoiRepository,
    private val encryptionService: PasswordEncryptionService,
    private val redisService: RedisService,
    private val xpController: XpController,
    private val xpRepository: XpRepository,
    private val editHistoryController: EditHistoryController,
    private val passwordViewLogRepository: PasswordViewLogRepository,
    private val poiResponseMapper: PoiWithWifiToPoiResponseMapper,
    // ... 6 more
)
```

Metro wires every parameter at compile time, but now there are no defaults to accidentally miss or forget or couple from the `BackendComponent`.

Features and plugins registered themselves in `Backend.kt` by name:

```kotlin
fun Application.module() {
    BackendComponent.applicationScope = this
    BackendComponent.appConfig = appConfig
    initDatabase()
    // ...
    authPlugin()
    defaultHeadersPlugin()
    // ...
    userFeature()
    poiFeature()
    healthFeature()
    speedTestFeature()
    socialFeature()
    offlineFeature()
    revenueCatFeature()
    xpFeature()
}
```

This works fine when the project is small. Then the project grows. `BackendComponent` becomes load-bearing
for every controller and repository. Testing means either calling the real `BackendComponent` (including its `lateinit` state)
or threading fakes through the default parameter chain. Feature files pile up in `Backend.kt`. The God Object earns its name, i am looking at you Context in Android.

## The DI Landscape for Ktor

If you look at the [Ktor docs on dependency injection](https://ktor.io/docs/server-dependency-injection.html),
there's a first-party DI plugin now, it's a runtime container, type-safe resolution, integrated lifecycle management. Solid option, but it never appealed to me cause it looked similar to Koin and I had to think of the [policies](https://ktor.io/docs/server-di-configuration.html#available-policies) which made me re-think my decision in first place (this is personal opinion don't take it close to your heart if it doesn't align with yours).

![Choose](/assets/img/backend_metro_di/red_vs_blue_pill_di.jpg)

Beyond that, often, the community options fall into two categories that Zac Sweers (Metro's author) describes well in
[Re: Dependency Injection vs. Service Locators](https://www.zacsweers.dev/re-dependency-injection-vs-service-locators/) which i suggest you read it carefully but here's a breakdown:

**Category 1 - true DI frameworks** (compile-time, constructor injection, inversion of control):
- **Dagger** - the industry standard for over a decade. Works on the JVM, not Android-exclusive. Hilt is the Android-specific wrapper on top of it and Anvil is its steroid sadly it's in maintenance mode
- **kotlin-inject** - KSP-based, compile-time, the spiritual predecessor of Metro with a Kotlin-first API
- **Metro** - compiler plugin, KMP-native, took kotlin-inject's API and Dagger's code gen as direct inspiration

**Category 2 - service locators** (runtime, `by inject()` pull model):
- **Koin** - popular and easy to get started. But as Zac puts it: *"Their focus and value prop is on ease of use to developers. The fact that they are runtime-only by default means there is no build costs and your builds are always faster! Your code may fail at runtime if you're missing a dependency."*

Similar thing can happen with a manual DI, you can introduce a circular dependency and up until the point you run the app to test the thing, you wouldn't know until you get a stackoverflow or X error, because maybe you provided it few hierarchies down and still compiles fine but you forgot which thing depends on what because it's no longer in the back of your mind. 

The practical consequence: *"The biggest bill in category 1 comes in the form of an extra few seconds of build time, and the biggest bill in category 2 might come in a 2am pagerduty alert."*

**Manual injection** - transparent, zero dependencies, and Zac does it himself, hell i did it too: *"When I write a new simple project, i almost never use a DI framework out the gate. But I do write manual DI still. Then after a certain level of complexity I find myself annoyed with the wiring and adopt a DI framework."* Which is exactly the `object BackendComponent` story above, it was not a question of whether i should use DI compile time framework or NO, it was a matter of time.

i wanted compile-time verification  and the deciding factor was the same framework running on Android, iOS, and the JVM backend with identical concepts, one framework, three platforms, zero runtime surprises (so far).
That's [Metro](https://zacsweers.github.io/metro/).
*Keep in mind i've used Dagger/Hilt/Anvil extensively since i'm doing Android development so this is nothing new, Metro just aggregates all of the best practices into one place and does it really well.*

This post is about the **backend** side specifically: how the graph replaced the God Object, how bindings auto-merge
without a central registry, how routes self-register, and how integration tests tap into the live Metro graph
with TestContainers instead of mocks or fakes.

## The Dependency Graph

Everything flows from `BackendGraph`. It's a plain interface annotated `@DependencyGraph(AppScope::class)` - Metro generates the implementation at compile time:

```kotlin
@DependencyGraph(AppScope::class)
interface BackendGraph : BackendTestGraphAccessors {

    // cross cutting accessors needed by Backend.kt
    @get:IoDispatcher val ioDispatcher: CoroutineDispatcher
    val appConfig: ApplicationConfiguration
    val dataSource: DataSource
    val dbSchedulerManager: AppDbScheduler
    val allRoutes: Set<RouteRegistrar>
    val allPlugins: Set<PluginInstaller>

    @DependencyGraph.Factory
    interface Factory {
        fun create(
            @Provides appConfig: ApplicationConfiguration,
            @Provides dataSource: DataSource,
            @Provides dslContext: DSLContext,
            @Provides applicationScope: CoroutineScope,
        ): BackendGraph
    }
}
```

The `Factory` interface is the boundary between compile-time bindings and runtime values.
`dslContext`, `dataSource`, and `applicationScope` are things that only exist after the Ktor application starts
(database pool initialization, application lifecycle scope), so they come in through `@Provides` parameters available inside the graph.

`AppScope` itself is just a class:

```kotlin
abstract class AppScope private constructor()
```

That's literally it. A private constructor so no one accidentally instantiates it, and nothing else.
It's an aggregation marker. Metro uses it to know which bindings belong together and which
`@ContributesTo` contributions to merge into this graph, i think metro provides one by default but i decided to go with my own.

In `Backend.kt`, the graph comes to life after the database initializes:

```kotlin
fun Application.module() {
    val appConfig = ApplicationConfiguration(/* read from application.conf with kotlinx.serialization */)
    val (dataSource, dslContext) = initDatabase(applicationConfiguration = appConfig)

    val graph = createGraphFactory<BackendGraph.Factory>().create(
        appConfig = appConfig,
        dataSource = dataSource,
        dslContext = dslContext,
        applicationScope = this,
    )
    attributes.put(BackendGraphKey, graph) // needed for integration tests

    //we add plugins first
    graph.allPlugins.sortedBy { it.order.order }.forEach { with(it) { install() } }
    //then register routes
    graph.allRoutes.forEach { with(it) { register() } }
}
```

## Providing Infrastructure: @BindingContainer + @ContributesTo

Cross-cutting infrastructure providers for the `Backend` live in `@BindingContainer`.
Each one is annotated `@ContributesTo(AppScope::class)` so Metro merges all contributions automatically -
no central module list, no manual `@Module` includes:

```kotlin
@ContributesTo(AppScope::class)
@BindingContainer
object CoreInfraBindings {

    @Provides
    @SingleIn(AppScope::class)
    fun provideJson(): Json = commonJsonSerialization()

    @Provides
    @SingleIn(AppScope::class)
    fun provideHttpClient(json: Json): HttpClient = HttpClient(OkHttp) {
        engine {
            config {
                ... // your config here
            }
        }
        install(ContentNegotiation) { json(json) }
    }

    @Provides
    @SingleIn(AppScope::class)
    fun provideGoogleIdTokenVerifier(appConfig: ApplicationConfiguration): GoogleIdTokenVerifier =
        GoogleIdTokenVerifier.Builder(NetHttpTransport(), GsonFactory())
            .... //things here
            .build()
}
```

Dispatchers get their own container because they're qualifier-tagged and consumed separately:

```kotlin
@ContributesTo(AppScope::class)
@BindingContainer
object DispatcherBindings {

    @Provides
    @IoDispatcher
    fun provideIoDispatcher(): CoroutineDispatcher = Dispatchers.IO

    @Provides
    @DatabaseDispatcher
    fun provideDatabaseDispatcher(appConfig: ApplicationConfiguration): CoroutineDispatcher =
        Dispatchers.IO.limitedParallelism(
            parallelism = appConfig.database.maximumPoolSize * 2,
            name = "database-dispatcher",
        )
}
```

`@IoDispatcher` and `@DatabaseDispatcher` are just `@Qualifier`-annotated annotations. Metro uses them
to distinguish between multiple `CoroutineDispatcher` bindings at call sites:

```kotlin
@Inject
class XpRepository(
    private val dslContext: DSLContext,
    @DatabaseDispatcher private val databaseDispatcher: CoroutineDispatcher,
)
```

Redis is a good example of a binding with conditional logic - in-memory for local/test environments, Jedis for production:

```kotlin
@ContributesTo(AppScope::class)
@BindingContainer
object RedisBindings {

    @Provides
    @SingleIn(AppScope::class)
    fun provideRedisService(
        appConfig: ApplicationConfiguration,
        json: Json,
        @IoDispatcher ioDispatcher: CoroutineDispatcher,
    ): RedisService = if (appConfig.redis.enabled) {
        JedisRedisService(config = appConfig.redis, json = json, ioDispatcher = ioDispatcher)
    } else {
        InMemoryRedisService(json = json)
    }
}
```

The rest of the codebase just depends on `RedisService` - it never cares which implementation is running.

## Constructor Injection: The Only Kind

Every Repository, Controller, Mapper, and UseCase is annotated `@Inject` and gets all dependencies
through the constructor. No field injection, no `lateinit var`, no `object` singletons with a `get()` mockery intended:

```kotlin
@Inject
class XpRepository(
    private val dslContext: DSLContext,
    @DatabaseDispatcher private val databaseDispatcher: CoroutineDispatcher,
) {

    suspend fun awardXp(
        userId: UUID,
        ...
    ): Boolean = withContext(databaseDispatcher) {
        runCatching {
            // jOOQ insert into the XP events table
        }.getOrDefault(false)
    }

    suspend fun getTotalXp(userId: UUID): Int = withContext(databaseDispatcher) {
        // jOOQ aggregate query - sum XP for user
        0
    }
}
```

jOOQ stays private to the repository layer, keep in mind you can split the repository with interface and impl if you plan to create fakes and unit tests so that you don't bring `dslContext`.

Controllers are equally plain, here's the daily XP claim controller:

```kotlin
@Inject
class DailyXpClaimController(
    private val dailyXpClaimRepository: DailyXpClaimRepository,
    private val xpRepository: XpRepository,
    private val redisService: RedisService,
    private val invalidateUserDetailsCacheUseCase: InvalidateUserDetailsCacheUseCase,
) {

    context(context: RoutingContext)
    suspend fun claim(userId: UUID) {
        // load current streak state from repository
        // other logic
        // award XP, update streak, invalidate caches
    }
}
```

That `context(context: RoutingContext)` is Kotlin context parameters. The controller method receives
`RoutingContext` implicitly - the route's lambda already has it in scope and passes it through without
any explicit `context = this`. The call site in the route looks like:

```kotlin
onUserOrTokenExpired { userId ->
    controller.claim(userId = userId)
}
```

Not `controller.claim(context = this, userId = userId)`. Just clean, this is the power of context parameters an upcoming feature, i'm using it since it got introduced and liking it so far.

## @SingleIn: Opt-In, Not Default

Stateless classes are plain `@Inject class`. Metro creates fresh instances on demand, no shared state,
no contention between concurrent requests.

`@SingleIn(AppScope::class)` is opt-in, reserved for classes that hold shared mutable state across calls:

- `HttpClient` (connection pool lives here)
- `BCrypt.Verifyer` (expensive to allocate)
- `RedisService` (connection pool, `@IoDispatcher` reference)
- Your expensive object allocation here...

## Self-Registering Routes

This is where Metro removes the most boilerplate. The Ktor equivalent of "register this route", with Metro's multibinding, each route declares its own membership:

```kotlin
fun interface RouteRegistrar {
    fun Application.register()
}
```

Every route annotated with `@ContributesIntoSet(AppScope::class)`
is automatically included in `Set<RouteRegistrar>` - the one `BackendGraph` exposes as `allRoutes`, how much i don't miss dagger's `@Binds` where i had to create an extra class just to bind an impl > interface, anyways here's how it looks with Metro:

```kotlin
@Inject
@ContributesIntoSet(AppScope::class)
class ClaimDailyXpRoute(private val controller: DailyXpClaimController) : RouteRegistrar {
    override fun Application.register() {
        routing {
            rateLimit(RateLimitRegistry.api) {
                authenticate(JWT_AUTH) {
                    post(AppRoutes.SOME_ENDPOINT) {
                        onUserOrTokenExpired { userId ->
                            controller.claim(userId = userId)
                        }
                    }
                }
            }
        }
    }
}
```

Add a new file, annotate it, it's registered. Delete the file, it's gone. `Backend.kt` doesn't know
or care how many routes exist:

```kotlin
graph.allRoutes.forEach { with(it) { register() } }
```

Two extension functions do the request lifecycle heavy lifting. `onUserOrTokenExpired` extracts
the userId from the JWT and responds 401 automatically if the token is expired or missing.
`withBody<T>` parses the request body and responds 400 if it's missing or malformed, but that's just helpers totally unrelated to what Metro does but some folks might find them helpful especially about Context parameters:

```kotlin
@Inject
@ContributesIntoSet(AppScope::class)
class RegisterDeviceRoute(private val controller: UserDetailsController) : RouteRegistrar {
    override fun Application.register() {
        routing {
            rateLimit(RateLimitRegistry.api) {
                authenticate(JWT_AUTH) {
                    post(AppRoutes.SOME_OTHER_ENDPOINT) {
                        onUserOrTokenExpired { userId ->
                            withBody<RegisterDeviceRequest> { request ->
                                controller.registerDevice(userId = userId, request = request)
                            }
                        }
                    }
                }
            }
        }
    }
}
```

The route file is entirely focused on request lifecycle: auth, rate limiting, body parsing, delegation.
Business logic lives in the controller. Database work lives in the repository.

## Self-Registering Plugins (with Ordering)

Routes don't need to run in a specific order - the router matches paths, not sequence. Ktor plugins are
different. 
CORS must run before authentication. 
Call ID must be assigned before logging uses it. 
JWT auth must be set up last. 

Order matters.

The mobile app solves a similar issue, which we'll explore in the next article as well.

Same pattern we'll use on the backend. `PluginInstallerOrder` carries the order which is crucial

```kotlin
interface PluginInstaller {
    val order: PluginInstallerOrder
    fun Application.install()
}

enum class PluginInstallerOrder(val order: Int) {
    CORS(order = 100),
    // ...
    CALL_ID(order = 300),
    CONTENT_NEGOTIATION(order = 500),
    // ...
    JWT_AUTHENTICATION(order = 1100),
}
```

Each plugin becomes an `@Inject @ContributesIntoSet` class. Plugins that need nothing receive nothing:

```kotlin
@Inject
@ContributesIntoSet(AppScope::class)
class CallIdPluginInstaller : PluginInstaller {
    override val order: PluginInstallerOrder = PluginInstallerOrder.CALL_ID

    override fun Application.install() {
        install(CallId) {
            header(HttpHeaders.XRequestId)
            generate()
            verify { callId: String -> callId.isNotEmpty() }
            reply { call, callId -> call.response.header(HttpHeaders.XRequestId, callId) }
        }
    }
}
```

Plugins that need dependencies receive them through the constructor. Metro wires them:

```kotlin
@Inject
@ContributesIntoSet(AppScope::class)
class ContentNegotiationPluginInstaller(
    private val json: Json,
) : PluginInstaller {
    override val order: PluginInstallerOrder = PluginInstallerOrder.CONTENT_NEGOTIATION

    override fun Application.install() {
        install(ContentNegotiation) {
            json(json)
        }
    }
}
```

`BackendGraph` gains one accessor. `Backend.kt` replaces eleven explicit calls with one line:

```kotlin
// before
corsPlugin(isDebug = graph.appConfig.environment.environment.allowsDebugFeatures)
metrics()
callId()
logging()
contentNegotiation(json = graph.json)
caching()
defaultHeaders()
rateLimit(rateLimitingConfig = graph.appConfig.rateLimiting, environment = graph.appConfig.environment.environment)
compression()
httpOverride()
jwtAuthentication(authService = graph.authService, jwtConfig = graph.appConfig.jwt)

// after
graph.allPlugins.sortedBy { it.order.order }.forEach { with(it) { install() } }
```

Cleaner imho, if we need to add a new Ktor plugin all we need to do is: create the installer class, pick an order value, annotate with
`@ContributesIntoSet`. That's it. Removing one: delete the file. `Backend.kt` doesn't change.

## Scaling to Multiple Gradle Modules

Rudio's backend lives in a single Gradle module (`backend:server`). That's the right call for where
the project is today. But the `RouteRegistrar` + `PluginInstaller` pattern was designed not to need a rewrite when
the codebase eventually grows large enough to split.

Ktor has [first-class multi-module support](https://ktor.io/docs/server-modules.html). Each `Application`
extension function is a module. `application.conf` declares which ones to load (depends how you structure your project to be initialized with, i use hocon):

```hocon
ktor {
    application {
        modules = [
            app.rudio.wifi.map.backend.server.BackendKt.module
        ]
    }
}
```

At that level, "Ktor module" and "Gradle module" are orthogonal concerns. The single `Backend.kt`
entry point can stay there while the Gradle graph splits. What changes is where the shared interfaces live.

Right now `RouteRegistrar`, `PluginInstaller`, `PluginInstallerOrder`, and the Metro `AppScope` marker
all sit inside `backend:server`. A Gradle split would pull them out into a thin `backend:infra` module
that every feature module depends on:

```
backend:infra
  └─ RouteRegistrar.kt
  └─ PluginInstaller.kt
  └─ PluginInstallerOrder.kt
  └─ di/scope/AppScope.kt   ← AppScope, shared by all modules

backend:feature-poi         (depends on backend:infra)
  └─ @ContributesIntoSet routes/plugins 

backend:feature-user        (depends on backend:infra)
  └─ @ContributesIntoSet routes/plugins

backend:server              (depends on all feature modules)
  └─ BackendGraph.kt        ← @DependencyGraph that aggregates contributions from every module
  └─ Backend.kt             ← unchanged: iterates Set<RouteRegistrar> and Set<PluginInstaller>
```

Then when two needs one you have an umbrella module etc.. etc.. standard gradle config.

Metro's aggregation happens at compile time across all Gradle modules. As long as every
`@ContributesIntoSet(AppScope::class, …)` annotation references the *same* `AppScope` class from `backend:infra`, Metro collects them all into the sets that `BackendGraph` exposes.

The `@BindingContainer` objects (dispatchers, third-party infra, Redis) would live either in
`backend:infra` or in a dedicated `backend:core` or split by 'specialty' or 'functionality' module alongside the config types they bind. Feature
modules never touch `BackendGraph` - they just contribute and let Metro do the wiring.

The end result: `Backend.kt` still looks exactly the same:

```kotlin
graph.allPlugins.sortedBy { it.order.order }.forEach { with(it) { install() } }
graph.allRoutes.forEach { with(it) { register() } }
```

Splitting a feature out into its own Gradle module is purely a build-system concern. The runtime
behaviour, the DI wiring, and the `Backend.kt` entry point are untouched.

## Integration Tests: The Live Graph Is The Fixture

Metro's [testing-with-fakes](https://zacsweers.github.io/metro/latest/quickstart/#testing-with-fakes) pattern
is the canonical Metro testing approach: construct the class under test directly via its constructor,
pass `Fake…` implementations for any deps you want to control, and never involve a DI container
in the test at all. It works well and is the right choice for unit and component tests.

We're not doing that, at least not in my case because...

The decision was deliberate, Rudio;s backend is a spatial data service using PostGIS functions, GIST indexes,
geography distance queries, Redis TTL semantics, and S3 presigned URL lifetimes all have to work correctly
together.

So instead: a real Ktor server, a real PostGIS/PostgreSQL container, a real Redis container, and a real
MinIO S3 container. `TestContainers` spins them up once per reuses them across runs. The data is
pre-seeded close to production shape - actual geographic coordinates, realistic WiFi data, properly
sequenced user onboarding state and whatever we need for other features as well.

**Before Metro**, the test HTTP client was wired directly to the God Object:

```kotlin
// old - test client used BackendComponent directly
val client: HttpClient = HttpClient(CIO) {
    install(ContentNegotiation) {
        json(BackendComponent.json)  // pulled from the singleton
    }
}
```

If you wanted to assert on repository state, you'd either construct a repository with default params
(again pulling from `BackendComponent` internally) looking something like:

```kotlin
class PoiRateLimitIntegrationTest : BaseIntegrationTest() {
    private val redisService: RedisService get() = BackendComponent.redisService //using the god object here once again
}
```

**After Metro**, two lines in `WithTestServer` change everything:

```kotlin
// in WithTestServer, after server starts:
lateinit var testGraph: BackendGraph
    private set

// ... then after startup:
testGraph = server.application.attributes[BackendGraphKey]
```

And in `BackendTestComponent`:

```kotlin
val graph get() = WithTestServer.testGraph
```

`BackendTestComponent` owns the containers:

```kotlin
object BackendTestComponent {

    // IMPORTANT: Keep these versions in sync with docker-compose.yml
    private const val POSTGRES_VERSION = "postgis/postgis:18-3.6"
    private const val REDIS_VERSION = "redis:8.6"

    val postgres = PostgreSQLContainer(
        DockerImageName.parse(POSTGRES_VERSION).asCompatibleSubstituteFor("postgres")
    ).apply {
        withDatabaseName("rudio_test")
        withUsername("test")
        withPassword("test")
        withReuse(true)
        start()
    }

    private val redis = RedisContainer(REDIS_VERSION).apply {
        withReuse(true)
        start()
    }

    private val minio = MinIOContainer("minio/minio:latest").apply {
        withReuse(true)
        start()
        // creates the test bucket via S3Client on startup
    }

    val graph get() = WithTestServer.testGraph
}
```

`WithTestServer` starts the actual Ktor server once, using `ApplicationConfig.mergeWith`
to override the database connection, Redis, S3, and a few other test-specific knobs:

```kotlin
private val mergedConfig = ApplicationConfig("application.conf").mergeWith(
    MapApplicationConfig(
        "database.jdbcUrl" to BackendTestComponent.postgresJdbcUrl,
        "database.username" to BackendTestComponent.postgresUsername,
        "database.password" to BackendTestComponent.postgresPassword,
        "environment.type" to "TEST",
        "redis.host" to BackendTestComponent.redisHost,
        "redis.port" to BackendTestComponent.redisPort.toString(),
        // override any other infrastructure coordinates for test containers
    )
)
```

After the server is up, it extracts the live `BackendGraph` from the application attributes:

```kotlin
testGraph = server.application.attributes[BackendGraphKey]
serverStarted = true
```

That's the same Metro-wired graph that handles every HTTP request during the tests.

`BackendTestGraphAccessors` is a separate interface that `BackendGraph` extends, keeping the production
surface clean while exposing what tests need:

```kotlin
interface BackendTestGraphAccessors {
    val json: Json
    val authService: AuthService
    val redisService: RedisService
    // repositories and services tests need to assert on
    val someFeatureRepository: SomeFeatureRepository
    val anotherFeatureRepository: AnotherFeatureRepository
    val redisService: RedisService
    // ...
}
```

Metro generates implementations for every accessor here because `BackendGraph : BackendTestGraphAccessors`.
The rule: no production code touches these accessors. Production code gets things through constructor injection.

`BaseIntegrationTest` is two lines:

```kotlin
abstract class BaseIntegrationTest : WithTestServer() {
    val client: HttpClient get() = BackendTestComponent.client
}
```

A typical test class extends it, declares the repositories it needs, and goes:

```kotlin
@TestMethodOrder(MethodOrderer.OrderAnnotation::class)
class CreateFeatureIntegrationTest : BaseIntegrationTest() {

    private val redisService: RedisService get() = BackendTestComponent.graph.redisService
    private val featureRepository get() = BackendTestComponent.graph.someFeatureRepository

    private var accessToken: String = ""
    private var createdEntityId: String = ""

    @Test
    @Order(1)
    @DisplayName("Create entity without auth returns Unauthorized")
    fun createWithoutAuth(): Unit = runTest {
        val response = client.post(FeatureRoutes.CREATE) {
            setBody(CreateFeatureRequest(/* ... */))
        }
        assertEquals(HttpStatusCode.Unauthorized, response.status)
    }

    @Test
    @Order(2)
    @DisplayName("Authenticated user creates entity - verifies Redis cache and DB state")
    fun createAuthenticated(): Unit = runTest {
        val cacheKeyBefore = redisService.get(/* relevant cache key */)
        assertNull(cacheKeyBefore) // cache should be cold before creation

        val response = client.post(FeatureRoutes.CREATE) {
            bearerAuth(accessToken)
            setBody(CreateFeatureRequest(/* ... */))
        }
        assertEquals(HttpStatusCode.Created, response.status)

        val body = response.body<CreateFeatureResponse>()
        createdEntityId = body.id

        // bypass HTTP - assert on repository state directly
        val entity = featureRepository.findById(UUID.fromString(createdEntityId))
        assertNotNull(entity)
    }
}
```

The HTTP call goes through the full Ktor pipeline (rate limiting, auth, controller, repository, jOOQ, PostgreSQL).
Then you assert on the repository directly to verify the exact DB state rather than trusting that the
response body captured everything that happened.

For race conditions, coroutines make it trivial:

```kotlin
@Test
@Order(5)
@DisplayName("Concurrent claims - only one succeeds")
fun concurrentClaimsOnlyOneSucceeds(): Unit = runTest {
    coroutineScope {
        val results = (1..10).map {
            async {
                client.post(FeatureRoutes.CLAIM) { bearerAuth(userToken) }
            }
        }.awaitAll()

        val successCount = results.count { it.status == HttpStatusCode.Created }
        val conflictCount = results.count { it.status == HttpStatusCode.Conflict }

        assertEquals(1, successCount, "only one claim should succeed")
        assertEquals(9, conflictCount, "all other concurrent attempts should conflict")
    }
}
```

One thing i really miss from Dagger is the IDE integration where you can navigate through dependencies, i really wish Metro gets that one day.

This is the DI setup powering the backend of [Rudio](https://rudio.app/) - find free WiFi hotspots around you.

Stay hydrated and keep your dependencies injected, until next article... where will be talking about the adoption of Metro in the other part of the project (KMP and CMP), stay curious.