---
title: KMP (Kotlin Multiplatform) multi module resources
date: 2024-05-16 22:20:00 +0200
categories: [ Kotlin, KMP ]
tags: [ Kotlin, KMP ]
---

### Intro

<img src="/assets/img/kmp/kmp.png" alt ="" class="center" >

Every mobile application today ships with static resources like localized strings, images, files, fonts, etc.

Resource usage in the KMP (Kotlin Multiplatform) world has been somewhat limited, with no official support until now. 
Previously, some of you might have used [Lyrcist](https://github.com/adrielcafe/lyricist), which had its flaws, such as placing strings inside a data class. 
A better alternative was [Moko resources](https://github.com/icerockdev/moko-resources), but the setup was lengthy, plurals weren't handled correctly, there were inconsistencies in AGP support, and fonts didn't load properly (these are issues I've encountered, which might be fixed now).

![img-description](/assets/img/kmp_mm_resources/2.png)
_Low effort meme_

Then Jetbrains introduced [Images and Resources](https://www.jetbrains.com/help/kotlin-multiplatform-dev/compose-images-resources.html).

I won't bore you with the details as they were explained by JetBrains. However, the most important things you should be aware of are:
- Almost all resources are read synchronously in the caller thread. The only exceptions are raw files and all of the resources on the JS platform that are read asynchronously.
- Reading big raw files, like long videos, as a stream is not supported yet. Use separate files on the user device and read them with the file system API, for example, the [kotlinx-io](https://github.com/Kotlin/kotlinx-io) library, i'd add [okio](https://github.com/square/okio) here as well
- Only files that are part of the application are considered resources (this is obvious I mean, but should be highlighted)

### Setup

As their article already mentioned how you can set it up in a single module project, I'm not going to cover that as it's pretty straightforward. Instead, I will focus on the multi-module project's approach, which lacks documentation.

Let's say you've decided to place all your resources in a separate module, `resources`, so that you can reuse it in every module you need later in your project.

*Keep in mind that using vectors in KMP has to satisfy Android's format*

Our `androidMain`, `iosMain` and `jvmMain` are gonna be empty for this demonstration unless you need to implement something specific.

<img src="/assets/img/kmp_mm_resources/1.png" alt ="" class="center" >

Your Gradle build file will be simple:

```kotlin
sourceSets {
        commonMain.dependencies {
            api(compose.components.resources)
        }
    }
```

Now, the most important part is configuring the generated class to be public:

```kotlin
compose.resources {
    publicResClass = true
    packageOfResClass = "dev.funkymuse.myawesomeproject.resources"
    generateResClass = auto
}
```

The `ResourcesExtension` configuration class is simple, having only three properties. By assigning the package of the res class, you can access it outside of this module by that package name.

As you saw in the image above, I had strings and static raw files. For the raw files, there's no generated `Res` class, so we have to write a bit of boilerplate code. For everything else, it's generated (I hope raw files are supported in the future too since they're placed in the raw folder anyway. It shouldn't be that hard to generate the code I'll be writing, but I guess the current method is due to folder nesting you can have inside the raw folder).

```kotlin
object ResourceFiles {
    
    @OptIn(ExperimentalResourceApi::class)
    suspend fun charsets() = Res.readBytes(path = "files/charsets.json")

    
    @OptIn(ExperimentalResourceApi::class)
    suspend fun languages() = Res.readBytes(path = "files/languages.json")

}
```
This will enable us to use it outside of this module.

Inside our `composeApp` module, we only have to include the project's module:

```kotlin
commonMain.dependencies {
    implementation(projects.resources)
}
```

You can enable project accessors in the `settings.gradle` file by adding this line:
`enableFeaturePreview("TYPESAFE_PROJECT_ACCESSORS")`.

The resources are generated once you build the project, in order to avoid building the whole project, you can just run:

```shell
./gradlew generateComposeResClass
```

And now in your UI code:
```kotlin
import dev.funkymuse.myawesomeproject.resources.Res


stringResource(resource = Res.string.delete_subtitles) //string
painterResource(Res.drawable.heart_plus) //drawable


val fontInter = FontFamily(Font(Res.font.inter)) //font
```

## Bye

This was inteded to be short.

Until the next article, ciao!

