---
layout: post
title: "Robolectric + Android Lollipop"
modified: 2015-02-23 00:29:49 -0500
tags: [robolectric,lollipop,sdk21,testing,coverage]
image:
  feature: 
  credit: 
  creditlink: 
comments: true
share: 
---

Getting actual Android code running on your computer
----------------------------------------------------

So far in the life of [MinimalBible](0) the
whole testing situation has been... *eherm*... convoluted. I've previously outlined why
I [replatformed](1) the application, how managing the `Application` lifecycle [makes testing
difficult](2), and finally how I've gotten [non-emulator testing set up](3) to report code coverage.

One of the things I've wanted to do for a while, but proved difficult after many long nights with
StackOverflow and other technology sites was get the [Robolectric](4) project working with my code.
Robolectric is basically a re-implementation of the Android SDK designed for easy testing. So instead
of having to run Android tests on an actual device, you can skip that.

Quick caveat: [Don't write UI tests using Robolectric](5). Robolectric is great for running small tests
that have to use Android API's, but should not be used to validate that your UI code is working. You won't
have an emulator or other device to make sure you actually wrote the tests the right way.

And with that out of the way, let me get into what it took for me to get Robolectric up and running for MinimalBible.
I imagine there will be other people with similar issues, so it's good to go ahead and talk about. All the code
this post is based off of can be found at commit [71fb362ffe](6).

Assuming you have the project set up as discussed [in my previous post](3) (which was inspired by [this](7)),
the `build.gradle` file change is pretty simple. Just add the following dependency:

```groovy

dependencies {
     testCompile 'org.robolectric:robolectric:2.+'
}
```
    
We need to add a special file called `project.properties` to the `<app_name>/src/main` folder with this content:

```

# suppress inspection "UnusedProperty" for whole file
android.library.reference.1=../../build/intermediates/exploded-aar/com.android.support/appcompat-v7/21.0.3
android.library.reference.2=../../build/intermediates/exploded-aar/com.android.support/support-v4/21.0.3

```

The "suppress" line at the top is so that Android Studio doesn't warn you that the properties are unused.
    
The `Activity` we're actually going to be testing is pretty simple too ([code here](8)):

```java

public class BasicSearch extends BaseActivity {
    
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_search_results);
       
        handleSearch(getIntent());
    }
}
```
    
So now that the `Activity` is set up, we need to build a simple test case. Let's do something like this ([code here](9)):

```java

@RunWith(RobolectricTestRunner.class)
@Config(emulateSdk = 18, manifest = "../app/src/main/AndroidManifest.xml")
public class BasicSearchTest {
    @Test
    public void testBuildActivity() {
        BasicSearch activity = Robolectric.buildActivity(BasicSearch.class)
            .create().get();
        assertNotNull(activity);
    }
}
```
    
This test is just responsible for getting the `Activity` started, and will fail if there are any issues during startup.
Please note that this also includes the `Application` getting started as well,
so it may throw off your coverage metrics.

By any means, we're now done with the hardest part. With any luck, the test will run, and you'll get a nice green bar
telling you that the test was successful.

There's one quick caveat: **When you run tests with Gradle, all tests are run in the application test project folder.**
In my case, that's the `app-test` folder. This is important, because **Android Studio by default likes to run tests
in the project root directory instead**. So if the tests work with Android Studio, and not with Gradle,
that's likely the issue.

**But** if it was actually that easy, it wouldn't be worth yet another post.

Robolectric + AppCompat - The deadly duo
----------------------------------------

As of right now, Robolectric has very limited support for Android Lollipop (SDK 21), and most specifically the
`appcompat` libraries it ships with. Unfortunately it appears that the `ActionBarDrawerToggle` is causing some
[rather hairy problems](10) that aren't easy to figure out.

So if you're trying to get Robolectric working on a Lollipop app, here are my tips from the school of hard knocks:

1. You can only test Activities that do not use the `ActionBarDrawerToggle`. This thankfully means that `Fragment`
testing will work just fine if you set them up maybe outside the original intended `Activity`.

2. Please make sure that all tests receive an `@Config(emulateSdk = 18)` until [this issue](11) is closed.

3. Please make sure that Robolectric picks up the support libraries when testing. If they are noticed correctly,
the test will output text which includes the two lines below:

{% highlight text %}
DEBUG: Loading resources for android.support.v7.appcompat from ./../app/src/main/../../build/intermediates/exploded-aar/com.android.support/appcompat-v7/21.0.3/res...
DEBUG: Loading resources for android.support.v4 from ./../app/src/main/../../build/intermediates/exploded-aar/com.android.support/support-v4/21.0.3/res... 
{% endhighlight %}

And at this point if you've followed all the tips, you should be good to go!

Summary
-------

So now that we've gone over how to get Robolectric working, and some of the tricks you need to get it working in
Lollipop, go forward and write tests. Who knows, it might bump up your coverage metrics [11%](12).

Comment below if you have any questions, I'll do what I can to help!

[0]: https://github.com/MinimalBible/MinimalBible
[1]: http://minimalbible.github.io//replatform-retrospective/
[2]: http://minimalbible.github.io//testing-with-dagger/
[3]: http://minimalbible.github.io//android-and-jacoco/
[4]: http://robolectric.org/
[5]: https://github.com/futurice/android-best-practices#use-robolectric-for-unit-tests-robotium-for-connected-ui-tests
[6]: https://github.com/MinimalBible/MinimalBible/commit/71fb362ffea4d7c7abfc8a7615e7db216be50e4e
[7]: http://blog.blundell-apps.com/android-gradle-app-with-jvm-junit-tests/
[8]: https://github.com/MinimalBible/MinimalBible/blob/71fb362ffea4d7c7abfc8a7615e7db216be50e4e/app/src/main/java/org/bspeice/minimalbible/activity/search/BasicSearch.java
[9]: https://github.com/MinimalBible/MinimalBible/blob/71fb362ffea4d7c7abfc8a7615e7db216be50e4e/app-test/src/test/java/org/bspeice/minimalbible/activity/search/BasicSearchTest.java
[10]: https://github.com/robolectric/robolectric/issues/1424
[11]: https://github.com/robolectric/robolectric/issues/1446
[12]: https://coveralls.io/builds/1946159
