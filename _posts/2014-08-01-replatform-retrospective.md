---
layout: post
title: "Replatform Retrospective"
modified: 2014-08-02 22:11:40 -0400
tags: [replatform, android studio, eclipse, hindsight, testing]
image:
  feature: 
  credit: 
  creditlink: 
comments: 
share: 
---
 
So you don't make the same mistakes.
------------------------------------
 
Over the past couple of weeks I re-built MinimalBible fully on top of Android Studio. Previously when starting this project, I used Eclipse for a couple reasons. First being Eclipse was what was currently used in production. Second, the foundational library I rely on ([jSword](https://github.com/crosswire/jsword)) was an existing Java library, so I would have to engineer the Gradle build myself to make it compatible with Android Studio.
 
Now all this was well and good, but I eventually wanted to migrate to Android Studio. It was new and robust, could do cool things, didn't have to mess with plugins, etc. After having used Android Studio for a while, it seems like most reputable libraries are moving there. Plus, IntelliJ is simply a better IDE than is Eclipse.
 
However, the migration process ended up being fairly challenging. I spent days migrating the jSword build to Ant, plus figuring out the random paths in the build.gradle file, plus the [apt](https://bitbucket.org/hvisser/android-apt) library for annotation processing... But the straw that broke the camel's back was testing. I had so many problems figuring out how to make Unit Tests work, I eventually gave up. It was time for a re-design, and I'm glad I did.
 
So, MinimalBible was built from the ground-up all over again. It ended up not being nearly as time-consuming or complicated as I expected. But, here's some of the lessons I've learned in the process. The idea is that nobody should have to make these errors again.
 
Lesson 1: Keep testing in mind
------------------------------
 
One of the things I was looking forward to in switching to Android Studio was getting testing working. The way I had originally structured the application, it was borderline impossible to use Mock objects. I have more of the specifics in another post **link here**, but here's the problem in a nutshell:
 
**Problem:**
 
* Everything being injected needs to get access to the objects it is being injected with
* Using the `Application` object seems like a good idea - it's available globally, so we can store all the dependencies there.
* There's no way to swap out the `Application` instance during testing to provide mock objects. And any modifications done to the instance (like forcibly over-riding objects) are global - it becomes impossible to guarantee a consistent environment.
 
**Solution:**
 
**Build in testing from the start.** It's been really helpful for me to be able to change code and still have a sanity check to make sure I didn't break anything. My code coverage right now is still terrible, but I have a platform to make sure I can actually do this going forward.
 
Lesson 2: Static objects are awful
----------------------------------
 
Every enterprise application I've worked on, and most Android applications, all have a dependency injection system of some form. I've previously outlined a number of them **link here**, but I settled on Dagger.
 
**Problem:**
 
* Many times you need to ensure that only one instance of an object exists during execution
* Add a static method to retrieve an instance of the object - everyone goes through the static method
* This guarantees testing is impossible - you can't ever change the static method everyone else relies on. Thus, you can test that method, but never change it.
 
**Solution:**
 
**Use your dependency injector the way it was meant to be used.** I'm talking as few static references to anything as possible. Two examples of how I do this:
 
### Configuration as code:
Both modules provide a singleton list that is used as configuration. It will not change during the application lifecycle, and I probably won't touch it again during development. But if I want to test how the objects inside the list are used, I can. This isn't possible if the "valid categories" are a static list.
 
**MainConfig.java**:
{% highlight java %}
@Module()
class MainConfig {
    @Provides @Singleton
    List<BookCategory> provideValidCategories() {
        return new ArrayList<BookCategory>() {{
            put(Category1);
            put(Category2);
        }};
    }
}
{% endhighlight %}
 
**TestConfig.java**:
{% highlight java %}
@Module()
class TestConfig {
    @Provides @Singleton
    List<BookCategory> provideValidCategories() {
        return new ArrayList<BookCategory>() {{
            put(mockCategory1);
        }};
    }
}
{% endhighlight %}
 
### Dynamic Injection references:
I've been talking a lot about testing, and here's some more. Bear with me.
 
One of the problems I ran into during testing was how objects being injected got access to the graph of their dependencies. Previously, I would call something like `MyApp.inject(this)`, relying on a static reference to `MyApp`. Instead, each object should be given a interface that they can inject from, and you can worry about the interface implementation elsewhere. Consider the below:
 
**MyFragment.java**
{% highlight java %}
class MyFragment extends Fragment() {
    public static MyFragment newInstance(Injector i) {
        MyFragment f = new MyFragment();
        i.inject(f);
        return f;
    }
}
{% endhighlight %}
 
**MyActivity.java**
{% highlight java %}
class MyActivity extends Activity implements Injector {
    // We're going to assume that the ObjectGraph is created
    // elsewhere
    private ObjectGraph mObjectGraph;
    private void inject(Object o) {
        mObjectGraph.inject(o);
    }
   
    @Override
    public void onCreate(Bundle savedInstanceState) {
        MyFragment f = MyFragment.newInstance(this);
    }
}
{% endhighlight %}
 
**MyActivityTest.java**
{% highlight java %}
class MyActivityTest extends TestCase implements Injector {
    // Again, created elsewhere
    private ObjectGraph mObjectGraph;
    private void inject(Object o) {
        mObjectGraph.inject(o);
    }
   
    public void testMyFragment() {
        MyFragment f = MyFragment.newInstance(this);
        // Actually do the testing stuff...
    }
}
{% endhighlight %}
 
**Injector.java**
{% highlight java %}
interface Injector {
    public void inject(Object o);
}
{% endhighlight %}
 
So now after all this code we have a `Fragment` that can be injected by both the actual activity, or run in isolation with a TestCase. Nice.
 
Lesson 3: Design choices matter
-------------------------------
 
Probably the most important lesson I can convey: **If you see a bad pattern, now is the time to fix it.** There were a number of different things that this re-platform allowed me to fix, and apply lessons that I've learned. And working in enterprise, I've seen how challenging it is to refactor code that's stuck in a broken design pattern. Every bit of time put in to make sure you start with good design pays incredible dividends.
 
So if you think that you have a better design idea, now's the time to make it happen. Don't let technological [cruft](http://en.wikipedia.org/wiki/Cruft) build up, be ruthless about cutting that junk out. If you see a problem early, fix it. There are so many ways to get the point across, but it really is that important.
 
Conclusion
----------
 
I learned a lot from re-implementing MinimalBible, and it's been great being able to re-think and start fresh. Plus, catching these things early makes life so much easier later on.
 
Hope this is helpful for making sure you can avoid issues in the future!
