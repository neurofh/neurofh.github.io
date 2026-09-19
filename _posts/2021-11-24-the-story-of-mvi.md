---
title: The famous and unthought MVI misconception in Android, alongside MVVM
date: 2021-11-24 23:30:00 +0100
categories: [MVI, Architecture, Android]
tags: [Android, Architecture, MVI]
---

<img src="/assets/img/mvi/mvvm_mvc.jpg" alt ="" class="center">

Android development is a vast topic that you can merely master and since it constantly evolves it's even harder to do so, that's why there are teams working on one application that does more than one thing and then there's the guy that wrote Telegram all by himself.

In this blog post we're not discussing the MVI as in Redux but rather the simplified event based MVI that's circulating the community that most seem to model their UI upon it.

Each application consists of code that's been glued together and passed around, just like every other non-Android application, that code is written per set of guidelines and conventions that you follow, a part of that is an **architecture**.

You might have heard of **MVC**, then we have **MVP**, **MVVM**, **MVI** etc... if you add any letter after ***MV*** it might exist already.

## MVVM is actually MVC?
---

**MVVM** on Android is actually ***MVC***, if you come to think about it, you usually write `CustomNamedViewModel` that inherits from androidx.`ViewModel`, I don't see the benefit of unnecessary sub-classing, what do you have access to? Nothing, what does it do? Nothing, but does it?
Yeah, it just has `onCleared` method of course and lots of other things going behind the scenes just because you've sub-classed something.

The androidx.`ViewModel` was the iteration of the failed `Loaders`, by that reference you can think of the androidx.`ViewModel` as a `Presenter`, you have a repository in there that brings you data that you expose as a LiveData, StateFlow, Channel etc... you can even call it a ***Provider***.

The whole naming is wrong and they've done it again with **Jetpack Compose**, Google... when will you learn?

Then what is androidx.`ViewModel`?

androidx.`ViewModel` is a **RetainedObject**, an object that's brought to you by the infamous and powerful data structure, `Map<*, *>` (dictionary), but as every other thing in Android, none of it could be easily accessed by you (the developer) as they hide it behind factories and not exposing it for unknown reasons?

What is androidx.`ViewModel`, again?

An abstraction that doesn't sit between **MVVM** and it's not that **VM** part, is created once and stored inside the [ViewModelStore](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:lifecycle/lifecycle-viewmodel/src/main/java/androidx/lifecycle/ViewModelStore.java) which means that they don't survive process death (only configuration change) and thats's why you've seen a `SavedStateHandle` passed inside them, so that you can delegate something to be persisted.


What is androidx.`ViewModel`, again?
A subtle but loud **mistake**.

