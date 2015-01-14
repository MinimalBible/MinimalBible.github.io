---
layout: post
title: "Bootstrapping Dagger"
modified: 2014-05-18 16:41:55 -0400
tags: [dagger, dependency injection, di, inversion of control, jsr330]
image:
  feature: 
  credit: 
  creditlink: 
comments: true
share: 
---
Because everyone needs more dependency injection
------------------------------------------------
 
*Clarification: If you work in enterprise Java, you probably don't. But most Android projects I've seen seem like they could benefit.*
 
So, I've covered [Dagger](https://github.com/square/dagger) previously in trying to evaluate a DI framework for MinimalBible. I did settle on using Dagger + ButterKnife for my injections, as Dagger provides compile-time validation, and I'm not forced into someone else's lifecycle (like [Android Annotations](https://github.com/excilys/androidannotations)).
 
Getting Dagger set up then was a bit of an interesting experience. Maybe I just wasn't looking hard enough (or scanning the [docs](http://square.github.io/dagger/) too quickly), but I was totally unaware of how to get the code running. When I initially added my `@Inject` handlers, I kept getting the feared [NPE](http://docs.oracle.com/javase/7/docs/api/java/lang/NullPointerException.html). This was the result of me fundamentally misunderstanding how Dagger as a whole works, so let me see if I can't break it down.
 
Compile Time (Modules)
----------------------
Like I said above, part of the benefit of using Dagger is compile-time validation. Well, in order to do that, you have to declare the entire structure of your dependency injection. Don't worry, it's not actually that complicated. To do so, we create `@Module` annotated classes.
 
**BasicModule.java**
{% highlight java %}
@Module (
    injects = MyApp.class;
)
public class MyModule{}
{% endhighlight %}
 
We've now got our Module! This module is responsible for handling the dependencies of `MyApp`, and that's all it cares about.
 
**MyApp.java**
{% highlight java %}
public class MyApp {
    @Inject MyObject object;
   
    public static void main(String[] args) {
        System.out.println(object);
    }
}
{% endhighlight %}
 
And now the basic app is set up. We'll get to the Android implementation later. Let's define `MyObject`, and then we'll be ready to run!
 
**MyObject.java**
{% highlight java %}
public class MyObject {
    @Override
    public String toString() {
        return "I'm alive!";
    }
}
{% endhighlight %}
 
So if you try and run `MyApp` right now, you'll get a NPE. But wasn't Dagger supposed to do the dependency injection for us? And the fact that we were even able to run this at all means that the validation *must* have run correctly...
 
Actually, that second point is correct. The validation has run at this point, and you're good to go. Because `MyObjection` has a no-arg constructor, Dagger is able to do everything it needs to set up dependency injection. The problem is, we need to actually set up the object graph and inject ourselves.
 
Generating the object graph is pretty simple - we just need to add a couple lines to `MyApp` so that Dagger builds in memory all the dependencies, and then can inject them.
 
**MyApp.java**
{% highlight java %}
    // We only need to modify `main()`
    public static void main(String[] args) {
        ObjectGraph objGraph = ObjectGraph.create(BasicModule.class);
        objGraph.inject(this);
        System.out.println(object);
    }
{% endhighlight %}
 
Huzzah! It's now working. Now here is where I started messing myself up. Let's say I wanted to use injection on another class. I'll modify the `@Module` to inject the next class.
 
**BasicModule.java**
{% highlight java %}
@Module (
    injects = {MyApp.class, AnotherClass.class};
)
{% endhighlight %}
 
And the next class:
 
**AnotherClass.java**
{% highlight java %}
public class Another Class() {
    @Inject MyObject object;
   
    public String toString() {
        return object;
    }
}
{% endhighlight %}
 
And update `MyApp` to use the new class:
 
**MyApp.java**
{% highlight java %}
    // Again, only need to modify `main()`
    public static void main(String[] args) {
        ObjectGraph objGraph = ObjectGraph.create(BasicModule.class);
        objGraph.inject(this);
        System.out.println(new AnotherClass());
    }
{% endhighlight %}
 
So now `MyApp` delegates the `object.toString()` call to `AnotherClass`. Well, trying to run this code will give you another `NullPointerException`. And this is where I misunderstood how Dagger actually works: **When using Dagger, each class is responsible for injecting itself.** This gives me flexibility over when in the lifecycle the injection happens, but I was originally under the impression that Dagger just auto-magically did injection for me.
 
Now, if each object handles its own injection, you'll remember that we had to construct an ObjectGraph for actually injecting. The ObjectGraph is relatively expensive to compute, so we don't want to keep rebuilding it every time we need to do injection (it doesn't change anyway, unless you're using *[plus()](http://square.github.io/dagger/javadoc/dagger/ObjectGraph.html#plus(java.lang.Object...)*, in which case you likely don't need to be reading this article).
 
This is where the Android-specific stuff comes in. Long story short, the `Application` itself will stay around for as long as your application is running. Seems like a great place to store the `ObjectGraph`, as there are a lot of people who will need it.
 
I'm not going to go into the specifics, as it should be pretty obvious how to implement that given what I've outlined above (the way I did it was create the graph during `onCreate()`, and hold it as a reference inside the actual `Application` singleton).
 
Hope that gives you guys a better understanding of how to bootstrap your Dagger projects, it's definitely been awesome so far getting to use it!
