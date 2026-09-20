---
title: Spotless and Ktlint for your Android app
date: 2023-04-03 13:50:00 +0100
categories: [Android]
tags: [Android]
---
<img src="/assets/img/spotless/spotless_logo.png" alt ="" class="center" >

## What is spotless?
---
Spotless is a code formatter that is usually used to automate corrections of mistakes such as spacing, new lines, unnecessary imports, break lines and many more useful features.

If you'd like to learn more about it, you can check out this [link](https://github.com/diffplug/spotless).

## Setup
---
You may have heard about [ktlint](https://pinterest.github.io/ktlint/) which will run alongside spotless in order to enforce some common rules for the Kotlin language.

In order to run Spotless, you need to install Ktlint on your machine.

You can do so by running 
```bash
brew install ktlint
```
on your mac, or by checking their [download](https://pinterest.github.io/ktlint/install/cli/) pages for different OS installation.

*Note that there's an option to add the [ktlint gradle plugin](https://github.com/JLLeitschuh/ktlint-gradle) instead of having to install it locally, but I did not proceed with it because the state of this plugin is uncertain. Even though the developers are still active, it's really sad to see something like [this](https://github.com/JLLeitschuh/ktlint-gradle/issues/569) on a cool open source project.*

After that, add the version to your version catalog. If you don't know how to set one up, you can check out this [tutorial](/posts/toml-gradle/).
```toml
[versions]
gradlePlugins-spotless = "6.17.0" #or latest version
[plugins]
spotless = { id = "com.diffplug.spotless", version.ref = "gradlePlugins-spotless" }
```

Inside your project `gradle` within the plugins block add
```kotlin
plugins {
    ...
    alias(libs.plugins.spotless).apply(false)
}
```
then inside your `subprojects`, configure and apply the plugin for the app and library modules:
```kotlin
subprojects {
    plugins.matching { anyPlugin -> anyPlugin is AppPlugin || anyPlugin is LibraryPlugin }.whenPluginAdded {
        apply(plugin = libs.plugins.spotless.get().pluginId)
        extensions.configure<SpotlessExtension> {
        kotlin {
            target("**/*.kt")
            targetExclude("$buildDir/**/*.kt")
            ktlint()
                .setEditorConfigPath("${project.rootDir}/spotless/.editorconfig")
        }
      }
    }
}
```
Of course, any additional logic can be tailored to your own use-case. For example, you can add credits, license as headers, etc.

Inside your root project folder (or anywhere else you want, but keep in mind the path), create a `spotless` folder and inside it (for better organization) create a `.editorconfig` file which we're including for Ktlint's setup in this line `ktlint().setEditorConfigPath("${project.rootDir}/spotless/.editorconfig")`.

Your `.editorconfig` can include any of the rules you can find on the [ktlint rules page](https://pinterest.github.io/ktlint/rules/standard/) (some for example):
```config
root = true

[*]
charset = utf-8
indent_style = space
trim_trailing_whitespace = true

[*.{kt,kts}]
indent_size = 4
ktlint_standard = enabled
ktlint_experimental = enabled
ktlint_code_style = official
ktlint_standard_comment-spacing = disabled
ktlint_standard_package-name = disabled
ktlint_experimental_package-naming = disabled
ij_kotlin_allow_trailing_comma = true
ij_kotlin_allow_trailing_comma_on_call_site = true
```
Feel free to change this config however you want or your team needs or agreed upon.

## Running
---

It's really easy, go inside your terminal and run
```gradle
./gradlew spotlessApply --stacktrace

```

then spotless will apply the formatting according to your setup.

## Pre commit hook
---
This is not a mandatory process since it can take a lot of time running on your machine (depending on project size and machine), it's better to config this on the CI/CD machine but here's how if you're wondering.

Create a script to run the spotless plugin
```bash
#!/bin/bash

echo "Running spotless"
./gradlew spotlessApply
git add .
```

inside your project `gradle` file add 

```kotlin
tasks.register("copySpotlessPreCommitHook") {
    doLast {
        copy {
            from("./spotless/spotless.sh")
            into("./.git/hooks")
        }
    }
}
```
you can add additional logic and improve the task code.

I hope this tutorial has helped you and your team to bring your code into one common harmony.

Until next article, summer is almost here, don't forget to stock on sun screen and apply it later on of course :)
