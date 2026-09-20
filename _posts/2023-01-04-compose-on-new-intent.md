---
title: onNewIntent in Jetpack Compose
date: 2023-01-04 00:15:00 +0100
categories: [Android, Compose, Jetpack Compose]
tags: [Android, Compose, Jetpack Compose]
---
<img src="/assets/img/compose/compose_logo.png" alt ="" class="center" >

Hello to everyone and happy new year, my previous year was very busy with Ktor and Compose, I can say I learned what more I need to learn.

I've decided to start sharing some small things for which there is no documentation or it's just tricky, I guess?

For this short blog post, we are going to explore how to handle the `onNewIntent` using Jetpack Compose.

## Activity Coupling
---
Everyone knows that we can override `onNewIntent` inside our Activity
```kotlin
override fun onNewIntent(intent: Intent?) {
        super.onNewIntent(intent)
    }
```
but this creates another issue, let's dig a bit.

Imagine you created a beautiful compression app using Middle Out algorithm from Sillicon Valley and you want to handle images, you add the intent filter and then you proceed to your Activity part.
```xml
        <activity
            android:name=".MainActivity"
            android:exported="true"
            android:launchMode="singleTask"
            android:theme="@style/Theme.MyAppTheme">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <action android:name="android.intent.action.VIEW" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>

            <intent-filter>
                <action android:name="android.intent.action.SEND" />
                <category android:name="android.intent.category.DEFAULT" />
                <data android:mimeType="image/*" />
            </intent-filter>
        </activity>
```
**We're using singleTask because we don't want to create a new activity stack entry every time**

Aaand you're lost.

You might think?
- Okay I will definitely hold an instance of the navigation controller to handle redirections
- I will just use an interface as a listener, cast the context to an activity, do that, circle around and not achieve anything
- I will use an event that'll be dispatcher to a flow/channel in the activity, pass it to the composable as a state, idk
- Something else that's wildly creative but highly impractical

<img src="/assets/img/compose/but_why.gif" alt ="" class="center" >

Let's say you have this setup
```kotlin
@Composable
fun ApplicationScaffold(){
    val bottomSheetNavigator = rememberBottomSheetNavigator()
    val navController = rememberAnimatedNavController(bottomSheetNavigator)
    ModalBottomSheetLayout(bottomSheetNavigator = bottomSheetNavigator) {
        Scaffold(modifier = Modifier.fillMaxSize(),
            bottomBar = {
                BottomNavigation( navController = navController)
            }) { paddingValues ->
            AnimatedNavHost(
                modifier = Modifier.padding(paddingValues),
                navController = navController,
                startDestination = "Home",
            ) {
                // add navigation
            }
        }
    }
}
```
since your `navController` lives inside this Composable, it means that you're kinda stuck and doesn't matter if you hoist it up to the root of your Activity's `setContent` because `onNewIntent` is a different function.

When you inherit from `ComponentActivity` you will notice something
```java
@CallSuper
    @Override
    protected void onNewIntent(
            @SuppressLint({"UnknownNullness", "MissingNullability"}) Intent intent
    ) {
        super.onNewIntent(intent);
        for (Consumer<Intent> listener : mOnNewIntentListeners) {
            listener.accept(intent);
        }
    }
```
it means that we can use these listeners in our favor and leverage Compose's side effects to handle the redirection.


Our code will look like this
```kotlin
@Composable
fun ApplicationScaffold(
    navigator : Navigator //only for demonstrations, you shouldn't do that, hoist as much as you can, affects the stability
){
    val bottomSheetNavigator = rememberBottomSheetNavigator()
    val navController = rememberAnimatedNavController(bottomSheetNavigator)
    ModalBottomSheetLayout(bottomSheetNavigator = bottomSheetNavigator) {
        Scaffold(modifier = Modifier.fillMaxSize(),
            bottomBar = {
                BottomNavigation( navController = navController)
            }) { paddingValues ->
            AnimatedNavHost(
                modifier = Modifier.padding(paddingValues),
                navController = navController,
                startDestination = "Home", //demonstration, don't put wild strings like this
            ) {
                // add navigation
            }
        }
    }

    val context = LocalContext.current
    val activity = (context.getActivity() as ComponentActivity)
    DisposableEffect(navController) {
        val listener = Consumer<Intent> {
            navigator.onListenForRedirections(it)
        }
        activity.addOnNewIntentListener(listener)
        onDispose { activity.removeOnNewIntentListener(listener) }
    }
}

fun Context.getActivity(): Activity {
    if (this is Activity) return this
    return if (this is ContextWrapper) baseContext.getActivity() else getActivity()
}
```
We use a disposable effect to add and remove listener for the lifecycle of the composable function.

Then inside our activity we can use an event dispatcher to migrate away from the tight coupling the navigation component has, you can check out my [previous post](/posts/compose_hilt_mm/) where this is explained, keep in mind this is an old blog post and this code evolved, if there's enough interest I can add another blog post with full solution on how to scale this even better.

## The catch
---
But of course, there's one catch, this will only catch (hope you saw what I did there) all the incoming intent filters when your app is opened, but if it's a cold boot they won't arrive.
In order to do that, after your `setContent` inside your *ComponentActivity* you can use
```kotlin
private fun checkForRedirectionsOnColdBoot(savedInstanceState: Bundle?) {
        if (savedInstanceState == null) {
            /*for (listener in mOnNewIntentListeners) { //wish we could've done this and avoid the next line :(
                listener.accept(intent)
            }*/
            navigator.onListenForRedirections(intent)
        }
    }
```
to handle even cold boot case.  

Navigation in Compose is a bit tricky at first, but once you've set it up correctly, it's even easier and better to use than the XML approach.

I hope you found this interesting, happy new year and hope you learnt a thing or two.

Until next post.