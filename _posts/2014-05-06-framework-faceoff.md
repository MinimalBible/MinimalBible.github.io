---
layout: post
title: "Framework Faceoff"
modified: 2014-05-07 22:55:55 -0400
tags: [roboguice, android annotations, di, dependency injection, ioc, inversion of control, dagger, butterknife, jsr330]
image:
  feature: 
  credit: 
  creditlink: 
comments: 
share: 
---

Making Black-and-White decisions in a Grey world
------------------------------------------------
 
Alright, so I've been doing a **lot** of work evaluating what I want to use as a Dependency Injection (DI)/Inversion of Control (IoC) and Android development framework. So far as I see it, there are four main technologies to consider: [RoboGuice](https://github.com/roboguice/roboguice), [Android Annotations](http://androidannotations.org/), [Dagger](http://square.github.io/dagger/), and [ButterKnife](http://jakewharton.github.io/butterknife/).
 
And a quick note on what each one is before I get to comparing them. **RoboGuice** is a full development framework for Android based on Google [Guice](https://code.google.com/p/google-guice/): it comes with DI, and a whole lot of features to eliminate boring code. **Android Annotation** is not quite as full-featured as RoboGuice, but as a DI framework is very similar. The biggest difference is in the technique it uses, more on that later. **Dagger** is a DI container built by the folks at Square, and that's all it does. So far as I can tell, you can actually use it independent of Android. Which means it doesn't have `View` injection like RoboGuice and Android Annotations. **ButterKnife** is a view injection toolkit for Android, and basically adds the 'eliminate boring Android code' aspect of the above frameworks. It **is not** an IoC container, so makes the most sense when you pair it with Dagger. So without further ado, the comparisons!
 
RoboGuice
---------
**Pros:**
* **Feature-filled** - Seriously. It comes with its own Event Bus (the `EventManager`) on top of the IoC container, and I'm sure there's even more there.
* **Well-used** - RoboGuice lists itself as being used by Groupon, FaceBook Messenger, and other high-profile apps.
* **Reflective** - RoboGuice uses reflection to accomplish everything, which means there's absolutely no code generation involved.
 
**Cons:**
* **Documentation** - The documentation is **awful**. I'm not saying this to be mean, but the documentation for RoboGuice is simply unusable. Even the JavaDocs are hosted by [someone else](http://www.imobilebbs.com/download/android/roboguice/javadoc/). If you're already familiar with Guice, you likely will be able to figure out a solution. If not, [StackOverflow](http://stackoverflow.com/questions/tagged/roboguice) is about to become a very close friend.
* **Reflective** - Can't do any compile-time checking like Dagger, so all bugs are discovered at run-time
* **Feature-filled** - RoboGuice has everything, including the kitchen sink, and an extra refrigerator too. It's complicated trying to figure out how exactly something should be done.
* **Inheritance** - This is a fairly small point, but you have to inherit from the `RoboGuiceActivity` and other `RoboGuice...` classes. Normally this isn't problematic, but can lead to some [interesting problems](http://stackoverflow.com/questions/8289660/any-simple-examples-using-roboguice-with-fragments-in-android).
 
**Final Thoughts:** 
My biggest complaint about RoboGuice is the documentation. I don't have a lot of history with the framework, and so the barrier to entry is huge. And honestly, that's the only substantive complain I have.
 
The other minor complaints I have are inheriting from the `RoboGuice...` objects. Kind of a minor point, and I'm a bit of a perfectionist, but I don't like inheriting from another object to basically just run the injection. I can run it myself. Also, RoboGuice just tries to be everything to everybody, and I think it would do well to pare down everything that's going on. The Android SDK is still a very capable library.
 
Also, there have been comments on the internet about the penalty of using reflection vs. compile-time injection, and honestly, I haven't seen anything to actually substantiate these claims. Technically, yes, it is an extra cost to do the reflection. From everything I can see though, the cost is negligible.
 
So I don't plan on using this, but if you need a one-stop-shop solution, RoboGuice is the way to go. It's battle-tested, and is the most feature-filled of anything here.
 
Android Annotations (AA)
------------------------
**Pros:**
* **Code generation** - AA works by generating the code you need at compile time, rather than doing anything at run-time. This means there's no overhead for using the reflection APIs.
* **Feature-filled** - Because AA runs at compile time, it can do some fancier things. For example, annotation a method with `@Background` allows you to run an arbitrary method off the main thread [without using](https://github.com/excilys/androidannotations/wiki/WorkingWithThreads) AsyncTasks
* **Documentation** - I hate that I need to actually include this as a Pro, I feel like it should be a given. That being said, the [documentation for AA](https://github.com/excilys/androidannotations/wiki) is pretty great. The community seems to be fairly active too, so that's great.
* **RoboGuice compatibility** - AA went above and beyond to make sure their framework [plays nice](https://github.com/excilys/androidannotations/wiki/RoboGuiceIntegration) with RoboGuice, so you can have something of the best of both worlds
 
**Cons:**
* **Code generation** - Part of what allows AA to do some amazing stuff makes it harder to understand. I don't fault the designers for this, it was going to be inherent in doing code generation to start. That being said, the semantics of using `MyClass` vs. `MyClass_`, and when the injection of `@Bean`s and `@ViewById`s are complete can be confusing.
* **[JSR-330](https://jcp.org/en/jsr/detail?id=330)** - All the other injection frameworks (Dagger, RoboGuice) are JSR-330 compliant, Android Annotations is not. Not that big of a deal, but is helpful if other people are working with your project, and I think the names are better in the JSR. Also means that AA is missing the `Producer<T>` pattern that both Dagger and RoboGuice supply.
* **Opinionated** - Because AA uses generated classes, you have to use their supplied patterns. For example, the `setArguments()` method goes away in `Fragment`s, you have to use `MyFragment_.builder().myArgument().build()`. To be fair, it makes type checking far simpler, but you do have to go through the builder.
 
**Final Thoughts:** 
Android Annotations was my original pick when starting to code MinimalBible. It has all the functionality of RoboGuice that I would actually use, the `@Background` annotation is awesome, and the documentation was incredibly helpful.
 
That being said, my largest complain is one of the biggest reasons I was attracted to AA in the first place - code generation. **I realize there is no other way this could have been done.** I certainly don't fault the developer for this, but trying to understand when to use the `Class_` notation I can see getting out of hand quickly. Plus, you need to understand the semantics of AA on top of DI.
 
So all said, I think I'd prefer Android Annotations over RoboGuice. But only slightly.
 
Dagger / ButterKnife
--------------------
*Note: I realize these are separate products, but they work so well together, and you need both to actually rival RoboGuice or AA*
 
**Pros:**
* **Code Generation** - Here's another code generation framework, so no reflection penalty. Also, Dagger is great about notifying you of errors in your injection code, so you don't have to wait until runtime to find the bugs (like RoboGuice). You also don't have to worry about `Class_` semantics (like AA).
* **Focus** - These are very focused libraries. They're worried about doing a few things, and doing them well. You won't have confusing APIs to navigate through.
* **Shiny** - The libraries are shiny and new. This isn't a giant selling point, but these libraries are very up-to-date and modern with current techniques, design patterns, etc. They're **[Good Code](https://xkcd.com/844/)**.
* **Explicit** - Dagger forces you to be explicit about your design, which is good style, and you don't have to understand the semantics underlying a framework like AA.
 
**Cons:**
* **Limited API** - While Dagger and ButterKnife are very clear about what they do, it also means they don't have functionality other frameworks do. Just means you're left to do some of the code yourself.
 
**Final Thoughts:** 
While I didn't put much down for the cons, I do want to stress the point - the limited API can be a big deal. Using Dagger/ButterKnife will definitely save time, and make your code more readable, but there are still significant chunks of functionality you're left to do yourself. That may be more your style (like me), but in fairness, Dagger and ButterKnife are relatively limited. Relative in the sense that RoboGuice and AA provide a huge amount of functionality. You will have to write more code by hand.
 
At this point, I'm planning on using Dagger/ButterKnife for the backbone of my app. The fact that I don't have a confusing and complex API to navigate is awesome, and I think there's something to be said for having a library that is very focused on what it actually wants to get done.
 
Side note, I'm planning on using [Auto Factories](https://github.com/google/auto/tree/master/factory) as well for some of the more complicated producers I have in mind. Right now the only formal "release" is a `0.1-beta`, but I think it's ready for some usage.
 
Bonus Round: Android Annotations + Dagger
-----------------------------------------
*Note: You can actually use Dagger as the IoC container and rely on AA for the Android meta-programming. I looked into trying to use this as well, and hope that it's helpful, as there is very little information about programming this way on the internet.*
 
**Pros:**
* **Two libraries** - Get the functionality of a JSR-330-compliant IoC container, and a full-featured Android toolkit! The power of Android Annotations, and the usability of Dagger!
 
**Cons:**
* **Two libraries** - It is still a bit confusing trying to mix two libraries. For example, both AA and Dagger provide a `@Singleton` annotation, so you need to make sure you understand which one you're using.
* **Code Generation** - You still have to understand some of the semantics of using AA. For example, you can't `@Inject MyClass`, but have to:
```
@Provides<MyClass>
public MyClass provideClass() {
    return (MyClass)new MyClass_();
}
```
If you tried to actually inject the original class, it wouldn't have the functionality actually supplied by AA. And when you're trying to do this with `@Singleton` as well, things can get confusing.
 
**Final Thoughts:** 
You can find a full-featured project demonstrating Dagger and AA over [here](https://github.com/sinojelly/androidannotations-dagger-example). It's a great little project that demonstrates some of the complexity of mixing DI frameworks.
 
Overall, I will have no reservations about using Android Annotations in the future if I think the readability cost of generated `MyClass_`s is surpassed by what AA can offer. For now though, I'm not convinced I *need* Android Annotations.
 
In the end
----------
For me personally, I plan on using Dagger and ButterKnife to make up the largest part, if not all of, the app. I think they provide the best feature count while not being too opinionated about how I'm supposed to program.
 
I'm not at all averse to integrating Android Annotations later on down the line should they provide functionality I need. I think it's a great project, and using code generation I think is pretty cool. However, the functionality gain would have to outweigh the cost of needing to understand the semantics of AA.
 
RoboGuice is the framework I wish I could use, but would take far too much of an investment to pick up. Again, the documentation is **awful**. From what I've read on their Github page, you need to extend from `RoboGuice...` whatever. From some instructions on upgrading from `1.1` to `2.0`, it appears that you [no longer need](http://code.google.com/p/roboguice/wiki/UpgradingTo20) some of the `extend`s. However, that documentation comes from the Google Code page, which is outdated (development happens in Github). To be fair, there is documentation about the upgrade [available on Github](https://github.com/roboguice/roboguice/wiki/WhatsNew20) as well, but it's not nearly as detailed. Plus, [they're developing](https://github.com/roboguice/roboguice/wiki/WhatsNew30) `3.0-alpha 2` at the time of writing. I just really don't know where to begin.
 
I hope this helps you think through which container/framework is right for you. I'm happy to answer any questions you have, just send an email!
 
*P.S.* 
While writing this post I discovered that someone else wrote something similar: http://vardhan-justlikethat.blogspot.com/2014/03/android-dependency-injection-roboguice.html 
However, it didn't seem to actually discuss the technologies in-depth enough, and the author didn't seem to actually understand what the technologies did. I think this article provides a bit more help actually evaluating each piece of software.
