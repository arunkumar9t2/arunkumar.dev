---
layout: post
title: 'Introducing Compass: Effective Paging with Realm and Jetpack Paging 3'
date: 2021-10-09 03:59 +0800
description: Compass provides a set of Kotlin types and extensions to make working with Realm mobile database easier.
categories: [Android, Kotlin]
image : /assets/images/compass.png
---

I like [Realm](https://realm.io) mobile database. I first started using `Realm` at a time when there were limited options for a reactive database - a feature common today with tools like [Room](https://developer.android.com/jetpack/androidx/releases/room) and [SqlDelight](https://github.com/cashapp/sqldelight/) (remember [SqlBrite](https://github.com/square/sqlbrite)?). With reactivity, `Realm` was pushing for [persistence as single source of truth](https://docs.mongodb.com/realm/sdk/android/quick-start-local/#complete-example) much earlier than the pattern caught on if I recall correctly. `Realm`'s reactivity is [fine grained](https://docs.mongodb.com/realm/sdk/android/examples/react-to-changes/#register-a-collection-change-listener) as well i.e can emit added, removed or modified changes on a collection without tools like [DiffUtil](https://developer.android.com/reference/androidx/recyclerview/widget/DiffUtil) and with correct usage it can integrate directly with [RecyclerView](https://developer.android.com/guide/topics/ui/layout/recyclerview). Apart from being reactive, it is an [object oriented database](https://docs.mongodb.com/realm/sdk/android/examples/define-a-realm-object-model/), so relations can be directly expressed as Java/Kotlin objects and has a capable [query](https://docs.mongodb.com/realm/sdk/android/fundamentals/query-engine/) system with support for aggregations and backlinks. 

However not all of these were smooth sailing in my opinion, particularly **paging, lifecycle and threading**. `Realm`'s live object model i.e at any point in time, we interact with the live view of the underlying database and objects are brought into memory only when they are read like `Object.getPropery()` (lazy by default). For paging, `Realm` initially recommended that it can be fast enough to read objects inside a `RecyclerView.Adapter.getItem()` call (this pattern is no longer recommended). I disagree with it since storage performance is highly variable (a background Play Store update can bring storage performance to its knees) and chances of ANR are high. 

[Threading](https://docs.mongodb.com/realm/sdk/android/advanced-guides/threading/) is not straight forward and had few rules that need to be followed. 
* `Realm` instances and the results `RealmResults` should not cross thread boundaries and each `Realm` instance carried a lifecycle with it should call `close()` when done.
* `Realm` objects can be observed only on a thread that has Android's [Looper](https://developer.android.com/reference/android/os/Looper) [prepared](https://developer.android.com/reference/android/os/Looper#prepare()) on them.
 
Because of these rules, writing `Realm` database code carried a bit of ceremony around it especially considering Android's already extensive lifecyle expectations.

### Motivation

Having discovered couple of patterns that can make working with `Realm` easier especially around threading, lifecycle and paging, I took the opportunity to use Kotlin's capable type system to provide safe defaults with limited options for developer error when using `Realm`. In the remainder of the article, I would like to talk about paging in particular with Jetpack Paging 3 and how to integrate `Realm` with it by using [Compass](https://github.com/arunkumar9t2/compass).

`compass` is available on `mavenCentral`:

```groovy

dependencies {
    implementation "dev.arunkumar.compass:compass:1.0.0"
    // Paging integration
    implementation "dev.arunkumar.compass:compass-paging:1.0.0"
}
```

### Compass

`Compass` has two main components that provide the core `compass` module with extensions for threading and lifecycle and `compass-paging` for integration with [Jetpack Paging 3](https://developer.android.com/topic/libraries/architecture/paging/v3-overview). To observe a `RealmQuery` with `compass` we can use the `asPagingItems` extension function as shown below.

```kotlin
val pagingFlow: Flow<PagingData<Person>> = RealmQuery { where<Person>().sort(Person.NAME) }
                                            .asPagingItems()
```

Since `asPagingItems()` returns `Flow<PagingData<T>>` it integrates well with
* `Paging 3` [stream of data](https://developer.android.com/topic/libraries/architecture/paging/v3-paged-data) support, particularly `cachedIn(coroutineScope)`. 
* [Jetpack Compose](https://developer.android.com/jetpack/compose)'s `LazyColumn` support via [collectAsLazyPagingItems()](https://developer.android.com/reference/kotlin/androidx/paging/compose/package-summary#(kotlinx.coroutines.flow.Flow).collectAsLazyPagingItems()).

The extension function internally manages threading, lifecycle of `Realm`s and confirms to typical expectations of a `Flow<T>`:

1. Returned objects can be safely passed around threads.
2. Automatic closing of any internal resources when `Flow` collection is stopped.

Let's unpack one by one on how `compass` achieves this.

### Implementation

#### Queries

Consider a typical `Realm` code with the following

```kotlin
val realm = Realm.getDefaultInstance() // 1

val persons = realm.where<Person>().findAll() // 2

persons.addChangeListener{ } // 3

realm.close() // 4

```
This looks straight-forward but as soon as threading is introduced, couple of challenges arise. Namely, `1`,`2`,`3`,`4` should be on the same thread. Since `2` is a database operation, our instinct should be to move this entire operation to background thread, and we would be correct. However if we want to observe any changes in `3` it would crash since the background thread most likely would not have `Looper` by default. The official solution to this problem is to explicitly use `*Async` APIs like `findAllAsync()` etc. This works for most cases but still has room for developer errors due to lack of checks especially in changes observation due to ties to Android Looper.

##### Deferred Execution

This problem can be solved by moving the construction of the query and observation both to usage site (background thread). Compass provides `RealmQuery {}` construction function that can be used to construct `RealmQueryBuilder<T>` which is a `typealias` for `Realm.() -> RealmQuery<T>`. The design is inspired by lazy APIs like Kotlin sequences/flows where construction happens lazily and only invoked when certain terminal operators are called.

In this case, the terminal operator can enforce threading rules as needed without any ceremony needed inside the construction function. For example:

```kotlin

val personQuery = RealmQuery { where<Person>() }

personQuery.getAll() // Acquire an instance of `Realm`, run the query, return results and close Realm.
personQuery.asFlow() // Observable variant
```

#### RealmExecutor/RealmDispatcher

`compass` provides dedicated constructs like [RealmDispatcher](https://arunkumar9t2.github.io/compass/compass/dev.arunkumar.compass.thread/-realm-dispatcher/index.html) that automatically handles basic threading expectations of `Realm`. The requirement is to ensure `Looper` is available and it is stopped when work is done. This is done via Android OS's [HandlerThread](https://developer.android.com/reference/android/os/HandlerThread) which is the official way to manage the `Looper` correctly. Converting `HandlerThread` to a dedicated `Executor` instance will allow us to use variety of extensions that Kotlin offers like `asCoroutineDispatcher()`. [HandlerExecutor](https://arunkumar9t2.github.io/compass/compass/dev.arunkumar.compass.thread/-handler-executor/index.html):

```kotlin
public class HandlerExecutor(private val tag: String? = null) : CloseableExecutor {

  private val handlerThread by lazy {
    HandlerThread(
      tag ?: this::class.java.simpleName + hashCode(),
      Process.THREAD_PRIORITY_BACKGROUND
    ).apply { start() }
  }

  private val handler by lazy { Handler(handlerThread.looper) }

  override fun execute(command: Runnable) {
    if (Looper.myLooper() == handler.looper) {
      command.run()
    } else {
      handler.post(command)
    }
  }

  override fun close() {
    handlerThread.quitSafely()
  }
}
```
Using `RealmDispatcher`, the following is valid:

```kotlin
withContext(RealmDispatcher()) {
  Realm { realm -> // Acquire default Realm with `Realm {}`
    val persons = realm.where<Person>().findAll()    

    val realmChangeListener = RealmChangeListener<RealmResults<Person>> {
      println("Change liseneter called")
    }

    persons.addChangeListener(realmChangeListener) // Safe to add    
    
    // Make a transaction
    realm.transact { // this: Realm
      copyToRealm(Person())
    }
    
    delay(500)  // Wait till change listener is triggered
  } // Acquired Realm automatically closed
}
```

Note that `RealmDispatcher` should be closed when no longer used to release resources. For automatic lifecycle handling via [Flow](https://kotlinlang.org/docs/flow.html), see below.

#### Streams via Flow

`compass` builds on top of the lazy query API to provide observable functions like `asFlow()` which confirms to basic threading expectations of a `Flow`. 

* Returned objects can be passed to different threads.
* Handles `Realm` lifecycle until `Flow` collection is stopped.

```kotlin
val personsFlow = RealmQuery { where<Person>() }.asFlow()
```
Internally `asFlow` creates a dedicated `RealmDispatcher` to run the queries and observe changes. The created dispatcher is automatically closed and recreated when collection stops/restarted all established through use of `callbackFlow` and `awaitClose`.

For ensuring `Realm` objects can be passed around threads, `compass` copies the object to memory using `Realm.copyFromRealm()`. Copying objects detaches it from `realm` and is no longer a live updating object.

##### Reading subset of data

Copying large objects from `Realm` can be expensive in terms of memory, to read only subset of results to memory we can use `asFlow()` overload that takes a [transform](https://arunkumar9t2.github.io/compass/compass/dev.arunkumar.compass/index.html#-1272439111/Classlikes/1670650899) function.

```kotlin
data class PersonName(val name: String)

val personNames = RealmQuery { where<Person>() }.asFlow { PersonName(it.name) }
```
In the above example, since `Realm` is lazy by default, only `name` is brought into memory from eash `Person` instance. Conceptually in SQL terms, this is like querying only one column from a table.

#### Paging

So far we have explored how `compass` handles common patterns around threading and lifecycle. All those concepts come together in `compass`'s paging integration.

##### RealmResults is like a live cursor

`compass` takes advantage of `Realm`s unique lazy loading feature to implement paging. Jetpack Paging 1 and 2 provided APIs to implement different types of paging [sources](https://developer.android.com/topic/libraries/architecture/paging/data#choose-data-source-type), this can be key based, position based and even dependant key based. Realm's paging support can be implemented in variety of ways, `compass` implements it using `PositionalDataSource`. It works under the premise that `RealmResults<T>` returned by `RealmQuery` is a live cursor. For example,

```kotlin
val persons = realm.where<Person>().findAll()
```
Here `persons` can contain 1 million entries but not all are brought into memory, they are read only when `persons.get(index)` is called, if one were to bind this to a `RecyclerView` only items in the view port would be read (nifty!). This feature alone would satisfy [PositionalDataSource](https://developer.android.com/reference/android/arch/paging/PositionalDataSource) expectations that the implementation should be able to read data at any arbitary position. 

Since paging can involve complicated threading, `compass`'s default implementation detaches the object from `Realm` using `Realm.copyFromRealm`, this makes it easy to apply any further [transformations](https://developer.android.com/topic/libraries/architecture/paging/v3-transform) as needed without concern on threading. 

As mentioned earlier, `copyFromRealm` is expensive for large nested objects and it is advised to take advantage of `Realm`'s lazy loading to bring only needed data to memory, this is done via [RealmModelTransform](https://arunkumar9t2.github.io/compass/compass/dev.arunkumar.compass/index.html#-1272439111/Classlikes/1670650899) API.

```kotlin
public typealias RealmModelTransform<T, R> = Realm.(realmModel: T) -> R
```
`compass` provides `ReamlCopyTransform` which does simply copying as shown below.
```kotlin
public fun <T : RealmModel, R> RealmCopyTransform(): RealmModelTransform<T, R> {
  return { model -> copyFromRealm(model) as R }
}
```
For customized reading of data, we can use the `asPagingItems` overload which takes a `transform` function.

```kotlin
val pagedPersonNames = RealmQuery { where<Person>() }.asPagingItems { it.name }
```

##### Observability and threading

For threading, `compass` uses `RealmDispatcher` to run queries and manage observability in a predictable way. Paging 3 expects that whenever there is data change, the corresponding `DataSource` implementation should [invalidate()](https://developer.android.com/topic/libraries/architecture/paging/data#notify-data-invalid) itself and new instance should be created. With `Realm` this can be easily accomplished by using change listeners:

```kotlin
// this: RealmTiledDataSource
private val realmChangeListener = { _: RealmResults<T> -> invalidate() }

private val realmResults by lazy {
    realmQuery.findAll().apply {
      addChangeListener(realmChangeListener)
    }
}

init {
  addInvalidatedCallback {
    if (realmResults.isValid) {
      realmResults.removeChangeListener(realmChangeListener)
    }
    realm.close()
  }
}
```

> See [RealmTiledDataSource](https://arunkumar9t2.github.io/compass/compass-paging/dev.arunkumar.compass.paging/-realm-tiled-data-source/index.html)

**Note**: `PositionalDataSource` is technincally deprecated, however similar to `Room`, `compass` relies on Paging 2 to 3 [migration support](https://developer.android.com/topic/libraries/architecture/paging/v3-migration#migrate) specifically `asPagingSourceFactory` to support Paging 3. Although technically it should be possible to implement Paging 3's `PagingSource` with the same patterns described in this article.

##### ViewModel

Jetpack `ViewModel` integration is straight-forward as shown below:

```kotlin
class MyViewModel: ViewModel() {

    val results = RealmQuery { where<Task>() }.asPagingItems().cachedIn(viewModelScope)
}
```

The `Flow` returned by `asPagingItems()` can be safely used for [transformations](https://developer.android.com/topic/libraries/architecture/paging/v3-transform#transform-data-stream), [seperators](https://developer.android.com/topic/libraries/architecture/paging/v3-transform#handle-separators-ui) and [caching](https://developer.android.com/topic/libraries/architecture/paging/v3-transform#avoid-duplicate). Although supported, for [converting to UI model](https://developer.android.com/topic/libraries/architecture/paging/v3-transform#convert-ui-model) prefer using `asPagingItems { /* convert */ }` as it is more efficient.

#### Sample

A basic paging implementation with Jetpack Compose with options to sort shown below:

<video class="phone" controls>
  <source src="https://github.com/arunkumar9t2/compass/blob/main/docs/videos/compass-sample.webm?raw=true" type="video/webm">
  Your browser does not support the video tag.
</video>


#### Assumptions

For simpler APIs, `compass` makes tradeoffs in managing `Realm` objects. It prefers to open short lived `Realm` instances using `Realm.getDefaultInstance()` instead of scoping `Realm`s to UI controllers. 

The official [guide](https://docs.mongodb.com/realm/sdk/android/fundamentals/realms/#the-realm-lifecycle) points that 

> If the realm is already open on a different thread within the same process, opening the realm is less expensive, but still nontrivial.

So far the pattern has worked well, but if your use case with different `Realm` schema has any troubles, I would love to hear in the comments.


#### Comparison to official APIs

Problems outlined in this article around observability has in part been addressed officially by the `Realm` team, namely with the introduction of [Freezing](https://docs.mongodb.com/realm/sdk/android/advanced-guides/threading/#frozen-objects) and `toFlow` / `toFlowable` APIs. 

##### Freezing

Freezing done by calling `realm.freeze()`, creates immutable views that do not have threading limitations.

```kotlin
val frogs = realm.where<Frog>().findAll().freeze()
```
Although freezing ticks the most boxes of concern, namely being able to move objects across thread:

> Freezing creates an immutable view of a specific object, collection, or realm that still exists on disk and does not need to be deeply copied when passed around to other threads. You can freely share a frozen object across threads without concern for thread issues.

The returned `frozen` objects still has ties to `Realm` and carries lifecycle risk with it:

> Frozen objects remain valid for as long as the realm that spawned them stays open. Avoid closing realms that contain frozen objects until all threads are done working with those frozen objects.

`compass`s `asFlow` APIs produce completely detached objects + the option of reading subset of data from each `RealmObject`. 

##### Streams via toFlow

Official recommendation around streams API is to use `toFlow` which internally uses `Freeze` to address threading concerns. However still lifecycle needs to be managed on `realm` seperately as shown below.

```kotlin
val realm = Realm.getDefaultInstance()

realm.where<Task>().findAllAsync()
  .toFlow()
  .flatMapLatest { frozenTasks ->
    frozenTasks.asFlow()
  }.map { it.id }

// realm?
```

In comparison, `compass`'s API `RealmQuery { where<Task>() }.asFlow { it.id }` automatically manages internal `Realm` instances and runs whole construction and execution in a background thread.

### Summary

`compass` provides simpler APIs and types to make working with `Realm` easier. Abstractions around threading, lifecycle and paging with the use of Kotlin's capable type system helps in avoiding common pitfalls. 

#### Possible improvements

`compass`'s paging, currently does not take advantage of fine-grained notifications from `Realm` and still relies on `DiffUtil` / Composition Keys to perform updates. Through some carefull structuring it should be possible to implement changes without needing to manually calculate changeset. 

> Compass is available here: <https://github.com/arunkumar9t2/compass>

Any feedback greatly appreciated!

-- Arun