I feel sorry for the folks on the Android team that had to agree to such thing as mentioned [here](https://www.reddit.com/r/androiddev/comments/b908fr/comment/ek2cm50/?utm_source=share&utm_medium=web2x&context=3), [here](https://www.reddit.com/r/androiddev/comments/b908fr/comment/ek4iqei/?utm_source=share&utm_medium=web2x&context=3) and [here](https://www.reddit.com/r/androiddev/comments/b908fr/comment/ek57uid/?utm_source=share&utm_medium=web2x&context=3).

But I think that their general case was to ease the developer life and make the API more developer friendly while making you learn new things and thinking you've written actual implementation of **MVVM**.

How is the **AAC MVVM** an **MVC**?

**M** = The data model of your choice (observable or not)

**V** = Fragment/Activity

**C** = ViewModel sub-classing you do

Some may argue that also

**M** = ViewModel sub-classing you do

**V** = Fragment/Activity

**C** = Fragment/Activity

Both of which if you come to think about are true.

Otherd may argue that it's also **MVP** that doesn't necessarily mean that you have knowledge about the `View` and they'll be correct too, so AAC MVVM on Android can be anything but not MVVM.

\* *AAC = Android architecture components*

<img src="/assets/img/mvi/pepe-cry.gif" alt ="" class="center">

## The holy grail, MVI or waiting for another coming? 
---

You've heard about "**MVI**" a form of `sealed` class in `Kotlin` on **Android**, looking something like the following

```kotlin
sealed class ApiState<T> {

    object Idle : ApiState()
    object Loading : ApiState()
    data class Success(val value: T) : ApiState()
    data class Error(val error: Throwable) : ApiState()

}
```

then inside your `"ViewModel"` *coughs* presenter *coughs* controller *coughs* provider, *coughs* is this */r/mAndroidDev* already, or I need more *coughs* and *memes*?

```kotlin
class UsersViewModel(
    private val repository: UsersRepository
) : ViewModel() {

    private val _state = MutableStateFlow<ApiState<List<Users>>>(ApiState.Idle)
    val state = _state.asStateFlow()

    init {
        getUsers()
    }

    private fun getUsers() {
        viewModelScope.launch {
            _state.value = ApiState.Loading
            _state.value = try {
                ApiState.Success(repository.getUsers())
            } catch (t : Throwable) {
                ApiState.Error(t)
            }
        }
    }
}
```

and/or something in form of an additional `Intent` that you listen to inside your `"ViewModel"` 

```kotlin
sealed interface MviIntent {

    object GetUsers : MviIntent

}
```

```kotlin
class UsersViewModel(
    private val repository: UsersRepository
) : ViewModel() {

    private val intentByUser = Channel<MviIntent>(Channel.UNLIMITED)

    private val _state = MutableStateFlow<ApiState<List<Users>>>(ApiState.Idle)
    val state = _state.asStateFlow()

    init {
        handleIntent()
    }

    fun sendUserIntent(event : MviIntent){
        viewModelScope.launch { intentByUser.send(event) }
    }

    private fun handleIntent() {
        viewModelScope.launch {
            intentByUser.receiveAsFlow().collect {
                when (it) {
                    is MviIntent.GetUsers -> getUsers()
                }
            }
        }
    }

    private fun getUsers() {
        viewModelScope.launch {
            _state.value = ApiState.Loading
            _state.value = try {
                ApiState.Success(repository.getUsers())
            } catch (t : Throwable) {
                ApiState.Error(t)
            }
        }
    }
}
```

then inside the `Fragment` or your other *View* it boils down to

```kotlin
class MviFragment : Fragment(R.layout.my_list_layout){

    private val usersViewModel by viewModels<UsersViewModel>()
    private val usersAdapter = UsersAdapter()
    private val binding by viewBinding<MyListLayoutBinding>(MyListLayoutBinding::bind)

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        binding.recycler.adapter = usersAdapter

        viewLifecycleOwner.lifecycleScope.launch {
             viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {
                 usersViewModel.state.collect { state-> updateUiState(state) }
             }
        }

        binding.testButton.setOnClickListener {
            usersViewModel.sendUserIntent(MviIntent.GetUsers)
        }
    }

    private fun updateUiState(state : ApiState){
        //binding.progressbar.isVisible = state is ApiState.Loading
        //binding.error.isVisible = state is ApiState.Error

        when(state){
            ApiState.Idle -> { //reset ui to idle state }
            ApiState.Loading -> { 
                //or just hide the error here and show the progress bar
                //binding.progressbar.isVisible = true
             }
            is ApiState.Success -> { 
                //or just hide the error and the progress bar here too
                usersAdapter.submitList(state.value)
                // but wait, the list can be empty? huh
            }
            is ApiState.Error -> { 
                //or just hide the progress bar and the list?
                binding.error.isVisible = true 
            }
        }
    }
}
```

This is iherently **wrong**, why?
1. Okay you can easily hide/show progress but what happens if there's **content** ***already*** and you want to show an error?
2. You've received *Success* state updated the *list*, tried to *refresh* the *content*, you've got an **error**, checked if the adapter is *not empty* and shown a *toast*, **rotated** the device, *still* showing the toast, but the **content** is *gone* and you might have wanted to show the *snackbar* **just once**, like an **event**.
3. You've received a success state but the list/data/value is **empty**, then you have to do an **additional check** to update the UI.
4. You wanted to show a fancy initial loading before data is populated but once it's populated there's pagination and you want to show progress bar maybe and not your fancy animation, this MVI doesn't provide that, you have to check if there's data in the adapter again and show the progress bar, but let's say **#2** happened, boom your data is gone, again.
5. How can you handle one shot events? **if** `isErrorShown` == `false`, show ***UI error*** then set  `isErrorShown` = `true`, rotate the device? process death? okay you've handled that (disable rotation, who needs it huh?, i mean sometimes you really don't, usually in production you never do because that's what you told your POs and other level of chain just so that you can ease your life, save some PTs etc... but you can't escape process death), but that's too much boilerplate to write.
6. Have you made already too many checks, to understand that MVI is a facade that looks beautiful on the outside?
7. You want to navigate the user to another screen X when let's say data is successfully loaded, you do, the user navigates back, the user navigates back to screen. X, why? because you've used a state mechanism that knows about the previous state that you do have in a `"ViewModel"`, hence **#5** but you've solved it again
8. You can't re-create the state after process death, you don't know what state you were at and you just refresh the data all over again(at times this does make sense, for onlnie-first apps).
9. You're **proxying** the intents which is an overkill for 3-4 intents since they can be a function, this does scale well only when doing something big, but then yet again this can be another "view model" instead of one, that does many things that at some point you'll have to refactor.
10. Can't create a good abstraction (you end up creating boilerplate usually anyways) without enforcing inheritance instead of providing composition that can be customized.
   
