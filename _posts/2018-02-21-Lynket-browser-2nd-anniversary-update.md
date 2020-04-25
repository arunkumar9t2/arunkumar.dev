---
layout: post
title:  "Introducing Lynket (previously Chromer)— A better way to browse on Android"
description: Introduction post about Lynket's major update and feature walkthrough.
categories: [Android, Apps]
image: https://cdn-images-1.medium.com/max/2000/1*9EiFLkWltIEqE59QIwBDBA.png
---

*This post originally appeared on [Medium](https://medium.com/@arunkumar9t2/introducing-chromer-2-0-a-better-way-to-browse-on-android-c34ff729df1b).*
  
I would like to introduce you to my open source Android app **Chromer**, now called Lynket. In this article, I want to talk about how Lynket does things differently and tries to remain relevant among top browsers such as Chrome, Brave, Firefox, Samsung Internet or Opera etc.

{% include youtube_embed.html url="https://www.youtube.com/embed/Hcd2R2Lh5ks" %}
  
### Brief history

Let’s say you are using WhatsApp and you receive a link. Naturally clicking on it would open the link in your browser app and you leave WhatsApp to read the link. What if WhatsApp wants to show you the link and customize it so that you don’t have to **leave the app** just for visiting this link? They have to use a system component called WebView which was only way developers to show web pages or build a custom browser.

While WebView served this purpose well, it was not perfect. There were flaws such as security patches required an Android System update (which is slow), performance were rather sub par and in worst cases [crashes](https://plus.google.com/+AnthonyRestaino/posts/NA5DyBLDJFr) the app. Popular custom browser Link Bubble’s Chris Lacy [agrees](https://www.androidpolice.com/2014/05/16/interview-link-bubble-and-action-launcher-developer-chris-lacy-shares-his-thoughts-on-android-app-development-and-more/) on the sentiments.

  

#### Google’s answer and progressive enhancements

Luckily all these cries did not fall on deaf ears. Browser app such as *Chrome* or *Firefox* are generally better because they use their own rendering engine updated through Play Store. It would be nice if the same happens with WebView, wouldn’t it? Google did that exactly in Android Lollipop.

However at IO/15, Google announced something unexpected. They said we are giving developers an API to load a light weight version of Chrome onto their apps! They called it Custom Tabs and it was faster and smoother in many ways.

#### Custom Tabs

Without saying, Custom Tabs is much superior to WebView. It is updated through Play Store (along with provider app), shares saved passwords and can essentially sync your browsing activity with Google account but it **has to be manually implemented by each and every app.**

{% include images_full_width.html url="https://developer.chrome.com/multidevice/images/customtab/performance.gif#center" caption="Custom Tabs Performance comparison" %}

### Enter Chromer

In December 2015, I [launched](https://plus.google.com/u/0/+arunkumar5592/posts/CQmK1kRgHSh) Chromer which enabled any app to use Chrome Custom Tabs *without the apps’ developer effort.* I was motivated to develop Chromer since apps that I use took a long time to implement CCT. It quickly grew to be liked by many users currently sitting at 4.6 rating from 8000+ reviewers.

### 2.0 — Anniversary Update and Lynket

Things change — Google has constantly improved the browsing experience and I am *forever* a fan of them. Google Chrome experimented opening Custom Tabs by default without the need for Chromer and system WebView in Nougat+ versions now uses Google Chrome internally.

Unfortunately, Chromer’s name had to be changed to comply with Play Store policies since it resembles Chrome. It was never my intention to imitate Chrome in anyway. I deliberately kept the icons different and made it clear in the description that initial versions required Chrome to work. Google rejected the update even though app had been live for 2 years with multiple updates in the past.

Lynket, the new name is derived from **Tabs** in Italian — `Linguetta`. With that said, can Lynket still offer a better browsing experience?

I would like to walk through certain different things I have tried in Lynket to improve your browsing experience.

{% include images_full_width.html url="https://cdn-images-1.medium.com/max/3840/1*V3S_bcE7iXJnD0F0qfsaMA.png" caption="New design of 2.0" %}

###  Lynket works with your favorite browser

{% include images_phone_screenshot.html url="https://cdn-images-1.medium.com/max/2000/1*rItxnyZcB7B0PzZAI6yZRg.gif" caption="Changing rendering engines" %}

In addition to many customization options, Lynket allows you to *choose* which rendering engine/browser you want to use for loading pages. This means, it **inherits several behaviors** from them while adding more.

* Brave Browser — Applies adblocking, tracking protection
* Google Chrome — Google account sync, login info sharing.
* Samsung Internet — Samsung Pass (Biometric login) and Samsung Cloud.

I cannot possibly build a better rendering engine, so Lynket leaves rendering to the top players and adds customizations on top of it through the Custom Tab protocol.

### The curious case of Multitasking
It is no doubt that Chris Lacy’s Link Bubble pioneered the bubble browsing in Android, inspired by Facebook’s Chat Heads. It allowed you to load links in floating bubbles, while you continue browsing so you don’t waste time looking at loading screens; but limitations of WebView still applied.

  

Later [Brave](https://brave.com) bought Link Bubble and tried to provide the same experience but [they too did not feel comfortable](https://brave.com/unpublishing-link-bubble/) working with WebView API and the same was unpublished later. 


In their own words (emphasis mine):

> given Google’s refusal to fix Android to support a **webview that runs and renders from a background service** (this is how Link Bubble runs in order to float bubbles over your current app). After a number of Android releases with no fixes in sight for significant bugs, we decided it was time to move on. …which offers the crucial features that users say they need to deal with today’s Web: **tabs**

Another problem was WebView in a Service simply does not respect the Android’s app lifecycle. A Service with WebView would take huge amounts of RAM and there was no automated way to release it, unless the developer handles it manually ([by listening for low memory events](https://developer.android.com/topic/performance/memory.html#release)).

I took the challenge to solve this problem by use of Custom Tabs. I had launched bubble browsing in Lynket last year but 2.0 contains important polishes that elevate your browsing experience.

Given the benefits of Custom Tabs, it would be awesome if we can:

* Load Custom Tabs in background
* Respect Android’s lifecycle
* Intuitive way to multitask between Tabs
* Load links in Floating Bubbles
 
That’s exactly what Lynket 2.0 does!

#### Better Bubble Browsing — Web Heads

{% include images_phone_screenshot.html url="https://cdn-images-1.medium.com/max/2000/1*XqIwvDNAhYm4U165RQUNfg.gif" caption="Web Heads - Floating bubbles" %}

#### Respect Android App Lifecycle

{% include images_phone_screenshot.html url="https://cdn-images-1.medium.com/max/2160/1*lKc2IVc3wH0FecA0Jj2hrA.png" caption="Lynkets uses Android's Overview screen to multitask" %}

Lynket uses your Android’s recents screen to create multiple tabs. This is achieved by use of *Lollipop’s Document API.* This is similar to Google Chrome’s Merge Tabs and Apps mode which was removed later.

##### Why this is better?

Custom Tabs are normal activities which Android System can **close and reload automatically** when in need of memory. This way your phone won’t lag when opening a Game or loading a photo in an editor app.

Lynket also shows title, icons and color just like Google Chrome used to do.

#### Floating bubbles + background loading

Since Custom Tabs are activities, they can’t be really loaded in background. Luckily after long investigations, experiments and refinements, Lynket 2.0 brings background loading to Custom Tabs! It works by launching the URL and **immediately going into background while you continue browsing**. This is the same mechanism Google Chrome used to do when opening a new tab in Merge Tabs and Apps mode.

{% include images_phone_screenshot.html url="https://cdn-images-1.medium.com/max/2000/1*m5Gy5vWT5GKekA4d37I3TA.gif#center" caption="Background loading with Web Heads" %}


Here you can see Techcrunch opening and getting minimized. It continues to load in background and when you want it, you can use the bubble to bring it to the front!
Long press the bubble to see active tabs.
This does not answer what intuitive multitasking means right? Read on.
### Intuitive Multitasking

{% include images_phone_screenshot.html url="https://cdn-images-1.medium.com/max/2000/1*YqQoa2QTgtyudYEiCfl6LQ.gif" caption="`Minimze` and `Tabs` to quickly multitask" %}

**Minimize —** Lynket also provides a button to quickly minimize the tab to background. This is a new convenient feature not seen in many apps/browsers. I can hear you thinking “I can do that by using recents screen!”. Yes you can, but do try how Minimize and Tabs works out for you. It’s clear, concise and gets the job done *without cognitive load of scanning through opened apps.*

**Tabs -** Something Google’s Merge Tabs and Apps missed was providing a way to access web page tabs alone in the midst of all apps in your Overview screen. Lynket solves that in **Tabs** screen. Here you can see all active tabs opened by Lynket.


I am really excited about background loading, minimize and tabs feature. I had to scour through Android’s API stack to find a way to implement this and I am excited to get feedback!

### Mobile First Browser

{% include images_phone_screenshot.html url="https://cdn-images-1.medium.com/max/4268/1*OpK7MSooJ30CkSbYK-vwUg.png" caption="" %}

Browsing on mobile is a lot different than desktop. Users expect fast, concise content that works even on slow networks. Lynket has two modes specifically tailored for this. **AMP and Article Mode**.

While I have personally received feedback from users that they don’t like AMP, it still loads much faster on Mobile on a slow network. Before loading an URL, Lynket can find if an AMP version exists and load it instead.

#### Article mode

Article mode is my favorite feature. Many browsers now have a mode called **Reader mode** which removes unnecessary content and grabs the gist of the article. Article is no different but I think it looks and functions better.

{% include images_full_width.html url="https://cdn-images-1.medium.com/max/3840/1*jBMuqr6p5tklZJd6wQRGgw.png" caption="Article mode with different themes." %}

{% include images_phone_screenshot.html url="https://cdn-images-1.medium.com/max/2000/1*MIlPFhmGaKuA0VkTh_ayiQ.gif" caption="Article mode in action." %}

You can tap the Keywords to do a Google search. Article mode was inspired by [Klinker Apps’](http://klinkerapps.com/) Article mode. However Lynket’s implementation is different and works locally without saving data on an external server.

### Other features

It does not stop here and Lynket has neat little features for your convenience.
* **Secondary browser** — Set a secondary browser to quickly switch to.
* **Favorite Sharing App** — Set a favorite share app to quickly share to.
* **Per app settings** — Configure **incognito and secondary browser** individually for each apps.
* **Quick settings toggle** for Web heads, Incognito Mode, AMP and Article Modes.
* Lot of customization features like Animations, toolbar colors etc.

### Conclusion
By using Custom Tabs API in innovative ways, I believe Lynket offers a superior browsing solution. You really don’t have to switch to Lynket completely but you can let it use your favorite browser and be in the same ecosystem (if it supports CCT). Web heads mode has matured a lot and merges bubble browsing and Custom Tabs perfectly. I have spent time on design polishes as well. Article mode is a real pleasure to use on mobile and I have received positive feedback from beta testers. I am excited to know your feedback too!

Best of all, Lynket is **fully free and open source**. You can find the source code [here.](https://github.com/arunkumar9t2/lynket-browser) If you are a developer, I would be glad to accept contributions.

<a href='https://play.google.com/store/apps/details?id=arun.com.chromer&utm_source=arunkumarsampath.in&pcampaignid=MKT-Other-global-all-co-prtnr-py-PartBadge-Mar2515-1'>
<img alt='Get it on Google Play' class="google-play-img" src='https://play.google.com/intl/en_us/badges/images/generic/en_badge_web_generic.png#center'/>
</a>

Until next time!

---