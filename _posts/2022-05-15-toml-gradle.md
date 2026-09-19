---
title: TOML + Gradle + project accessors
date: 2022-05-15 12:30:00 +0100
categories: [Android, Gradle]
tags: [Android, Gradle, Dependencies]
---

<img src="/assets/img/11/gradle.png" alt ="" class="center">

## Intro
---

The new and shiny feature by Gradle is their way of having [conventional dependencies](https://docs.gradle.org/current/userguide/platforms.html#sub:conventional-dependencies-toml) that are organized in a fashionable and easy to grasp manner, now being stable in *Gradle* `7.4.2`, the TOML `libs.versions` a.k.a *version catalogs* are the way to handle dependencies instead of hardcoding the artifacts in your existing Gradle scripts.

## Before we get started
---
1. Make sure you have your `distributionUrl` targeting at least `gradle-7.4.2-bin.zip`, otherwise up until that version you have to add `enableFeaturePreview("VERSION_CATALOGS")` inside `settings.gradle`
2. Create a `libs.versions.toml` file inside your **gradle** folder

## Understanding the TOML structure
---
There are four things to know when looking inside a `TOML` version catalog file

1. `[versions]` - where you would have the version number/s of your dependencies
2. `[libraries]` - kind of self explanatory, the libraries you would include later on and use `version.ref` to obtain the version from `versions`
3. `[bundles]` - you can group together multiple `libraries` from **.2**
4. `[plugins]` - where the plugins that are being added to the project live

`TOML` is minimal language and is designed to map unambiguously to a hash table, this makes it easy to couple it with a Gradle script, however do note that you can add tons of stufs in that .toml file that would make them easier for your development process later on, you can check the additional [specs](https://toml.io/en/v1.0.0) of what more could be added.


*For the sake of this article and brevity we would keep it short and what you may use daily*
## How?
---
Let's go on by adding few versions of what would be our dependency handler later on.

Open the `libs.versions.toml` file inside your **gradle** folder that you created before you started.

First we add versions for each library we'd use
```toml
[versions]
kotlin = "1.6.21"
coroutines = "1.6.1"
kaHelpers = "3.4.0"
gradlePlugins-agp = "7.2.0"
```

We include the libraries

```toml
[libraries]
coroutines-core = { module = "org.jetbrains.kotlinx:kotlinx-coroutines-core", version.ref = "coroutines" }
coroutines-android = { module = "org.jetbrains.kotlinx:kotlinx-coroutines-android", version.ref = "coroutines" }
kahelpers-toaster = { module = "com.github.FunkyMuse.KAHelpers:toaster", version.ref = "kaHelpers" }
```

now as you can see we, have two libraries that makes sense to group them we use **bundles** to do so
```toml
[bundles]
coroutines = ["coroutines-android", "coroutines-core"]
```
as aforementioned that the `TOML` is basically a hash table, `bundles` can look up libraries by the name (key) you declared, just as `version.ref` does for `versions`.

Since we added versions for plugins we can utilise the `plugins` section
```toml
[plugins]
android = { id = "com.android.application", version.ref = "gradlePlugins-agp" }
kotlinAndroid = { id = "org.jetbrains.kotlin.android", version.ref = "kotlin" }
kapt = { id = "org.jetbrains.kotlin.kapt", version.ref = "kotlin" }
```

The plugins section is gonna help us handle the plugins that are being added to the project without lots of boilerplate.

Our complete version catalog file ends up looking like

```toml
[versions]
kotlin = "1.6.21"
coroutines = "1.6.1"
kaHelpers = "3.4.0"
gradlePlugins-agp = "7.2.0"

[libraries]
coroutines-core = { module = "org.jetbrains.kotlinx:kotlinx-coroutines-core", version.ref = "coroutines" }
coroutines-android = { module = "org.jetbrains.kotlinx:kotlinx-coroutines-android", version.ref = "coroutines" }
kahelpers-toaster = { module = "com.github.FunkyMuse.KAHelpers:toaster", version.ref = "kaHelpers" }

[bundles]
coroutines = ["coroutines-android", "coroutines-core"]

[plugins]
android = { id = "com.android.application", version.ref = "gradlePlugins-agp" }
kotlinAndroid = { id = "org.jetbrains.kotlin.android", version.ref = "kotlin" }
kapt = { id = "org.jetbrains.kotlin.kapt", version.ref = "kotlin" }
```

## Usage
---
Gradle allows really easy way to use the version catalog inside your scripts, you can access everything using `libs.`

It's really simple, you can now use 
```gradle
implementation(libs.bundles.coroutines)
implementation(libs.kahelpers.toaster)
```
Each `-` in the libraries name is replaced with `.` inside the script, it's like having a domain package name structure for the libraries.

As we progress towards the plugins section, inside your top level gradle `build.gradle.kts` script you can declare your plugins

```gradle
plugins {
    alias(libs.plugins.android).apply(false)
    alias(libs.plugins.kotlinAndroid).apply(false)
    alias(libs.plugins.kapt).apply(false)
}
```

we're using aliases because we're gonna reference them from the gradle plugin portal instead of manually adding them to the `buildScript` **classpath**.

Inside your `settings.gradle.kts` make sure you have `gradlePluginPortal()`
```gradle
pluginManagement {
    repositories {
        gradlePluginPortal() // <- this
        google()
        mavenCentral()
    }
}
```
this pulls all the plugins from the gradle portal by their alias ID and adds them manually to the build script until you apply them later on wherever needed, this is why they're not applied now since they live in the top-level gradle script.

Now inside your `:app` module you can enable the plugins you've just added
```gradle
plugins {
    alias(libs.plugins.android)
    alias(libs.plugins.kotlinAndroid)
    alias(libs.plugins.kapt)
}
```
Thanks to [Atul](https://twitter.com/atlgpt17?t=tGroZSms5urafTahluzYCA&s=09) for improving the alias for the `:app` module.

## Project accessors
---
Since the version catalog is a powerful feature that one can utilise for a modern architecture where the project isn't monolithic you need to gather some of your features inside the `:app` module, let's say in order to call them there (do note that this can vary from project to project since every project has different structure, this is only for demonstrations).

Tot enable the feature, inside your `gradle.settings.kts` add this line
```gradle
enableFeaturePreview("TYPESAFE_PROJECT_ACCESSORS")
```

so now instead of typing
``` gradle
implementation project(":feature_home")
```

you can use
``` gradle
implementation(projects.feature_home)
```

## Drawbacks of using version catalogs
---
As of this article's date, there are several drawbacks that you need to be aware when using version catalogs for your projects

1. There's no auto completion by the IDE (yet)
2. There's no auto suggestion to update the versions
3. Dependabot still hasn't support for TOML with Gradle, there's an [issue](https://github.com/dependabot/dependabot-core/issues/3121) but who knows, the alternative way is using [renovate](https://github.com/renovatebot/renovate) or a custom Github cron job that can update the versions
4. It doesn't control transitive dependencies, a catalog only references direct dependencies

With this in mind and as mentioned in the beginnig, treat the version catalog as a catalog for discoverability and easier maintenance.

## Updating versions using community plugins
---
Since some of the aforementioned limitations are annoying, the community has presented us with solutions.
There's a wonderful [plugin](https://github.com/littlerobots/version-catalog-update-plugin) that would enable us to manually update the version catalog via a gradle task.


Our final version catalog ends up looking like this
```toml
[versions]
kotlin = "1.6.21"
coroutines = "1.6.1"
kaHelpers = "3.4.0"
gradlePlugins-agp = "7.2.0"
gradlePlugins-versionCatalog = "0.3.1"

[libraries]
coroutines-core = { module = "org.jetbrains.kotlinx:kotlinx-coroutines-core", version.ref = "coroutines" }
coroutines-android = { module = "org.jetbrains.kotlinx:kotlinx-coroutines-android", version.ref = "coroutines" }
kahelpers-toaster = { module = "com.github.FunkyMuse.KAHelpers:toaster", version.ref = "kaHelpers" }

[bundles]
coroutines = ["coroutines-android", "coroutines-core"]

[plugins]
android = { id = "com.android.application", version.ref = "gradlePlugins-agp" }
kotlinAndroid = { id = "org.jetbrains.kotlin.android", version.ref = "kotlin" }
kapt = { id = "org.jetbrains.kotlin.kapt", version.ref = "kotlin" }
versionCatalogUpdate = { id = "nl.littlerobots.version-catalog-update", version.ref = "gradlePlugins-versionCatalog" }
```

so that our top level gradle `build.gradle.kts` looks like this

```gradle
plugins {
    alias(libs.plugins.android).apply(false)
    alias(libs.plugins.kotlinAndroid).apply(false)
    alias(libs.plugins.kapt).apply(false)
    alias(libs.plugins.versionCatalogUpdate)
}
```

we can update the version numbers by running a simple script

```gradle
./gradlew versionCatalogUpdate
```

You can now use version catalogs and project accessors to enable modern way of keeping your dependencies clear and access the sub-projects in fashionable and readable manner, this article doesn't provide a way to build a Github action to create a dependabot auto update of the versions but you have all the required knowledge to create one for your project.


## Goodbye
---
Don’t forget to drink water, it’s starting to look like summer and it’s hot (just as your machine running a build using Gradle).

This is my first article in 2022 as i've been spending this year learning Compose and Ktor backend, mentoring my friend and spending quality time on my passion projects.

Here's a picture of two red pandas to make your day more awesome.

<img src="/assets/img/11/red_pandas.webp" alt ="" class="center">
