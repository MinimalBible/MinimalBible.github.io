---
layout: post
title: "Best Practices of the Future"
modified: 2014-06-18 23:07:05 -0400
tags: [future, libraries, legacy, annotations]
image:
  feature: 
  credit: 
  creditlink: 
comments: 
share: 
---
What we'll think was a terrible idea in 5 years
-----------------------------------------------
 
So, I've been working with some legacy code as of recently, and becoming particularly frustrated with it (think Spring 2.5 legacy - late 2007). Trying to wade through multiple files-worth of XML and private constructors were the two things that really stuck out as "I'm not sure why this was ever considered a good idea."
 
**XML configuration:** So, the problem is that you have classes with multiple dependencies, and if you just look at the java code, you have no idea why anything should work. You never instantiate the dependencies, you never do anything to touch the fields, they *just work*. Dependency injection is a great idea, don't get me wrong, but it's confusing to have to read through (and write!) XML to "wire up" the application. Most specifically, writing the XML is simply painful. Plus, if you have people who don't understand the Spring way of doing things (like I did), it's borderline impossible to figure out what's going on. Plus, how often does the configuration actually change? I can almost guarantee that it doesn't change more often than you compile the project (and if it does, we need to have another talk). And when it does change, do you always remember to edit the XML?
 
**Private Constructors:** I don't have as much to say about these, but I do think they need to be pointed out as a "this just isn't a good idea." I think the idea of private constructors was to force you into using the dependency injection framework, rather than having hard-coded references everywhere. This way, you force people to abide by the framework in place.
 
The initial problem is that you destroy the possibility of unit testing, or at least make it incredibly complicated. Gone is the ability for Test classes to instantiate the objects they're trying to test. Instead, you need to use some [reflection hacks](http://docs.spring.io/spring/docs/2.5.6/api/org/springframework/test/util/ReflectionTestUtils.html) to bypass the intentions of the person who wrote the class in the first place. It just doesn't make sense to write something one way, and then test it a completely different way.
 
But enough of the legacy problems. Here's what I see being problematic five years from now:
 
Annotations
-----------
 
Annotations are fantastic, don't get me wrong. Currently, there are a significant number of annotations in use for writing MinimalBible. The problem is, I think we're using too many. Here's a sampling of some Android libraries using Annotations:
 
