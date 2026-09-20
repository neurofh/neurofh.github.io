---
title: "Android ViewModel Data Loading: Best Practices and Flow-Based Architecture"
date: 2025-08-29 01:11:00 +0200
categories: [Android Development, Architecture, MVVM, MVI]
tags: [Android, ViewModel, Kotlin Flows, StateFlow, MVVM, Architecture, Best Practices, Testing]
---

Architecture discussions in Android development often spark passionate debates—sometimes garnering both praise and criticism. Writing about these topics isn't easy, but that's what makes it worthwhile.

This article presents my opinionated perspective on data loading patterns, refined through experience and recovery from recent surgeries (one recovery is still in progress). 

Consider this a snapshot of my understanding and skills in 2025.

I may be joining this conversation later than others, but better late than never.

## The Challenge: Common Data Loading Antipatterns in Android ViewModels

"Most" Android developers use ViewModels to manage UI state, which gets collected by Views (Fragments, Activities, or Composables), to display meaningful content, you need to load data from a source of truth, transform it into view state, and expose it for consumption.

Previously it was LiveData, nowadays it's Flow that acts as a glue between the View and the ViewModel (mostly), there are solutions utilizing [molecule](https://github.com/cashapp/molecule) but this is out of our scope.

![Twitter discussion](/assets/img/load_data/zhui.png)

