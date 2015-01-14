---
layout: post
title: "A Star Wars reference, and practical-scale builds"
modified: 2014-04-29 17:35:35 -0400
tags: [shade, ant, jsword, and-bible, maven, xml, jarjar]
image:
  feature: 
  credit: 
  creditlink: 
comments: true
share: 
---

_Also some really cool build logic_

So I’ve covered in [another post][1] how I got to create a build.xml file for the MinimalBible. The actual process of writing the build.xml file is another story entirely.

First, a few disclaimers. Google’s version of the Apache HttpClient is [old and outdated][2]. You can’t just include the httpclient library in your project, as it produces [errors][3]. Also, I wanted an entirely automated, one-click build. That meant no copying in a [new library][2] by hand. Especially since said new library would not be compatible with code expecting a vanilla HttpClient (see Compatibility notes number 2). It’s possible there would be no compatibility issues, but I never followed this path. Finally, I had run into this problem when I was using Maven to build the jSword library. The solution I used then was to do a [shaded build][4], which basically manipulated the bytecode to look for the library in a different path. Thus, all the code referencing org.apache.http could instead reference org.apache.shaded.http, or something similar.

So, knowing that I was going to write a build script using Ant, I now had to figure out how to do something similar to the Maven shaded build. Very long story short, [JarJar][5] was the plugin I found to do the trick (ironically, searching “JarJar” on Google yielded the library and not the character as the first result. I’m OK with this). I had a couple of reservations since the most recent version of jarjar was released November 2012, but so far it’s panned out all right.

Now I needed to edit the jar task in Ant to use the new JarJar plugin. The existing jar task that I copied from And-bible did most of the work, but did not include the shaded build. The JarJar [Getting Started page][6] had most of the information I needed on getting this set up for MinimalBible. Honestly, the only thing I really needed that wasn’t documented was including multiple jars at a time. I didn’t want to manually list out a ][7] for every jar file that needed to get built in with jSword, so after a lot of digging around, I found the . It doesn’t seem to be officially documented, so using some information [found on Stack Overflow][8], I hacked together a solution that allowed me to include the jSword dependencies, while excluding the libraries used for testing.

So at this point, I’ve now got a shaded build for jSword working as expected. I’m now ready to start focusing on MinimalBible code exclusively, and I can easily include building jSword as a part of building MinimalBible. In the future I’ll need to work on stripping out more of the code that goes into jSword that isn’t needed (it’s currently an 18MB library), but for now I’ve got a stable, consistent, one-click build working. I’ll call that good.

[1]: http://minimalbible.blogspot.com/2014/04/xml-is-terrible-programming-language.html
[2]: https://hc.apache.org/httpcomponents-client-4.3.x/android-port.html
[3]: http://stackoverflow.com/questions/19836012/how-to-override-android-api-class-with-a-class-available-in-added-jar
[4]: http://maven.apache.org/plugins/maven-shade-plugin/
[5]: http://code.google.com/p/jarjar/
[6]: http://code.google.com/p/jarjar/wiki/GettingStarted
[7]: http://ant.apache.org/manual/Types/zipfileset.html
[8]: http://stackoverflow.com/questions/1821803/creating-a-bundle-jar-with-ant 
