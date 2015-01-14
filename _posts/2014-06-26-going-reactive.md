---
layout: post
title: "Going Reactive"
modified: 2014-06-28 23:22:39 -0400
tags: [rxjava, reactive, rx, lambda, monad, observable]
image:
  feature: 
  credit: 
  creditlink: 
comments: true
share: 
---
 
What happens when everything is asynchronous...
-----------------------------------------------
 
So, I don't know if some people just had a conversation one day on how to deal with the problems inherent in asynchronous systems, but if they did, I think Reactive (Rx) is what likely came about as a result of that conversation. For anyone who's had to write any amount of Javascript, you've likely had to deal with the problems of "Spaghetti code." You make calls to JQuery, JQuery returns a result, and then you need to update the page with that new result. Sounds easy enough, but when almost your entire page is meant to be AJAX-enabled, you very quickly have issues of trying to figure out what the current state of the page is, updating it, etc. The "spaghetti" moniker is from the image of code reaching across the global state, and it being impossible to untangle who modifies what and when. Technologies like [Angular-js](https://angularjs.org/), [Knockout-js](http://knockoutjs.com/), and [Backbone.js](http://backbonejs.org/) attempt to remedy these issues through numerous techniques.
 
The [Reactive Extensions](http://msdn.microsoft.com/en-us/data/gg577609.aspx) are a set of tools built for the .NET platform that allow you to define your interactions as a series of transformations applied to data. This way, each transformation is capable of responding to errors, and can "react" to the data of the transformation before it. The entire structure is incredibly similar to functional programming (which I'm totally okay with). And if you've worked with [Monads](http://en.wikipedia.org/wiki/Monad_(functional_programming) before, you'll feel right at home. If you have no idea what Monads are, read [this blog post](http://mttkay.github.io/blog/2014/01/25/your-app-as-a-function/). It's by far and above the best explanation I've ever heard.
 
So, of course Reactive is not just a .NET thing, because otherwise I wouldn't be writing this post. Netflix is at work implementing the same API on Java via [RxJava](https://github.com/Netflix/RxJava), and I think it may just be the best thing to happen to the JVM behind it's creation. Lambdas are a close third.
 
A time for experimenting...
---------------------------
 
I recently finished the first feature-level functionality of MinimalBible. As of a couple weeks ago, you can now download and remove Bibles and other books from the device. Given that I had to learn a lot to make it to this point (Dependency injection, Android Studio/Gradle, etc.) I think that's definitely an accomplishment. But, before moving on the next stage, I wanted to take a moment and play around with the existing code base that I had. Two projects that I thought were really cool that I wanted to give a shot were RxJava (and [retrolambda](https://github.com/orfjackal/retrolambda)), and [Grooid](http://melix.github.io/blog/2014/06/grooid.html).
 
Long story short, I don't think Grooid is ready for the big time. It looks cool, but I wasn't able to get it set up. While it could definitely shorten development time (the language itself looks fantastic) in the mean time there are plenty of libraries available for Android that can similarly help.
 
Rx on the other hand, has been a blast to use. It will definitely force you to think differently (unless you come from a functional-language world - Ruby/Python count too), but it has the potential to make life much easier.
 
Going Reactive
--------------
 
The idea behind Rx is that you have data ([Observables](https://github.com/Netflix/RxJava/wiki/Observable)), transformations on that data, and then Subscribers doing things with that data. A real-world example for MinimalBible is that I have a list of Installer objects (that are able to download/install Bibles) and I want to do things with it. An example might be something like this (for sake of brevity, I'm using lambda notation):
 
{% highlight java %}
List<Book> finalList = new ArrayList<Book>();
   
Observable.from(getAllInstallers()) // Get the installers
    .map((Installer i) -> i.getBooks()) // Get the books from each installer
    .reduce((new ArrayList<Book>(), accumulator, bookList) -> accumulator.addAll(bookList)) // Compile the overall list
    .subscribe((List bookList) -> displayList(bookList)) // Display all books in one shot
{% endhighlight %}
 
That's pretty cool, right? I have a group of installers to start with, and I end up with all the books I can install. But, those installers get their information from the Internet, and Android doesn't let me do networking on the main thread. Let's fix that.
 
{% highlight java %}
List<Book> finalList = new ArrayList<Book>();
   
Observable.from(getAllInstallers())
    .map((Installer i) -> i.getBooks())
    .subscribeOn(Schedulers.io()) // Do the operations on a different thread
    .reduce((new ArrayList<Book>(), accumulator, bookList) -> accumulator.addAll(bookList))
    .subscribe((List bookList) -> finalList.addAll(bookList))
{% endhighlight %}
 
It took me exactly one line of code to handle doing everything asynchronously. No `AsyncTask` listeners, callbacks, nothing. Just pure clean [asynchronicity](http://en.wikipedia.org/wiki/Synchronicity_(The_Police_album\)).
 
Now, trying to run this code can be pretty expensive, but the results don't change that often. What Rx allows me to do is [cache](https://github.com/Netflix/RxJava/wiki/Observable-Utility-Operators#cache) the results so that anyone who needs them later is automatically provided with them.
 
{% highlight java %}
List<Book> finalList = new ArrayList<Book>();
   
Observable.from(getAllInstallers())
    .map((Installer i) -> i.getBooks())
    .reduce((new ArrayList<Book>(), accumulator, bookList) -> accumulator.addAll(bookList))
    .cache() // Subsequent calls will complete immediately
    .subscribeOn(Schedulers.io())
    .subscribe((List bookList) -> finalList.addAll(bookList))
{% endhighlight %}
 
Now, the list of books will only be compiled once and anyone can use it. In a very clean, coherent manner, I've accomplished an incredible amount. That being said, if I want to use this list on the UI side, I somehow need to get on to the main thread. Not too complicated:
 
{% highlight java %}
List<Book> finalList = new ArrayList<Book>();
   
Observable.from(getAllInstallers())
    .map((Installer i) -> i.getBooks())
    .reduce((new ArrayList<Book>(), accumulator, bookList) -> accumulator.addAll(bookList))
    .cache()
    .subscribeOn(Schedulers.io())
    .observeOn(AndroidSchedules.mainThread()) // Receive the objects on the UI thread
    .subscribe((List bookList) -> finalList.addAll(bookList))
{% endhighlight %}
 
So I started with a list of installers, I now have a list of books. In between the list is cached, and Android is happy about networking done off the main thread, UI manipulation on the main thread. All in 8 lines of code.
 
There are two important downsides to this though:
 
Inner-class shenanigans
-----------------------
 
The first (obvious) downside to using RxJava is that you are trying to make a functional-style library work in a imperative/procedural language. This means that you will have a large number of inner classes that are created in order to accomplish your goals. I kind of cheated a bit to get the lines of code down to 8 above - in actual code, it would be closer to 20 counting all of the Anonymous classes made. That being said, 20 lines of code is not a lot for the amount of functionality it generates.
 
So I don't think inner classes are a big issue for two reasons. First, and this is not a great reason, is that you can enable lambdas on Java prior to Java 8 via the retrolambda library. Unfortunately, this requires that everyone who works on your project uses Java 8.
 
The second reason is that Android Studio will "fold" the code in the editor for you. So even though you write out those classes once, you can hide them to keep everything from being ugly. Quick, simple, easy, and it's built into the system.
 
Reactive Everywhere
-------------------
 
The second problem with using RxJava, or reactive in general, is that there really isn't a good way to do things half-heartedly. Technically you can, but it's not simple. In my experience, once you start with reactive programming, you want everything to be reactive. Simply put, it is nigh on impossible to port a significant code base to using a reactive style, and so I would not recommend this for existing projects.
 
That being said, it's doable - you can use [BlockingObservable](https://github.com/Netflix/RxJava/wiki/Blocking-Observable-Operators)s to make your API play nice, and then port your high-level code to taking advantage of the `Observable`s. From what I did though, it was a lot easier (and a lot more fun) to just convert everything all at once.
 
Conclusion
----------
 
So, working with RxJava has been a fantastic experience, and I absolutely love it. Shout out to the guys at [Netflix](https://github.com/Netflix) for making this possible, it's certainly made development a lot easier for me. Looking forward to continuing to use Rx, and I'll be writing more soon!
