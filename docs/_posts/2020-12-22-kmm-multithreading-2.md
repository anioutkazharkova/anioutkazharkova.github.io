---
title: "Kotlin Multiplatform. Practical multithreading (part 2)"
categories: "kotlin multiplatform"

excerpt: "Simple tutorial how to setup KMM application with common concurrency"
author_profile: true
permalink: /posts/2020-12-22-kmm-multithreading-2/
header:
    image: "/assets/images/kotlin.jpg"
    teaser: "/assets/images/kotlin.jpg"
    caption: "Kotlin Island"
---



![](https://cdn-images-1.medium.com/max/800/1*tbmc_gILwf_Q-_tDBFNViA.png)


In previous [article](https://dev.to/anioutkajarkova/kotlin-multiplatform-practical-multithreading-part-1-4357) I demontstrated one of the possible solutions to implement multitheading in Kotlin Multiplatform application. In this part I’m going to describe another solution, it will be KMM application with fully shared common code and common multithreading in shared business logic.   ![](https://cdn-images-1.medium.com/max/800/0*Xv2L7roxS1nyUFNa.png)

In previous sample we used Ktor to make our common network client. This library makes all asynchronous work under its hood. In this case we have no need event to use DispatchQueue in our iOS native application. But in other cases we should use queues to request our business logic correctly and process the result responses. We also used MainScope to call suspended methods in our native Android app.

So if we want to implement multithreading in our shared code with common logic, we should use coroutines for all parts of our project, so we need to setup correct scopes and contexts of our coroutines.

Let’s begin with something simple. First of all, we will create our intemediate architectual component. I will use MVP pattern, so I need to make my presenters. This presenter will call all the methods of the specified service in its own CoroutineScope initialized with the CoroutineContext:
```
class PresenterCoroutineScope(context: CoroutineContext) : CoroutineScope {
    private var onViewDetachJob = Job()
    override val coroutineContext: CoroutineContext = context + onViewDetachJob

    fun viewDetached() {
        onViewDetachJob.cancel()
    }
}

//base class 
abstract class BasePresenter(private val coroutineContext: CoroutineContext) {
    protected var view: T? = null
    protected lateinit var scope: PresenterCoroutineScope

    fun attachView(view: T) {
        scope = PresenterCoroutineScope(coroutineContext)
        this.view = view
        onViewAttached(view)
    }
}
```

So, as previously mentioned, the presenter requests service methods in specified scope and then deliver the results to our UI:
```
class MoviesPresenter:BasePresenter(defaultDispatcher){
    var view: IMoviesListView? = null

    fun loadData() {
        //call in scope
        scope.launch {
            service.getMoviesList{
                val result = it
                if (result.errorResponse == null) {
                    data = arrayListOf()
                    data.addAll(result.content?.articles ?: arrayListOf())
                    withContext(uiDispatcher){
                    view?.setupItems(data)
                   }
                }
            }
        }

//IMoviesListView - protocol to implement with UIViewController/Activity. 
interface IMoviesListView  {
  fun setupItems(items: List<MovieItem>)
}
class MoviesVC: UIViewController, IMoviesListView {
private lazy var presenter: IMoviesPresenter? = {
       let presenter = MoviesPresenter()
        presenter.attachView(view: self)
        return presenter
    }()

    override func viewWillAppear(_ animated: Bool) {
        super.viewWillAppear(animated)
        presenter?.attachView(view: self)
        self.loadMovies()
    }

    func loadMovies() {
        self.presenter?.loadMovies()
    }

   func setupItems(items: List<MovieItem>){}
//....

class MainActivity : AppCompatActivity(), IMoviesListView {
    val presenter: IMoviesPresenter = MoviesPresenter()

    override fun onResume() {
        super.onResume()
        presenter.attachView(this)
        presenter.loadMovies()
    }

   fun  setupItems(items: List<MovieItem>){}
//...
```
We need to specify CoroutineDispatcher to initialize our CoroutineScope with CoroutineContext.

We need to use a platform-specific code, that’s why we’re going to customize it with expect/actual mechanism.
```
expect val defaultDispatcher: CoroutineContext

expect val uiDispatcher: CoroutineContext
```
uiDispatcher should be used with UI-thread logic and other logic will be requested with defaultDispatcher.

It could be easily done in our androidMain, because it uses Kotlin JVM, so there are default dispatchers for both cases. Dispatchers.Default is a default dispatcher for Coroutines mechanism:

```
actual val uiDispatcher: CoroutineContext
    get() = Dispatchers.Main

actual val defaultDispatcher: CoroutineContext
    get() = Dispatchers.Default
```

CoroutineDispatcher uses specified fabric MainDispatcherLoader under the hood to create MainCoroutineDispatcher for requesting platform:
```
internal object MainDispatcherLoader {

    private val FAST_SERVICE_LOADER_ENABLED = systemProp(FAST_SERVICE_LOADER_PROPERTY_NAME, true)

    @JvmField
    val dispatcher: MainCoroutineDispatcher = loadMainDispatcher()

    private fun loadMainDispatcher(): MainCoroutineDispatcher {
        return try {
            val factories = if (FAST_SERVICE_LOADER_ENABLED) {
                FastServiceLoader.loadMainDispatcherFactory()
            } else {
                // We are explicitly using the
                // `ServiceLoader.load(MyClass::class.java, MyClass::class.java.classLoader).iterator()`
                // form of the ServiceLoader call to enable R8 optimization when compiled on Android.
                ServiceLoader.load(
                        MainDispatcherFactory::class.java,
                        MainDispatcherFactory::class.java.classLoader
                ).iterator().asSequence().toList()
            }
            @Suppress("ConstantConditionIf")
            factories.maxBy { it.loadPriority }?.tryCreateDispatcher(factories)
                ?: createMissingDispatcher()
        } catch (e: Throwable) {
            // Service loader can throw an exception as well
            createMissingDispatcher(e)
        }
    }
}
```
Same mechanism used for DefaultDispatcher:
```
internal object DefaultScheduler : ExperimentalCoroutineDispatcher() {
    val IO: CoroutineDispatcher = LimitingDispatcher(
        this,
        systemProp(IO_PARALLELISM_PROPERTY_NAME, 64.coerceAtLeast(AVAILABLE_PROCESSORS)),
        "Dispatchers.IO",
        TASK_PROBABLY_BLOCKING
    )

    override fun close() {
        throw UnsupportedOperationException("$DEFAULT_DISPATCHER_NAME cannot be closed")
    }

    override fun toString(): String = DEFAULT_DISPATCHER_NAME

    @InternalCoroutinesApi
    @Suppress("UNUSED")
    public fun toDebugString(): String = super.toString()
}
```
But not for all native platforms we can use existed default coroutine dispatchers. For example, such platforms as iOS work with KMM via Kotlin/Native, not Kotlin/JVM.

So if we try to use the same implementation, as we used for Android, we will receive an error:   ![](https://cdn-images-1.medium.com/max/800/0*z151yPt_8_rQ4wh7.png)

Let’s take a look, what has happened.

GitHub Kotlin Coroutines  [Issue 470](https://github.com/Kotlin/kotlinx.coroutines/issues/470)   contains information, that these special dispatchers for iOS haven’t been created in Kotlin/Native yet:   ![](https://cdn-images-1.medium.com/max/800/0*XmhD-SVkLgICSJsJ.png)

Issue 470 depends on  [Issue 462](https://github.com/Kotlin/kotlinx.coroutines/issues/462)  , so it is also not resolved:   ![](https://cdn-images-1.medium.com/max/800/0*rS3g5aXwyr1P1ez4.png)

Recommended solution for our case is to create our own dispatchers:
```
actual val defaultDispatcher: CoroutineContext
get() = IODispatcher

actual val uiDispatcher: CoroutineContext
get() = MainDispatcher

private object MainDispatcher: CoroutineDispatcher(){
    override fun dispatch(context: CoroutineContext, block: Runnable) {
        dispatch_async(dispatch_get_main_queue()) {
            try {
                block.run()
            }catch (err: Throwable) {
                throw err
            }
        }
    }
}

private object IODispatcher: CoroutineDispatcher(){
    override fun dispatch(context: CoroutineContext, block: Runnable) {
        dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT.toLong(),
0.toULong())) {
            try {
                block.run()
            }catch (err: Throwable) {
                throw err
            }
        }
    }
```
We have created MainDispatch with DispatchQueue.main and IODispatcher with DispatchQueue.global(). When we launch our code, we will get the same error.

The problem is, we cannot use dispatch_get_global_queue to dispatch our coroutines, because it is not bound to any particular thread in Kotlin/Native:   ![](https://cdn-images-1.medium.com/max/800/0*EPoIt46vcXRwDT_U.png)

Secondly, Kotlin/Native doesn’t allow to move any mutable objects between threads. Included the coroutines.

So we can try to use MainDispatcher for all our cases:
```
actual val ioDispatcher: CoroutineContext
get() = MainDispatcher

actual val uiDispatcher: CoroutineContext
get() = MainDispatcher


@ThreadLocal
private object MainDispatcher: CoroutineDispatcher(){
    override fun dispatch(context: CoroutineContext, block: Runnable) {
        dispatch_async(dispatch_get_main_queue()) {
            try {
                block.run().freeze()
            }catch (err: Throwable) {
                throw err
            }
        }
    }
```
But it is not enough. We also should  [freeze](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.native.concurrent/freeze.html)   our objects before sharing the between threads. So we need to use freeze() command in this case:   ![](https://cdn-images-1.medium.com/max/800/0*OcZIhmnh9SHlgiXE.png)

But if we try to perform freeze() on already frozen object, FreezingException will be thrown. For example, all singletones are frozen by default.

That’s why we should use @ThreadLocal annotation to share singletones and @SharedImmutable for global variables:
```
/**
 * Marks a top level property with a backing field or an object as thread local.
 * The object remains mutable and it is possible to change its state,
 * but every thread will have a distinct copy of this object,
 * so changes in one thread are not reflected in another.
 *
 * The annotation has effect only in Kotlin/Native platform.
 *
 * PLEASE NOTE THAT THIS ANNOTATION MAY GO AWAY IN UPCOMING RELEASES.
 */
@Target(AnnotationTarget.PROPERTY, AnnotationTarget.CLASS)
@Retention(AnnotationRetention.BINARY)
public actual annotation class ThreadLocal

/**
 * Marks a top level property with a backing field as immutable.
 * It is possible to share the value of such property between multiple threads, but it becomes deeply frozen,
 * so no changes can be made to its state or the state of objects it refers to.
 *
 * The annotation has effect only in Kotlin/Native platform.
 *
 * PLEASE NOTE THAT THIS ANNOTATION MAY GO AWAY IN UPCOMING RELEASES.
 */
@Target(AnnotationTarget.PROPERTY)
@Retention(AnnotationRetention.BINARY)
public actual annotation class SharedImmutable
```
We can simply use MainDispatcher for all our needs, when we use Ktor or another library that supports its own asynchronous processing. In common case for all long-running work we should use GlobalScope with the context of Dispatchers.Main/MainDispatcher:

```
//iOS
actual fun ktorScope(block: suspend () -> Unit) {
    GlobalScope.launch(MainDispatcher) { block() }
}

//Android
actual fun ktorScope(block: suspend () -> Unit) {
           GlobalScope.launch(Dispatchers.Main) { block() }
       }
```
So we can easily perform the switching between contexts to our service logic:
```
suspend fun loadMovies(callback:(MoviesList?)->Unit) {
       ktorScope {
            val url =
                "http://api.themoviedb.org/3/discover/movie?api_key=KEY&page=1&sort_by=popularity.desc"
            val result = networkService.loadData<MoviesList>(url)
            //performing some long-running to demonstrate that MainDispatcher 
            //is not enough without GlobalScope in this case
            delay(1000)
           withContext(uiDispatcher) {
               callback(result)
           }
        }
    }
```
We created the common scope for all code performing in same suspended function. So everything will work correctly. It is not the only way to implement coroutine based logic, you can organize it with any approach you prefer.

You can also use wrapping blocks to share code with DispatchQueue.global():
```
//You can specify any type of the response you need
actual fun callFreeze(callback: (Response)->Unit) {
    val block = {
      //Just the sample of shared code
        callback(Response("from ios").freeze())
    }
    block.freeze()
    dispatch_async {
        queue = dispath_get_global_queue(DISPATCH_QUEUE_PRIORITY_BACKGROUND.toLong, 
            0.toULong())
        block = block     
    }
}
```
Of course, you need to implement actual fun callFreeze(…) in androidMain, but you need just to put a callback into the completion block.

Finally, we got a completed application that works same way in both platforms:   ![](https://cdn-images-1.medium.com/max/800/0*sWW6gtdAexoxhjpC.png)

[Sample code](https://github.com/anioutkazharkova/movies_kmp)
One more sample:
[github.com/anioutkazharkova/kmp_news_sample](https://github.com/anioutkazharkova/kmp_news_sample)

[tproger.ru/articles/creating-an-app-for-kotlin-multiplatform](https://tproger.ru/articles/creating-an-app-for-kotlin-multiplatform/)
[github.com/JetBrains/kotlin-native](https://github.com/JetBrains/kotlin-native)
[github.com/JetBrains/kotlin-native/blob/master/IMMUTABILITY.md](https://github.com/JetBrains/kotlin-native/blob/master/IMMUTABILITY.md)
[github.com/Kotlin/kotlinx.coroutines/issues/462](https://github.com/Kotlin/kotlinx.coroutines/issues/462)
[helw.net/2020/04/16/multithreading-in-kotlin-multiplatform-apps](https://helw.net/2020/04/16/multithreading-in-kotlin-multiplatform-apps/)


------------------------------


Originally published at  * [https://habr.com*](https://habr.com/ru/post/533952/)     . *


By  [Anna Zharkova](https://medium.com/@anioutkazharkova)   on  [December 21, 2020](https://medium.com/p/1c20e6e36d4d)    .

[Canonical link](https://medium.com/@anioutkazharkova/kotlin-multiplatform-practical-multithreading-part-2-1c20e6e36d4d)

Exported from  [Medium](https://medium.com)   on January 20, 