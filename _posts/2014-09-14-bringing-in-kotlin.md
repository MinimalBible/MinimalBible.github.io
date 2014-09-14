---
layout: post
title: "Bringing in Kotlin"
modified: 2014-09-14 14:59:47 -0400
tags: [kotlin, java, dependency injection, lambda]
image:
  feature: 
  credit: 
  creditlink: 
comments: 
share: 
---

An Alternative Language that works
----------------------------------

So, not too long ago I wrote a post on why I wasn't using [Groovy]({% post_url 2014-09-01-why-not-groovy %}) for my app. Suffice to say that I wanted to use Groovy, there were a lot of good things that come from it, but the method count made it simply impossible to use. Android applications can only have 65,536 methods in them, and while that sounds like a lot, external libraries vrey quickly take that up. Groovy itself takes ~30,000.

Not too long after writing that post, Mike Gouline wrote a post about another language called Kotlin, calling it the ["Swift of Android."](http://blog.gouline.net/2014/08/31/kotlin-the-swift-of-android/) I wanted to see if it would actually hold up.

Some quick background though: [Kotlin](http://kotlinlang.org/) is a language created by the same people behind the IntelliJ IDE. Which is the IDE driving Android Studio. Kotlin's selling points are pretty nice - it plays nicely with Java, is incredibly expressive, and helps a lot with null safety. So, I wanted to take some time and outline the Good, the Bad, and the Ugly, and why I'm using Kotlin going forward.

The Good
--------

So why would I want to use Kotlin? Three reasons:

###First:###
Lambda functions. Call them closures if you will, but I'm a big fan of how much less code I need to write, and functions being first-class objects. I've had a lot of experience in Python, and there's a lot of really cool things you can do because of this. A quick example using RxJava though:

**Java**

{% highlight java %}
public static void main(String[] args) {
	List<Integer> list = new ArrayList<Integer>() {1, 2, 3, 4, 5, 6};
    Observable.from(list)
    	.filter(new Func1<Integer, Boolean>() {
        	@Override
            public Boolean call(Integer value) {
            	return value % 2 == 0;
            }
        })
        .forEach(new Action0<Integer>() {
        	@Override
            public void call(Integer value) {
            	System.out.println(value);
            }
        });
}
{% endhighlight %}

**Kotlin**

{% highlight kotlin %}
public fun main(args: String[]) {
	list = array(1, 2, 3, 4, 5, 6)
    Observable.from(list)
    	.filter { it % 2 == 0 }
        .forEach { System.out.println(it) }
}
{% endhighlight %}

###Second:###

I don't have a lot of space, but [Extension Functions](http://kotlinlang.org/docs/reference/extensions.html) have already been very useful as well. Check out an example over [here](https://github.com/MinimalBible/MinimalBible/blob/cb13dd64aaa1af03c6c44f272fc0fabccc8504c1/app/src/main/kotlin/org/crosswire/jsword/versification/VersificationUtil.kt).

The basic premise is that you can *extend* a class by adding extra functions to it that the original Java code didn't have. So, you can remove classes like this:

**Java**

{% highlight java %}
public class VersificationUtil() {
	public static Versification getVersification(Book b) {
    	// Implementation here...
    }
}

public static void main(String[] args) {
	System.out.println(VersificationUtil.getVersification(myBook));
}
{% endhighlight %}

And instead write code that looks like this:

**Kotlin**

{% highlight kotlin %}
public fun Book.getVersification() {
	// Same implementation, but use `this` instead of `b`
}

public fun main(args: String[]) {
	System.out.println(myBook.getVersification())
}
{% endhighlight %}

###Third:###

The method count. Groovy is a fantastic language, and I'd really love to use it. But because it takes up so many methods (~30,000), it's simply impossible.

Kotlin on the other hand takes up ~6000 methods. A fifth of the size for approximately the same functionality that I would actually end up using. I really like that. All of the features, (almost) none of the cost.

The Bad
-------

So, there are a couple frustrating things with Kotlin.

**Tests can not be written in Kotlin.** All tests being executed by Android must be written in Java still. I haven't tried Robolectric tests (which run on the host computer, not on the Android device), but I imagine the situation is the same. This is certainly not a game changer, but it does make testing things like extension functions a bit painful.

**Documentation is still a bit lacking.** I was having issues finding what I was looking for in the documentation. For instance, there are no longer any `MyClass.class` references, but [you should use](http://kotlinlang.org/docs/reference/java-interop.html) `javaClass<MyClass>()` (find the section starting with `getClass()`). While this works (I guess) it just took a long time to find. Most of that is my fault, but having some more examples would have been helpful. Additionally, the community is a bit small yet so just Googling around won't cut it - you need to actually read the documentation.

**Final by default.** I ran into this issue when I was trying to write tests for code written in Kotlin. In Kotlin, by default, [everything is final](http://kotlinlang.org/docs/reference/classes.html) (see the section labeled Inheritance). This ended up causing issues because [Mockito](https://code.google.com/p/mockito/) was then unable to mock anything. While I only needed to add the `open` modifier to my classes and functions being mocked, it took a while to figure out what was going on.

That said, all these things are easily fixed and I can get over pretty quickly.

The Ugly
--------

And this is the reason why I almost didn't use Kotlin at all. For Groovy, the method count meant that I simply was unable to incorporate it into my application.

For Kotlin, the fatal flaw is annotation processing. Because Kotlin is an alternative language that is compiled to Java bytecode, it doesn't make a whole lot of sense for annotation processing tools to run on it. That is, the tool would be generating Java code for a file written in another language. If said tool read from the **\*.class** file, it would work. But Annotation Processing doesn't make any sense from the source code.

This is incredibly problematic if (like me) a lot of your codebase relies on annotation processing. The two most notable subjects are [Dagger](http://square.github.io/dagger/) and [Butterknife](http://jakewharton.github.io/butterknife/). Dagger has been an incredibly useful tool to remove a lot of ugly code and make testing possible. Butterknife is more a utility, but still incredibly useful. Neither of these work on Kotlin currently (there is an [open ticket](http://youtrack.jetbrains.com/issue/KT-5714) to fix this), nor do I believe they ever will.

So, does using Kotlin mean I have to sacrifice my dependency injection system? Simply put: no. In the blog referenced earlier, the author details how you can write the annotated class in Java, and extend it in Kotlin. I think this is a terrible idea, because then you have to always worry about modifying two languages whenever you edit a class, and I imagine Butterknife or Dagger can get a bit confused. So having two files for a single class doesn't make a lot of sense.

However, if dependency injection is the only big problem, we can work around that. Dagger allows you to write `@Provides` functions that return an object. So, all we need to do is write a provider that instantiates the Kotlin class.

Here's a pure Java example of what I'm talking about:

{% highlight java %}
public class MyClass {
	public String doSomething() {
    	// This is the part where you do something
    }
}

public class DependentClass {
	public DependentClass(MyClass mC) {
    	this.mC = mC;
    }
}

@Module(injects=DependentClass.class) {
	@Provides MyClass provideClass() {
    	return new MyClass();
    }

	@Provides DependentClass provideDependent(MyClass mC) {
    	return new DependentClass(mC);
    }
}
{% endhighlight %}

So now we've set up a a way to do dependency injection without relying on `@Inject` annotations in the concrete classes. Moving this to Kotlin is trivial then:

{% highlight kotlin %}
// Kotlin classes
class MyClass() {
	public fun doSomething(): String {
    	// This is the part where you do something
    }
}

class DependentClass(mC: MyClass) {
	val mC = mC
}
{% endhighlight %}

{% highlight java %}
// This next section is the same @Module from above
@Module(injects=DependentClass.class) {
	@Provides MyClass provideClass() {
    	return new MyClass();
    }

	@Provides DependentClass provideDependent(MyClass mC) {
    	return new DependentClass(mC);
    }
}
{% endhighlight %}

The end result: Kotlin can participate in Dependency Injection without having to worry about the framework.

The caveat: Because Kotlin can't participate in the framework, that means that you need to write some extra code in your `@Module` classes to do the manual injection. However, it's faster for me to write the `@Module` code than it is to write the Java class.

The Summary
-----------

So far, despite the "fatal flaw" named above, I'm happy with Kotlin. I plan on using it for all classes that don't need to actively participate in any of the code generation frameworks. So while `Fragment`s and `Activity`s and the like will still need to be Java, I can write the bulk of the code in Kotlin to make it nice and quick.
