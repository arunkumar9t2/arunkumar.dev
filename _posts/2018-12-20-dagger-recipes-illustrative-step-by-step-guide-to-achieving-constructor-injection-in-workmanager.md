---
layout: post
title: 'Dagger Recipes: Illustrative step by step guide to achieving constructor injection
  in WorkManager'
description : Step by step guide to configure your Android project to enable constructor injecton in Jetpack's WorkManager.
categories: [Android, Dagger]
date: 2018-12-20 22:14 +0530
---

Recently, while working on an app, I decided to try out Jetpack's WorkManager for all background jobs (it's pretty great :+1:). The app already uses Dagger 2 for DI and I wanted to carry that over to WorkManager as well when executing jobs.


You might already know, Android system components are not constructor injectable prior to Pie and currently available solutions use `Members injection` and there are variatons in it.

* The first is accessing your injector component from `Application` class and then calling `component.inject(this)` which would do a member injection. This approach breaks DI best pratices - the class getting injected should not know its injector and it makes it difficult to access this `Activity` in a modular project.
* The second is using `dagger.android` and `Multibinding`s to implement a factory + service locator apporach to get the dependencies runtime. Injectable android components only needed to do `AndroidInjection.inject(this)`. I initially avoided this approach (lack of good documentation and unknowns), but have learnt to live with it especially in a modular project and this is the goto approach I take.

### Why constructor injection?

One pratice I follow after setting up Dagger is everything apart from Android components should be constructor injected. This means once the class is instantiated the dependencies are already resolved, there is no waiting for a lifecycle event to do member injection and you can reduce stateful code by making the dependency immutable i.e `val`. This approach has few benefits

* Reduced stateful code in resolving dependencies. 
* Improves readability - exactly know what dependencies the class wants.
* No dependency on the injector itself - constuctor injected class can exist in different components. In a isolated module for example, a `TestComponent` can be used to provide dependecies for testing and during usage, the dependencies can provided by the app component. 

#### About this article

With above in mind, in this article, I will *walk you through my thought process* of how I acheived construction injection in `WorkManager`. **I believe knowing to walk through dagger errors and then configuring it to successfull compile using various options dagger provides is a helpful skill to have.** To do this, I will start with a **failing build** and then step through ways to help dagger help us. 

### What we will build

We will have a simple app with a single `HelloWorldWorker`. The app will use `Repository` pattern in the data layer and Dagger is setup to inject repositories. The `HelloWorldWorker` has a dependency on `WordsRepository`. We will concentrate on injecting `HelloWorldWorker`.

The initial failing setup can be found in this [commit](https://github.com/arunkumar9t2/dagger-workmanager/tree/bd5f6787261f1e0cc3fa4c17900f1063db62c916).

Concerned `Worker` subclass what we would like to inject:

```kotlin
class HelloWorldWorker
@Inject
constructor(
    private val application: Application,
    workerParameters: WorkerParameters,
    private val wordsRepository: WordsRepository // App dependency
) : RxWorker(application, workerParameters) {

    override fun createWork(): Single<Result> {
        return wordsRepository.sayHelloWorld()
            .doOnSuccess { message ->
                // Toasts are bad, don't use it.
                Toast.makeText(application, message, Toast.LENGTH_SHORT).show()
            }.map { Result.success() }
            .onErrorReturnItem(Result.failure())
    }
}
```

In the remainder of the article, we will see how this class can be initialized and make it compatible with `WorkManager`.

### Missing pieces

`HelloWorldWorker`'s constructor is marked with `@Inject`, we are letting Dagger know that this `Worker` participates in DI. Since this class is supposed to be initialized by `WorkManager`, we don't actually request an instance from Dagger but instead `WorkManager` does. When we run the app and try to start the `Worker` we get:

```java
Could not instantiate in.arunkumarsampath.dagger.workmanager.jobs.HelloWorldWorker
    java.lang.NoSuchMethodException: <init> [class android.content.Context, class androidx.work.WorkerParameters]
        at java.lang.Class.getConstructor0(Class.java:2320)
        at java.lang.Class.getDeclaredConstructor(Class.java:2166)
        at androidx.work.WorkerFactory.createWorkerWithDefaultFallback(WorkerFactory.java:91)
        at androidx.work.impl.WorkerWrapper.runWorker(WorkerWrapper.java:191)
        at androidx.work.impl.WorkerWrapper.run(WorkerWrapper.java:125)
        at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1162)
        at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:636)
        at java.lang.Thread.run(Thread.java:764)
```

WorkManager here tries to instantiate `HelloWorldWorker` with two params `context` and `workerParameters`, but we do have a third one `wordsRepository`. Before we try to fix the params issue, let's see if the graph is complete. Dagger, while processing recursively tries to satisfy all dependencies (the ones you *provide* - more on this later). To force Dagger to check if `HelloWorldWorker`'s dependencies can be resolved, let's request it directly in `HomeActivity`.

```kotlin
class HomeActivity : DaggerAppCompatActivity() {

    @Inject
    lateinit var helloWorldWorker: HelloWorldWorker

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setupUi()
    }
}
```

Now when we try to build the app, we get a **compile** time error:

```java
[Dagger/MissingBinding] androidx.work.WorkerParameters cannot be provided without an @Inject constructor or an @Provides-annotated method.
public abstract interface AppComponent extends dagger.android.AndroidInjector<in.arunkumarsampath.dagger.workmanager.WorkManagerApp> {
                ^
      androidx.work.WorkerParameters is injected at
          in.arunkumarsampath.dagger.workmanager.jobs.HelloWorldWorker(…, workerParameters, …)
      in.arunkumarsampath.dagger.workmanager.jobs.HelloWorldWorker is injected at
          in.arunkumarsampath.dagger.workmanager.home.HomeActivity.helloWorldWorker
      ... // left for brevity
```

Looks like Dagger cannot find `WorkerParameters` instance but it did not complain about `application` or `wordsRepository`. The remaining missing piece is how to **expose** `WorkerParameters` at compile time and then letting `WorkerManager` use it? Let's visualize this below.

{% include images_center.html url="/assets/images/dagger-workmanager-1.png" caption="Missing binding for WorkerParameters" %}

### WorkerFactory

Since one of the recent releases, Google added something called `WorkerFactory` whose implementation is used to instantiate `Worker`s. It is possible to provide a custom isntance via `WorkManager.initialize`. `WorkerFactory` has `createWorker` nethod which has all the information we need.

```java
public abstract @Nullable ListenableWorker createWorker(
            @NonNull Context appContext,
            @NonNull String workerClassName,
            @NonNull WorkerParameters workerParameters);
```

Before we discuss about exposing `workerParameters` binding, let's get the setting custom `WorkerFactory` instance out of the way. WorkManager uses a `ContentProvider` by default to intialize itself (that's how it gets `application` context). We can disable this by using `tools:remove="node"` in AndroidManifest.

