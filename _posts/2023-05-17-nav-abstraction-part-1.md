---
title: Abstract your Android Navigation for Compose, part 1
date: 2023-05-17 17:00:00 +0200
categories: [Hilt, Compose, Android, Navigation]
tags: [Hilt, Kotlin, Compose, Android, Navigation]
---
<img src="/assets/img/compose/compose_logo.png" alt ="" class="center" >

## Intro
---
Navigation in Compose is a *touchy* subject to many, there seems to be a new library about navigation the same way that JS frameworks are born, this article is not about creating a new navigation library.

This is sort of a continuation from a previous article that felt incomplete [Navigation in multi module Android Compose UI project + Hilt](/posts/compose_hilt_mm/).

This is an approach that I found useful and helped scale our multi module application in my previous job using the [Navigation Component](https://developer.android.com/jetpack/compose/navigation) that Google created.

Why didn't I go the route of using another library?

Intense use of Hilt and Google's team created a nice ecosystem where everything plays well.

Keep in mind that this approach might not fit your app just as every architecture should be app specific.

This approach also does utilise Hilt to help us in the goal, but it can be omitted for manual DI.

We're not gonna reinvent the wheel, rather use the aforementioned [Navigation Component](https://developer.android.com/jetpack/compose/navigation) and with some help from a brilliant repo [compose destinations](https://github.com/raamcosta/compose-destinations), we're gonna utilise the way arguments are handled.

## Problem
---
I explored several approaches and was not amused of everything just being in one file, for example this famous repo called TiVi has [the navigation in one file](https://github.com/chrisbanes/tivi/blob/main/app/src/main/java/app/tivi/AppNavigation.kt) but I thought the app can grow and you will be forced to extract this into multiple files, so that's the goal, small and single responsibility blocks.

## Goal
---

The goal is pretty short, most of the devs on my team did not know where to host the logic for the UI (collect state, instantiate viewmodels, add remember values etc.) and that created confusion.

Also when doing an abstraction, you should understand that some parts are gonna have more boilerplate than the others, but the most important thing is to make them easily deletable or modifiable, alongside those lines you as an architect should provide a way for others to not *go into the void of over "featurization"* which can make life more harder.

## Building blocks
---

To understand the abstraction and how we'll approach towards this solution, you should first be aware of the components that are used for the [Navigation Component](https://developer.android.com/jetpack/compose/navigation).

### Nav Host
---

Each starting point of a navigation is a [NavHost](https://developer.android.com/jetpack/compose/navigation#create-navhost), it's goal is to host global composable functions or nested graphs.

Every `NavHost` has a required start destination, which means that it's like a graph which has a starting node.

### Graphs
---

In order to achieve better separations it's crucial for us to utilize graphs, why?
When you have a flow of your app that is self contained is a prime example for using a graph, let's say login.

The building block for a graph in Compose is `navigation()` which has a required `startDestination` and required `route` which acts as a unique ID for this graph.
And inside a `navigation()` you can add the destinations to that graph.

Through my years of experience, I found it the best to extract every destination in it's own graph, even if it's one, one day it can become a flow and you'll leverage the power of the graph, but each to their own.

### Screens
---

Every screen of yours in the composable world is a `@Composable` function, the navigation component out of the box gives us a `composable` and a `dialog`, but since bottom sheets are a thing, we'll leverage [Accompanist navigation material](https://google.github.io/accompanist/navigation-material/) to provide a way for `bottomSheet` to be a destination too.

In this case we'll also leverage their `AnimatedNavHost` instead of the `NavHost`, no need to worry, `AnimatedNavHost` will eventually be baked into `NavHost` judging from previous Accompanist migrations to Compose foundation.

To sum it up, every screen can be one of the following:
- `composable()`
- `dialog()`
- `bottomSheet()`

### Animations
---

Every screen can be animated with the **exception** of `dialog()` and `bottomSheet()`.
Their functions don't accept animations as of writing this artcile and probably won't since the way `dialog()` works is through a window overlay (maybe one day we can animate that, idk?).

Every screen can receive one of the following:
- *enterTransition*
- *exitTransition*
- *popEnterTransition*
- *popExitTransition*

Also the `AnimatedNavHost` can receive default animations that'll be passed down to the destinations in case a screen doesn't define one for itself.

### Arguments
---

There are variables that we pass to and from a screen, we should be supporting this use cases.
In Compose when using the navigation component, everything is a `String`.

Of course, sometimes you would want to pass an argument to the previous screen in case of a callback, which would also support the aforementioned constraints.

It's important to note that each destination in the `Navigation component` has a `currentNavStackEntry` which means the current "Screen" you're showing and also its predecessor (which can be null if it's the starting destination or if you're not using graphs that might mean that your flow wasn't restored after process death up until the`currentNavStackEntry` that was saved when that occured).

### Navigating
---

Navigating should be easy (many will get triggered here), global, usually when you say `global` most of the time you think about event emitter/receiver.

As with this question, there's a harder one to ponder, which layer of the architecture should be sending the `"navigation commands"`?

The UI can drive the navigation when it's really simple, let's say navigating back, also some times you want to consume a single shot event inside the UI which can then trigger a navigation command.

I think it's a point of discussion and principle that you'll set up in your team and you'll strictly follow that convention, you can even add an additional layer that will be just for navigation logic but that'll be totally unnecessary, but it depends, do fit your use-case and not blindly follow what someone says (even this article).

## Closing notes
---
This will be the end of this blog post, thank you for your wholehearted attention, feel free to continue to [Part #2](/posts/nav-abstraction-part-2/).

