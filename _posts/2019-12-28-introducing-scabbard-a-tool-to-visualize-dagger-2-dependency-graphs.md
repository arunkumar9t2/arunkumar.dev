---
layout: post
title: 'Introducing Scabbard: Easily visualize Dagger 2 dependency graphs'
description: An IntelliJ/Gradle plugin to easily visualize and understand Dagger 2 graphs from within IDE.
date: 2019-12-28 09:43 +0800
categories: [Android, Dagger, Library]
image : /assets/images/scabbard-banner.png
---

I am excited to announce [Scabbard](https://arunkumar9t2.github.io/scabbard/), a tool to visualize Dagger 2 dependency graphs. Here's Scabbard in action:

{% include gfycat_embed.html url="https://gfycat.com/ifr/thankfulconfusedkillifish" %}

Here is an advanced example from Google's I/O 19 app.

{% include gfycat_embed.html url="https://gfycat.com/ifr/ignorantscrawnylemming" %}

Apart from ensuring runtime safety, one of the advantages of a compile-time framework for dependency injection is that the entire graph of dependencies can be known at compile time even before binary is generated. This enables features like compilation errors when there are missing bindings which is a very good step towards reducing developer mistakes. But the cost we pay for that is the mental effort of knowing all the bindings, knowing current scope, navigating errors to make dagger happy etc. Very soon we realize there is a visualization problem here and it is evident from many dagger tutorial articles which tries to visualize the dagger structure in their own ways to convey the message. I tried to tackle this problem with Scabbard.

Please follow [Getting Started](https://arunkumar9t2.github.io/scabbard/) guide on the project website to start generating graphs right away. Hopefully you would have graphs by the time you finish reading this article.

### Features

* **Visualize** entry points, dependency graph, component relationships and scopes in your Dagger setup.

* **Minimal setup** - Scabbard's Gradle plugin prepares your project for graph generation and provides ability to customize graph generation behavior.

* **IDE integration** - Easily view a `@Component` or a `@Subcomponnet` graphs directly from source code via gutter icons (IntelliJ/Android Studio).

* **Supports** both Kotlin and Java.

### How it works

In order to get to the point shown in above videos, Scabbard has 3 different components working together.

* **Scabbard Processor** - A `BindingGraphPlugin` which is provided as part of [Dagger SPI](https://dagger.dev/spi.html). It receives callbacks from Dagger's main compiler about graph information and is responsible for generating `png` images and `dot` files.

* **Scabbard Gradle Plugin** - A gradle plugin to abstract away internal implementation details required in setting up the above processor and also acts as an API to configure Scabbard behavior. Being a Gradle plugin allows it do stuff that is otherwise not possible with pure annotation processors.

* **Scabbard IntelliJ Plugin** - A IntelliJ plugin to link generated graphs back to their source code. This plugin also sets up stage for building any additional UI tools based on Dagger information in the future.

### Goals

For Scabbard, I had the following goals in mind.

* Establish ability to build informational tools with Dagger graph information - as mentioned above the IDE and Gradle plugins work together with shared knowledge to provide UX improvements. This can be easily extended to build additional tools (like linking `@Inject` with bindings).
* Easy to use visualization - Scabbard aims to be simple to setup and tries to abstract away the internal implementation details involved in graph generation
* Readable Graph information - it is very easy to end up in complicated graph diagrams which is technically correct but hard to decipher due to their complexity especially in large projects like _Grab's_. To tackle this, Scabbard splits the graph by `@Component` and `@Subcomponent` to keep the graphs small and understandable.
* Opionated strucure - internal dagger representation can be inferred and represented in many different ways, Scabbard aims to start with a base representation and evolve it based on community feedback.

### Tech Stack

#### GraphViz

[GraphViz](https://www.graphviz.org/) is a popular tool for drawing graphs and Scabbard primarly uses it for generating images. In addition to `png` files, Scabbard generates a raw `dot` file which can be analyzed in variety of tools ([Gephi](https://gephi.org/) etc). Generating other formats like `svg` is planned.

#### Kotlin

Project is fully written in Kotlin and it enabled me to build better APIs (DSL) allowing me to concentrate on expressing the graph cleanly instead of worrying about API specifics. Scabbard also ships a `dot-dsl` artifact for writing dot files easily with Kotlin, example:

{% include images_phone_screenshot.html url="https://github.com/arunkumar9t2/scabbard/raw/master/docs/images/dot-dsl.png" caption="Dot-dsl for constructing dot files in Kotlin"%}

### Cheat sheet

Below is the example `AppComponent` used as a base for building graph and its attributes (please `open in new tab` for better quality). There is a plan to provide a better detailed cheat sheet to help in understanding the graph better, contributions welcome!

{% include images_center.html url="https://github.com/arunkumar9t2/scabbard/raw/master/docs/images/temp-cheat-sheat.png" %}

### Highlights

There are a lot to cover, below are some selected highlights and examples.

#### Visualizing errors

Scabbard can highlight `Missing Bindings` in red color which can be easily used to point to why the error arises. Requires [full binding graph validation.](https://arunkumar9t2.github.io/scabbard/configuration/#enable-full-binding-graph-validation)

{% include images_center.html url="https://github.com/arunkumar9t2/scabbard/raw/master/docs/images/missing-binding.png" caption="Missing bindings"%}

#### Google IO 2019

{% include images_center.html url="https://arunkumar9t2.github.io/scabbard/images/io_appcomponent.png" caption="AppComponent"%}

#### Plaid

{% include images_center.html url="https://arunkumar9t2.github.io/scabbard/images/plaid_home.png" caption="HomeComponent"%}

### Summary

Overall I am excited to ship `0.1.0` today and it has been challenging project with lots of stuff learnt particularly dagger internals. Hopefully it is helpful. Would love to hear feedback and iterate on it further. Please share your thoughts in the comments or [github issues](https://github.com/arunkumar9t2/scabbard/issues), or on [Twitter](https://twitter.com/arunkumar_9t2).