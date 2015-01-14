---
layout: post
title: "ViewHolder vs. HolderView"
modified: 2014-05-23 19:04:19 -0400
tags: [viewholder, holderview, caching, listview]
image:
  feature: 
  credit: 
  creditlink: 
comments: true
share: 
---
Further semantics and practices
-------------------------------
 
Well, I personally think that `ListView`s are some of the most complicated `View`s in the Android ecosystem. So, I'm going to make two blog posts talking about them! The first is the ViewHolder vs. HolderView pattern, the second will be some nasty issues on ViewHolder caching I ran into. So, without further ado:
 
What is ViewHolder?
-------------------
 
One of the biggest problems in displaying a `ListView` in Android is just how inefficient it is. Specifically, calling `findViewById()` is really expensive, so we try and do everything we can to avoid it. That's why the [ViewHolder](http://developer.android.com/training/improving-layouts/smooth-scrolling.html#ViewHolder) pattern was invented. The basic principle is:
 
* The `ListView` gives us a `View` object when it requests a view from the adapter with `getView()`
 
* If that `View` is `null`, we need to inflate a new one. Then, cache it in the view with `setTag()`
   
* If that `View` is not `null`, we can use the View's `getTag()` method to retrieve a cached version of it (since it's been inflated prior)
 
So, what we actually *cache* in that sequence above is the `ViewHolder` object. Basically, it just stores a static reference to the inflated fields so we don't have to reinflate everything. Then, the actual adapter is responsible for updating the inflated `View` using the references in the `ViewHolder`.
 
If you have any other questions on ViewHolder, check out [this page](http://lucasr.org/2012/04/05/performance-tips-for-androids-listview/) because the author did an amazing job of explaining everything.
 
What is HolderView?
-------------------
 
The [HolderView](http://www.jayway.com/2013/11/06/viewholder-vs-holderview/) pattern has the same basic principle. We want to make as few calls to `findViewById()` as possible. There's a big difference though: **The `HolderView` is responsible for its own presentation.**
 
* The `ViewHolder` just stores a reference to the `View` elements that are being inflated (by the adapter). The `HolderView` actually inflates itself.
* The Adapter driving the ListView is responsible for the presentation of data in each element. The `HolderView` handles it's own presentation through a `bind()` or some similar method.
* The `ViewHolder` is a static class, and so can't be garbage collected. Each `ViewHolder` takes up incredibly little space, but if you're OCD, you still have unused objects you can't remove from memory. The `HolderView` is allowed to go out of scope and be deconstructed.
 
There are a few downsides too:
 
* The `ViewHolder` pattern is nearly ubiquitous on the Internet. Honestly, if I wasn't forced to use `ViewHolder` by [Android Annotations](https://github.com/excilys/androidannotations/wiki/Adapters-and-lists), I would never have known it exists (I can't justify it, but I'm guessing a similar pattern is used on iOS development, feel free to let me know).
* The `HolderView` likely means you'll have another class file. You don't necessarily *have* to do this, but it's usually better that way.
* The `HolderView` is used in place of the view that the `ListView` is actually trying to inflate. That is, the `ViewHolder` just caches references to elements inside the `View`. The `HolderView` actually replaces the `View` (which means typecasting).
 
By any means, the `HolderView` pattern kind of acts like an [MVC](http://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93controller) layer for your views. You can check out my original implementation over [here](https://github.com/MinimalBible/MinimalBible/blob/d16730781b31efc120a13a917992372956127310/MinimalBible/src/org/bspeice/minimalbible/activities/downloader/BookListAdapter.java#L51).
 
So the biggest benefit of using a `HolderView` pattern is that you can move the presentation logic of a `View` to its own class, and thus do some pretty cool stuff (for example, Dagger won't let you inject non-static inner classes).
 
So those are the two patterns! Coming up next: more than you ever wanted to know about `ListView` caching.