As shown in this [Twitter discussion](https://x.com/github_skydoves/status/1829315087611707848), most developers load data within the `ViewModel's init {}` block. While this approach seems logical, it creates several architectural issues that Ian Lake and others have identified as antipatterns—including the use of `LaunchedEffect` for data loading.

![Irony](/assets/img/load_data/jokes.png)

The irony becomes apparent when even official samples sometimes contradict these best practices:

![Irony](/assets/img/load_data/irony.png)

### Why Developers Choose init {} Block (And Why It's Problematic)

The appeal of the `ViewModel's init {}` block is understandable—it ensures data loading survives configuration changes, preventing unnecessary API calls or database reads. However, this approach introduces four critical issues:

#### Problem #1: Navigation Backstack Complications

When using `init {}` for data loading, returning to a screen with an existing ViewModel won't trigger re-initialization. This forces developers to add workaround logic in `onStart` or `onResume` to check data freshness—creating spaghetti code that's hard to maintain.

#### Problem #2: Dispatcher Race Conditions  

Data loading in `init {}` typically uses `viewModelScope`, which runs on `Dispatchers.Main.immediate`. This immediate dispatcher can cause race conditions where data processing completes before UI composition, especially in Jetpack Compose applications.

![Darkness](/assets/img/load_data/darkness.png)

#### Problem #3: Data Staleness Issues

Modern CRUD applications require fresh data. Users might return from other screens or resume from a paused state after significant time has passed. The `init {}` approach doesn't provide built-in mechanisms for data freshness validation.

#### Problem #4: Testing difficulties

Every time you have to run a test, you have to construct your ViewModel in order to successfully run the `init {}` block for that specific test case.

## The Flow-Based Solution: Turning Cold Flows Hot

The solution leverages Kotlin Flows—specifically transforming cold flows into hot ones using StateFlow with proper sharing strategies. Think of it as the Katy Perry "Hot N Cold" approach, but with predictable behavior for all edge cases.

### Building the Foundation: Use Case and ViewModel Structure

**DO NOTE THAT THIS CODE IS FOR DEMONSTRATION PURPOSES ONLY, HOW YOU STRUCTURE IT IS UP TO YOU, IT'S NOT BEST PRACTICES EXCEPT THE PART ABOUT LOADING**

```kotlin
inline fun <reified T : ViewModel> provideFactory(
    crossinline creator: () -> T
) = viewModelFactory {
    initializer {
        creator()
    }
}
```

*Note: This factory pattern is for demonstration purposes only*

Our use case handles data retrieval, formatting, and business logic transformation:

```kotlin
class GetUserDetailsUseCase private constructor(
    private val authRepository: AuthRepository = AuthRepository(),
    private val dispatcher: CoroutineDispatcher = Dispatchers.IO,
    private val billingCache: BillingCache = BillingCache.create(),
    private val dateFormatter : DateFormatter = DataFormatter()
) {
    suspend fun execute(): Result<UserDetails> =
        withContext(dispatcher) {
            val userDetails: Result<UserDetailsResponseModel> = authRepository.getUserDetails()

            userDetails.map { details ->
                UserDetails(
                    creationDate = dateFormatter.format(details.creationDate, DateFormatter.Format.UTC_SHORT).getOrNull(),
                    avatarUrl = details.avatar,
                    isPremium = billingCache.isPremium(),
                    email = details.email
                )
            }
        }

    companion object {
        fun create() = GetUserDetailsUseCase()
    }
}
```

This use case encapsulates data retrieval from the repository, date formatting, premium status validation, and prepares user information for display.

```kotlin
internal class UserAccountDetailsViewModel private constructor(
    private val getUserDetailsUseCase: GetUserDetailsUseCase = GetUserDetailsUseCase.create(),
) : ViewModel() {

    data class ViewState(
        val isLoading: Boolean = false,
        val isError: Boolean = false,
        val userInfo: UserInfo? = null
    ) {
        data class UserInfo(
            val displayEmail: String,
            val avatarUrl: String?,
            val showPremiumBadge: Boolean,
            val memberSince: String?
        )
    }

    val userDetails: Flow<ViewState> = flow {
        emit(
            getUserDetailsUseCase.execute()
                .fold(
                    onSuccess = {
                        ViewState(
                            isLoading = false,
                            isError = false,
                            userInfo = ViewState.UserInfo(
                                displayEmail = it.email,
                                avatarUrl = it.avatarUrl,
                                showPremiumBadge = it.isPremium,
                                memberSince = it.creationDate?.toString()
                            )
                        )
                    },
                    onFailure = {
                        ViewState(isLoading = false, isError = true)
                    }
                )
        )
    }.stateIn(
        viewModelScope,
        SharingStarted.WhileSubscribed(5_000),
        ViewState(isLoading = true, isError = false)
    )

    companion object {
        fun factory() = provideFactory { UserAccountDetailsViewModel() }
    }
}
```

This approach provides several key benefits:

- **Data Freshness**: The 5-second timeout matches Android's ANR threshold, ensuring data refreshes when collectors reappear after the timeout period
- **Configuration Change Handling**: Data persists through configuration changes within the timeout window
- **Resource Efficiency**: Prevents unnecessary network calls for quick navigation patterns

*Pro tip: Set timeout to 0 for applications requiring real-time data freshness*

### Adding User Interaction: Implementing Refresh Functionality

Real-world applications need user-initiated data refresh capabilities. Product managers love swipe-to-refresh, but our basic flow doesn't accommodate this pattern. Let's enhance our architecture:

We implement this using a `MutableSharedFlow` that triggers within our main flow collector:

```kotlin
internal class UserAccountDetailsViewModel private constructor(
    private val getUserDetailsUseCase: GetUserDetailsUseCase = GetUserDetailsUseCase.create(),
) : ViewModel(), IntentAware<UserAccountDetailsViewModel.ViewState.Intents>  {

    data class ViewState(
        val isLoading: Boolean = false,
        val isError: Boolean = false,
        val userInfo: UserInfo? = null
    ) {
        data class UserInfo(
            val displayEmail: String,
            val avatarUrl: String?,
            val showPremiumBadge: Boolean,
            val memberSince: String?
        )

        sealed class Intents {
            data object Refresh : Intents()
        }

    }

    private val refreshListener = MutableSharedFlow<Unit>()
    val userDetails: Flow<ViewState> = flow {
        emit(getUserDetailsState())

        refreshListener.collect {
            emit(ViewState(isLoading = true, isError = false))
            emit(getUserDetailsState())
        }
    }.stateIn(
        viewModelScope,
        SharingStarted.WhileSubscribed(5_000),
        ViewState(isLoading = true, isError = false)
    )

    private suspend fun getUserDetailsState(): ViewState = getUserDetailsUseCase.execute()
        .fold(
            onSuccess = {
                ViewState(
                    isLoading = false,
                    isError = false,
                    userInfo = ViewState.UserInfo(
                        displayEmail = it.email,
                        avatarUrl = it.avatarUrl,
                        showPremiumBadge = it.isPremium,
                        memberSince = it.creationDate?.toString()
                    )
                )
            },
            onFailure = {
                ViewState(isLoading = false, isError = true)
            }
        )

    override fun onIntent(intent: ViewState.Intents) {
        when (intent) {
            ViewState.Intents.Refresh -> {
                viewModelScope.launch {
                    refreshListener.emit(Unit)
                }
            }
        }
    }

    companion object {
        fun factory() = provideFactory { UserAccountDetailsViewModel() }
    }
}                      
```

Perfect! We've successfully refactored our duplicated data loading logic and implemented refresh functionality. However, we're not finished yet.

### Optimizing State Management: Eliminating Redundant Emissions

To prevent unnecessary UI updates, we'll add `distinctUntilChanged()` to filter duplicate state emissions.

### Handling Complex State Updates

For scenarios where intents modify UI state without requiring data reloading, we need access to current state within our flow operations. Consider toggling email visibility—this requires state modification rather than data reloading.

```kotlin
 data class ViewState(
        val isLoading: Boolean = false,
        val isError: Boolean = false,
        val isEmailVisible : Boolean = false,
        val userInfo: UserInfo? = null
    ) {
        data class UserInfo(
            val displayEmail: String,
            val avatarUrl: String?,
            val showPremiumBadge: Boolean,
            val memberSince: String?
        )

        sealed class Intents {
            data object Refresh : Intents()
            data class ToggleEmailVisibility(val isEmailVisible: Boolean) : Intents()
        }

        sealed class StateParameters {
            data class EmailVisibilityChanged(val isEmailVisible: Boolean) : StateParameters()
            data object Refresh : StateParameters()
        }
    }

    private val refreshListener = MutableSharedFlow<ViewState.StateParameters>()
    val userDetails: Flow<ViewState> = flow {
        emit(getUserDetailsState())

        refreshListener.collect { refreshParams ->
            when(refreshParams){
                is ViewState.StateParameters.EmailVisibilityChanged -> {
                    //do some changes here
                }
                ViewState.StateParameters.Refresh -> {
                    emit(ViewState(isLoading = true, isError = false))
                    emit(getUserDetailsState())
                }
            }
        }
    }
        .distinctUntilChanged()
        .stateIn(
        viewModelScope,
        SharingStarted.WhileSubscribed(5_000),
        ViewState(isLoading = true, isError = false)
    )
```

The challenge here is accessing current state within the flow. Let's solve this by tracking state internally:

```kotlin
internal class UserAccountDetailsViewModel private constructor(
    private val getUserDetailsUseCase: GetUserDetailsUseCase = GetUserDetailsUseCase.create(),
) : ViewModel(), IntentAware<UserAccountDetailsViewModel.ViewState.Intents>  {

    data class ViewState(
        val isLoading: Boolean = false,
        val isError: Boolean = false,
        val isEmailVisible : Boolean = false,
        val userInfo: UserInfo? = null
    ) {
        data class UserInfo(
            val displayEmail: String,
            val avatarUrl: String?,
            val showPremiumBadge: Boolean,
            val memberSince: String?
        )

        sealed class Intents {
            data object Refresh : Intents()
            data class ToggleEmailVisibility(val isEmailVisible: Boolean) : Intents()
        }

        sealed class StateTriggers {
            data class EmailVisibilityChanged(val isEmailVisible: Boolean) : StateTriggers()
            data object Refresh : StateTriggers()
        }
    }

    private var currentState = ViewState(isLoading = true, isError = false)
    private val refreshListener = MutableSharedFlow<ViewState.StateTriggers>()
    val userDetails: Flow<ViewState> = flow {
        emit(getUserDetailsState())
        refreshListener.collect { refreshParams ->
            when(refreshParams){
                is ViewState.StateTriggers.EmailVisibilityChanged -> {
                    emit(currentState.copy(isEmailVisible = refreshParams.isEmailVisible))
                }
                ViewState.StateTriggers.Refresh -> {
                    emit(ViewState(isLoading = true, isError = false))
                    emit(getUserDetailsState())
                }
            }
        }
    }
        .distinctUntilChanged()
        .onEach {
            currentState = it
        }
        .stateIn(
        viewModelScope,
        SharingStarted.WhileSubscribed(5_000),
        currentState
    )

    private suspend fun getUserDetailsState(): ViewState = getUserDetailsUseCase.execute()
        .fold(
            onSuccess = {
                ViewState(
                    isLoading = false,
                    isError = false,
                    userInfo = ViewState.UserInfo(
                        displayEmail = it.email,
                        avatarUrl = it.avatarUrl,
                        showPremiumBadge = it.isPremium,
                        memberSince = it.creationDate?.toString()
                    )
                )
            },
            onFailure = {
                ViewState(isLoading = false, isError = true)
            }
        )

    override fun onIntent(intent: ViewState.Intents) {
        when (intent) {
            ViewState.Intents.Refresh -> {
                viewModelScope.launch {
                    refreshListener.emit(ViewState.StateTriggers.Refresh)
                }
            }

            is ViewState.Intents.ToggleEmailVisibility -> {
                viewModelScope.launch {
                    refreshListener.emit(ViewState.StateTriggers.EmailVisibilityChanged(intent.isEmailVisible))
                }
            }
        }
    }

    companion object {
        fun factory() = provideFactory { UserAccountDetailsViewModel() }
    }
}
```

### Intelligent Data Caching: Conditional Loading Strategy

Now we can achieve sophisticated caching behavior. Since `currentState` survives ViewModel lifecycle, we can emit cached data immediately and conditionally load fresh data only when necessary:

```kotlin
internal class UserAccountDetailsViewModel private constructor(
    private val getUserDetailsUseCase: GetUserDetailsUseCase = GetUserDetailsUseCase.create(),
) : ViewModel(), IntentAware<UserAccountDetailsViewModel.ViewState.Intents> {

    data class ViewState(
        val isLoading: Boolean = false,
        val isError: Boolean = false,
        val isEmailVisible: Boolean = false,
        val userInfo: UserInfo? = null
    ) {
        val isDataLoaded get() = userInfo != null

        data class UserInfo(
            val displayEmail: String,
            val avatarUrl: String?,
            val showPremiumBadge: Boolean,
            val memberSince: String?
        )

        sealed class Intents {
            data object Refresh : Intents()
            data class ToggleEmailVisibility(val isEmailVisible: Boolean) : Intents()
        }

        sealed class StateTriggers {
            data class EmailVisibilityChanged(val isEmailVisible: Boolean) : StateTriggers()
            data object Refresh : StateTriggers()
        }
    }


    private var currentState = ViewState(isLoading = true, isError = false)
    private val refreshListener = MutableSharedFlow<ViewState.StateTriggers>()
    val userDetails: Flow<ViewState> = flow {
        emit(currentState)

        //i added error check just because this is for demonstration of this edge case
        if (currentState.isDataLoaded.not() || currentState.isError) { 
            emit(getUserDetailsState())
        }
        
        refreshListener.collect { refreshParams ->
            when (refreshParams) {
                is ViewState.StateTriggers.EmailVisibilityChanged -> {
                    emit(currentState.copy(isEmailVisible = refreshParams.isEmailVisible))
                }

                ViewState.StateTriggers.Refresh -> {
                    emit(ViewState(isLoading = true, isError = false))
                    emit(getUserDetailsState())
                }
            }
        }
    }
        .distinctUntilChanged()
        .onEach {
            currentState = it
        }
        .stateIn(
            viewModelScope,
            SharingStarted.WhileSubscribed(5_000),
            currentState
        )

    private suspend fun getUserDetailsState(): ViewState = getUserDetailsUseCase.execute()
        .fold(
            onSuccess = {
                ViewState(
                    isLoading = false,
                    isError = false,
                    userInfo = ViewState.UserInfo(
                        displayEmail = it.email,
                        avatarUrl = it.avatarUrl,
                        showPremiumBadge = it.isPremium,
                        memberSince = it.creationDate?.toString()
                    )
                )
            },
            onFailure = {
                ViewState(isLoading = false, isError = true)
            }
        )

    override fun onIntent(intent: ViewState.Intents) {
        when (intent) {
            ViewState.Intents.Refresh -> {
                viewModelScope.launch {
                    refreshListener.emit(ViewState.StateTriggers.Refresh)
                }
            }

            is ViewState.Intents.ToggleEmailVisibility -> {
                viewModelScope.launch {
                    refreshListener.emit(ViewState.StateTriggers.EmailVisibilityChanged(intent.isEmailVisible))
                }
            }
        }
    }

    companion object {
        fun factory() = provideFactory { UserAccountDetailsViewModel() }
    }
}
```

This pattern provides intelligent caching—even after the 5-second timeout, you can choose whether to refetch data based on your specific requirements:

- **Expensive API calls**: Cache data in ViewModel to reduce network overhead
- **Static backend data**: Avoid unnecessary requests for rarely-changing information  
- **Real-time requirements**: Force refresh for applications requiring fresh data

### Creating Reusable Abstractions

Writing this pattern repeatedly becomes tedious. Let's create reusable abstractions, starting with ViewModel extension functions:

```kotlin
fun <T, R> ViewModel.loadData(
    initialState: T,
    loadData: suspend FlowCollector<T>.(currentState: T) -> Unit,
    refreshMechanism: SharedFlow<R>? = null,
    timeout: Long = 5_000,
    refreshData: (suspend FlowCollector<T>.(currentState: T, refreshParams: R) -> Unit)? = null,
): StateFlow<T> {
    if (refreshMechanism != null) {
        requireNotNull(refreshData) {
            "You've provided a refresh mechanism but no way to refresh the data"
        }
    }
    if (refreshData != null) {
        requireNotNull(refreshMechanism) {
            "You've provided a refresh data but no mechanism to refresh the data"
        }
    }

    var latestValue = initialState
    return flow {
        emit(latestValue)
        loadData(latestValue)
        refreshMechanism?.collect { refreshParams ->
            if (refreshData != null) {
                refreshData(latestValue, refreshParams)
            }
        }
    }
        .distinctUntilChanged()
        .onEach {
            latestValue = it
        }
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(timeout),
            initialValue = initialState
        )
}

fun <T> ViewModel.loadData(
    initialState: T,
    loadData: suspend FlowCollector<T>.(currentState: T) -> Unit,
    timeout: Long = 5_000,
): StateFlow<T> {
    var latestValue = initialState
    return flow {
        emit(latestValue)
        loadData(latestValue)
    }
        .onEach {
            latestValue = it
        }
        .distinctUntilChanged()
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(timeout),
            initialValue = initialState
        )
}
```

With our abstracted extension functions, ViewModels become much cleaner:

*Note: This abstraction covers 90% of common use cases but doesn't support complex flow chaining operations*

```kotlin
internal class UserAccountDetailsViewModel private constructor(
    private val getUserDetailsUseCase: GetUserDetailsUseCase = GetUserDetailsUseCase.create(),
) : ViewModel(), IntentAware<UserAccountDetailsViewModel.ViewState.Intents> {

    data class ViewState(
        val isLoading: Boolean = false,
        val isError: Boolean = false,
        val isEmailVisible: Boolean = false,
        val userInfo: UserInfo? = null
    ) {
        val isDataLoaded get() = userInfo != null

        data class UserInfo(
            val displayEmail: String,
            val avatarUrl: String?,
            val showPremiumBadge: Boolean,
            val memberSince: String?
        )

        sealed class Intents {
            data object Refresh : Intents()
            data class ToggleEmailVisibility(val isEmailVisible: Boolean) : Intents()
        }

        sealed class StateTriggers {
            data class EmailVisibilityChanged(val isEmailVisible: Boolean) : StateTriggers()
            data object Refresh : StateTriggers()
        }
    }


    private val refreshListener = MutableSharedFlow<ViewState.StateTriggers>()
    val userDetails = loadData(
        initialState = ViewState(isLoading = true, isError = false),
        loadData = { currentState ->
            if (currentState.isDataLoaded.not() || currentState.isError.not()) {
                emit(getUserDetailsState())
            }
        },
        refreshMechanism = refreshListener,
        refreshData = { currentState, refreshParams ->
            when (refreshParams) {
                is ViewState.StateTriggers.EmailVisibilityChanged -> {
                    emit(currentState.copy(isEmailVisible = refreshParams.isEmailVisible))
                }

                ViewState.StateTriggers.Refresh -> {
                    emit(ViewState(isLoading = true, isError = false))
                    emit(getUserDetailsState())
                }
            }
        }
    )

    private suspend fun getUserDetailsState(): ViewState = getUserDetailsUseCase.execute()
        .fold(
            onSuccess = {
                ViewState(
                    isLoading = false,
                    isError = false,
                    userInfo = ViewState.UserInfo(
                        displayEmail = it.email,
                        avatarUrl = it.avatarUrl,
                        showPremiumBadge = it.isPremium,
                        memberSince = it.creationDate?.toString()
                    )
                )
            },
            onFailure = {
                ViewState(isLoading = false, isError = true)
            }
        )

    override fun onIntent(intent: ViewState.Intents) {
        when (intent) {
            ViewState.Intents.Refresh -> {
                viewModelScope.launch {
                    refreshListener.emit(ViewState.StateTriggers.Refresh)
                }
            }

            is ViewState.Intents.ToggleEmailVisibility -> {
                viewModelScope.launch {
                    refreshListener.emit(ViewState.StateTriggers.EmailVisibilityChanged(intent.isEmailVisible))
                }
            }
        }
    }

    companion object {
        fun factory() = provideFactory { UserAccountDetailsViewModel() }
    }
}
```
We can eliminate the repetitive `refreshListener` declaration by creating a more sophisticated base class:

```kotlin
abstract class ViewModelLoader<State : Any, Intent : Any, Trigger : Any> : ViewModel() {

    private val _trigger by lazy { MutableSharedFlow<Trigger>() }

    fun <T> loadData(
        initialState: T,
        loadData: suspend FlowCollector<T>.(currentState: T) -> Unit,
        triggerData: (suspend FlowCollector<T>.(currentState: T, triggerParams: Trigger) -> Unit)? = null,
        timeout: Long = 5000L, //matching ANR timeout in Android
    ): StateFlow<T> {
        var latestValue = initialState
        return flow {
            emit(latestValue)
            loadData(latestValue)
            if (triggerData != null) {
                _trigger.collect { triggerParams ->
                    triggerData(this, latestValue, triggerParams)
                }
            }
        }
            .distinctUntilChanged()
            .onEach {
                latestValue = it
            }
            .stateIn(
                scope = viewModelScope,
                started = SharingStarted.WhileSubscribed(timeout),
                initialValue = initialState
            )
    }

    abstract val state: StateFlow<State>

    val currentState get() = state.value

    open fun onIntent(intent: Intent) {}

    protected fun sendTrigger(trigger: Trigger) {
        viewModelScope.launch {
            _trigger.emit(trigger)
        }
    }

}
```

Our final implementation becomes remarkably clean and maintainable:

```kotlin
internal class UserAccountDetailsViewModel private constructor(
    private val getUserDetailsUseCase: GetUserDetailsUseCase = GetUserDetailsUseCase.create(),
) : ViewModelLoader<UserAccountDetailsViewModel.ViewState, UserAccountDetailsViewModel.ViewState.Intents, UserAccountDetailsViewModel.ViewState.StateTriggers>() {

    data class ViewState(
        val isLoading: Boolean = false,
        val isError: Boolean = false,
        val isEmailVisible: Boolean = false,
        val userInfo: UserInfo? = null
    ) {
        val isDataLoaded get() = userInfo != null

        data class UserInfo(
            val displayEmail: String,
            val avatarUrl: String?,
            val showPremiumBadge: Boolean,
            val memberSince: String?
        )

        sealed class Intents {
            data object Refresh : Intents()
            data class ToggleEmailVisibility(val isEmailVisible: Boolean) : Intents()
        }

        sealed class StateTriggers {
            data class EmailVisibilityChanged(val isEmailVisible: Boolean) : StateTriggers()
            data object Refresh : StateTriggers()
        }
    }


    override val state = loadData(
        initialState = ViewState(isLoading = true, isError = false),
        loadData = { currentState ->
            if (currentState.isDataLoaded.not() || currentState.isError.not()) {
                emit(getUserDetailsState())
            }
        },
        triggerData = { currentState, refreshParams ->
            when (refreshParams) {
                is ViewState.StateTriggers.EmailVisibilityChanged -> {
                    emit(currentState.copy(isEmailVisible = refreshParams.isEmailVisible))
                }

                ViewState.StateTriggers.Refresh -> {
                    emit(ViewState(isLoading = true, isError = false))
                    emit(getUserDetailsState())
                }
            }
        }
    )

    private suspend fun getUserDetailsState(): ViewState = getUserDetailsUseCase.execute()
        .fold(
            onSuccess = {
                ViewState(
                    isLoading = false,
                    isError = false,
                    userInfo = ViewState.UserInfo(
                        displayEmail = it.email,
                        avatarUrl = it.avatarUrl,
                        showPremiumBadge = it.isPremium,
                        memberSince = it.creationDate?.toString()
                    )
                )
            },
            onFailure = {
                ViewState(isLoading = false, isError = true)
            }
        )

    override fun onIntent(intent: ViewState.Intents) {
        when (intent) {
            ViewState.Intents.Refresh -> {
                sendTrigger(ViewState.StateTriggers.Refresh)
            }

            is ViewState.Intents.ToggleEmailVisibility -> {
                sendTrigger(ViewState.StateTriggers.EmailVisibilityChanged(intent.isEmailVisible))
            }
        }
    }

    companion object {
        fun factory() = provideFactory { UserAccountDetailsViewModel() }
    }
}
```

### Handling UI State Complexity

This abstraction uses boolean flags (`isLoading`, `isError`) which can create ambiguous states. For clearer state management, consider using sealed classes:
```kotlin
@Immutable
sealed interface UIState {

    @Immutable
    data object Success : UIState

    @Immutable
    data object Error : UIState

    @Immutable
    data object Idle : UIState

    @Immutable
    data object Loading : UIState
}
```
For scenarios requiring simultaneous error display (like Snackbars) alongside existing data, you can create more sophisticated state holders:

```kotlin
@Immutable
data class UIStateHolder<out T>(
    val uiState: UIState = UIState.Idle,
    val payload: T? = null
)
```

This approach enables flexible UI state management while maintaining clear separation between UI state and data payload but might increase cognitive load and introduce more mapping behavior and unwrapping logic.

### Extending Beyond ViewModels

This pattern isn't limited to ViewModels. By providing a custom CoroutineScope, you can use this data loading approach in any component—Composables, repositories, or business logic layers.

## Flow Combination Patterns

The beauty of this approach extends to combining multiple data sources. Here are examples for handling one and two flows:

```kotlin
inline fun <reified T, R> ViewModel.loadFlow(
    initialState: R,
    flow: Flow<T>,
    crossinline transform: suspend CoroutineScope.(newValue: T, currentState: R) -> R,
    timeout: Long = 0,
): StateFlow<R> {
    var latestValue = initialState
    return flow
        .map { newValue ->
            coroutineScope {
                transform(newValue, latestValue)
            }
        }
        .onEach {
            latestValue = it
        }.stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(timeout),
            initialValue = latestValue
        )
}
```

For combining two flows:

```kotlin
inline fun <reified T1, reified T2, R> ViewModel.loadFlow(
    initialState: R,
    flow1: Flow<T1>,
    flow2: Flow<T2>,
    crossinline transform: suspend CoroutineScope.(newValue1: T1, newValue2: T2, currentState: R) -> R,
    timeout: Long = 0,
): StateFlow<R> {
    var latestValue = initialState
    return combine(flow1, flow2) { value1, value2 ->
        coroutineScope {
            transform(value1, value2, latestValue)
        }
    }
        .onEach {
            latestValue = it
        }.stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(timeout),
            initialValue = latestValue
        )
}
```

These extensions allow you to easily combine multiple data sources while maintaining the same intelligent caching and state management principles.

### Testing Your Flow-Based ViewModels

Before we conclude, let's explore how to properly test our `UserAccountDetailsViewModel` implementation using fakes and Turbine for flow testing.

#### Setting Up Test Dependencies with Fakes and Turbine

```kotlin
@OptIn(ExperimentalCoroutinesApi::class)
class UserAccountDetailsViewModelTest {

    private val testDispatcher = StandardTestDispatcher()
    private val fakeGetUserDetailsUseCase = FakeGetUserDetailsUseCase()
    private lateinit var viewModel: UserAccountDetailsViewModel

    @Before
    fun setUp() {
        Dispatchers.setMain(testDispatcher)
        viewModel = UserAccountDetailsViewModel(fakeGetUserDetailsUseCase)
    }

    @After
    fun tearDown() {
        Dispatchers.resetMain()
    }
}

// Fake implementation for realistic testing, this sounds funny to write haha
class FakeGetUserDetailsUseCase {
    private var shouldReturnError = false
    private var userDetailsToReturn: UserDetails? = null
    private var executionCount = 0
    
    fun setSuccessResponse(userDetails: UserDetails) {
        this.userDetailsToReturn = userDetails
        this.shouldReturnError = false
    }
    
    fun setErrorResponse() {
        this.shouldReturnError = true
        this.userDetailsToReturn = null
    }
    
    // Expose execution count when testing caching/performance behavior
    fun getExecutionCount() = executionCount
    fun reset() { executionCount = 0 }
    
    suspend fun execute(): Result<UserDetails> {
        executionCount++
        delay(50) // Simulate network delay
        
        return if (shouldReturnError) {
            Result.failure(Exception("Network error"))
        } else {
            Result.success(userDetailsToReturn ?: createDefaultUserDetails())
        }
    }
    
    private fun createDefaultUserDetails() = UserDetails(
        email = "default@example.com",
        avatarUrl = null,
        isPremium = false,
        creationDate = "2023-01-01"
    )
}
```

#### Parameterized Testing for Success and Failure Scenarios

```kotlin
@ParameterizedTest
@ValueSource(booleans = [true, false])
fun `should handle both success and error scenarios`(shouldSucceed: Boolean) = runTest {
    // Given
    if (shouldSucceed) {
        fakeGetUserDetailsUseCase.setSuccessResponse(
            UserDetails(
                email = "success@example.com",
                avatarUrl = "https://avatar.url",
                isPremium = true,
                creationDate = "2023-01-01"
            )
        )
    } else {
        fakeGetUserDetailsUseCase.setErrorResponse()
    }

    // When
    viewModel.state.test {
        advanceUntilIdle()

        // Then Focus on behavior, not implementation details
        if (shouldSucceed) {
            awaitItem() // Loading state
            val successState = awaitItem()
            assertThat(successState.isLoading).isFalse()
            assertThat(successState.isError).isFalse()
            assertThat(successState.userInfo?.displayEmail).isEqualTo("success@example.com")
            assertThat(successState.userInfo?.showPremiumBadge).isTrue()
        } else {
            awaitItem() // Loading state
            val errorState = awaitItem()
            assertThat(errorState.isLoading).isFalse()
            assertThat(errorState.isError).isTrue()
            assertThat(errorState.userInfo).isNull()
        }
    }
}
```


```kotlin
@Test
fun `should refresh data when refresh intent is triggered`() = runTest {
    // Given Initial successful load
    fakeGetUserDetailsUseCase.setSuccessResponse(
        UserDetails(
            email = "initial@example.com",
            avatarUrl = null,
            isPremium = false,
            creationDate = "2022-01-01"
        )
    )

    viewModel.state.test {
        advanceUntilIdle()
        
        awaitItem() // Loading
        val initialState = awaitItem() // Success
        assertThat(initialState.userInfo?.displayEmail).isEqualTo("initial@example.com")
        
        // Change response and trigger refresh
        fakeGetUserDetailsUseCase.setSuccessResponse(
            UserDetails(
                email = "refreshed@example.com",
                avatarUrl = "https://new-avatar.url", 
                isPremium = true,
                creationDate = "2023-01-01"
            )
        )
        
        viewModel.onIntent(ViewState.Intents.Refresh)
        advanceUntilIdle()

        // Then
        awaitItem() // Loading during refresh
        val refreshedState = awaitItem() // New Success
        assertThat(refreshedState.isLoading).isFalse()
        assertThat(refreshedState.userInfo?.displayEmail).isEqualTo("refreshed@example.com")
        assertThat(refreshedState.userInfo?.showPremiumBadge).isTrue()
        
        // Verify both initial load and refresh were called
        assertThat(fakeGetUserDetailsUseCase.getExecutionCount()).isEqualTo(2)
    }
}
```

#### Testing UI-Only State Changes

```kotlin
@Test
fun `should toggle email visibility without triggering data reload`() = runTest {
    // Given Successful initial load
    fakeGetUserDetailsUseCase.setSuccessResponse(
        UserDetails(
            email = "test@example.com",
            avatarUrl = null,
            isPremium = false,
            creationDate = "2023-01-01"
        )
    )

    viewModel.state.test {
        advanceUntilIdle()
        
        awaitItem() // Loading
        val loadedState = awaitItem() // Success
        assertThat(loadedState.userInfo?.displayEmail).isEqualTo("test@example.com")
        assertThat(loadedState.isEmailVisible).isFalse()

        // When Toggle email visibility
        viewModel.onIntent(ViewState.Intents.ToggleEmailVisibility(isEmailVisible = true))
        advanceUntilIdle()

        // Then
        val toggledState = awaitItem()
        assertThat(toggledState.isEmailVisible).isTrue()
        assertThat(toggledState.userInfo?.displayEmail).isEqualTo("test@example.com")
        
        assertThat(fakeGetUserDetailsUseCase.getExecutionCount()).isEqualTo(1)
    }
}
```

#### Testing Data Caching Behavior

```kotlin
@Test
fun `should use cached data when returning to screen quickly`() = runTest {
    // Given
    fakeGetUserDetailsUseCase.setSuccessResponse(
        UserDetails(
            email = "cached@example.com",
            avatarUrl = null,
            isPremium = true,
            creationDate = "2023-01-01"
        )
    )

    // When First collection
    viewModel.state.test {
        advanceUntilIdle()
        
        awaitItem() // Loading
        val firstState = awaitItem() // Success
        assertThat(firstState.userInfo?.displayEmail).isEqualTo("cached@example.com")
        
        cancel() // Simulate leaving screen
    }
    
    // When Quick return (simulating navigation back within timeout)
    viewModel.state.test {
        advanceUntilIdle()
        
        // Then Should have cached data immediately (no Loadin)
        val cachedState = awaitItem()
        assertThat(cachedState.isLoading).isFalse()
        assertThat(cachedState.userInfo?.displayEmail).isEqualTo("cached@example.com")
        
        expectNoEvents()
    }
    
    assertThat(fakeGetUserDetailsUseCase.getExecutionCount()).isEqualTo(1)
}
```

#### Testing Error Recovery

```kotlin
@Test  
fun `should recover from Error on successful refresh`() = runTest {
    // Given Initial error
    fakeGetUserDetailsUseCase.setErrorResponse()
    
    viewModel.state.test {
        advanceUntilIdle()
        
        awaitItem() // Loading
        val errorState = awaitItem() // Error
        assertThat(errorState.isError).isTrue()
        
        // When Fix the response and refresh
        fakeGetUserDetailsUseCase.setSuccessResponse(
            UserDetails(
                email = "recovered@example.com",
                avatarUrl = null,
                isPremium = false,
                creationDate = "2023-01-01"
            )
        )
        
        viewModel.onIntent(ViewState.Intents.Refresh)
        advanceUntilIdle()
        
        // Then Should recover successfully  
        awaitItem() // Loading during refresh
        val recoveredState = awaitItem() // Success
        assertThat(recoveredState.isError).isFalse()
        assertThat(recoveredState.userInfo?.displayEmail).isEqualTo("recovered@example.com")
        
        assertThat(fakeGetUserDetailsUseCase.getExecutionCount()).isEqualTo(2)
    }
}
```

### Why Testing Principles for Flow-Based ViewModels

Oh... this will spark some war but here is this great [article](https://dev.to/jameson/fakes-mocks-on-android-well-partner-that-depends-h6n) which gives a better picture and should not steal this article's purpose on why i'm using fakes, here's a qucik summary that'd add on top of the article

1. **Use Fakes Over Mocks**: Fakes provide realistic behavior and are easier to maintain, especially nowadays with LLMs assistance...
2. **Parameterized Tests**: Test both success and failure paths with a single test
3. **Turbine for Flow Testing**: Clean, expressive flow testing with proper state verification
4. **Test State Transitions**: Verify complete flow of states, not just final states
5. **Execution Counts Sparingly**: Only verify call counts when testing caching, performance, or retry behavior
6. **Focus on Behavior**: Test what the user experiences, not implementation details (i guess this one is obvious)

### Conclusion

This exploration of Flow-based data loading in Android ViewModels addresses the fundamental challenges of the traditional `init {}` block approach. While covering all architectural variations would be extensive, this pattern successfully handles approximately 90% of common use cases.

The abstract base class approach (`ViewModelLoader`) provides the most predictable and maintainable solution, offering:

- **Predictable State Management**: Clear state triggers and intent handling
- **Built-in Testing Support**: Testable architecture with proper coroutine handling  
- **Flexibility**: Easy extension for effects and hybrid MVI patterns
- **Caching and refresh**: Caching and refresh mechanisms (as much as we can cover in an article)


### Key Takeaways

1. **Flow-based loading** eliminates race conditions, adds testing ease and removes backstack issues inherent in `init {}` blocks
2. **StateFlow with proper sharing strategies** provides optimal lifecycle-aware data management
3. **Abstraction layers** reduce boilerplate while maintaining flexibility
4. **Comprehensive testing** ensures reliable behavior across all use cases

Remember, this represents one architectural approach among many, it fixed my problem it might not fix yours. 
The goal is understanding the underlying problems and evaluating whether this solution fits your specific requirements.
As time goes on, this solution will undergo many changes and might be obsolete.
Lately my efforts are poured more into backend development with Ktor which is an amazing experience on its own.

This architecture successfully powers [WallHub](https://wallhub.app/) on both iOS and Android platforms, demonstrating its real world "viability" and cross-platform adaptation if one can say such thing.

Stay hydrated, until next article...

