---
layout: post
title: "Hold that thought..."
modified: 2014-07-03 17:55:41 -0400
tags: [delay, unit test, testing]
image:
  feature: 
  credit: 
  creditlink: 
comments: 
share: 
---

Hey all! Quick update:

I've been trying for the past few weeks to get unit testing enabled on the app. This unfortunately hasn't been going well, and there have been too many late nights on [StackOverflow](http://stackoverflow.com) and Google with no luck.

The issues are entirely to do with Dagger and the `Application` lifecycle on Android. Currently, I need to inject mock objects into the `ObjectGraph` so that I can run the tests. Problem being, Dagger validates everything at compile time. Note that I'd have the same issues using any other dependency injection framework, it just happens that I'm using dagger right now. That being said, the only way to inject new objects is to `plus()` the original `ObjectGraph` and then store that back in the Application. I personally don't consider this acceptable - besides just being bad code (messing with the `Application` just doesn't sound like a great idea) I'm guessing I would run into issues later if multiple modules try to over-ride the inject for a class.

So, I'll be taking time to start fresh with Android Studio. Using Gradle is awesome (mostly for dependency management), but I need to rebuild the application from the ground up to fully take advantage of the new platform (i.e. build variants and the support library).

To that end, what I'm doing to make sure I don't run into this issue in the future is similar to what's in the [U2020](https://github.com/JakeWharton/u2020) app - inject the application itself. This way, each build variant (i.e. debug, release, and testing) is free to implement the application however they want. And, that application is then free to set up mock objects how/if it needs.

So this work will take a week or three I expect, at which point I'll be back to regular development.

Thanks for the patience, and see you on the other side of a beautiful codebase!
