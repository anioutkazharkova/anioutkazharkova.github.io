---
title: "Kotlin Multiplatform. Practical multithreading (part 1)"
categories: "kotlin multiplatform"

excerpt: "Simple tutorial how to setup KMM application with common concurrency"
author_profile: true
permalink: /posts/2020-12-21-kmm-multithreading-1/

header:
    image: "/assets/images/kotlin.jpg"
    teaser: "/assets/images/kotlin.jpg"
    caption: "Kotlin Island"
---


There are a lot of different samples how to create a simple basic KMM app. But I would like to demonstrate something more advanced, that we face in our daily tasks. So it will be a multithreading Kotlin Multiplatform app for both iOS/Android platforms.   ![](https://cdn-images-1.medium.com/max/800/0*aOgBnqEuas6HamWR.png)

At first, little bit introduction to Kotlin Multiplatform SDK. If you already know the basics of KMM, you can either scroll to the sample or read  [part 2](https://medium.com/p/1c20e6e36d4d)   about more advanced case.

The basic idea of KMM is common to other cross-platform technologies. Performing optimization of the development, write your code once and use it on all your platforms.

According to the conception of JetBrains company, Kotlin Multiplatform is not a framework. It is an SDK to create your own codebase modules and connect them to your native projects.   ![](https://cdn-images-1.medium.com/max/800/0*hNSTQd6Pd6dy8a_6.png)

KMM SDK uses the specific for every native platform versions of Kotlin: Kotlin/JVM, Kotlin/JS or Kotlin/Native. Every version includes special Kotlin language’s extensions, tools and libraries specified for particular platform.
The module created with Kotlin language will be compiled into JVM bitecode for Android and into LLVM bitecode to iOS.

Shared module contains all common reused business logic. Platform-specific modules for iOS/Android use the logic from the connected Shared modules as it is or implement its own solution depending on platform and its needs.

Common business logic could contain:  -  common networks services;
-  common database storages and service;
-  data models, entities.


It could also contain common architectural solutions for both platforms and all its parts not containing UI code:  -  ViewModel;
-  Presenter;
-  Interactors and etc.


Kotlin Multiplatform and its conception can be compared with Xamarin. Though there is no code for UI implementation. All UI should be implemented in native projects.

Let’s take a look how to apply it on practice.

If you haven’t worked with KMM before, you need to install and setup tools. You need to download and install  [Android Studio 4.1)](https://developer.android.com/studio)   and  [Kotlin Multiplatform Mobile](https://plugins.jetbrains.com/plugin/14936-kotlin-multiplatform-mobile?_ga=2.222289432.1256240536.1608381251-893400752.1607774396)  plugin. Now you need to create your project with KMM Application template. The project with its basic structure will be created automatically.   ![](https://cdn-images-1.medium.com/max/800/0*99sbImR_OLXIfDrK.png)

Kotlin Multiplatform project contains several modules of codes:  -  shared business logic (Shared, commonMain и т.п);
-  platform-specific ios logic (iOSMain, iOSTest);
-  platform-specific android logic (androidMain, androidTest).


All our shared reusable business logic is in commonMain/kotlin and it could be also separated to different packages. All the functions, classes, objects and variables that should be reimplemented in platforms-specific code at first should be declared with expect modifier:   ![](https://cdn-images-1.medium.com/max/800/0*rAlsVAQ4eovIz3dK.png)

All implementations should use actual modifier

Here is an example of simple multithreading application that uses open api:   ![](https://cdn-images-1.medium.com/max/800/0*rdtuBuN6btI9F_SU.png)

My sample uses  [www.themoviedb.org](https://www.themoviedb.org)  . You can find the link to its repo at the end of the article.

All shared business logic is placed in Common project:   ![](https://cdn-images-1.medium.com/max/800/0*kCVpwCqlOro5Lb-5.png)

We will have common network service. It is easy.

iOS/Android apps will contain only UI components and its special adapters. iOS app will be written on Swift, Android app will uses Kotlin.

All business will be places in commonMain, that’s why we need to uses common Kotlin Multiplatform solutions:

Ktor — library for networking and serialization.

Let’s add following dependendencies in build.gradle (:app):
```
val ktorVersion = "1.4.0"
val serializationVersion = "1.0.0-RC"
 sourceSets {
        val commonMain by getting {
            dependencies {
                implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:$coroutinesVersion")
                implementation("io.ktor:ktor-client-core:$ktorVersion")
                implementation("org.jetbrains.kotlinx:kotlinx-serialization-core:$serializationVersion")
                implementation("io.ktor:ktor-client-serialization:$ktorVersion")
            }
        }
        val androidMain by getting {
            dependencies {
                //...
                implementation("io.ktor:ktor-client-android:$ktorVersion")
            }
        }
        val iosMain by getting {
            dependencies {
                implementation("io.ktor:ktor-client-ios:$ktorVersion")
            }
        }
        ...
```
We also need to support serialization:
```
plugins { //... kotlin("plugin.serialization") version "1.4.10" }
```
At that moment we should decide, how to implement multithreading. It is a platform-specific logic: we use GCD (Grand Central Dispatch) in iOS projects and JVM Threads and Coroutines for our Android app.
Kotlin Multiplatform provides common solution for multithreading. It uses Kotlin, so we can use Coroutines, special version of every platform:
```
val coroutinesVersion = "1.3.9-native-mt"
 sourceSets {
        val commonMain by getting {
            dependencies {
                implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:$coroutinesVersion")
               //...
            }
        }
        val androidMain by getting {
            dependencies {
                implementation("org.jetbrains.kotlinx:kotlinx-coroutines-android:$coroutinesVersion")
               //...
            }
        }
        val iosMain by getting {
            dependencies {
                //...
            }
        }
        ...
```
Now let’s provide some basics of Coroutines, because not every iOS developer knows about this technology. Coroutine it is a simple block of code that can be suspended without blocking its thread. It has its own context of running (CoroutineContext), it has a manager of its lifecycle (Job). It has its own CoroutineScope and its thread is specified with special CoroutineDispatcher.
It could be compared with performing of code block in specified DispatchQueue in iOS or in specified NSThread. Or just like an Operation in OperationQueue. Even GlobalScope looks resemble to DispatchQueue.global(), аs MainScope to DispatchQueue.main.
```
//Android
fun loadMovies() {
  GlobalScope.async {
    service.makeRequest()
   withContext(uiDispatcher) {
//...
}
  }
}

//iOS
func loadMovies() {
  DispatchQueue.global().async {
    service.makeRequest()
  DispatchQueue.main.async{
//...
}
}
}
```
All coroutines use suspend key word. This modifier doesn’t make method asynchronous, because it depends on other details of realization. It indicates that such function could be suspended without blocking its own thread. It also shows that this function could be called only from coroutine context.

Ktor also uses coroutines for its asynchronous work, that’s why we call HttpClient from suspended function:
```
//Network service
class NetworkService {
     val httpClient = HttpClient {
        install(JsonFeature) {
            val json = kotlinx.serialization.json.Json { ignoreUnknownKeys = true }
            serializer = KotlinxSerializer(json)
        }
    }

    suspend inline fun <reified T> loadData(url: String): T? {
       return httpClient.get(url)
    }
}

//Movies service
 suspend fun loadMovies():MoviesList? {
        val url = MY_URL
        return networkService.loadData<MoviesList>(url)
    }
```
When we added Kotlin Coroutines to our project, we haven’t specified any special version for iOS. It is not a mistake. Since Kotlin 1.4 it has basic support for suspending functions in Swift and Objective-C. All suspending functions are available as functions with callbacks and completion handlers:
```
func getMovies() {
    self.networkService?.loadMovies {(movies, error) in 
      //...
    }
}
```
Because Ktor makes all its work asynchronously under its hood, we have no need to use DispatchQueue in our iOS app.

But in our Android app we should use coroutines to be able to call suspended functions:
```
fun getMovies() {
    mainScope.launch {
    val movies = this.networkService?.loadMovies()
//...
    }
}
}
```
So, such implementation of multithreading can be used if we don’t need to have common architectural solution in both our native apps. In this case we implement only Common business logic. It is a work solution.

If we need to create a fully shared common code project, with common architectural implementations connected with our UI with specified protocols, we should use another solution to provide multithreading.

See it in  [part 2](https://dev.to/anioutkajarkova/kotlin-multiplatform-practical-multithreading-part-2-47po)

Source code  [github.com/anioutkazharkova/movies_kmp](https://github.com/anioutkazharkova/movies_kmp)
More info about [coroutines](https://github.com/Kotlin/kotlinx.coroutines)


------------------------------


Originally published at  * [https://habr.com*](https://habr.com/ru/post/533864/)     . *


By  [Anna Zharkova](https://medium.com/@anioutkazharkova)   on  [December 21, 2020](https://medium.com/p/db8af265d5cc)    .

[Canonical link](https://medium.com/@anioutkazharkova/kotlin-multiplatform-practical-multithreading-part-1-db8af265d5cc)

Exported from  [Medium](https://medium.com)   on January 20, 2021.

---

Originally published at https://habr.com.