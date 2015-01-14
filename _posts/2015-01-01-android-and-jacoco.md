---
layout: post
title: "Android &amp; Jacoco"
modified: 2015-01-01 13:56:32 -0500
tags: [jacoco,testing,code coverage]
image:
  feature: 
  credit: 
  creditlink: 
comments: true
share: 
---

Empirical Development
---------------------

For a while now, I've been wanting to get code coverage working with MinimalBible,
and it's finally at a point that I'm mostly satisfied with. I certainly wouldn't argue that it's in a "good" state,
but it's enough for now. So, I wanted to write a post outlining how this was all set up since I haven't been able to
find many people using this style. Plus, it took a very long time to set up, so I hope I can save someone else some
pain in the future!

Before I get too much farther though, let me explain the concept of code coverage.
Code coverage is intended to answer the question of "What code have I written tests for?"
This allows you to quickly spot code that is untested by your existing suite, and let you know where is best to focus
your time. Coverage statistics are generated line-by-line and branch-by-branch -
if your test doesn't execute a specific branch of code during the test,
your coverage system will let you know that not everything is "covered" by a test.
All said, you get an easy way to see what potential problems exist and proactively solve them.

Test Setup
----------

So, onward to how I set up testing in MinimalBible.
First things first, I have to say that I really changed how I did testing in Android. Instead of using the existing
Android build tools, I'm using pure-Java testing. Nothing Android. You can find the setup process
[here](http://blog.blundell-apps.com/how-to-run-robolectric-junit-tests-in-android-studio/)
(massive shoutouts to Blundell Apps, this has been incredibly useful) but I'll give a quick overview of how this is set up.

The basic principle is that the testing support in Android is so broken as to be useless. I won't go into all my gripes,
but suffice to say I have had issues with the JUnit API (Android uses something below 3.8), code coverage, and UI tests
with [Espresso](https://code.google.com/p/android-test-kit/). So instead of trying to use the native Android tools for
testing, we create a new project based on the source code of our existing project. This allows us to run everything in a
native Java environment, without worrying about any of the platform considerations. If you're interested in what that
setup would look like, you can find my version
[here](https://github.com/MinimalBible/MinimalBible/tree/cb8ea71f620ac0a25c628920bfe5fa82b1d6cebe).

There are two projects: `app` and `app-test`.
App is where the actual source code resides, and App-test is where the test code resides.
However, the test code includes the original code as a dependency so that we can still write tests against it.
I've included the important bits of `app-test/build.gradle` below:

{% highlight groovy %}
apply plugin: 'java'

def androidModule = project(':app')
dependencies {
    compile androidModule
    testCompile androidModule.android.applicationVariants.toList().first().javaCompile.classpath // 1
    testCompile androidModule.android.applicationVariants.toList().first().javaCompile.outputs.files // 2
    testCompile files(androidModule.plugins.findPlugin("com.android.application").getBootClasspath()) // 3
    testCompile 'junit:junit:4.+' // 4
    testCompile 'org.robolectric:robolectric:2.2'
}
{% endhighlight %}

And a quick breakdown of what's going on:

1. Make sure to include all dependencies of the actual project in the test project
2. Include all code for the actual project in the test project.
3. Include all dependencies of the `android` Gradle plugin in the test project. Not sure why this is here.
4. Finally, include all the other libraries and things we actually need for testing.

So at this point we have a test-ready project setup. We're not quite ready for code coverage yet, but we're about to change that.

Next steps: Enabling Code Coverage
----------------------------------

So, we have a test project set up to run the tests we put inside it. The next step is setting up Jacoco to report on
what code is being tested during the test suite. I'm going to present the solution first to make it easy on anyone
reading this - if you want a full explanation of what's going on check it out [here](#jacoco-full-explanation).

In order to enable Jacoco testing we need to change `app-test/build.gradle` to look like
[this](https://github.com/MinimalBible/MinimalBible/blob/c29b043d2313b3653a9671c36921f6ce8e4b9348/app-test/build.gradle):

{% highlight groovy %}
apply plugin: 'java'
apply plugin: 'jacoco'

def androidModule = project(':app')
def firstVariant = androidModule.android.applicationVariants.toList().first()

def testIncludes = [
    '**/*Test.class'
]
def jacocoExcludes = [
    'android/**',
    'org/bspeice/minimalbible/R*',
    '**/*$$*'
]
{% endhighlight %}

First steps first, this is the easy part. We're defining what files we want to include in testing,
alongside the files we want to exclude from the final report. For example, we exclude the "R" file, since
it's all generated code. In addition, anything containing a "$$" is generated by Dagger/Butterknife, so we ignore
those too.

If you want to adapt the solution I'm outlining here to your own project, these should be the only sections you
need to edit.

The next section is a whole lot more complicated:

{% highlight groovy %}
dependencies {
    compile androidModule
    testCompile 'junit:junit:4.+'
    testCompile 'org.robolectric:robolectric:+'
    testCompile 'org.mockito:mockito-core:+'
    testCompile 'com.jayway.awaitility:awaitility:+'
    testCompile 'org.jetbrains.spek:spek:+'
    testCompile firstVariant.javaCompile.classpath
    testCompile firstVariant.javaCompile.outputs.files
    testCompile files(androidModule.plugins.findPlugin("com.android.application").getBootClasspath())
}
def buildExcludeTree(path, excludes) {
    fileTree(path).exclude(excludes)
} // 1
jacocoTestReport {
    doFirst {
        // First we build a list of our base directories
        def fileList = new ArrayList<String>()
        def outputsList = firstVariant.javaCompile.outputs.files
        outputsList.each { fileList.add(it.absolutePath.toString()) }
        
        // And build a fileTree from those
        def outputTree = fileList.inject { tree1, tree2 ->
            buildExcludeTree(tree1, jacocoExcludes) +
            buildExcludeTree(tree2, jacocoExcludes)
        }
        
        // And finally tell Jacoco to only include said files in the report
        classDirectories = outputTree
    } // 3
}
tasks.withType(Test) {
    scanForTestClasses = false
    includes = testIncludes
} // 2
{% endhighlight %}

1. Define a quick function that will exclude a *list* of file paths from a given path.
2. Set up Gradle to run the tests we defined earlier
3. Set up Jacoco to exclude the paths we specified earlier from the report. This step is so complicated
because we have to get the outputs paths from the Android project, and exclude our paths from each of those.

Wrapping Up
-----------

So given the above build.gradle file, we now have a project capable of testing your actual application code and
producing coverage statistics on it. While I haven't outlined it above, because the testing code is separate from the
Android project, you're free to write your tests in JUnit, [Spock](https://code.google.com/p/spock/),
or [Spek](http://jetbrains.github.io/spek/). I'm going to be using Spek moving forward.

We can include tests using the `testIncludes` list, and make sure that classes don't get reported using the
`jacocoExcludes` list. All said, that's what we were out for in the first place, so I'll call it a success.

If you want to take this solution further, the next step would be to add Robolectric tests into the suite,
but [I've been having issues](https://github.com/robolectric/robolectric/issues/1385) with that too.

Appendix: <a name="jacoco-full-explanation">Jacoco Full Explanation</a>
=============================================================

In order to fully understand what's going on with how Jacoco excludes things from reporting, we have to step back and
take a visit to Gradle first to understand your build lifecycle.

Gradle: Configure, Run
----------------------

Gradle is an incredibly powerful tool, but it is massively confusing if you don't already know what you're doing.
In my opinion, the documentation is still missing many examples that would be super-helpful,
and is generally dense to try and get through.

That aside, to understand what's going on, you must understand that the Gradle build process happens in two phases:
**Configuration**, and then **Build**.

For our purposes, you don't need to understand what each one does, but understanding the semantics is crucial.
Because there's a two phase build, we can't write a `build.gradle` that tries to exclude files from Jacoco like this:

{% highlight groovy %}
jacocoTestReport {
    // First we build a list of our base directories
    def fileList = new ArrayList<String>()
    def outputsList = firstVariant.javaCompile.outputs.files
    outputsList.each { fileList.add(it.absolutePath.toString()) }
    
    // And build a fileTree from those
    def outputTree = fileList.inject { tree1, tree2 ->
        buildExcludeTree(tree1, jacocoExcludes) +
        buildExcludeTree(tree2, jacocoExcludes)
    }
    
    // And finally tell Jacoco to only include said files in the report
    classDirectories = outputTree
}
{% endhighlight %}

Did you notice the difference? In the second example, we're missing the `doFirst` closure.
Keep this in mind during the next sections.

Under the hood, Jacoco reports on all classes specified in the `classDirectories` variable. So, all we need to do is
make sure that we include all the classes to report on in `classDirectories`, and exclude the ones we don't want to see.

However, if you skip the `doFirst` closure, you'll be in deep trouble. Without that closure, Groovy will run the code
in the `jacocoTestReport` closure before testing is actually run, since it will be in the **configuration** build phase.

What the code actually does is exclude everything in `jacocoExcludes` from the global class path. This isn't a great
solution, but I'm not sure how else to do it.

The problem comes when you exclude files like the `android` package that we don't want to report on, but are needed for
testing. When things in `android` aren't loaded during the tests, you'll get lots of nasty `NoClassDefFoundException`
exceptions, because Java can't find the code it needs for testing.

The solution? We need to modify the class path **only right before Jacoco runs**. This way, the tests are allowed to
run successfully, and Jacoco never knows about those classes.

To do this, we need to move the class path configuration into the **build** phase instead of **configure**.
The way to do that? You guessed it, surround the code in a `doFirst` closure.

So the end result is that we can exclude specific classes from reporting without interfering in the test setup process.
It took me forever to figure out how exactly to implement this, but I hope this can help someone avoid the same issues
in the future.

Side note: Much of the above solution was adapting the procedures outlined
[here](https://issues.gradle.org/browse/GRADLE-2955) to the world of Android. Thanks to everyone for putting in the
effort to make it easier for me!

