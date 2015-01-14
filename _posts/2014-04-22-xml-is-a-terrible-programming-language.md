---
layout: post
title: "XML is a terrible programming language"
modified: 2014-04-29 17:32:29 -0400
tags: [xml, ant, gradle, maven, dependency management, android studio]
image:
  feature: 
  credit: 
  creditlink: 
comments: true
share: 
---

_But Gradle is worse_

So, one of the first goals in building MinimalBible was getting the underlying library building correctly. I’m using [jSword][1] to drive most of the underlying functionality of the app – downloading Bibles, searching Bibles, displaying the text, etc. I have incredibly little experience with Java build systems, but in my experience prior, I had worked with Maven a bit. So when I noticed a pom.xml file, I thought I had it made. Turns out the pom.xml was outdated, and for an old build of the library.

Looking at [And-bible][2] was one of the things that first clued me in that I was approaching this the wrong way (the fact that the build was producing a jsword-1.6.jar when I knew the build was at least at version 2.1 was the other). The build.xml file also looked important, so I decided to do some more research. Here’s what I’ve learned.

There are a lot of different systems to build Java projects (I come from Python/C). The three most prevalent are [Gradle][3], [Ant][4], and [Maven][5]. By default, Android projects (using [Android Development Tools in Eclipse][6]) use none of these. Small-scale Android apps that don’t rely on a lot of extra libraries or other projects simply don’t need the level of functionality provided by any of these. But for the apps that do need extra functionality, you’re going to need to put a build system in place.

I’ll start with Maven, since it seems to be the least used in the Android world. There’s a [tutorial][7] and a [plugin][8] for doing it, but on the whole I didn’t see a whole lot of information on how to use it. That being said, the Maven project retains a [central repository][9] of Java libraries that all the other build systems rely on. This way, you can declare dependencies on other libraries, and have the build system automatically download and install them. Lovely.

Ant seems to be the most well-used (if soon to be phased-out) build system in use with Android. ADT has support for generating an Ant build file, and Google has info on how to use Ant with ADT [online][10]. However, Ant in and of itself does not have the dependency management that Maven and Gradle do. For that, we rely on a different library known as [Ivy][11]. This way, while Ant handles the build process, Ant can invoke Ivy to fetch all the needed libraries before actually compiling. This build system is what is in use by jSword, and they have a fairly nice build set up.

Gradle is the final build toolkit I’ll talk about. First things first, Gradle is **powerful**. Their manifesto even [says so][12]: “Make the impossible possible, make the possible easy and make the easy elegant.” Gradle is also the build system used by [Android Studio][13], the IDE that will eventually replace ADT. So starting as a new project, I thought it would be a good idea to give Android Studio/Gradle a spin. This is eventually going to be the way of the future.

Getting the initial Eclipse project imported wasn’t too complicated, the documentation for doing that [is pretty simple][14]. I generated my build.gradle file, and imported the project. And that’s about as far as I got.

The first order of business was making sure that I could build the jSword project using the new Gradle system. Building the Android project itself I’m sure is incredibly simple, I can get to that later. I wanted to first make sure it would be possible to even get the underlying library working first. To that end, I started doing some research and playing around with getting [Ant to work with Gradle][15]. Theoretically, Ant build tasks are first-class citizens in Gradle world. The problem is, Gradle has no way of namespacing the imported Ant tasks. That means if there’s a naming conflict, the build fails (there’s a [patch to fix this][16] coming up). While this isn’t an issue if you’re just migrating from Ant to Gradle, if you’re trying to use both Gradle and Ant at the same time, this will cause catastrophic problems. Since I don’t intend to fork jSword for Gradle alone, I needed to come up with something different.

For a while I tried to hack together namespacing by putting the jSword Ant tasks into another project. You can find an example (that I assume worked for the author) [over here][17]. This didn’t really work for me. By this time, I had been checking the And-bible [build script][18] trying to see how its author solved this problem. Without having any actual knowledge of how to write Ant tasks, what I learned by just looking at the code was that he copied over the original source, merged in his modified code, and then compiled the entire thing. And to be honest, I didn’t even try to attempt this with Gradle. While I’m sure Gradle is more than capable of doing this, it wasn’t worth the effort to figure it out; I had a build I could imitate and use as a starting point.

And so, using the And-bible build.xml as a starting point, I wrote my own build script, which is now available over [here][19]. I’ll have more information about actually writing the build script in another post!

[1]: http://crosswire.org/jsword/
[2]: https://github.com/mjdenham/and-bible/tree/master/AndBible
[3]: http://www.gradle.org/
[4]: https://ant.apache.org/
[5]: http://maven.apache.org/
[6]: http://developer.android.com/sdk/index.html
[7]: http://www.vogella.com/tutorials/AndroidBuildMaven/article.html
[8]: https://code.google.com/p/maven-android-plugin/
[9]: http://search.maven.org/
[10]: http://developer.android.com/tools/building/building-eclipse.html
[11]: https://ant.apache.org/ivy/
[12]: http://www.gradle.org/docs/current/userguide/tutorial_using_tasks.html
[13]: http://developer.android.com/sdk/installing/studio.html
[14]: http://developer.android.com/sdk/installing/migrate.html
[15]: http://www.gradle.org/docs/current/userguide/ant.html
[16]: http://issues.gradle.org/i#browse/GRADLE-771
[17]: http://gradle.1045684.n5.nabble.com/Accessing-ant-build-xml-from-gradle-td4650344.html
[18]: https://github.com/mjdenham/and-bible/blob/master/jsword-tweaks/build.xml
[19]: https://github.com/DjBushido/MinimalBible/blob/master/jsword-minimalbible/build.xml

