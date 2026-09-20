---
title: Assisted Inject for AAC ViewModels with vanilla Dagger2
date: 2021-11-22 20:10:00 +0100
categories: [Dagger]
tags: [Android, Kotlin, Dagger]
---

<img src="/assets/img/dagger/dagger_title.jpeg" alt ="" class="center">

Into the [Part #7](/posts/dagger-part-7/) of the Dagger series, we saw how complicated `ViewModel` injection can be on Android, but that can be easily fixed with `@AssistedInject`.

For this to work we need `@AssistedInject` on a constructor, an `@AssistedFactory` for the `ViewModel` factory and `@Assisted` for each parameter we'll be providing as an outside dependency.

Note that this is easily solved with `@ViewModelInject` from *Hilt* but in a pure Dagger world this is how it'll work.

## By assistedViewModel
---
Each `ViewModel` can have a `SavedStateHandle` for your needs, in order to get an access to that dependency we need to implement `AbstractSavedStateViewModelFactory` while leveraging the lazy creation of the delegate that's already provided to us `by viewModels<>`.

Upon looking closely
```kotlin
object : AbstractSavedStateViewModelFactory(TODO()) {
        override fun <T : ViewModel> create(
            key: String,
            modelClass: Class<T>,
            handle: SavedStateHandle
        ) 
```
after implementing the factory we have the handle, all we need to do is give this factory to the `Fragment.viewModels` delegate as it awaits a `factoryProducer`

```kotlin
inline fun <reified T : ViewModel> Fragment.assistedViewModel(
    crossinline viewModelProducer: (SavedStateHandle) -> T
) = viewModels<T> {
    object : AbstractSavedStateViewModelFactory(this, arguments) {
        override fun <T : ViewModel> create(
            key: String,
            modelClass: Class<T>,
            handle: SavedStateHandle
        ) =
            viewModelProducer(handle) as T
    }
}
```
we can have a nice delegate functionality `by assistedViewModel` in our `Fragment`s now.

## Assisted ViewModel
---
Let's take an example of a `ViewModel` that looks like the following, pretty simple, `ExampleNetworkSearchService` is provided from our `SingletonComponent`.

```kotlin
class SearchViewModel @AssistedInject constructor(
    private val searchService: ExampleNetworkSearchService,
    @Assisted savedStateHandle: SavedStateHandle
) : ViewModel() {
    private val args = SearchFragmentArgs.fromSavedStateHandle(savedStateHandle)
    private val query = args.query
    private val sort = args.sort
}
```   

## Assisted ViewModel factory
---
In order to facilitate the usage of a factory, an assisted factory has the job to create the dependency it provides, in this case a `SearchViewModel`.
```kotlin
class SearchViewModel @AssistedInject constructor(
    private val searchService: ExampleNetworkSearchService,
    @Assisted savedStateHandle: SavedStateHandle
) : ViewModel() {

    @AssistedFactory
    interface Factory {
        fun create(
            savedStateHandle: SavedStateHandle
        ): SearchViewModel
    }

    private val args = SearchFragmentArgs.fromSavedStateHandle(savedStateHandle)
    private val query = args.query
    private val sort = args.sort
}
```

### Assisted inside a Fragment
---
and inside the `SearchFragment` we can leverage


```kotlin
class SearchFragment : Fragment(R.layout.fragment_search) {

    override fun onAttach(context: Context) {
        fragmentComponent.inject(this)
        super.onAttach(context)
    }

    private val binding by viewBinding(FragmentSearchBinding::bind)

    @Inject
    lateinit var searchViewModelFactory: SearchViewModel.Factory
    private val searchViewModel by assistedViewModel { savedStateHandle -> searchViewModelFactory.create(savedStateHandle) }
}
```

The way this works is that you're providing the `@AssistedInject` annotation dependency that you already have in the Dagger graph in our case an `ExampleNetworkSearchService` because Dagger knows about this and doesn't need an assistance, but what Dagger graph doesn't have is access to `SavedStateHandle` and we assist Dagger with a dependency that's just created (creation time assistance).

The downside to this is having one factory per one `ViewModel`, a lot of boilerplate to be written, also if you inject assisted dependencies that outlive the `ViewModel`'s lifecycle that means you'll have memory leaks that you haven't handled (dependencies like Fragment's or Activitiy's own stuff, even worse a View).

This is just to show the easy it can be without having to rely on `MultiMap` bindings from Dagger.

However *Hilt* makes all of this go away and should be the preferred solution for writing injectable `ViewModel`s in your code unless you have a ton of Dagger code that can't be migrated.

Happy coding.

<img src="/assets/img/hilt/1/hilt_1.jpg" alt ="" class="center">
