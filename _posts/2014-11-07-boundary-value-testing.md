---
layout: post
title: "Boundary Value Testing"
modified: 2014-11-10 20:20:43 -0500
tags: [functional, core, shell, boundary value, testing]
image:
  feature: 
  credit: 
  creditlink: 
comments: true
share: 
---

*Disclaimer: The ideas presented below were sparked by a presentation available [here](https://www.destroyallsoftware.com/talks/boundaries). Please, please check this out if you get the opportunity, this has been the single most influential 35 minutes in my programming life so far.*
 
Over the past couple of weeks I've been refactoring some of the MinimalBible codebase to use [Kotlin](http://kotlinlang.org/) more extensively. It's a fantastic language that I'm using to take the place of Java development. Much of the existing code is being replaced with functional programming yielding a clean codebase that integrates well with RxJava.
 
What kicked off some of this refactoring was the talk referenced above and an online class I've been taking on Scala. Functional Programming is strange at first, but I've been falling in love with it. Thinking recursively is downright weird, but the quote I found that describes functional programming best for me is [this](http://community.schemewiki.org/?scheme-fortune-cookies):
 
>Functional programming is like describing your problem to a mathematician.  Imperative programming is like giving instructions to an idiot.
 
>--- arcus, #scheme on Freenode
 
So what does functional programming have to do with testing?
 
Splitting the Core
------------------
 
At the heart of the presentation above is a distinction between what the author deems the *core* of a program as opposed to its *shell*. The idea is that a program should be constructed in such a way that its *core* is purely functional, but its *shell* can be imperative.
 
### What a program is ###
 
At the heart of a program is decision making. You have some form of inputs, whether a database, JSON stream, or hard-coded values. The goal is to produce some meaningful output, whether a database, JSON stream, HTML page, etc. How do you get from one point to the next?
 
However you do it, decisions need to be made. Maybe you only need to display the records that were created yesterday. Maybe behavior switches in a mobile application depending on whether you are connected to WiFi. All of these decision points create different possible outputs.
 
So the trick in testing code is to make sure that given the correct inputs, you get the proper outputs. This characteristically involves setting up an in-memory database instead of a real one, simulating a browser connecting to a server, and many different *mock* objects used to simulate the real things.
 
This often ends up leading to tests that take many times longer to set up the environment than to actually run your tests. Instead, what about tests that require no environment set up at all?
 
### Boundary Values ###
 
What I'm proposing sounds kind of crazy at first, but you can structure your app(lication) to need no environment setup whatsoever. **The idea is that you separate values from how you get them.** For example:
 
I have code in MinimalBible that needs to reload books from a server every 30 days, but only if you're on WiFi. If we split out the values from how they're obtained, you can do something like this:
 
{% highlight java %}
int secondsInThirtyDays = 155520000;
   
// Functional Core
public boolean doRefresh(Date currentDate, Date refreshDate,
        Int networkState) {
    if ((currentDate.getTime() - refreshDate.getTime())
            > secondsInThirtyDays &&
        networkState == ConnectivityManager.WIFI)
            return true;
    else
        return false;
}
   
// Imperative Shell
public boolean doRefresh() {
    return doRefresh(
        new Date(),
        SharedPreferences.get("lastRefreshDate"),
        ConnectivityManager.getNetworkState());
}
{% endhighlight %}
 
The functional core is the only code we actually need to test. There are four possible branch conditions we need to test, and they're all very easy to test:
 
{% highlight java %}
Date currentDate = new Date();
Date shortDate = new Date(secondsInThirtyDays - 1);
Date longDate = new Date(secondsInThirtyDays + 1);
 
int wifiState = ConnectivityManager.WIFI;
int nonWifiState = ConnectivityManager.WIFI + 1;
 
assertFalse(currentDate, shortDate, wifiState);
assertFalse(currentDate, shortDate, nonWifiState);
assertFalse(currentDate, longDate, nonWifiState);
assertTrue(currentDate, longDate, wifiState);
{% endhighlight %}
 
So now we have a test suite that 100% covers our code and guarantees exactly what we want. We haven't had to mess with the system clock, haven't had to mess with network state, and there are exactly 0 mock objects.
 
We retain all functionality we originally intended - we can call `doRefresh()` in our application without having to worry about network state or current time. And we no longer need to write tests for `doRefresh()` - there's really not anything to go wrong, it just handles interfacing with the external API's.
 
Scaling Up
----------
 
So far the only example of this principle I've given has been pretty trivial. The principle extends way beyond that example though. At its heart, the idea is to separate your logic from any external considerations. Then, when your logic is liberated this way, you are free to wire up the pieces however you choose.
 
For example, using the `filter()` function happens pretty often:
 
{% highlight kotlin %}
val ints = List(1, 2, 3, 4, 5)
val odds = ints.filter { it % 2 == 1 }
{% endhighlight %}
 
Now all you need to test is the condition `it % 2 == 1`.
 
What I'm still working on though, is how granular to make this. The code below will yield a 100% tested solution, but is incredibly verbose:
 
{% highlight java %}
public boolean isOdd(int value) {
    return (value % 2 == 1);
}
   
@Test
public testIsOdd() {
    assertTrue(1);
    assertFalse(2);
}
{% endhighlight %}
 
I now have 6-8 lines of code (depending on how you count) to test 7 characters of code. This is awful. I'm working on how to scale these tests, but I really like the idea of having 100% coverage without complicated mocking.
 
Wrapping Up
-----------
 
If you haven't watched that presentation now, do it. I'll even give you the [link](https://www.destroyallsoftware.com/talks/boundaries) again.
 
But I hope this explains some of how I'm refactoring the design of MinimalBible. I intend to take full advantage of functional programming for this app, because I think great things can come of it. I'll continue to keep everyone updated on how it's going!
