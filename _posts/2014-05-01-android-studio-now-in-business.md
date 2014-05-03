---
layout: post
title: "Android Studio now in business"
modified: 2014-05-01 11:31:39 -0400
tags: [Android Studio, gradle, ant, build, dependency management, Eclipse]
image:
  feature: 
  credit: 
  creditlink: 
comments: 
share: 
---

But I'm still not a fan of Gradle
---------------------------------
 
So, I've wanted to switch to Android Studio for a while. Eclipse as an IDE is a great tool, but largely, it seems to be showing its age. I've just not had great experience with it before, and trying to configure plugins, etc. Plus, Android Studio offered a lot of new features that looked cool.
 
Problem: **Configuring builds to use Gradle.**
 
[JSword](http://www.crosswire.org/jsword/) is going to be used for most of the backend functionality, MinimalBible is the user interface driving everything. The JSword library is built using [Ant](http://ant.apache.org/), and I have another [Ant build](https://github.com/MinimalBible/jsword-minimalbible/blob/b09021b56d8c75d21024e2d693ad00fcb3389790/build.xml) responsible for tweaking the original library (currently it just shades dependencies and builds an uberjar).
 
So my build of JSword was going just fine, but I wanted to have a one-click build of the entire MinimalBible app (and eventually use something like [Travis](https://travis-ci.org/)). That means getting a build of JSword working alongside the main build.
 
Originally, it would have made the most sense to stick with Ant builds and re-use targets from that. But me being adventurous and wanting to try the new shiny tooling, I gave Gradle a shot. The results really didn't [pan out]({% post_url 2014-04-22-xml-is-a-terrible-programming-language %}).
 
To be honest, I don't really remember why I gave Android Studio a second shot. But I did, and got a [preliminary build](https://github.com/MinimalBible/MinimalBible/commit/77c797d4f1621511f659557397f597fd0843a6f6) working after an hour or two. I knew the build wasn't perfect, but developing in Android Studio was now a live option.
 
The next step was trying to get a build working on Windows. That wasn't incredibly challenging, but Windows was complaining about a `builtBy` clause in the dependencies. I'm not sure why Linux didn't complain the same way, but oh well. So at [this point](https://github.com/MinimalBible/MinimalBible/commit/2818a25555902c371d94330d56d7997912f133dc), the build is working on Windows as well.
 
The final problem was the bootstrap build. All the builds prior succeeded because the `jsword.jar` file already existed in the `libs/` folder of the project. When you deleted the JAR and tried to build it would fail. But if you tried to rebuild, the JAR already existed as part of the last build, and so the current build would succeed. This, however, would lead to a build failure on any sane CI system, plus it's ugly.
 
So, I did what any sane developer would do after failing to understand the Gradle build system, and posted a [StackOverflow question](http://stackoverflow.com/questions/23397440/dynamically-add-jar-to-gradle-dependencies). As of the time of writing, there were no answers, but I did get it figured out on my own.
 
What follows next is not an explanation of why the code works, but more so a "here's what I did and it fixed my problem." There were two big changes to make:
 
**Main app:**
 
The way the build originally worked was that a task ran a stubbed Gradle build file, and this build file imported the Ant targets from my build of JSword (the idea I used is [over here](http://www.kellyrob99.com/blog/2011/09/18/using-gradle-to-bootstrap-your-legacy-ant-builds/)). This is problematic since Gradle is then unable to track what is generated from this (and I assume this is the reason why the new JAR wasn't added as a dependency).
 
The new way of doing things is to run that build as part of a different Gradle project, and export the artifacts of that project. So basically instead of the main app being responsible for building the external artifact, the main app just depends on what my JSword build produces. You can find the exact changes over [here](https://github.com/MinimalBible/MinimalBible/commit/7533f73f98835c02abfb4333784557b53f830215)
 
**JSword build:**
 
My build of JSword is now responsible for exporting the artifact from Gradle. However, I still want to depend on Ant targets, since that's what the underlying library actually uses. In the future, I might switch to pure Gradle, but for now, it's working. So, I added an `artifacts` section, made a specific `configuration`, and then the artifact now depends on the task which actually generates the JAR file for JSword.
 
So the [first iteration of the build](https://github.com/MinimalBible/jsword-minimalbible/commit/7e0eaee2015dfccf63c7b2f458bdb8bfba4033ad) was complete, and I could now build the entire MinimalBible app using Gradle, without including the `jsword.jar` in the app itself. Daringly so, I set up my git tree in another location to guarantee everything was working correctly. Turns out I forgot to add the artifact dependency on the build task. I fixed that [here](https://github.com/MinimalBible/jsword-minimalbible/commit/b09021b56d8c75d21024e2d693ad00fcb3389790).
 
**Wrapping up**
 
But now the entire build works correctly, and I can use Gradle! All told, it took me a lot longer than I'd like to get everything set up. With great power comes great responsibility, and honestly, the Gradle documentation was of very little help in trying to get my build set up. I did eventually get a solution (and a good solution at that), and I'm glad I put the time in to make the switch - it seems everyone is migrating to Android Studio, and trying to do this in the future after having set up an Ant build for MinimalBible would have been a nightmare. Starting with Gradle I think will make things easier.
 
So Android Studio is now working. I have a good, clean, build of MinimalBible going that I should be able to evolve as my needs change. I still really don't like Gradle, but I can understand that when you're working with an Ant build with [~1500 lines](https://github.com/scala/scala/blob/ca9003e453873c496c72c431f0e5f9f3eaf31511/build.xml) you need a build system as complex as Gradle. At this point, we'll call it good.
