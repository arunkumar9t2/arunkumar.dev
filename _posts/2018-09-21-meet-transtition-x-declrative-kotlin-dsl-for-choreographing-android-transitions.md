---
layout: post
title:  "Meet Transition X — Declarative Kotlin DSL for choreographing Android Transitions"
description: An Android library for simplifying transitions with a custom Kotlin DSL. 
categories: [Android, Libraries]
image: https://cdn-images-1.medium.com/max/2000/1*6aGUGICoIOKY3gqfryNRRg.png
---

### Motion

Material Design’s announcement at Google IO 2014 redefined Android UX. New emphasis were given to **motion** and the guidelines encouraged using motion as a tool to be expressive and adding character to your app.

> **Motion provides meaning**
> Motion focuses attention and maintains continuity, through subtle feedback and coherent transitions. As elements appear on screen, they transform and reorganize the environment, with interactions generating new transformations.

[**Understanding motion**
*Motion helps make a UI expressive and easy to use. Principles Motion helps orient users by showing how elements are…*material.io](https://material.io/design/motion/understanding-motion.html#usage)

On the technical side of things, for the first time `Android Lollipop` used a separate thread (`RenderThread`) to handle animations even if there are delays in main thread. Introduction of RenderThread allowed things like `Circular Reveal Animation`, `AnimatedVectorDrawables`, `Ripple Feedback` without noticeable slowdowns.

### Transitions Framework

Before Android Lollipop, **Kitkat introduced Scenes & Transitions API** which were an abstraction on top of Animations to reduce effort required for simple animations. Basically it was possible to give a start and end layout and the framework would animate the difference.

{% include images_full_width.html url="https://cdn-images-1.medium.com/max/2000/1*eN_khPCZ-qb67y1P_Zw8sA.png" caption="https://developer.android.com/training/transitions/" %}

In order to let the framework animate simple changes, all we must do is to call `TransitionManager.beginDelayedTransition(veiwGroup)` before changing the layout and the framework would take care of animating the difference. I recommend reading this [article](https://medium.com/google-developers/transitions-in-the-android-support-library-8bc86a1d688e) by [Andrey Kulikov](https://medium.com/@andkulikov), which covers different types of animations you could do with the framework.

### Transition X — Better transitions with Kotlin

What Kotlin has to do with transitions and Material Motion? *Let’s find out.*

Material Choreography provides certain guidelines on how these animations should play; namely **ordering, easing, duration and type.**
[**Choreography**
*Complex layout changes use shared transformation to create smooth transitions from one layout to the next. Elements are…*material.io](https://material.io/design/motion/choreography.html#)

The transition framework is already capable of [this](https://developer.android.com/reference/android/support/transition/Transition). But the boilerplate required for choreographing detailed transition were immense. Let’s take an example. Android lets us create [TransitionSet](https://developer.android.com/reference/android/support/transition/TransitionSet) instance by using XML as shown below.

{% gist 8f3e4aea6edf98540bbb73cd23b58347 %}

{% include images_full_width.html url="https://cdn-images-1.medium.com/max/2000/1*YM_HbwYVDLDF8bSIUiW1Ig.png" caption="Android studio has auto completion for common properties." %}

It is no argument that XML takes lot of words to convey structure and that grows linearly when you want to choreograph complex transitions. Later in the code you would use `TransitionInflater` to get an instance then call `TransitionManager.beginDelayedTransition`. **This has high chance of getting repeated all over the place.**

Using XML also breaks type safety when working with custom transition instances.

I set to solve this problem using Kotlin by taking advantage of lamdas with receivers feature and reducing much of the boilerplate. To achieve the same transition, Transition X code would look like below:

{% gist 32d1562ce9994ccdcae4069b8789cfb1 %}

Transition X provides a `prepareTransition` extension method on `ViewGroup` classes which can be used to construct complex transition instances. After `prepareTransition` you typically apply the layout changes for kick starting the transition. Scheduling call to `TransitionManager` is handled automatically, you only need to concentrate on declaring different transitions and their characteristics and the library handles the rest.

#### Transition X library features

* **Declarative —** Easily declare variety of transitions and specify their properties. No math required. Transition X has variety of methods for inbuilt transitions from support library like `ChangeBounds`, `ChangeTransform`, `ChangeClipBounds` etc.

* **Rich tooling support —** Transition X inherits all of Kotlin’s powerful IDE tooling integrations and provides you with auto complete for all available transition and related properties.

{% include images_full_width.html url="https://cdn-images-1.medium.com/max/2000/1*K12hNjCMdWjAn6NSgOQvrg.png" caption="IDE autocomplete for all available transitions and properties" %}

* **Extensible —** Have a custom transition? Transition X can work with them too, thanks to Kotlin’s `reified` inline types.

{% gist efeadcc4a0a9b3c69c1cb80ae5c79954 %}

* **Natural syntax —** The syntax is simple to understand and reduces cognitive load needed to follow the animation choreography. Lifecycle methods like `onEnd`, `onStart` provide greater clarity and control over how rest of the code reacts to transition.

{% gist 997dc7a99371aeab92d12ecf9bb93a25 %}

The remainder of the article will explore some common transitions and demonstrate how they are accomplished with Transition X.

### Examples

##### Snackbar transition

Snackbar + Fab animation is one of the first things that caught our eye when material design principles were introduced. The real implementation however used `CoordinatorLayout` and bunch of math to mimic the moving the behavior. With `ConstraintLayout` and `Transition X`, it becomes simple as ever.

{% gist 1ce29f06ade092e61757e90d365113ca %}

{% include images_phone_screenshot.html url="https://cdn-images-1.medium.com/max/2000/1*T-UNjRal8DnF7rcbvRpc8Q.gif" caption="" %}

Here the FAB has a bottom constraint attached to the SnackbarMessage layout. When the SnackbarMessage becomes visible, the FAB moves to the top because of the constraints.

Breaking down the code above:

* **moveResize**: Internally adds a `ChangeBounds` transition which animates `View`’s bounds. By using `+` operator overload, we can ask this transition to only affect fab and nothing else.

* **slide:** Internally adds a `Slide` animation which triggers when view visibility changes. Like fab this transition is exclusively set to affect `snackbarMessage` alone.

* **decelerateEasing():** Adds a `LinearOutSlowInInterpolator` on this transition.

##### Advanced Choreography:

At most times, it is not beneficial to simply move resize items on screen without context. Material Choreography recommends greater control over which elements move/fade and how. In the following example, there are several things happening.

{% include images_phone_screenshot.html url="https://cdn-images-1.medium.com/max/2000/1*f1U8T2lR8vCLdatdGeekfA.gif" caption="" %}

1. The card size changes to accommodate larger images. As per material choreography items resizing should have `FastOutSlowInInterpolator`.

1. Notice the `Calm your mind..`. text does not simply move to the bottom, but instead fades away and emerges naturally under expanded images.

1. Images’ scale type and bounds changes.

1. Collapse/Expand text transition waits for the resize animation to end and does not change together with the resize animation.

The code for the above transition:

{% gist 1d2cd1f519a7d4e26818bd39abd1fe22 %}

Notice that it has become considerably easier to pull off this transition with no math or calculations, just clever declarations with different targets and the `TransitionManager` takes care of animating the rest.

### Why not ObjectAnimators?

Transitions internally use object animators and they simply are an abstraction over `Animators` with start and end values. It is possible to use `Animators` directly but there are certain concerns.

* **Performance**: Object animators are one of the powerful ways to handle animations, but the transition framework has some important optimizations which make them desirable. Namely the `ChangeBounds` transition. `ChangeBounds` internally uses a hidden private API called `ViewGroup.suppressLayout` which temporarily pauses all `invalidate()` calls to the layout which means less pressure on the CPU to constantly remeasure moving items on the screen.

* **Increased complexity:** Object animators requires calculation and knowing end value before layout changes whereas `Transition` framework automatically gets the information after the layout has changed.

{% include images_phone_screenshot.html url="https://cdn-images-1.medium.com/max/2000/1*QtPP5kF4xw86nZ5UYfESSw.gif" caption="" %}

In this example, the ImageView's scale and gravity alone is changed. By using `moveResize` and `ArcMotion` it is possible perform this hero transition without needing to calculate where the ImageView would be when thelayoutchanges are applied.

```kotlin
frameLayout.prepareTransition {
    moveResize {
        pathMotion = ArcMotion()
        +userIconView
    }
}
```

With `ObjectAnimator` however, we would need to calculate the end location of the ImageView beforehand to run this transition.

### Gotchas

Like any framework, `Transition` does have some [limitations](https://developer.android.com/training/transitions/#Limitations). All of them can considerably overcome using Transition X DSL. For example, to exclude all `RecyclerView` from participating in the transition it is sufficient to call `exclude<RecyclerView>()` and all the RVs inside current layout will not participate in the transition.

### Conclusion

I strongly believe Transition X style DSL reduces complexity and simplifies workflow when working with transitions. The aim of this library is to increase exposure to the Transition framework and unlock their potential using Kotlin’s language features. The entire library and a sample app with basic transition are available in the below repo.
[**arunkumar9t2/transition-x**
*Declarative Kotlin DSL for choreographing Android transitions - arunkumar9t2/transition-x*github.com](https://github.com/arunkumar9t2/transition-x)

If you have any feedback on the project, I would be happy to hear from you in the comments below or on [Twitter](https://twitter.com/arunkumar_9t2) or on [Google+](https://plus.google.com/u/0/+arunkumar5592).
