---
layout: post
title: "AsyncTask caching"
modified: 2014-04-29 17:36:19 -0400
tags: [singleton, asynctask, eventbus, caching, fragment, asynchronous]
image:
  feature: 
  credit: 
  creditlink: 
comments: true
share: 
---

Singletons, AsyncTasks, & EventBus! Oh my!
------------------------------------------
 
So, MinimalBible is my first full-scale Android app. I'm trying to do what I can to make sure the code is readable and follows good design patterns. To that end, let me describe the problem:
 
I need to fetch the list of modules available for download (may or may not be from the internet, not worried about that here). To do so is a relatively expensive operation (since it may involve the internet) so using the `Fragment` that displays the module list is right out. Some potential solutions:
 
* Eat the cost of fetching the list, each `Fragment` fires an `AsyncTask`
* The `Fragment` is initialized with the list it needs
* Have some external singleton class cache the list in memory, and each fragment can retrieve it from that class
* Have a service responsible for doing the download, everyone comes to the service
* If no value exists, Fragment displays that there is no information, and manually force a reload when the information has been retrieved
 
This is complicated by the fact that the `Fragment` should be able to retrieve the list synchronously if it has already been loaded. That is, the Fragment needs to know if the value exists and it can do its work, or if it needs to wait a bit. Since the Fragment is on the UI thread, I can't assume a synchronous operation unless the value is already prepared.  
So the basic problem is: **I need to get a value asynchronously, but how do I cache it to re-use? Or, who is responsible for caching?** None of the solutions above seem to have a sound answer to the problem. 
The solution I chose then doesn't fundamentally solve the problem either. But I'm more than happy to accept suggestions on how to do this differently!
 
First off, somebody has to be responsible for doing the work of actually fetching the modules, whether from the internet, or from the jSword cache. Because fetching from cache is still an expensive operation (2-3 sec.), it's easier to have one `AsyncTask` responsible for doing all of that work.
 
Second, I only ever want this `AsyncTask` to run once. The module list (hopefully) can stay persisted in memory to be re-used by the different `Fragment`s that need it. That means that there needs to be a Singleton somewhere along the line that's responsible for caching the value.
 
Finally, that Singleton has a method available to inspect whether the content the `AsyncTask` was fetching exists yet.
 
That's the basic methodology I used. However, instead of caching the returned value in the parent Singleton, I used the [EventBus](https://github.com/greenrobot/EventBus) to handle it for me. This way, **I avoid the boilerplate of an Interface/Listener pattern, and I avoid the [Sleeping Barber](http://en.wikipedia.org/wiki/Sleeping_barber_problem) problem.**
 
The EventBus is an asynchronous communication bus that allows us to connect senders and receivers of POJO objects. This way, the Singleton I was describing above retains a reference to the bus the `AsyncTask` will post on, and connects listeners to it. After that, the EventBus has a concept of *sticky events*, which are events persisted in memory until explicitly cleared. 
So, let's get into some actual pseudo-code! 
 
**FetchTask.java**

{% highlight java %}
class FetchTask extends AsyncTask<...> {
    private EventBus downloadBus;
    public FetchTask(EventBus downloadBus) {
        this.downloadBus = downloadBus;
    }
   
    public doInBackground(...) {
        // Do the deed
        downloadBus.postSticky(results);
        return results;
    }
}
{% endhighlight %}
 
**DownloadManager.java**

{% highlight java %}
class DownloadManager {
    private EventBus downloadBus;
    private DownloadManager instance;
 
    private DownloadManager() {}
    public static DownloadManager getInstance() {
        if (instance == null) {
            instance = new DownloadManager();
            instance.downloadBus = new EventBus();
            new FetchTask().execute();
        }
    }
    public EventBus getDownloadBus() {
        return this.downloadBus;
    }
}
{% endhighlight %}
 
**Fragment.java**

{% highlight java %}
class Fragment {
    public void init() {
        EventBus downloadBus = DownloadManager.getInstance().getDownloadBus();
        Results results = downloadBus.getStickyEvent(Results.class);
        if (results == null) {
            // The operation hasn't finished yet, so notify us when done
            downloadBus.registerSticky(this);
        } else {
            // Operation is already done, initialize now
            initUI(results);
        }
    }
   
    public void onEventMainThread(Results results) {
        // EventBus will call this
        initUI(results);
    }
   
    public void initUI(Results results) {
        // Initialize the UI - can safely assume we're on main thread
    }
}
{% endhighlight %}
 
So, let's quickly review what's going on - the Fragment gets initialized, and gets the Download manager, so it can get the event bus. Then, since the AsyncTask is started alongside the new `DownloadManager`, we register that we should be notified when done.
 
Then the DownloadManager starts the initial AsyncTask to download the list, and cache it in the EventBus. Finally, we `postSticky()` and call it a day.
 
After this, any new `Fragment`s checking the `downloadBus` will see that the value already exists, and can do work.
 
So this isn't a perfect implementation, and honestly, just shoves the problem onto the `EventBus`. That being said, it's a sane implementation, and is relatively readable.
 
**TODO:** Switch around some of the singletons to use dependency injection. I've got my eye on [Dagger](http://square.github.io/dagger/).