But reading one famous article without questioning it or writing the code yourself, made you believe MVI is the silver bullet on Android, let me get this straight (this is **my opinion**), there's no silver bullet nor architecture, nor doing my things right nor your way is right, there's not ***one architecture*** that solves all problems, but this is the developer's life aint't it? (kind of ironic when reading this article too)

It seems that nowadays it's popular to fight about Library X is better than Library Y, let's talk about architecture because it makes us feel more senior, we're all juniors, some of us just made more mistakes then the rest, we're disposable resources at the mercy of a one way street called time, enough with the philosophical terms let's get back to ***MVI***.

But then again you have the androidx.`ViewModel` standing in your way in that whole "**MVI**" naming doing it's thing.

<img src="/assets/img/mvi/mvi_mock.jpeg" alt ="" class="center">


## Sealed classes, MVI, my mind hurts already, but why?
---

In Kotlin usually when you hear about **MVI**, you're thinking of `sealed` **classes** and **interfaces**.

`sealed` classes are an `abstract class` that has `inner classes` declared inside it, that extend that `abstract class` and are `final`. 
`sealed` interfaces are just an `interface` that has `classes` defined inside it and those classes `implement` the `interface` of course they're `final` too.

In a way they're a **state machine**, only **one** of the declared state can be **present** at a given time and no other state (of course unless you keep a track of it via some mechanism), but that's really a poor choice (**my opinion**) to drive your UI based upon this, or is it? (since many are doing it already)

As you've seen that sometimes your requirements are: if data is already loaded and user has seen it and error is next, just show a baloon/snackbar/toast/bear/tiger/ninja turtle/idk? that gives them a warning, maybe it was a timeout of the server and they try again, it will work, you don't hide the data that's already present.

With **"one declared state present at time"**, this **isn't** possible.

Sealed **classes/interfaces** are good at being simple state machines describing your "***models***" and also you can easily test the behavior of a state machine.

Just as an ***inline class*** can be a good database ID.

Some may argue that you'll be introducing reducers that act as a mediator between the state and the next/previous state/action but things can't be simpler can it?

But your previous state was (initial) idle > loading > success and on success you have empty data and then you try to go: success (with empty data) > loading > success ... all of that averted without a reducer and one simple class holding the [payload](https://github.com/FunkyMuse/KAHelpers/blob/master/retrofit/src/main/java/com/crazylegend/retrofit/viewstate/ViewStateContract.kt), but this is just **opinionated**.

**MVI** (or whatever you wanna call it at this point) falls short at describing your UI, also the bad thing about this is that MVI's implementation somewhere else can be totally different than on Android, as aforementioned there's no silver bullet, there's no one size fits all.

## What should I use?
---

From personal experience what I found to work best is what you discuss within a team or build a hybrid solution that you'll iterate in time, just to wake up one day and say that it was a bad idea and write something better.

At the time of writing i've come up with a solution for my own needs when i'm an calling API and some variation of it (close sourced at the moment) when there are other things involved like non-API, you can look at the [API sample](https://github.com/FunkyMuse/KAHelpers/blob/master/app/src/main/java/com/crazylegend/setofusefulkotlinextensions/nav/MVIFragment.kt).

This is an opinionated article and many people would be butthurt and won't like what they read.

<img src="/assets/img/mvi/elmo.gif" alt ="" class="center">

Thank you for your ***attention***, stay **safe**, until next time.


