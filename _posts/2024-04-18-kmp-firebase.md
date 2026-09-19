---
title: KMP (Kotlin Multiplatform) Firebase setup
date: 2024-04-18 12:10:00 +0200
categories: [ Kotlin, Android, iOS, KMP, Firebase ]
tags: [ Kotlin, Android, iOS, KMP, Firebase ]
---

### Intro

<img src="/assets/img/kmp/kmp.png" alt ="" class="center" >

Nearly every mobile application nowadays utilizes Firebase in some capacity, whether it's implementing CRUD operations through their NoSQL database, analyzing A/B tests, or detecting crashlytics. It has become a crucial component for launching your app successfully.

Unfortunately, when it comes to Kotlin Multiplatform (KMP), Firebase lacks official support, as highlighted in this [UserVoice thread](https://firebase.uservoice.com/forums/948424-general/suggestions/46591717-support-kotlin-multiplatform).
However, GiveLiveApp has taken on much of the heavy lifting with their [SDK wrapper](https://github.com/GitLiveApp/firebase-kotlin-sdk).

This blogpost goes in hand with using this SDK instead of reinventing the wheel, you're free to use expect/actual anytime.

### Steps

## **Create Firebase project**

Navigate to the [Firebase console](https://console.firebase.google.com/u/0/) and click “Add project”

It should look like this <img src="/assets/img/kmp-firebase-setup/add_project_1.png" alt = "package structure" class="center" >

Choose a name for your Firebase project and click “Continue”.

<img src="/assets/img/kmp-firebase-setup/add_project_2.png" alt = "package structure" class="center" >

In the following step, you'll be prompted to enable analytics. This is optional; you can choose based on your preference.

<img src="/assets/img/kmp-firebase-setup/add_project_3.png" alt = "package structure" class="center" >

Wait until the project is created

<img src="/assets/img/kmp-firebase-setup/add_project_4.png" alt = "package structure" class="center" >


## **Add configuration for Android**

Now, add the configuration parameters for your Android app.


<img src="/assets/img/kmp-firebase-setup/add_project_5.png" alt = "package structure" class="center" >

After clicking on the Android icon, you'll be taken to a registration screen for your app. Ensure that you provide the correct package name and then click "Register."

<img src="/assets/img/kmp-firebase-setup/add_project_6.png" alt = "package structure" class="center" >

On the next step, make sure to download the generated google-services.json file.

Proceed to add it to your `:composeApp` or wherever your entry point to your Android app is within your KMP structured project.

<img src="/assets/img/kmp-firebase-setup/add_project_7.png" alt = "package structure" class="center" >

## **Add configuration for iOS**

The process for iOS configuration is similar to Android. Begin by adding your Apple target.


<img src="/assets/img/kmp-firebase-setup/add_project_9.png" alt = "package structure" class="center" >

Ensure that your `BUNDLE_ID`, which can be found in your `Config.xconfig`, is correctly added in the configuration file.

<img src="/assets/img/kmp-firebase-setup/add_project_11.png" alt = "package structure" class="center" >

As a naming convention, add the suffix "iOS" to your project nickname for clarity. This helps distinguish between different projects, I did not add it to the Android one on purpose as this is slight inconvenience that everyone does, not necessarily a *must* change.

<img src="/assets/img/kmp-firebase-setup/add_project_10.png" alt = "package structure" class="center" >

In the next step, download the generated `GoogleService-Info.plist` file.

Right-click on the `iosApp.xcodeproj` file in your iosApp folder and open it with Xcode.

<img src="/assets/img/kmp-firebase-setup/add_project_12.png" alt = "package structure" class="center" >

Copy the `GoogleService-Info.plist` to the *iosApp* folder.

<img src="/assets/img/kmp-firebase-setup/add_project_13.png" alt = "package structure" class="center" >

A screen will appear, click "Finish."

<img src="/assets/img/kmp-firebase-setup/add_project_14.png" alt = "package structure" class="center" >

You can change the reference path if needed.


<img src="/assets/img/kmp-firebase-setup/add_project_15.png" alt = "package structure" class="center" >

In Xcode, navigate to File > Add package dependencies.

<img src="/assets/img/kmp-firebase-setup/add_project_16.png" alt = "package structure" class="center" >

Make sure to add the Firebase dependency from `https://github.com/firebase/firebase-ios-sdk`.

<img src="/assets/img/kmp-firebase-setup/add_project_17.png" alt = "package structure" class="center" >

Choose the Firebase components you want to use from the repository. In this example, we'll use analytics and crashlytics. Then, click "Add Packages."

<img src="/assets/img/kmp-firebase-setup/add_project_18.png" alt = "package structure" class="center" >

Due to an [issue](https://github.com/JetBrains/compose-multiplatform/issues/4026) where the Frameworks, Libraries, and Embedded Content section is missing when creating a KMP project, you may need to add it manually from the Build phases tab in Xcode and restart Xcode a few times.

Ensure to add all necessary components from Firebase that you'll be using in your project.

<img src="/assets/img/kmp-firebase-setup/add_project_19.png" alt = "package structure" class="center" >


That's all for the setup process. It's a bit lengthy, but as of the time of writing this article, this is the current setup procedure.

## Writing some code

```toml
[versions]
#Firebase android
firebase-android-bom = "32.8.1"
#Gradle
gradlePlugins-crashlytics = "2.9.9"
gradlePlugins-google-services = "4.4.1"
#Gitlive
firebase-gitlive-sdk = "1.12.0"

[libraries]
#Firebase
firebase-android-bom = { module = "com.google.firebase:firebase-bom", version.ref = "firebase-android-bom" }
firebase-android-crashlytics-ktx = { module = "com.google.firebase:firebase-crashlytics" }
#Gitlive
gitlive-firebase-kotlin-crashlytics = { module = "dev.gitlive:firebase-crashlytics", version.ref = "firebase-gitlive-sdk" }

[plugins]
crashlytics = { id = "com.google.firebase.crashlytics", version.ref = "gradlePlugins-crashlytics" }
google-services = { id = "com.google.gms.google-services", version.ref = "gradlePlugins-google-services" }
```

In your project's `build.gradle.kts`, include the following plugins:

```kotlin
plugins {
    ...
    alias(libs.plugins.google.services) apply false
    alias(libs.plugins.crashlytics) apply false
}
```

In your `:composeApp` or Android app entry point, be sure to include the following plugins:

<img src="/assets/img/kmp-firebase-setup/add_project_20.png" alt = "package structure" class="center" >

Ensure that your `:shared` module's `build.gradle.kts` contains the necessary setup.

```kotlin
sourceSets {
        commonMain {
            api(libs.gitlive.firebase.kotlin.crashlytics)
        }
}   
```

## Testing the implementation

Once your setup is complete, you can run the Android app to verify the implementation.

<img src="/assets/img/kmp-firebase-setup/run_project_1.png" alt = "package structure" class="center" >

Additionally, you can configure settings such as disabling crashlytics collection on Android. Further details on configuration options are available in other articles. Below, I'll demonstrate how to configure this on iOS.

On the iOS side, you can configure the collection settings as follows. Start by adding the initialization point to your `iOSApp.swift`:

<img src="/assets/img/kmp-firebase-setup/run_project_2.png" alt = "package structure" class="center" >

Then, within your `MainViewController.kt` file, you can define your Kotlin logic to enable or disable collection in debug mode:

<img src="/assets/img/kmp-firebase-setup/run_project_3.png" alt = "package structure" class="center" >

When you run the app through Xcode, you'll notice that Firebase has been successfully initialized:

<img src="/assets/img/kmp-firebase-setup/run_project_4.png" alt = "package structure" class="center" >

## Conclusion

Integrating Firebase into your Kotlin Multiplatform project facilitates sharing some platform benefits seamlessly. However, it's worth noting that Firebase lacks support for Desktop, including but not limited to Crashlytics. This limitation may affect your project's scope.

Stay hydrated in the early summer heat! Don't forget to drink water. Thanks for reading, stay cool!

Until the next article!

<img src="/assets/img/gradle_abstraction/goodbye.jpg" alt ="" class="center">
