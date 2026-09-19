---
title: Abstract your Android Navigation for Compose, part 3
date: 2023-05-17 17:02:00 +0200
categories: [Hilt, Compose, Android, Navigation]
tags: [Hilt, Kotlin, Compose, Android, Navigation]
---
<img src="/assets/img/compose/compose_logo.png" alt ="" class="center" >

## Intro
---
Welcome to the last part of the navigation abstraction in Compose using Google's navigation component, in this blog post we'll actually see the implementation of the abstracted code from [Part #2](/posts/nav-abstraction-part-2/).

## Implementation
---
For the implementation we would utilize Hilt to help us, why? 
- Compile time checks
- We can easily navigate through the code and it'll help us scale when app gets large, but it will be writing some boilerplate code to set it up of course.

## Navigator implementation
---
We had our `Navigator` defined in the previous part, the implementation follows

```kotlin
@Singleton
internal class NavigatorImpl @Inject constructor() : Navigator, NavigatorDirections {

    private val _directions = Channel<NavigatorIntent>(Channel.BUFFERED)
    override val directions = _directions.receiveAsFlow()

    override fun popCurrentBackStack() {
        _directions.trySend(NavigatorIntent.PopCurrentBackStack)
    }

    override fun navigate(navigatorIntent: NavigatorIntent) {
        _directions.trySend(navigatorIntent)
    }

    override fun navigateUp() {
        _directions.trySend(NavigatorIntent.NavigateUp)
    }

    override fun popBackStack(destination: String, inclusive: Boolean, saveState: Boolean) {
        _directions.trySend(NavigatorIntent.PopBackStack(destination, inclusive, saveState))
    }

    override fun navigateTopLevel(destination: String) {
        _directions.trySend(NavigatorIntent.NavigateTopLevel(destination))
    }

    override fun navigate(destination: String, builder: NavOptionsBuilder.() -> Unit) {
        _directions.trySend(NavigatorIntent.Directions(destination, builder))
    }

    override fun navigateSingleTop(destination: String, builder: NavOptionsBuilder.() -> Unit) {
        _directions.trySend(NavigatorIntent.Directions(destination, builder))
    }

    override fun navigate(destination: String) {
        _directions.trySend(NavigatorIntent.Directions(destination))
    }
}
```
We want it singleton so that we'll be able to re-use it everywhere, also we had it split into two, so we would just provide it as separates

```kotlin
@Module
@InstallIn(SingletonComponent::class)
internal abstract class NavigatorModule {

    @Binds
    abstract fun bindNavigator(navigatorImpl: NavigatorImpl): Navigator

    @Binds
    abstract fun bindNavigatorDestination(navigatorImpl: NavigatorImpl): NavigatorDirections
}
```

## Bottom navigation
---

```kotlin
@Immutable
class BottomNavigationEntry(
    val destination: NavigationDestination,
    val text: String,
    val selectedIcon: ImageVector,
    val unselectedIcon: ImageVector,
    val route: String
)
```
Everything is pretty self explanatory, except desination and route, we would use destination to know whether current one is selected or not and the route is for the top level navigation upon click (matching the Graph's route/ID).

Our `BottomNavigation` composable will look like

```kotlin
@Composable
internal fun BottomNavigation(
    modifier: Modifier = Modifier,
    navBackStackEntry: NavBackStackEntry?,
    hideBottomNav: Boolean,
    bottomNavigationEntries: ImmutableHolder<List<BottomNavigationEntry>>,
    onTopLevelClick: (route: String) -> Unit
) {
    AnimatedVisibility(
        visible = !hideBottomNav,
        enter = slideInVertically { it },
        exit = slideOutVertically { it },
        modifier = modifier.fillMaxWidth(),
    ) {
        BottomAppBar(modifier = Modifier.fillMaxWidth()) {
            bottomNavigationEntries.item.forEach { bottomEntry ->
                val isSelected = navBackStackEntry?.destination?.route == bottomEntry.destination.destination()
                NavigationBarItem(
                    selected = isSelected,
                    alwaysShowLabel = true,
                    onClick = {
                        onTopLevelClick(bottomEntry.route)
                    },
                    label = {
                        Text(
                            modifier = Modifier
                                .wrapContentSize(unbounded = true),
                            softWrap = false,
                            maxLines = 1,
                            textAlign = TextAlign.Center,
                            text = bottomEntry.text,
                        )
                    },
                    icon = {
                        Crossfade(targetState = isSelected, label = "bottom-nav-icon") { isSelectedIcon ->
                            if (isSelectedIcon) {
                                Icon(imageVector = bottomEntry.selectedIcon, contentDescription = bottomEntry.text)
                            } else {
                                Icon(imageVector = bottomEntry.unselectedIcon, contentDescription = bottomEntry.text)
                            }
                        }
                    },
                )
            }
        }
    }
}
```

## Graph entries
---

Each graph with it's entries would be added accordingly to the types we created

```kotlin
fun NavGraphBuilder.addGraphs(
    navController: StableHolder<NavHostController>,
    navigationGraphs: Map<NavigationGraph, Set<NavigationGraphEntry>>,
    showAnimations: Boolean,
) {
    navigationGraphs.forEach { (navigatorGraph, destinationGraphEntries) ->
        navigation(startDestination = navigatorGraph.startingDestination.destination(), route = navigatorGraph.route) {
            destinationGraphEntries.forEach { destinationGraphEntry ->
                addDestinationBasedOnType(destinationGraphEntry, navController, showAnimations)
            }
        }
    }
}

private fun NavGraphBuilder.addDestinationBasedOnType(
    navigationGraphEntry: NavigationGraphEntry,
    navController: StableHolder<NavHostController>,
    showAnimations: Boolean,
) {
    when (navigationGraphEntry.navigationDestination) {
        is DialogDestination -> addDialogDestinations(navController, navigationGraphEntry)
        is ScreenDestination -> addComposableDestinations(navController, navigationGraphEntry, showAnimations)
        is BottomSheetDestination -> addBottomSheetDestinations(navController, navigationGraphEntry)
    }
}

@OptIn(ExperimentalMaterialNavigationApi::class)
private fun NavGraphBuilder.addBottomSheetDestinations(navController: StableHolder<NavHostController>, entry: NavigationGraphEntry) {
    val destination = entry.navigationDestination
    bottomSheet(destination.destination(), destination.arguments, destination.deepLinks) {
        entry.Render(navController)
    }
}

private fun NavGraphBuilder.addDialogDestinations(navController: StableHolder<NavHostController>, entry: NavigationGraphEntry) {
    val destination = entry.navigationDestination as DialogDestination
    dialog(
        destination.destination(),
        destination.arguments,
        destination.deepLinks,
        destination.dialogProperties,
    ) {
        entry.Render(navController)
    }
}

private fun NavGraphBuilder.addComposableDestinations(
    navController: StableHolder<NavHostController>,
    entry: NavigationGraphEntry,
    showAnimations: Boolean,
) {
    val destination = entry.navigationDestination
    if (destination is AnimatedDestination && showAnimations) {
        composable(
            destination.destination(),
            destination.arguments,
            destination.deepLinks,
            destination.enterTransition,
            destination.exitTransition,
            destination.popEnterTransition,
            destination.popExitTransition,
        ) {
            entry.Render(navController)
        }
    } else {
        composable(
            destination.destination(),
            destination.arguments,
            destination.deepLinks,
        ) {
            entry.Render(navController)
        }
    }
}
```

## Reacting to the navigator commands
---

We created a way to send commands, we should be able to utilize them

```kotlin
@Composable
private fun NavHostControllerEvents(
    navigator: NavigatorDirections,
    navController: NavHostController,
) {
    LaunchedEffect(navController) {
        navigator.directions.collectLatest { navigatorEvent ->
            when (navigatorEvent) {
                is NavigatorIntent.NavigateUp -> navController.navigateUp()
                is NavigatorIntent.Directions -> navController.navigate(navigatorEvent.destination, navigatorEvent.builder)
                NavigatorIntent.PopCurrentBackStack -> navController.popBackStack()
                is NavigatorIntent.PopBackStack -> navController.popBackStack(
                    navigatorEvent.route,
                    navigatorEvent.inclusive,
                    navigatorEvent.saveState,
                )

                is NavigatorIntent.NavigateTopLevel -> {
                    val topLevelNavOptions = navOptions {
                        // Pop up to the start destination of the graph to
                        // avoid building up a large stack of destinations
                        // on the back stack as users select items
                        popUpTo(navController.graph.findStartDestination().id) {
                            saveState = true
                        }
                        // Avoid multiple copies of the same destination when
                        // reselecting the same item
                        launchSingleTop = true
                        // Restore state when reselecting a previously selected item
                        restoreState = true
                    }
                    navController.navigate(navigatorEvent.route, topLevelNavOptions)
                }
            }
            Log.d("NavDirections", navigatorEvent.toString())
        }
    }
}
```

## Graphs factory
---

We have so many Graphs in our application, we should be able to aggregate and connect them with their entries through their destinations.

```kotlin
/**
 * Responsible for concatenating your destinations and graphs
 */
@Suppress("UNCHECKED_CAST")
@Singleton
class GraphFactory @Inject constructor(
    private val destinations: Map<Class<*>, @JvmSuppressWildcards Set<NavigationGraphEntry>>,
    private val graphs: Map<@JvmSuppressWildcards Class<*>, @JvmSuppressWildcards NavigationGraph>,
) {

    fun <T : NavigationGraph> getGraphByRoute(uniqueRoute: Class<*>): T {
        val graph = graphs[uniqueRoute] as? T
        when {
            graph == null -> throw IllegalArgumentException("Graph with $uniqueRoute is not registered")
            graph.javaClass != uniqueRoute -> throw IllegalArgumentException("Graph should be of a type ${uniqueRoute.javaClass}")
            else -> return graph
        }
    }

    val graphsWithDestinations: Map<NavigationGraph, Set<NavigationGraphEntry>> =
        (destinations.keys.plus(graphs.keys).asSequence())
            .associate {
                graphs[it] to destinations[it]
            }.filter {
                it.key != null && it.value != null
            } as Map<NavigationGraph, Set<NavigationGraphEntry>>
}
```

In this example we'll utilize the `ClassKey` for connecting the navigation graph entries to the graph, of course you can use a `StringKey` which would be of the same effect and you won't need to bind the `ClassKey` to a map and have one less module, I'll demonstrate this later down the road.

### Why do we use MultiMap bindings
---
The goal is to never re-use the same key, why?
- You're splitting features into modules, each feature is its own entity
- Each feature has a flow, a flow can consits of more than one destination (can be 3-4 or more)
- One complete flow should not be scattered across multiple modules, it should stay in one

Multi bindings prevent us from using same `Key` twice, that means you can't contribute to the `Graph` from your `auth` feature and at the same time from your `home` feature.

## Login feature example
---
So, let's say we want to implement a **"Login"** feature, for that we'll need a `LoginGraph`, `LoginScreenDestination`, `LoginScreenGraphEntry` and of course UI for the entry.

Let's say that this is a multi modular project and you're approaching the setup with a feature based modularization.

You would have the following

1. Graph
```kotlin
class LoginGraph @Inject constructor(override val startingDestination: LoginScreenDestination) : NavigationGraph {
    override val route: String = "LoginGraph"
}
```
2. Destination
```kotlin
class LoginScreenDestination @Inject constructor() : ScreenDestination {
    override fun destination(): String = "LoginScreenDestination"

    override val enterTransition: (AnimatedContentTransitionScope<NavBackStackEntry>.() -> EnterTransition?)
        get() = {
            slideIntoContainer(AnimatedContentTransitionScope.SlideDirection.Up)
        }

    override val exitTransition: (AnimatedContentTransitionScope<NavBackStackEntry>.() -> ExitTransition?)
        get() =  {
            slideOutOfContainer(AnimatedContentTransitionScope.SlideDirection.Down)
        }
}
```
3. Graph entry
```kotlin
internal class LoginScreenGraphEntry @Inject constructor(
    override val navigationDestination: LoginScreenDestination,
    private val navigator: Navigator
) : NavigationGraphEntry {

    @Composable
    override fun Render(controller: StableHolder<NavHostController>) {
        LoginScreen(navigator::navigateUp)
    }
}
```
4. UI
```kotlin
@Composable
internal fun LoginScreen(onClick: () -> Unit) {
    Column(
        modifier = Modifier
            .fillMaxSize()
            .background(Color.DarkGray.copy(alpha = 0.65f)),
        horizontalAlignment = Alignment.CenterHorizontally,
        verticalArrangement = Arrangement.Center
    ) {
        Button(onClick = onClick) {
            Text(text = "Go back")
        }
    }
}
@Composable
@Preview
private fun LoginScreenPreview(){
    ComposedLibThemeSurface {
        LoginScreen {

        }
    }
}
```
5. Module binding the entries to the graph
```kotlin
@InstallIn(SingletonComponent::class)
@Module
internal object LoginModule {

    @Provides
    @IntoMap
    @ClassKey(LoginGraph::class)
    fun loginEntries(loginScreenGraphEntry: LoginScreenGraphEntry): Set<NavigationGraphEntry> = setOf(loginScreenGraphEntry)
}
```
6. Adding the graph to the global graph map
```kotlin
@Module
@InstallIn(SingletonComponent::class)
internal abstract class GraphsModule {

    @Binds
    @IntoMap
    @ClassKey(FavoritesGraph::class)
    abstract fun favoritesGraph(graph: FavoritesGraph): NavigationGraph

    @Binds
    @IntoMap
    @ClassKey(HomeGraph::class)
    abstract fun homeGraph(graph: HomeGraph): NavigationGraph

    @Binds
    @IntoMap
    @ClassKey(SettingsGraph::class)
    abstract fun settingsGraph(graph: SettingsGraph): NavigationGraph

    @Binds
    @IntoMap
    @ClassKey(LoginGraph::class)
    abstract fun loginGraph(graph: LoginGraph): NavigationGraph
}
```

You can always choose to use a `StringKey` to drop the **6th step** and use a companion object where you won't need to create a module for your graphs.
```kotlin
class LoginGraph @Inject constructor(override val startingDestination: LoginScreenDestination) : NavigationGraph {
    companion object {
        const val ID = "LoginGraph"
    }

    override val route: String = ID
}
```
Upon modification
```kotlin
@InstallIn(SingletonComponent::class)
@Module
internal object LoginModule {

    @Provides
    @IntoMap
    @StringKey(LoginGraph.ID)
    fun loginEntries(loginScreenGraphEntry: LoginScreenGraphEntry): Set<NavigationGraphEntry> = setOf(loginScreenGraphEntry)
}
```
and of course, you would change the `GraphFactory` to provide you a `String` based map.
This is the actual solution I started with, but later changed it with a `GraphsModule` so that I can keep track of which `Graphs` are actually included into the global Map of graphs.

## Top level destinations
---
We want to have our top level destinations provided with a help from `GraphFactory`, we're getting a graph by it's unique `ClassKey`.

```kotlin
@ActivityRetainedScoped
class TopLevelDestinationsProvider @Inject constructor(
    private val graphFactory: GraphFactory,
) {

    private val homeGraph get() = graphFactory.getGraphByRoute<HomeGraph>(HomeGraph::class.java)
    private val favoritesGraph get() = graphFactory.getGraphByRoute<FavoritesGraph>(FavoritesGraph::class.java)
    private val settingsGraph get() = graphFactory.getGraphByRoute<SettingsGraph>(SettingsGraph::class.java)

    fun getStartingDestination(): String = homeGraph.route


    val bottomNavigationEntries = listOf(
        BottomNavigationEntry(
            destination = HomeScreenBottomNavDestination,
            text = "Home",
            selectedIcon = Icons.Filled.Home,
            unselectedIcon = Icons.Outlined.Home,
            route = homeGraph.route
        ),

        BottomNavigationEntry(
            destination = FavoritesScreenBottomNavDestination,
            text = "Favorites",
            selectedIcon = Icons.Default.Favorite,
            unselectedIcon = Icons.Outlined.FavoriteBorder,
            route = favoritesGraph.route
        ),
        BottomNavigationEntry(
            destination = SettingsScreenBottomNavDestination,
            text = "Settings",
            selectedIcon = Icons.Default.Settings,
            unselectedIcon = Icons.Outlined.Settings,
            route = settingsGraph.route
        ),
    )
}
```
## Connecting it all
---
All in all, our `MainActivity` will be looking like this

```kotlin
@AndroidEntryPoint
class MainActivity : AppCompatActivity() {

    @Inject lateinit var navigatorDirections: NavigatorDirections
    @Inject lateinit var navigator: Navigator
    @Inject lateinit var graphFactory: GraphFactory
    @Inject lateinit var topLevelDestinationsProvider: TopLevelDestinationsProvider

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            ComposedLibThemeSurface {
                AppScaffold(
                    startingDestination = topLevelDestinationsProvider.getStartingDestination(),
                    graphs = {
                        graphFactory.graphsWithDestinations
                    },
                    showAnimations = true,
                    navigatorDirections = navigatorDirections,
                    bottomNavigationEntries = topLevelDestinationsProvider.bottomNavigationEntries.asImmutable,
                    navigator = navigator
                )
            }
        }
    }
}

@OptIn(ExperimentalMaterialNavigationApi::class)
@Composable
private fun AppScaffold(
    startingDestination: String,
    graphs: () -> Map<NavigationGraph, Set<NavigationGraphEntry>>,
    showAnimations: Boolean,
    navigatorDirections: NavigatorDirections,
    bottomNavigationEntries: ImmutableHolder<List<BottomNavigationEntry>>,
    navigator: Navigator,
) {
    val bottomSheetNavigator: BottomSheetNavigator = rememberBottomSheetNavigator()
    val navController: NavHostController = rememberAnimatedNavController(bottomSheetNavigator)

    val navBackStackEntry: NavBackStackEntry? by navController.currentBackStackEntryFlow.collectAsStateWithLifecycle(null)
    val hideBottomNav: Boolean by remember {
        derivedStateOf { navBackStackEntry.hideBottomNavigation }
    }
    val currentRoute by remember {
        derivedStateOf { navBackStackEntry?.destination?.route }
    }
    LaunchedEffect(currentRoute) {
        Log.d(
            "Current destination", "${navBackStackEntry?.destination} with arguments ${navBackStackEntry?.arguments}",
        )
    }

    ModalBottomSheetLayout(
        modifier = Modifier.fillMaxSize(),
        bottomSheetNavigator = bottomSheetNavigator,
        sheetShape = MaterialTheme.shapes.large.copy(topStart = CornerSize(16.dp), topEnd = CornerSize(16.dp)),
        sheetElevation = 0.dp,
        sheetBackgroundColor = MaterialTheme.colorScheme.background
    ) {
        Scaffold(
            modifier = Modifier.fillMaxSize(),
        ) { paddingValues ->
            Box(
                modifier = Modifier
                    .fillMaxSize()
                    .padding(paddingValues),
            ) {
                AnimatedNavHost(
                    navController = navController,
                    startDestination = startingDestination,
                    enterTransition = { fadeIn() },
                    exitTransition = { fadeOut() },
                ) {
                    addGraphs(navController.asStable, graphs(), showAnimations)
                }

                BottomNavigation(
                    bottomNavigationEntries = bottomNavigationEntries,
                    modifier = Modifier.align(Alignment.BottomStart),
                    hideBottomNav = hideBottomNav,
                    navBackStackEntry = navBackStackEntry,
                    onTopLevelClick = navigator::navigateTopLevel
                )
            }
        }
    }
    NavHostControllerEvents(
        navigator = navigatorDirections,
        navController = navController,
    )
}

@Composable
private fun NavHostControllerEvents(
    navigator: NavigatorDirections,
    navController: NavHostController,
) {
    LaunchedEffect(navController) {
        navigator.directions.collectLatest { navigatorEvent ->
            when (navigatorEvent) {
                is NavigatorIntent.NavigateUp -> navController.navigateUp()
                is NavigatorIntent.Directions -> navController.navigate(navigatorEvent.destination, navigatorEvent.builder)
                NavigatorIntent.PopCurrentBackStack -> navController.popBackStack()
                is NavigatorIntent.PopBackStack -> navController.popBackStack(
                    navigatorEvent.route,
                    navigatorEvent.inclusive,
                    navigatorEvent.saveState,
                )

                is NavigatorIntent.NavigateTopLevel -> {
                    val topLevelNavOptions = navOptions {
                        // Pop up to the start destination of the graph to
                        // avoid building up a large stack of destinations
                        // on the back stack as users select items
                        popUpTo(navController.graph.findStartDestination().id) {
                            saveState = true
                        }
                        // Avoid multiple copies of the same destination when
                        // reselecting the same item
                        launchSingleTop = true
                        // Restore state when reselecting a previously selected item
                        restoreState = true
                    }
                    navController.navigate(navigatorEvent.route, topLevelNavOptions)
                }
            }
            Log.d("NavDirections", navigatorEvent.toString())
        }
    }
}
```

## Arguments feature
---
Let's create a `YellowDialog` and `RedDialog`.

You will go from `YellowDialog` to `RedDialog` with arguments.

You will also send arguments from `RedDialog` to `YellowDialog`.

### YellowDialog
---
Our starting point is always a Graph.

```kotlin
class DialogsGraph @Inject constructor(override val startingDestination: YellowDialogDestination) : NavigationGraph {
    override val route: String = "DialogsGraph"
}
```

We should not forget to add it to all of our available graphs if you're using the approach of `ClassKey`.

```kotlin
@Module
@InstallIn(SingletonComponent::class)
internal abstract class GraphsModule {
//the others 
    @Binds
    @IntoMap
    @ClassKey(DialogsGraph::class)
    abstract fun dialogsGraph(graph: DialogsGraph): NavigationGraph

}
```
What we would do without a destination (in life) and even in code?

```kotlin
class YellowDialogDestination @Inject constructor() : DialogDestination {
    override val dialogProperties: DialogProperties
        get() = DialogProperties(usePlatformDefaultWidth = false)

    override fun destination(): String = "YellowDialogDestination"
}
```
For this purpose we would create directions class to safely navigate from one to the other

```kotlin
class YellowDialogDirections @Inject constructor(
    private val navigator: Navigator,
    private val redDialogDestination: RedDialogDestination
) {
    fun openRedDialog(
        texts : Array<String>
    ) = navigator.navigateSingleTop(redDialogDestination.openDestination(texts))
}
```

Our graph entry
```kotlin
internal class YellowDialogGraphEntry @Inject constructor(
    override val navigationDestination: YellowDialogDestination,
    private val yellowDialogDirections: YellowDialogDirections
) : NavigationGraphEntry {
    @Composable
    override fun Render(controller: StableHolder<NavHostController>) {
        var callbackString by remember { mutableStateOf<String?>(null) }
        val argumentsFromRedDialog = remember { ArgumentsFromRedDialog(controller) }
        argumentsFromRedDialog
            .OnSingleStringCallbackArgument(key = ArgumentsFromRedDialog.TEXT, onValue = {
                callbackString = it
            })
        Column(
            modifier = Modifier
                .fillMaxHeight(0.6f)
                .fillMaxWidth(0.6f)
                .background(Color.Yellow.copy(alpha = 0.9f), RoundedCornerShape(16.dp)),
            horizontalAlignment = Alignment.CenterHorizontally,
            verticalArrangement = Arrangement.Center
        ) {
            Button(onClick = {
                yellowDialogDirections.openRedDialog(
                    arrayOf(
                        "Navigation",
                        "Compose",
                        "Something",
                        "Dialog",
                    )
                )
            }) {
                Text(text = "Open other dialog")
            }
            AnimatedVisibility(
                visible = !callbackString.isNullOrEmpty(),
                modifier = Modifier.fillMaxWidth()
            ) {
                Text(
                    text = "Callback string $callbackString",
                    modifier = Modifier.padding(horizontal = 16.dp)
                )
            }
        }
    }
}
```
### RedDialog
---

The arguments that we'll use for the callback

```kotlin
class ArgumentsFromRedDialog(override val navHostController: StableHolder<NavHostController>) : AddArgumentsToPreviousDestination {

    companion object {
        const val TEXT = "text"
    }

    fun addText(text :String){
        set(TEXT, text)
    }
    override fun consumeArgumentsAtReceivingDestination() {
        consumeArgument(TEXT)
    }
}
```

We also need the destination which has a receiving arguments `texts: Array<String>`, we create the destination using the helper function `createDestination()` where the first argument is the starting route, followed by a vararg of either [optional](https://github.com/FunkyMuse/Composed/blob/9e4f59e98d1310a282c7ea6652516edeebf4a8d6/navigation/src/main/java/com/funkymuse/composed/navigation/destination/arguments/DestinationArgument.kt#L5) or [required](https://github.com/FunkyMuse/Composed/blob/9e4f59e98d1310a282c7ea6652516edeebf4a8d6/navigation/src/main/java/com/funkymuse/composed/navigation/destination/arguments/DestinationArgument.kt#L6) arguments.

When we are trying to navigate to a destination we use `applyArgumentsToDestination`, again the first argument is the starting route, followed by a one of:
- `fun String.asNonNullDestinationValue()`
- `fun String.asNullWithDestinationValue(value: String?)`
- `fun String.asNullDestinationValue()`

Where `String`, the receiver object is the name of the argument when you're using null values or for non null values, the actual value you would pass to the screen.

When you're passing value and you use it in form of `String.asNonNullDestinationValue()` you can use one of the [following functions](https://github.com/FunkyMuse/Composed/blob/9e4f59e98d1310a282c7ea6652516edeebf4a8d6/navigation/src/main/java/com/funkymuse/composed/navigation/destination/NavigationDestinationExtensions.kt) to add the argument safely.

```kotlin
class RedDialogDestination @Inject constructor() : DialogDestination {

    companion object {
        const val ROUTE = "RedDialogDestination"
        const val TEXTS = "texts"
    }

    override val dialogProperties: DialogProperties
        get() = DialogProperties()

    override fun destination(): String = createDestination(ROUTE, TEXTS.asOptionalArgument())

    fun openDestination(texts: Array<String>) = applyArgumentsToDestination(
        ROUTE, TEXTS.asNullWithDestinationValue(
            addStringArrayArgument(texts)
        ),
    )

    override val arguments: List<NamedNavArgument>
        get() = listOf(
            navArgument(TEXTS) {
                type = DestinationsStringArrayNavType
                nullable = true
                defaultValue = null
            },
        )
}
```
We want the arguments to be consumed inside a ViewModel

```kotlin
@ViewModelScoped
class RedDialogViewModelArguments @Inject constructor(override val savedStateHandle: SavedStateHandle) : ViewModelNavigationArguments {

    val texts = getStringArrayArgument(RedDialogDestination.TEXTS)
}
```
Accordingly the ViewModel
```kotlin
@HiltViewModel
class RedDialogViewModel @Inject constructor(
    redDialogViewModelArguments: RedDialogViewModelArguments
) : ViewModel() {

    private val _texts = MutableStateFlow(redDialogViewModelArguments.texts)
    val texts = _texts.asStateFlow()

}
```

and finally the graph entry

```kotlin
internal class RedDialogGraphEntry @Inject constructor(
    override val navigationDestination: RedDialogDestination,
    private val navigator: Navigator
) :
    NavigationGraphEntry {
    @Composable
    override fun Render(controller: StableHolder<NavHostController>) {
        val redDialogViewModel = hiltViewModel<RedDialogViewModel>()
        val texts by redDialogViewModel.texts.collectAsStateWithLifecycle()
        val argumentsFromRedDialog = remember { ArgumentsFromRedDialog(controller) }
        Column(
            modifier = Modifier
                .fillMaxSize()
                .background(Color.Red.copy(alpha = 0.8f), RoundedCornerShape(24.dp)),
        ) {
            LazyColumn(
                contentPadding = PaddingValues(16.dp),
                content = {
                    item {
                        Button(
                            onClick = {
                                argumentsFromRedDialog.addText("Text argument sent back to Yellow")
                                navigator.navigateUp()
                            },
                            modifier = Modifier
                                .fillMaxWidth()
                                .padding(horizontal = 16.dp)
                        ) {
                            Text(
                                text = "Send arguments back", color = Color.White,
                            )
                        }
                    }
                    items(texts.orEmpty(), key = { it }) {
                        Text(text = it, color = Color.White)
                    }
                })
        }
    }
}
```

Which we finally concatenate in the graph by doing

```kotlin
@InstallIn(SingletonComponent::class)
@Module
internal object LoginModule {

    @Provides
    @IntoMap
    @ClassKey(DialogsGraph::class)
    fun dialogEntries(
        yellowDialogGraphEntry: YellowDialogGraphEntry,
        redDialogGraphEntry: RedDialogGraphEntry
    ): Set<NavigationGraphEntry> = setOf(yellowDialogGraphEntry, redDialogGraphEntry)
}
```

You can check out the [sample app](https://github.com/FunkyMuse/Composed/tree/main/app) which also acts as a library for many of these abstraction, there you can also find how to use [NavEntryArguments](https://github.com/FunkyMuse/Composed/blob/main/app/src/main/java/com/funkymuse/composedlib/navigation/destinations/bottom_sheets/test2/Test2BottomSheetDialogNavEntryArguments.kt), [Create destinations]() etc.

## Thank you
---

Thanks for your wholehearted attention and reading up until this lengthy article, I hope this will help you modularize the app and make ease of use with some abstraction on top of the Compose Navigation component from Google, which works amazingly well, of course with a little bit of help, as you know Compose is new and the Navigation Component was XML based up until recently, so it's normal that this article exists because of this.

Special thanks to my friend who helped me proof read this artcle.

This is an article that might not fit your case, it solved my problem, might not work in your situation.

A capybara to make your day more chill, as they're one of the chillest animals.

<img src="/assets/img/nav_abstraction/capybara.jpg" alt ="" class="center" >

At last, a video of the sample app and demonstration of everything so far and everything else that did not fit in these series of blog posts (as some parts were skipped because they can be looked into the code and understood as they're straight forward)

<video src="/assets/videos/nav-abstraction/sample.mp4" controls="controls" style="width:50%;height:33%" class="center" muted></video>

This is now redundant, please use the [type-safe solution](/posts/nav-type-safe/).