#### Setting custom WorkerFactory instances

* Remove content provider initializer by adding the below in `AndroidManifest.xml`

```xml
 <provider android:name="androidx.work.impl.WorkManagerInitializer"
            android:authorities="${applicationId}.workmanager-init"
            android:exported="false"
            tools:node="remove" />
```

* Initialize WorkManager with custom `WorkerFactory` in `Application.onCreate`.

```kotlin
 WorkManager.initialize(
                    application,
                    Configuration.Builder().run {
                        setWorkerFactory(workerFactory) // Pass custom worker factory instance
                        build()
                    }
            )
```

Now writing a custom `WorkerFactory` and exposing `workerParameter`s binding from there is pending, visualizing below.

{% include images_center.html url="/assets/images/dagger-workmanager-2.png" caption="How to expose WorkerParameters received in createWorker as Dagger Binding?" %}

### Exposing WorkerParameters

#### Dagger bindings refresher
First, I'd like to do a refresher on various options we have to `expose` bindings to Dagger. For classes we don't own the constructor, we have to guide Dagger ourselves and that is where `@Module` comes in. We typically write methods in `@Module` annotated classes and return the `Type` we want exposed. This way when this `Type` is requested, Dagger uses the method we wrote. In order to not repeat information, I suggest reading this excellent [article](https://medium.com/@Zhuinden/that-missing-guide-how-to-use-dagger2-ef116fbea97) by Gabor Varadi.

#### Enter Subcomponents

Above we notice that dagger is able to find `Application` instance when we injected `HelloWorldWorker`. If we look at the [source](https://github.com/arunkumar9t2/dagger-workmanager/blob/bd5f6787261f1e0cc3fa4c17900f1063db62c916/app/src/main/java/in/arunkumarsampath/dagger/workmanager/di/app/AppComponent.kt#L29), while building our root component (`AppComponent`), it is possible to pass in parameters using `@BindsInstance`. `application` is the parameter in this case. One crucial detail is that, **types passed in with `@BindsInstance` are exposed as binding to the entire graph**.

Since `Application` is known only at runtime, we use `@BindsInstance` to pass the created instance which then becomes available to the whole graph. During compile time validation, the **checks pass since `@BindsInstance` is considered as binding.**

Example:
We already request `application` instance in [DefaultWordsRepository](https://github.com/arunkumar9t2/dagger-workmanager/blob/bd5f6787261f1e0cc3fa4c17900f1063db62c916/app/src/main/java/in/arunkumarsampath/dagger/workmanager/data/words/DefaultWordsRepository.kt). This request is satisfied by `@BindInstance` in the `AppComponent`. It looks like for runtime known types `@BindsInstance` is a good candidate for exposing that type. The `workerParameters` is known only when `createWorker` is called, if we create a component there with `@BindsInstance` of type `WorkerParameters` we should be able to expose it to Dagger. 

`Subcomponents` are great candidate for this. It automatically inherits all the bindings from parent component and makes it available within the subcomponent. Let's update the visualization with this info about how `application` and `wordsRepository` are satisfied.

{% include images_center.html url="/assets/images/dagger-workmanager-3.png"%}

Now in order to provide `WorkerParameters`, we can create a `SubComponent` at `createWorker`. Now Dagger should be able to resolve `workerParameter`s for `HelloWorldWorker` at the constructor **as long as `HelloWorldWorker` and the exposed `WorkerParameters` are in the same component**. We will come back to this later, now we will visualize updated approach.

{% include images_center.html url="/assets/images/dagger-workmanager-4.png" caption="Using a intermediate subcomponent to expose WorkerParameters"%}

#### Writing WorkerSubcomponent

As discussed, our `WorkerSubcomponent` takes a `WorkerParameters` param and it is pretty straight forward to write.

```kotlin
@Subcomponent
interface WorkerSubcomponent {

    @Subcomponent.Builder
    interface Builder {
        @BindsInstance
        fun workerParameters(param: WorkerParameters) : Builder

        fun build(): WorkerSubcomponent
    }
}
```

##### Connecting AppComponent and WorkerSubcomponent

In order to let `AppComponent` know the `WorkerSubComponent`, it is sufficient to write an abstract method that returns `WorkSubComponent`'s `Builder` in `AppComponent`. This is important since **without this `WorkerSubComponent` won't inherit `Application` or `WordsRepository` etc**. 

Udpated AppComponent:

```kotlin
@Singleton
@Component(
    modules = [
        AndroidSupportInjectionModule::class,
        HomeBuilder::class,
        DataModule::class
    ]
)
interface AppComponent : AndroidInjector<WorkManagerApp> {
    
    // Establish WorkerSubcomponent as subcomponent
    fun workerSubcomponentBuilder(): WorkerSubcomponent.Builder

    @Component.Builder
    interface Builder {
        fun build(): AppComponent
        @BindsInstance
        fun application(application: Application): Builder
    }
}
```

Let us visualize the update binding graph now.

{% include images_center.html url="/assets/images/dagger-workmanager-5.png" caption="Final binding graph to HelloWorldWorker's dependencies"%}


### Connecting the dots

Now since we have a approach to provide a full graph, let's look into how to use in and actually automate instantiating instances and let `WorkManager` use them.

#### DaggerWokerFactory - custom worker factory

We will start with writing our DaggerWorkerFactory which has following resonsibilities

* With `workerParameters` create a subcomponent and complete binding graph for `Worker` instances.
* With `workerClassName` instantiate the `Worker` instance and return it for `WorkerManager` to use. 

##### Creating the workerSubcomponent

Since we already declared `AppComponent.workerSubcomponentBuilder()`, we can directly request an instance in constructor and use it to create `WorkerSubComponent`. Note that the Dagger maintains the internal implementation of the builders. Covering that is beyond the scope of the article.

```kotlin
@Singleton
class DaggerWorkerFactory
@Inject
constructor(private val workerSubcomponent: WorkerSubcomponent.Builder) : WorkerFactory() {

    override fun createWorker(
        appContext: Context,
        workerClassName: String,
        workerParameters: WorkerParameters
    ): ListenableWorker? {
        workerSubcomponent
            .workerParameters(workerParameters)
            .build()
        // How to create workerClassName instance?
    }
}
```

After creating the subcomponent, creating the instance with `workerClassName` still remains unknown which is covered in the next section.

##### Creating instances with class name

Dagger already has a mechanism to request dependencies by their type. Within Dagger, it is done using `Provider<Type>`, by calling `provider.get()` we can get an instance of `Type`. To create Worker instance with class name, all we need is `Provider<HelloWorldWorker>`. 

Note: We can't get a `Provider<T>` without a complete graph for `T`. Which means for a incomplete graph, compilaton fails and `Provider<T>` is not generated at all.

In order to write it in a generic way i.e using this `DaggerWorkerFactory` for instantiating all available `Workers` in the app, we will have to do it without hardcoding `HelloWorldWorker` type.

##### Multibindings

From [Google docs](https://google.github.io/dagger/multibindings.html)

> Dagger allows you to bind several objects into a collection even when the objects are bound in different modules using multibindings. Dagger assembles the collection so that application code can inject it without depending directly on the individual bindings.

This seems like a perfect fit for our use case which is getting the provider by key from a map. Suppose if we have s `Map<Class<out RxWorker>, Provider<RxWorker>>` instance, we can use the `workerClassName` from `createWorker()` and get the `Provider<HelloWorldWorker>` and return the instance.

In order to get `Map<Class<out RxWorker>, Provider<RxWorker>>`, we have to configure multibinding. We can do that by using `@Binds` and `@IntoMap` annotations. We also need to specify the key type (`Class<out RxWorker>`) for the `Map` which can be done using a custom annotation as shown below.

```kotlin
@Target(AnnotationTarget.FUNCTION, AnnotationTarget.PROPERTY_GETTER, AnnotationTarget.PROPERTY_SETTER)
@Retention()
@MapKey
annotation class WorkManagerKey(val value: KClass<out RxWorker>)
```

In order to let dagger generate this map instance, we have to declare the multibinding generating code in any `@Module`. My personal preference to add it as a `Builder` class inside the class that participates in map generation, in this case `HelloWorldWorker`.

Updated `HelloWorldWorker`:

```kotlin
class HelloWorldWorker
@Inject
constructor(
    private val application: Application,
    workerParameters: WorkerParameters,
    private val wordsRepository: WordsRepository
) : RxWorker(application, workerParameters) {

    override fun createWork(): Single<Result> {
        return wordsRepository.sayHelloWorld()
            .doOnSuccess { message ->
                // Toasts are bad, don't use it.
                Toast.makeText(application, message, Toast.LENGTH_SHORT).show()
            }.map { Result.success() }
            .onErrorReturnItem(Result.failure())
    }

    @Module
    abstract class Builder {
        @Binds
        @IntoMap
        @WorkerKey(HelloWorldWorker::class)
        abstract fun bindHelloWorldWorker(worker: HelloWorldWorker): RxWorker
    }
}
```

If you remember, in order for binding graph to be complete, the `WorkerParameters` exposed and generated map bindings should be in the same component. Since `WorkerSubComponent` is responsible for exposing `WorkerParameters` we will install the multibinding module to `WorkerSubComponent` as follows.

```kotlin
@Subcomponent(modules = [HelloWorldWorker.Builder::class])
interface WorkerSubcomponent {

    fun workers(): Map<Class<out RxWorker>, Provider<RxWorker>>

    @Subcomponent.Builder
    interface Builder {
        @BindsInstance
        fun workerParameters(param: WorkerParameters): Builder

        fun build(): WorkerSubcomponent
    }
}
```

As noted above, for convenience we expose a method to get the binding for `Map<Class<out RxWorker>, Provider<RxWorker>>` with `fun workers()`. Connecting everything the `DaggerWorkerFactory` looks like below.

```kotlin
@Singleton
class DaggerWorkerFactory
@Inject
constructor(private val workerSubcomponent: WorkerSubcomponent.Builder) : WorkerFactory() {

    override fun createWorker(
        appContext: Context,
        workerClassName: String,
        workerParameters: WorkerParameters
    ) = workerSubcomponent
        .workerParameters(workerParameters)
        .build().run {
            createWorker(workerClassName, workers())
        }

    private fun createWorker(
        workerClassName: String,
        workers: Map<Class<out RxWorker>, Provider<RxWorker>>
    ): ListenableWorker? = try {
        val workerClass = Class.forName(workerClassName).asSubclass(RxWorker::class.java)

        var provider = workers[workerClass]
        if (provider == null) {
            for ((key, value) in workers) {
                if (workerClass.isAssignableFrom(key)) {
                    provider = value
                    break
                }
            }
        }
        if (provider == null) {
            throw IllegalArgumentException("Missing binding for $workerClassName")
        }

        provider.get()
    } catch (e: Exception) {
        throw RuntimeException(e)
    }
}
```

With the given `workerClassName` we find a appropriate `Provider<T>` and get an instance and return it to `WorkManager`.

### Results

The changes for above sections are done in this [commit](https://github.com/arunkumar9t2/dagger-workmanager/commit/778a40726dc0d4c528f5ea550bbd81ef262e1a1a). Debugging the `HelloWorldWorker`, we can see that the injection has happened.

{% include images_center.html url="/assets/images/dagger-workmanager-6.png" caption="Constructor injection in Worker instance"%}


### Summary

In this article, we discussed a way to configure dagger for advanced cases where a dependency instance can be accessed only during runtime. In `WorkManager`, we get to know about `WorkerParameter` only at the time of initialization by `WorkerFactory`. In order to handle this case, we break down our root component to sub components and pass in the required type (workParameters) as parameter to a subcomponent. Then we use `Multibindings` to instantiate the instances. The implemented solution is a generic one and as long as all `Worker`s have a multibinding generator, the injection will work for all available `Workers`. Since the initialization happens in `onCreate` of the application, even when the `WorkManager` wakes up the application to do jobs, the custom factory we provided would be set.


### Misc

Thanks for following through till the end of the article. Liked what you read? I would be glad to receive feedback or criticism if any. If you do have any review comments/feedback please reach out to me or express in the comments below. I plan to write more about practical uses of Dagger in this Recipes series.