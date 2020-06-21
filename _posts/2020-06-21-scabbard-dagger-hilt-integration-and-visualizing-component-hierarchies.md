---
layout: post
title: 'Scabbard: Dagger Hilt integration and visualizing component hierarchies'
date: 2020-06-21 06:20 +0800
description: Scabbard 0.4.0 adds support for Google's new Dagger Hilt compiler and ability to visualize Component hierarchy and their scopes.
categories: [Android, Dagger, Library]
image : /assets/images/scabbard-040-header.png
---

Recently Google released an opinionated and recommended way for dependency injection in Android with Dagger. Much has already been said on this topic and the official [guides](https://developer.android.com/training/dependency-injection/hilt-android) are pretty good way to get started. In this short article, I plan to share a bit about [Scabbard]({% post_url 2019-12-28-introducing-scabbard-a-tool-to-visualize-dagger-2-dependency-graphs %})'s new release **0.4.0** which brings couple of integrations for Hilt.

### Hilt overview

As mentioned earlier, Hilt is opinionated in both component hierarchy and component entry points. This means it encourages us to simply define the bindings and where we want it placed and Hilt takes care of placing it correctly in a set of predefined components. While this is great for reducing "boilerplate" (quotes because I did not think of them as boilerplate before), I could not help but think there is more magic happening behind the scenes. For example, `dagger-android` automated `@Subcomponent` creation for us while Hilt automates the whole `@Component` creation as well. In order to do this, Hilt provides new annotations like `@AndroidEntryPoint` and `@HiltAndroidApp`. I do believe that with Scabbard, it should be possible to gain insights on what Hilt is doing behind the scenes and how they relate to these annotations.


{% include images_phone_screenshot.html url="/assets/images/hilt-components.svg" caption="Hilt hierarchy from official guide"%}

### Scabbard's Hilt support

While Scabbard's image generation [worked](https://twitter.com/arunkumar_9t2/status/1260915216663015426) out of the box with Hilt, the IDE plugin was not aware of these new annotations. Luckily it was easy to add support because of [monolithic](https://dagger.dev/hilt/monolithic.html) components. Since Hilt encourages monolith components, we can establish a many:1 mapping between Android system components and Hilt generated components. In other words, if `@AndroidEntryPoint` is added on an `Activity` then the bindings for it will always be in `ActivityComponent` (generated) no matter which Activity. With these assumptions, Scabbard supports the following annotations.

#### `@AndroidEntryPoint`

Scabbard will show a gutter icon on Android entry point annotated components that links to the equivalent Hilt generated component as shown here.

{% include images_center.html url="/assets/images/android-entry-point-activity-sample.png" caption="Scabbard Gutter for Android Entry Points"%}

##### `@WithFragmentBindings`

From the graph above we see that it is possible for `View` to optionally have bindings from Fragment and this is differentiated via the `@WithFragmentBindings` annotation. By checking for presence of this annotation, Scabbard links to `ViewComponent` or `ViewWithFragmentComponent`.


{% include images_center.html url="/assets/images/android-entry-point-view.png" caption="Scabbard Gutter for Views"%}


##### `@HiltAndroidApp`

Similar to above but links to the application root component.

{% include images_center.html url="/assets/images/hilt-android-app-sample.png" caption="Scabbard Gutter for viewing root application component"%}

##### `@DefineComponent`

Since Hilt handles the component generation now, there must be a way to define custom components and that is where `@DefineComponent` comes in. When we define a custom interface annotated with `@DefineComponent`, Hilt generates a `@Component` interface and makes it implement the interface we defined earlier. To find the generated component, it was as simple as finding the subclasses of the interface.

{% include images_center.html url="/assets/images/hilt-custom-component.png" caption="Scabbard Gutter for viewing custom component graph"%}

This is the gist of the recent additions for Hilt. With regards to the graph themselves not much has changed other than some minor cleanups in names of the components.

{% include images_center.html url="/assets/images/dev.arunkumar.scabbard.sample.hilt.HiltSampleApp_HiltComponents.ApplicationC.svg" caption=""%}

### Visualizing Component Hierarchies

This has been in plan for a while and I am happy to report Scabbard 0.4.0 supports visualizing Component hierarchies and scopes similar to the one presented in Hilt docs. Actually there is a long [thread](https://github.com/google/dagger/issues/288) on dagger repo around this subject and this was my inspiration to build scabbard in the first place. The hierarchy images will be generated alongside dependency graph images and gutter icon pop up presents option to open it. This is only presented for root components. For example, we can visualize the entire magic that `Hilt` does behind the scenes by simple adding scabbard.

{% include images_center.html url="/assets/images/tree_dev.arunkumar.scabbard.sample.hilt.HiltSampleApp_HiltComponents.ApplicationC_old.svg" caption="Hilt generated components' hierarchy"%}

Now let's try modifying the hierarchy by adding a custom component with `@DefineComponent`:

{% include images_center.html url="/assets/images/tree_dev.arunkumar.scabbard.sample.hilt.HiltSampleApp_HiltComponents.ApplicationC.svg" caption=""%}

As expected, the component became the subcomponent of `ApplicationComponent`. *Additionally as mentioned in 0.2.0 release notes, you could open the subcomponent graph by clicking on the components when you open the svg in browser.*

### Summary

In addition to these new features, there have been couple of minor bug fixes as well which are described in release notes. Overall I am excited for this release and would be glad to hear any feedback or suggestions for improvements. Please reach out to me via Twitter, Email or comments below, thanks and stay safe!