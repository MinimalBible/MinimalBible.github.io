---
layout: post
title: "The OGHolder Pattern"
modified: 2014-08-02 22:09:10 -0400
tags: [fragment, dagger, mortar, flow]
image:
  feature: 
  credit: 
  creditlink: 
comments: 
share: 
---
Scoped ObjectGraphs to the max
--------------------------------
 
*Technical Disclaimer: The technique detailed below is inspired by the [Mortar](http://corner.squareup.com/2014/01/mortar-and-flow.html) library developed by Square. One of the things Mortar accomplishes for you is allowing scoped ObjectGraphs to be tied to your activity, meaning that when the Activity dies, the things in the ObjectGraph can be garbage collected. And when the screen is rotated, you don't have to re-create the ObjectGraph. I didn't want to use Mortar for my app, but re-created the technique. Check it out.*
 
It's totally possible I'm not the first person to implement everything this way, but I still hope to claim naming rights anyway. I think I came up with a great name. Just so you know, "OGHolder" is the ObjectGraph Holder. But programmers need all the OG status they can get.
 
By any means, let me set out the problem:
 
Scoped `ObjectGraph`s are a feature of [Dagger](http://square.github.io/dagger/) that allow you to do some nifty things. Essentially, the `ObjectGraph` is built in parts, rather than all at once. This means you have a root graph for your application, and each `Activity` has a graph that builds on top of this. There are a couple benefits:
 
1. You don't have to build the entire `ObjectGraph` in one shot. Depending on the modules you inject, this can turn into a significant speed increase.
2. When scoped graphs go out of scope, they are garbage collected - meaning you don't have to store all dependencies in memory for the entirety of an `Application`'s lifecycle.
 
So, given that scoped graphs are pretty hot stuff, how do you go about doing it?
 
**RootModules.java**
 
{% highlight java %}
@Module(
    library=true
)
class RootModules {
    MyApp application;
    public RootModules(MyApp application) {
        this.application = application;
    }
   
    @Provides @Singleton
    MyApp provideApplication() {
        return application;
    }
}
{% endhighlight %}
 
**ActivityModules.java**
{% highlight java %}
@Module(
    injects=MyActivity.class,
    addsTo=RootModules.class // 1
)
class ActivityModules {
 
    @Provides @Singleton
    SomeService provideService(MyApp application) {
        return new SomeService(application);
    }
}
{% endhighlight %}
 
**MyActivity.java**
{% highlight java %}
public class MyActivity extends Activity {
    @Inject
    SomeService someService;
   
    ObjectGraph mObjectGraph;
   
    public void inject(Object o) {
        if (mObjectGraph == null) {
            ObjectGraph root = ((MyApp)getApplication())
                    .getObjectGraph();
            mObjectGraph = root.plus(new ActivityModules()); // 2
        }
        mObjectGraph.inject(o); // 3
    }
   
    @Override
    public void onCreate(Bundle savedInstanceState) {
        inject(this); // 4
       
        // Do other Activity things here...
    }
}
{% endhighlight %}
 
Alright, time for some explanation.
 
1. Our `ActivityModules` class declares itself as adding to the `RootModules` class. **This means Dagger is expecting us to call .plus() to add ActivityModules to RootModules.**
 
  * For the technically inclined, this is semantically different from `@Module(includes=...)`, as the `includes` option tells Dagger to build the included graph at the same time as the root.
   
2. Given that we get the root `ObjectGraph` from our `Application`, we now need to add the `ActivityModules` to it. The way this is done is by calling `.plus()` on the original graph, and storing the new graph.
 
3. We're now ready to inject anybody that needs it, so inject people using the new scoped graph.
 
4. Inject the `SomeService` object into the `Activity` so we can use it.
 
Given all this, we've now arrived at a working activity that can make use of SomeService. However, we do have an issue - whenever the screen is rotated, we'll create a new `SomeService`. If the service is marked as a `@Singleton`, why is it re-created when you rotate the screen?
 
Enforcing @Singleton
--------------------
 
To understand why your singletons aren't actually singletons, you need a little picture of the `Activity` lifecycle. Try checking the documentation [here](http://developer.android.com/training/basics/activity-lifecycle/recreating.html). The important part is this:
 
> Your activity will be destroyed and recreated each time the user rotates the screen.
 
So, let's break down what happens:
 
1. Your initial `Activity` grabs the root `ObjectGraph`, **creates a new graph using `plus()`**, sets up `SomeService`, and then injects itself.
 
2. You rotate the screen, and your `Activity` is destroyed, then re-created.
 
3. The `onCreate()` method is called again now that the `Activity` is created - and when you go to inject again, you find that `mObjectGraph` is null again.
 
4. So, we create another graph, with another `SomeService`, to inject with.
 
So, each `ObjectGraph` enforces the `@Singleton` annotation. However, if you create a new `ObjectGraph`, all bets are off. How do we combine scoped graphs and singletons then?
 
Introducing: The new OG
-----------------------
 
This section of the post is largely based on another [fantastic tutorial](http://www.androiddesignpatterns.com/2013/04/retaining-objects-across-config-changes.html). The basic premise is this: `Fragment`s can be persisted across configuration changes. They can also hold on to Objects without a need to be serialized. **Thus, store our ObjectGraph in the Fragment when it's created, and check there before we build another.**
 
To demonstrate the principle, we need to add only a single class:
 
**OGHolder.java**
{% highlight java %}
public class OGHolder extends Fragment {
                private final static String TAG = "OGHolder";
                private ObjectGraph mObjectGraph;
 
    // Use FragmentActivity for the support library
                public static OGHolder get(Activity activity) { // 1
                    // Use getSupportFragmentManager for support library
                                FragmentManager manager = activity.getFragmentManager();
                                OGHolder holder = (OGHolder) manager.findFragmentByTag(TAG); // 2
                                if (holder == null) {
                                                holder = new OGHolder();
                                                manager.beginTransaction().add(holder, TAG).commit();
                                }
                                return holder;
                }
 
                @Override
                public void onCreate(Bundle savedInstanceState) {
                                super.onCreate(savedInstanceState);
                                setRetainInstance(true); // 3
                }
 
                public void persistGraph(ObjectGraph graph) {
                                mObjectGraph = graph;
                }
 
                public ObjectGraph fetchGraph() {
                                return mObjectGraph;
                }
}
{% endhighlight %}
 
And edit the Activity a little bit:
 
**MyActivity.java**
{% highlight java %}
    // ...
    public void inject(Object o) {
        if (mObjectGraph == null) { // 4
            // Check the holder to see if this is a restart
            OGHolder holder = OGHolder.get(this);
            mObjectGraph = holder.fetchGraph();
           
            // If the holder doesn't have a graph...
            if (mObjectGraph == null) {
                ObjectGraph root = ((MyApp)getApplication())
                        .getObjectGraph();
                mObjectGraph = root.plus(new ActivityModules());
                holder.persistGraph(mObjectGraph); // 5
            }
        }
        mObjectGraph.inject(this);
    }
    // ...
{% endhighlight %}
 
So let's get into the Secret Sauce:
 
1. We have a static method that is responsible for looking up the `OGHolder` tied to this `Activity`. This method works because the `FragmentManager` is scoped to each `Activity` - we can have multiple `OGHolder` fragments with the same tag without worrying about collision.
 
2. Get the `OGHolder` instance. If it doesn't exist, create a new one. When creating a new instance though, make sure to `.commit()`!
 
3. This is the super-special magic piece of code. Long story short, this notifies Android to persist the fragment across restart.
 
4. Now it's time to get our `ObjectGraph`. Get the OGHolder for this `Activity`, and then see if the holder's graph is null.
 
5. If the holder's graph was null, we need to create a new one, and then tie it to the holder to persist it! Finally, inject, and then we're done.
 
And if you run the above code, you'll find that `SomeService` is only ever created once, since we don't lose the graph we created.
 
Summary
-------
 
First things first, as detailed in the tutorial above, please don't use a different method for persisting objects across configuration change/screen rotate. This is the official way of doing things.
 
Second, while you can extend the technique to persist objects other than the `ObjectGraph`, strongly consider whether those objects shouldn't already be in the `ObjectGraph` anyway.
 
Finally, use this technique liberally. It allows you to be efficient both in terms of speed and memory usage. You only need to add a small Fragment and a couple lines to the injection. And in my case, I shaved off seconds of time processing because the expensive singletons didn't get re-created on screen rotate.
 
Hope this is helpful, feel free to contact me if you have questions!
