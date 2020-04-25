---
layout: post
title: Dagger SPI - Building custom validations for Dependency Graphs
date: 2020-04-25 08:18 +0800
categories: [Android, Dagger, Library]
image : /assets/images/scabbard-banner.png
---
In one of recent Dagger versions, Google added support for processing internal Dagger dependency graph information as part of the `dagger-spi` artifact (also read [SPI vs API](https://stackoverflow.com/questions/2954372/difference-between-spi-and-api)). According to the [docs](https://Dagger.dev/dev-guide/spi), it allows us access to the same model information that Dagger internally uses and lets us add few functionality on top of Dagger's compiler - like a plugin. Recently I used this functionality to [build a tool](https://arunkumar.dev/introducing-scabbard-a-tool-to-visualize-dagger-2-dependency-graphs/) that visualizes how the overall dependency graph of your project is structured. In this article, I plan to discuss one other functionality of the SPI artifact (validations) and explain how to consume this SPI from scratch.

### What we will build?

A compile time validator for Dagger declarations that can 

* Validate a Dagger binding according to project specific needs.
* Have customized behavior based on options received from Java compiler options.
* Provide rich formatted error messages similar to Dagger.
* Minor guidance on how to organize these validations for scaling. 

### Why custom validations?

While Dagger is excellent at compile time static validations and preventing misconfigured DI graph from crashing your application, it does not have any knowledge of the consuming framework's functional needs. At its core, it only sees types we declare and don't know anything about framework requirements - which is good. `dagger-android` and upcoming `dagger.hilt` are exceptions but the point still stands because they are built on top of Dagger core. Because of this, there is nothing stopping us from declaring bindings that can negatively affect your application stability and we usually realy on our expertise to manually vet those declarations. If we somehow are able to connect these framework specific functionalities to Dagger's already existing validations then we can prevent misuse.

#### Various Faces of Android's Context

If you are an Android developer today, you probably would have stumbled upon this [question](https://stackoverflow.com/questions/3572463/what-is-context-on-android). Each of the `ContextWrapper` implementations serve a different purpose. For example, `Application`, `Service` or `Activity` etc. It is common knowledge to excercise caution when using `Activity` instance - holding it too long outside of the active scope will easily leak memory and usual solution is to prefer `Application` instance. But then again you lose all the configuration and theme data if you use `Application`. Given all the specifics, all the below declarations in Dagger are valid:

```kotlin
@Binds
fun bindApplication(context: Application): Context

@Binds
fun bindActivity(context: Activity): Context

@Binds
fun bindService(context: Service): Context
```

With these bindings, it is possible to inject short lived implementation like `Activity` into a larger scoped object causing memory leak. I personally prefer to avoid `Context` bindings altogether and [bind implementation](https://github.com/arunkumar9t2/scabbard/blob/449be3e48bc465daaf09992475de3ae16ff556ef/samples/android-kotlin/src/main/java/dev/arunkumar/scabbard/di/AppComponent.kt#L36) specifically. Using Dagger `@Qualifier` is also another approach but **it is not enforced** by Dagger.

In this article, let's build a custom validation for `android.content.Context` that fails the build if it detects a `Context` binding without any `@Qualifier`.


### Trident

Continuing the knife related naming thing going on with Dagger, for this project I chose **Trident**, a special form of late middle ages [Dagger](https://en.wikipedia.org/wiki/Parrying_dagger#Trident_dagger) designed for defending or parrying. I saw the word "defend", so I am rolling with it for validations (heh).

### BindingGraphPlugin

Before jumping to implementation details, I think it helps to talk a bit about specifics of the SPI first. [`BindingGraphPlugin`](https://github.com/google/dagger/blob/master/java/dagger/spi/BindingGraphPlugin.java) is our single source of extension where the `visitGraph(bindingGraph: BindingGraph, diagnosticReporter: DiagnosticReporter)` will be called for every `@Component`. From here, we can utilize the graph information available in `BindingGraph` and then report errors using `DiagnosticReporter`. In order to let Dagger discover our custom plugin, we have to make our implementation available in the classpath so that Dagger can use `ServiceLoader` to instantiate our class (more on this later).

#### BindingGraph

`BindingGraph` is how Dagger represents our dependency graph. Dagger performs a lot of checks before construction and then finally reports the constructed graph to plugin, where we are able to inspect it. It is **not possible to modify the graph** once constructed. Some notable constructs of `BindingGraph` are listed below.

* `Node` - As the name suggests, a node in the graph and it can be any of the below.
    * `ComponentNode` - A node that defines the `Component` itself. For example, if you have an `AppComponent` and a `AppSubComponent` they will be present as `ComponentNode` in the graph.
    * `MaybeBinding` - A marker to denote that `Node` can be a dependency binding. *Can* is an important term here.
        * `MissingBinding` - If we forget to declare a binding that another dependency uses, it is denoted as `MissingBinding` and used in error reporting and build usually fails. One exception is full binding graph validation but that is not the scope of this article.
        * `Binding` - A valid dependency declaration is denoted as `Binding`. For example, if we have `@Provides fun provides(): Vehicle`, then we have `Binding` instance for `Vehicle`.
* `Edge` - As the name suggests, an edge connects any two nodes in the graph. There are many variations of it as listed below.
    * `DependencyEdge` - An edge between two dependency, meaning the source depends on the target. For example, `Vehicle(tyres: List<Tyre>)`, edge starts from `Binding` of vehicle and ends at binding of tyres.
    * `SubcomponentCreatorBindingEdge` - An edge to represent a link between a parent component and creator of the subcomponent. Example, `Subcomponent.Builder` or `Subcomponent.Factory`.
    * `ChildFactoryMethodEdge` - Similar to above but represents a link between the parent component and the child subcomponent.

#### Diagnostic Reporter

Based on data from `BindingGraph` we can report our custom validation result using [`DiagnosticReporter`](https://github.com/google/dagger/blob/master/java/dagger/spi/DiagnosticReporter.java). The API is straight forward - for the `Node`s or `Edge`s we think are invalid, we call one of the `reportXXX()` methods and Dagger will format that error in a neat way and fail the build or warn accordingly.

### Setting up the project

As mentioned earlier, all of these validation run during compile time and not at runtime. In order for Dagger to discover our custom plugin using Java's `ServiceLoader`, we have to make sure we declare our plugin code in the annotation processor classpath. i.e either `annotationProcessor` or `kapt` for Koltin projects.

1. Create a Java or Kotlin library module.
2. Add an `implmentation` dependency on `dagger-spi` like `implementation "com.google.dagger:dagger-spi:${versions.dagger}"`.
3. Now we should be able to extend `BindingGraphPlugin` and add our custom validations.
    *  For automating `ServiceLoader` declarations, we can use Google's [auto service](https://github.com/google/auto/tree/master/service). Thus the declartion becomes
        ```kotlin
        @AutoService(BindingGraphPlugin::class)
        class TridentValidator : BindingGraphPlugin {
            // implementation
        }
        ```
4. Finally this module should be consumed in the annotation processor configuration like `kapt project(":dagger-validator")`

### Writing validations

We can verify if the above setup is working by adding a `println` to `visitGraph`. Once done, we can straight away start with validations in `visitGraph` block. For Android's Context validation, we simply need to look for `Binding` instance that has a type `android.content.Context` and whether it has any `@Qualifier` associated with it. If it does not have any then we simply call `diagnosticReporter.reportBinding` as shown below.

```kotlin
override fun visitGraph(
    bindingGraph: BindingGraph,
    diagnosticReporter: DiagnosticReporter
) {
    bindingGraph.bindings()
            .filter { binding ->
                val key = binding.key()
                key.type().toString() == "android.content.Context" && !key.qualifier().isPresent
            }.forEach { contextBinding ->
                diagnosticReporter.reportBinding(
                    ERROR,
                    contextBinding,
                    "Please annotate context binding with any qualifier"
                )
            }
}
```

In the consuming `app` module, let's bind `Context` with `@BindsInstance` and run the build.

```kotlin
@Singleton
@Component
interface AppComponent : AndroidInjector<SpiValidation> {

    @Component.Factory
    interface Factory {
        fun create(@BindsInstance context: Context): AppComponent
    }
}
```

As expected the build would fail with the error as show below. By using `DiagnosticReporter`, we are able to retain the same error format Dagger uses.

```java
error: [dev.arunkumar.dagger.validator.TridentValidator] Please annotate context binding with any qualifier
public abstract interface AppComponent extends dagger.android.AndroidInjector<dev.arunkumar.dagger.spi.validation.SpiValidation> {
                ^
      android.content.Context is injected at
          dev.arunkumar.dagger.spi.validation.MainActivity.context
      dev.arunkumar.dagger.spi.validation.MainActivity is injected at
          dagger.android.AndroidInjector.inject(T) [dev.arunkumar.dagger.spi.validation.di.AppComponent → dev.arunkumar.dagger.spi.validation.MainActivity_Builder_MainActivity.MainActivitySubcomponent]
> Task :app:kaptDebugKotlin FAILED
```

### Customizing Behavior

Similar to Dagger, there is a way for us to customize our validation behavior based on compiler arguments provided via Gradle and `javac`. This is done via `supportedOptions()` and `initOptions()` respectively. The overall flow of options looks as shown in the diagram below.

{% include images_center.html url="/assets/images/dagger-spi-options.png" caption="BindingGraphPlugin options flow"%}

First, we declare all the supported options via `supportedOptions()` as `Set<String>`. Dagger then uses this information to filter out raw options received from compiler and then calls `initOptions`. It is good practice to properly namespace the option key for clarity. For example, instead of simply stating `enabled`, we could expose `trident.enabled` which is much clearer in intention. Personally, I don't much prefer working with `Map<String, String>` for options. It is better to abstract the options into type safe data structure. One such way is shown below.

```kotlin
/**
 * Name space for compiler options
 */
private const val TRIDENT_NAMESPACE = "trident"
/**
 * Source of truth for all supported options
 */
enum class SupportedOptions(val key: String) {
    ENABLED("$TRIDENT_NAMESPACE.enabled"),
}

val SUPPORTED_OPTIONS = values().map { it.key }.toMutableSet()

data class TridentOptions(val enabled: Boolean = false)
/**
 * @return true if this map contains `key` and its value is a `Boolean` with `true`.
 */
private fun Map<String, String>.booleanValue(key: String): Boolean {
    return containsKey(key) && get(key)?.toBoolean() == true
}
/**
 * Parses raw key value pair received from javac and maps it to typed data structure (`TridentOptions`)
 */
fun Map<String, String>.parseTridentOptions(): TridentOptions {
    val enabled = booleanValue(ENABLED.key)
    return TridentOptions(enabled)
}
```

Then in `TridentValidator` it becomes easy to declare and parse options to `TridentOptions`.

```kotlin
/**
 * Map to store user defined option values
 */
private lateinit var options: Map<String, String>

override fun initOptions(options: MutableMap<String, String>) {
    this.options = options
}

override fun supportedOptions() = SUPPORTED_OPTIONS

override fun visitGraph(
    bindingGraph: BindingGraph,
    diagnosticReporter: DiagnosticReporter
){
    val tridentOptions = options.parseTridentOptions()
    if (tridentOptions.enabled) {
       // Do validations
    }
}
```

So far we have covered setting up the project, writing validation and the ability to customize behavior based on build arguments. This forms the crux of the article and the remainder of the article is about my own take on organizing the validations for better seperation with Dagger.

### Organizing the validations

In order to better oganize and scope individual validations, I decided to refactor the base setup with Dagger and introducing seperate class for each type of validation and ability to add/remove validations by using Multibindings. As a first step, define a base `Validator` class as shown below.

```kotlin
class Validator
@Inject
constructor(
    private val tridentOptions: TridentOptions,
    private val validations: Set<@JvmSuppressWildcards Validation>
) {
    fun doValidation() {
        if (tridentOptions.enabled) {
            validations.forEach { validation ->
                validation.validate()
            }
        }
    }
}
```

With this we can configure Dagger to contribute `n` number of `Validation` implementations that the `Validator` can execute.

```kotlin
/**
 * Marker interface to annotate that a class performs validation.
 */
interface Validation {
    /**
     * The [BindingGraph] of the component for which the validation is to be performed
     */
    val bindingGraph: BindingGraph

    /**
     * The [diagnosticReporter] instance which can be used to report errors/warning to dagger.
     */
    val diagnosticReporter: DiagnosticReporter

    /**
     * Implementation of the method is expected to utilize `bindingGraph` and report any validation
     * failures/concerns to `diagnosticReporter`.
     */
    fun validate()
}
```

`bindingGraph` and `diagnosticReporter` are the minimum required objects for validation hence they are part of interface. Implementations could request any other dependency from Dagger if required. Finally we configure our injector `TridentComponent` to accept these values via combination of `@BindsInstance` and `@Module` for placing these instances in the graph.

```kotlin
@Component(
    modules = [
        TridentModule::class,
        ValidationModule::class
    ]
)
interface TridentComponent {

    fun validator(): Validator

    @Component.Factory
    interface Factory {
        fun create(
            tridentModule: TridentModule,
            @BindsInstance bindingGraph: BindingGraph,
            @BindsInstance diagnosticReporter: DiagnosticReporter
        ): TridentComponent
    }
}
```

Overall, the dependency graph looks like below (generated with [Scabbard](https://arunkumar9t2.github.io/scabbard/)).

{% include images_center.html url="/assets/images/dagger-trident-component.svg" caption="Trident dependency graph"%}

I also added another `Validation` that ensure primitives are not bound as-is and has a qualifier. Works for `Integer` alone for now but can be easily extended.

```kotlin
class PrimitivesValidation
@Inject
constructor(
    override val bindingGraph: BindingGraph,
    override val diagnosticReporter: DiagnosticReporter
) : Validation {
    override fun validate() {
        bindingGraph.bindings()
            .filter { binding ->
                val key = binding.key()
                key.type().toString() == Integer::class.java.name && !key.qualifier().isPresent
            }.forEach { binding ->
                diagnosticReporter.reportBinding(
                    WARNING,
                    binding,
                    "Primitives should be annotated with any qualifier"
                )
            }
    }
}
```

Connecting all together, the validation in `TridentValidator.visitGraph` becomes as shown below.

```kotlin
override fun visitGraph(
    bindingGraph: BindingGraph,
    diagnosticReporter: DiagnosticReporter
) {
    val tridentModule = TridentModule(types, elements, options.asTridentOptions())
    DaggerTridentComponent.factory()
        .create(tridentModule, bindingGraph, diagnosticReporter)
        .validator()
        .doValidation()
}
```

> The full source of this sample is available [here](https://github.com/arunkumar9t2/blog-resources/tree/master/dagger-spi-validations)

### Conclusion

In this article, Dagger SPI specifics, internal Dagger model representations, and building custom validations for Dagger dependency graph was presented in detail. Dagger SPI, although limited (only inspection allowed) is a great way to naturally extend Dagger's compiler according to project specific and framerwork specific needs. Although only few methods of `DiagnosticReporter` were covered in this article, it is powerful and allows us to target many different types of nodes and edges. For example, it can be used to block any usage of `@Subcomponent` at all. Finally an opinionated way to organize the valdiation code better was presented.

#### Further work

Some of the things that I considered initially for the article were:
* Adding a compiler option to toggle between `Diagnostic.Kind.ERROR` or `Diagnostic.Kind.WARNING` thus allowing us to configure severity of validation.
* Custom validation that vets any illegal dependency edge. For example, any instance in which it is forbidden to use another specific binding as a dependency.

Would be glad to hear any feedback/concerns in the comments below, thanks.

**- Arun**