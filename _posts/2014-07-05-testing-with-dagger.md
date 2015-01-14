---
layout: post
title: "Testing with Dagger"
modified: 2014-07-13 19:19:56 -0400
tags: [dagger, testing, dependency injection]
image:
  feature: 
  credit: 
  creditlink: 
comments: true
share: 
---

Or, why testing on Android is nearly impossible
-----------------------------------------------

So recently I started work to re-design MinimalBible on the new Android Studio platform from the ground up (previously I had just ported the Eclipse version to Android Studio). The biggest impetus for this was me discovering there were significant issues in trying to test MinimalBible. Let's break down how the issues came up.

Dependency Injection
--------------------

Using dependency injection (correctly) is awesome. It allows me to split apart pieces of code for easy testing, and especially with some of the legacy projects I've worked with, it would make life much easier. The framework I'm using for this is [Dagger](https://github.com/square/dagger), a minimal-style JSR-330 compatible injector. One of the cool things about Dagger is that it validates the `ObjectGraph` (an object that stores references to everything that will need injecting) at compile-time, meaning a lot of really dumb errors I've made have been caught way ahead of time.

In actual operation, you need to store a reference to the built `ObjectGraph`, and the place that makes the most sense to do that is the Android `Application`. The Android ecosystem itself enforces that there will only be one application created, so store the graph there, and reference it as need be.

Now, one of the fundamental issues in testing is the use of mock objects. These are objects that look like the original, but the behavior is controlled during the test. So instead of an application getting connected to the `CreditCardPaymentSystem`, it receives a `DummyPaymentSystem`. It can still make calls to the payment system to simulate a purchase, but nobody will actually be charged.

However, the ObjectGraph-inside-Application can get problematic since the Singleton pattern carries with it a number of issues. There can only ever be one `Application`, but who is in control of that `Application`? If the Android OS is responsible for the `Application`, it becomes near-impossible to test - you can't change the `ObjectGraph` used to inject the rest of the application. There are three ways to address this that I've come up with so far.

Run-time re-inject
------------------

This was something I kind of invented myself. It's a really quick and easy way to implement the fixes needed, but is also most likely to lead to significant issues in testing.

The basic idea is this - expose the `Application` enough that you can add the mock objects you need at runtime. An example `Application` would look like this:

**MinimalBible.java**
{% highlight java %}
public class MinimalBible extends Application {

    private ObjectGraph mObjectGraph;

    @Override
    public void onCreate() {
        mObjectGraph = ObjectGraph.create(new MinimalBibleModules());
    }

    public void inject(Object o) {
        mObjectGraph.inject(o);
    }

    public void plusObjGraph(Object[] modules) {
        mObjectGraph = mObjectGraph.plus(modules);
    }
}
{% endhighlight %}

**MinimalBibleTest.java**
{% highlight java %}
public class MinimalBibleModule extends ActivityUnitTestCase<ClassUnderTest> {
    @Module(overrides = true)
    public static class MockModules {
        @Provides SomeObject provideObject() {
            return new MockObject();
        }
    }

    public void setUp() {
        // Assume you get a reference to the application under test somehow...
        mApplication.plusObjGraph(new MinimalBibleMockModules());
    }

    public void testModule() {
        // Test case code here
    }
}
{% endhighlight %}

In a typical flow, the `Application` would be started normally. Then, the test takes over to create the actual `Activity`. The first step is in the `setUp()`, where we re-create the `ObjectGraph` used by the app under test. We over-ride the existing objects with the mock implementation, and the Activity will never know the difference. Then the `testModule()` function can create the Activity, and everything is good to go!

Now, there are two problems with this:
1. We expose the `Application` un-necessarily to injection and over-riding. The whole idea of Dagger is that we can know what the `ObjectGraph` looks like ahead of time. Instead, this new model just kind of hacks around the problem in the first place. We would also have to make sure that nobody used the `plusObjGraph()` method outside of a test case, using it in actual code would be incredibly bad design.

2. I can't prove that this would or would not be an issue, but: continuing to add on and over-ride objects multiple times would likely lead to issues. How does the graph respond to multiple `overrides = true` `@Module`s being used? Also, since tests can be executed without any regard to order, we are left with an inconsistent and unpredictable graph. The over-rides we were expecting to be there now aren't, and over-rides we weren't expecting now show up.

All told, this method works for a proof-of-concept, but is terrible design from the start.

Replace the Application Context
-------------------------------

This technique comes from the [dagger-unit-test](https://github.com/vovkab/dagger-unit-test) project. Shout-out to [vovkab](https://github.com/vovkab) for demonstrating this, I learned an incredible amount about both testing on Android and Dagger from this project.

The method itself involves a lot of under-the-hood shenanigans to make it work, but here's the basic idea:

**[MainActivityUnitTest.java](https://github.com/vovkab/dagger-unit-test/blob/ff42cab4fbc4c128004151634d44af647a019f37/app/src/androidTest/java/vovkab/daggerexample/MainActivityUnitTest.java)**
{% highlight java %}
public class MainActivityUnitTest extends ActivityUnitTestCase<MainActivity> {
    public static final String TEST_HELLO_TEXT = "Test hello";

    private Application mApplication;
    private Context mContext;

    public MainActivityUnitTest() {
        super(MainActivity.class);
    }

    @Module(injects = MainActivity.class)
    final class TestModule {
        @Provides HelloController provideHelloController() {
            HelloController helloController = mock(HelloController.class);
when(helloController.getHello()).thenReturn(TEST_HELLO_TEXT);
            return helloController;
        }
    }

    @Override
    protected void setUp() throws Exception {
        super.setUp();

        mContext = new ContextWrapper(getInstrumentation().getTargetContext()) {
            @Override
            public Context getApplicationContext() {
                return mApplication;
            }
        };

        mApplication = new InjectableAppMock(mContext) {
            @Override public Object[] getModules() {
                return new Object[]{new TestModule()};
            }
        };

        setApplication(mApplication);
    }

    public void testHelloControllerValueSet() {
        setActivityContext(mContext);
        startActivity(MainActivity.createIntent(mContext), null, null);

        // Application context is AppMock
        assertTrue(getActivity().getApplicationContext() instanceof InjectableAppMock);

        TextView helloView = (TextView) getActivity().findViewById(R.id.hello);
        assertEquals(TEST_HELLO_TEXT, helloView.getText());
    }
}
{% endhighlight %}

Here's the basic premise of what's going on:
1. The `Application` objects, both mock and production, implement an interface with a simple `getModules()` method. This method returns the modules used to build the `ObjectGraph` - in the production case, it returns the production modules. The test case returns the modules used for mocking.

2. The real shenanigans happen in the `setUp()` method. What this does is change the existing test `Context`. Specifically, it replaces the `Application` under test with a completely different application that implements the same `getModules()` method. This is how we get around the issues talked about earlier - we just run the tests inside an application we control, rather than the existing `Application`.

3. Now we're prepared to run the actual test - we set the `Context` for the activity to the new one we control, start the `Activity`, and we're good to go.

This method does a good job of avoiding the problems with the first method. Specifically, we don't have to worry about people abusing the internals of the `Application` we created. Second, each `Activity` is tested in a completely isolated `Application`, so we don't have to worry about conflicts, etc. Finally, Dagger is free to do graph validation this way, since we declare the structure of the `ObjectGraph` at compile-time.

However, there are a couple things not so good about this:

1. These tests require intimate knowledge of the way Android operates in order to be effective. If anything goes wrong, it's up to you to fix it, since you're really playing with fire.

2. This is killer: *you can only run unit tests with this style.* Because this method relies on you controlling the `Application` lifecycle, you can not use classes like the `ActivityInstrumentationTestCase2<>`. The Instrumentation test gives Android control of the `Context`, `Application`, and `Activity` lifecycles, so we are unable to mock any objects used in a multi-activity test.

3. Finally, and because of #2 above, Activities are the only objects you can test and mock. Specifically that means `Fragment`s become untestable. Or at the very least, incredibly complicated to set up. I can't prove that it's impossible, but I spent a couple days trying to get Fragments working with this and had no luck. I did however receive plenty of incredibly cryptic errors.

Product Flavors
---------------

This final style of testing is inspired by [U2020](https://github.com/JakeWharton/u2020), but so far as I'm aware, my project is the first to implement/pioneer this. [Product Flavors](http://tools.android.com/tech-docs/new-build-system/build-system-concepts) were introduced in the new Android build system (so they are Android Studio-specific) and allow you to mix and match different versions of the same class. So for example, one version of an `Activity` would show ads, and the second wouldn't.

What I do instead is define a testing flavor that defines the modules to inject for Dagger:

**[MinimalBibleApplication.java](https://github.com/MinimalBible/MinimalBible/blob/46f8e625c278118889bb76423d772cf0204f795b/app/src/main/java/org/bspeice/minimalbible/MinimalBible.java)**
{% highlight java %}
public class MinimalBibleApplication extends Application {
    private ObjectGraph mObjectGraph;

    @Override
    public void onCreate() {
        super.onCreate();
        buildObjGraph();
    }

    public void buildObjGraph() {
        mObjectGraph = ObjectGraph.create(Modules.list(this));
    }

    public void inject(Object o) {
        mObjectGraph.inject(o);
    }

    public static MinimalBible get(Context ctx) {
        return (MinimalBible)ctx.getApplicationContext();
    }
}
{% endhighlight %}

This class still acts as a Singleton - everybody can come here to find the `ObjectGraph`, but the graph is build using the results of `Modules.list()`. The question then becomes - what the heck is `Modules`?

**[main/Modules.java](https://github.com/MinimalBible/MinimalBible/blob/46f8e625c278118889bb76423d772cf0204f795b/app/src/mainConfig/java/org/bspeice/minimalbible/Modules.java)**
{% highlight java %}
public class Modules {
    private Modules() {}

    public static Object[] list(MinimalBible app) {
        return new Object[] {
                new MinimalBibleModules(app)
        };
    }
}
{% endhighlight %}

**[test/Modules.java](https://github.com/MinimalBible/MinimalBible/blob/46f8e625c278118889bb76423d772cf0204f795b/app/src/testConfig/java/org/bspeice/minimalbible/Modules.java)**
{% highlight java %}
public class Modules {
    private Modules() {}

    public static Object[] list(MinimalBible app) {
        return new Object[] {
                new MinimalBibleModules(app),
                new TestModules()
        };
    }
}
{% endhighlight %}

Here we are presented with two files of exactly the same name. How are we to tell which wone we are to use? That's where the Product Flavors come in. Since these files are stored in different directories (main vs. test) they are used by the *Main* and *Test* flavors respectively. Thus, when we test, use the *Test* flavor. Otherwise, use the *Main* flavor.

This style excels at a couple things:

1. Incredibly clean design. You program the wiring separate from the `Application` itself.

2. You don't have to mess with Android internals to enable testing. The only *real* difference is that you just add the `TestModules()` at test time, and they over-ride the main modules.

3. Scalable - because you aren't messing with internals, and you're using the injection system the way it was intended, you won't have to worry about really strange errors. Trust me, there's a lot of really obscure things that can come up.

4. You can actually test - the problem with touching the Android run-time is that you make the test cases intrinsically tied to the `Application`/`Activity` lifecycle. The first method only worked with instrumentation test cases (since it was assumed the `Application` had already been started for you to inject it), the second only with unit test cases (since you had to create your own `Application`). This way allows you to program to whatever style you need.

5. Not `Application` level-specific, like the second method. The same rules apply there as here.

However, there are some legitimate challenges:

1. Having to maintain different flavors for testing vs. production. You don't have a unified folder containing all the code. This is maybe a bit OCD of me, but I think it's frustrating to have the main code folder, test case folder, the production `@Module`s, and the mock `@Module`s in their own folders. Trying to keep everything in sync is a challenge (albeit much easier than the challenges presented by above methods)

2. This only works for Android Studio. Product flavors are specific to Android Studio, and thus require other people to learn how your product flavors are used.

3. Module conflicts - If different test cases need different mock modules, you are in royal trouble (the first method could theoreticaly get around this using scoped `ObjectGraphs`). I don't have a specific use-case in mind of why you would need this, but it will require some extra coding to make it work. The only thing to be done in this case is to add extra methods to the mock module specific to the test case.

Summary
-------

After having spent some time starting from scratch trying to re-implement testing in MinimalBible, I'm grateful for the effort I put in. Finding these issues now instead of integrating testing later has certainly saved me an incredible amount of time and frustration. Plus, I learned enough about how the testing system works to understand how to write tests in the future.

One point needs to be made though - [RoboGuice](https://github.com/roboguice/roboguice) and other similar dependency injection frameworks won't have this issue, since your dependencies are created at run-time. The issues described above are mostly specific to Dagger (although I suspect [Android Annotations](http://androidannotations.org/) would run into the same problems).

So all this said, let me take a moment to echo the concerns of an [influential Android developer](http://jakewharton.com/android-needs-a-simulator/) that testing on Android is awful right now. He's more so talking about test speed, while I've spent far more time trying to figure out how to make my tests work. Technically different concerns, but we're both frustrated with how testing is currently structured. And while I think it will end up serving me well, I was practically forced to re-structure my application to make this all work. I can only imagine how frustrating testing was before product flavors.

Thanks for sticking with me through the post, hope you learned a lot!