* [Dagger](https://github.com/square/dagger) - An annotation-based dependency injection library for Android. Designed to also work on Java, and be small and light. Actively used and developed by Square and Google.
* [Butterknife](https://github.com/JakeWharton/butterknife) - An annotation-based view injection library for Android.
* [RoboGuice](https://github.com/roboguice/roboguice) - An annotation-based dependency injection library designed to imitate Google [Guice](http://code.google.com/p/google-guice/)
* [Dart](https://github.com/f2prateek/android-dart) - Annotation-based injection of "Extras" - both for [Activity extras](http://developer.android.com/reference/android/content/Intent.html#getExtras() and [Fragment arguments](http://developer.android.com/reference/android/app/Fragment.html#getArguments()
* [Michelangelo](https://github.com/RomainPiel/Michelangelo) - Annotation-based layout inflation injection. Even comes with support for Butterknife, and can call that simultaneously.
* [Flow](https://github.com/square/flow) & [Mortar](https://github.com/square/mortar) - Annotation-based libraries for arranging the "screens" of your application, injecting them with Dagger (*note: Mortar actually depends on Dagger*), and then managing the back-stack
* And finally: [Android Annotations](http://androidannotations.org/) - A library to do annotation-based injection of **everything**. Includes POJO, layout, REST, Activity, Fragment, SharedPreference, Event, and more injections. The list is available [here](https://github.com/excilys/androidannotations/wiki/AvailableAnnotations).
 
I guess what I'm trying to say is: If you don't want to code it, chances are that someone has created an annotation-based library for it. This isn't entirely fair, but you have everything from an annotation framework (Android Annotations) down to simple dependency injection (Dagger) all using Annotations.
 
Now, the implications of all this:
1. It's that much less code that you have to write yourself. That's awesome.
2. It can enforce good practice where things are lacking (I have Android Annotations [Fragment builder](https://github.com/excilys/androidannotations/wiki/FragmentArg) in mind)
3. Most of the work is done at compile-time. This means that you don't have run-time reflection running, which is also cool. Dagger even validates the ObjectGraph at compile time, meaning it will catch errors before you run anything.
 
These are all good things. The problem though, is this: **using annotations forces you to understand what the framework does and doesn't accomplish for you.**
 
The worst offender of this is Android Annotations. When you use one of its annotations on a class, you must remember to *not use the original class,* but use the generated class in the rest of your code. So even though you may have something like this:
 
{% highlight java %}
@EActivity(R.layout.my_layout)
class MyActivity extends Activity {
    // ...
}
{% endhighlight %}
 
You use it like this:
 
{% highlight java %}
    // ...
    Intent i = new Intent(MyActivity_.class);
    // ...
{% endhighlight %}
 
The next problem is knowing when the activity injects its resources (at which point you have an `@AfterInject` annotation), so on and so forth. **The penalty for using annotations to write code for you is that you're locked in to understanding the code another programmer wrote.**
 
Libraries like Dagger and Butterknife are better about this - you only have to call `objectGraph.inject(this)` or `Butterknife.inject(this)`, but I've frequently run into strange issues because I forgot to call those.

Fragmentation
-------------

The other problem I see with Android is that there are a bunch of different ways to accomplish the same thing. You can go with the vanilla SDK, or you can mix in some basic libraries like Dagger, or you can add some more in-depth tooling with Flow/Mortar, etc. If you want to consume REST data, there's [Retrofit](https://github.com/square/retrofit), though some prefer [Volley](https://android.googlesource.com/platform/frameworks/volley/). The (fantastic) [Android Weekly](http://androidweekly.net/) Newsletter has a section each week on new libraries available to do cool new things.

All of this (in my experience) leads to a bit of framework paralysis, when you try to mix in different libraries to see how they work. There's a lot of fancy cool new technology; there's also a lot of projects with a completely unproven track record waiting to distract you. Which is not to say that they're useless - but I did spend significant time trying to get things like [Robolectric](http://robolectric.org/) to work before scrapping it.

What I'd appreciate, and what will simply take time, is to see which projects are really in it for the long haul and do actually produce significant developer productivity (without sacrificing time to bring other developers up to speed).
 
Where do we go from here?
-------------------------
 
So the question of the hour then is: so what?
 
A couple years down the line, I think we'll end up looking at some of the annotation-heavy code that we've written and have no idea what's going on or how we're supposed to use it. Yes, the code is more descriptive and much smaller in size. However, it's hard to remember how 7 different annotation libraries expect you to use them, or why you're not getting the features working correctly (forgot to call a `.inject()` method, or use `MyClass_` instead of `MyClass`).
 
I think the way forward might be less reliance on annotations to handle the code we don't want to write, and relying more on well-designed API's. So, I think things like Dagger and Butterknife will stay - the penalty for using them is a couple lines of extra code, for a huge benefit. Things like Android Annotations and RoboGuice I'm guessing will largely disappear simply because it's hard to remember how they're supposed to be used, or nobody will be able to contribute to your codebase because they don't understand the framework.
 
I also think there's a huge potential in projects like [RxJava](https://github.com/Netflix/RxJava) and [Groovy on Android](http://melix.github.io/blog/2014/06/grooid.html). With RxJava you get a very well put-together API allowing you to define behavior and manage the complexity of a largely asynchronous application. It will definitely require you to structure an application differently, and think differently (there's a learning curve to a functional style), but the flexibility is incredible. Also, I haven't used Groovy at all yet (I'm planning to try and demo it a bit and see about writing MinimalBible with it), but so far it seems a lot like Python. I've had very good experience with mixed-paradigm (partly functional, partly object-oriented) languages (mostly Python) in the past, and it allowed me to write clean code quickly. I'm all down for that.
 
The tradeoff in using things like these is that they are bleeding-edge new. RxJava specifically is going to be a pain due to inner-class shenanigans. I don't think there was any other way that it could've been done, but that doesn't mean that 8 levels of nesting is any more readable. Unless you use lambdas, Rx code in Android is going to be ugly. And Groovy was announced less than a week ago. There's a huge risk of the project dying off quickly and leaving you with a deprecated app. Given Apple pioneering the Swift language, I think Groovy will be a strong competitor, but that all remains to be seen.

So this post definitely looks far into the future, but that's where I think the scene will be shifting. I'm going to give RxJava (likely with [retrolambda](https://github.com/orfjackal/retrolambda) support) and Grooid (Groovy Android? I think they need a [better name](http://martinfowler.com/bliki/TwoHardThings.html)) a try, so I'll write more posts on those coming up.

