---
layout: post
title: "Why not Groovy?"
modified: 2014-09-01 13:36:35 -0400
tags: [groovy, dex, 64k, play services]
image:
  feature: 
  credit: 
  creditlink: 
comments: true
share: 
---

So, a while back one of the top Groovy developers released a version of Groovy that was [ready to run on Android](http://melix.github.io/blog/2014/06/grooid.html). Even beyond that, it comes with it's nice own [Gradle plugin](https://github.com/melix/groovy-android-gradle-plugin) so all you really have to do is add the plugin to build and wa-la! You're good to go!

Now, I would like to say first off that I do sincerely believe that using Groovy would drastically speed up development time. Being able to use closures is great, and I'm a huge fan of dynamic languages (I've spent a lot of time in Python development). Even the New York Times [is getting on board](http://open.blogs.nytimes.com/2014/08/18/getting-groovy-with-reactive-android/?_php=true&_type=blogs&module=BlogPost-Title&version=Blog%20Main&contentCollection=General&action=Click&pgtype=Blogs&region=Body&_r=0).

Unfortunately, it's simply not a possibility for me.

Because of the 64k method limit
-------------------------------

So why won't I be using Groovy? Simply put, the 64k method limit for Dex.

This is a [well](http://stackoverflow.com/questions/25607908/limit-of-methods-64k-per-a-dex-file-in-android) [documented](http://stackoverflow.com/questions/15436956/how-to-solve-the-issue-with-dalvik-compiler-limitation-on-64k-methods) [issue](http://stackoverflow.com/questions/21146959/dynamic-class-loading-with-intellij-64k-method-dex-issue) in many Android projects. The crux of the issue is this: the Android format only supports being able to call 65,536 methods in an application. And while you likely won't write anywhere near that many methods yourself, trying to depend on external libraries fast approaches that limit. So far as I can tell, there are 4 solutions to addressing this problem:

### Proguard ###

Proguard is a code obfuscation tool, that also uses static analysis to do some cool things. For example, Proguard can locate unused methods and strip them out of your application. Sounds nice, right? There are two problems.

1. Proguard takes a while to run. Even if it's only 30 seconds, that's an extra 30 seconds per build that you lose in development time. It adds up and its frustrating to boot.

2. Proguard isn't that great at removing methods. Proguard can only remove methods that aren't referred to by anyone else. So if you have a method that is unused during the application, but can still be reached, Proguard can't remove it. This is an issue especially in the Play services library - many methods will never be used, but Proguard can't guarantee that.

While eventually I'll be using Proguard for my release builds, in the mean time it would be painfully slow to wait 30 seconds every time to debug a build on my device.

### Dynamic Classloading ###

Google put up a [blog post](http://android-developers.blogspot.com/2011/07/custom-class-loading-in-dalvik.html) detailing how you can use Dynamic Classloading to circumvent the 64k method limit. The basic workflow is this:

1. Build a `.dex` file separate from your main application - this can contain an external library or whatever else is needed. This should be stored in your `assets/` folder.

2. Use code to load the external `.dex` file into memory

3. This is where it gets interesting. To actually use this new Dex file you **must either get classes via reflection and bind them to interfaces** (such that the interface contains the methods you need), or **call everything via reflection**.

But if you can accomplish all this, you're done. That said, while this works, it's practically impossible to write maintainable code with this. There's an incredible amount of reflection involved and needed to make this work, **and can not be accomplished in Eclipse** because it doesn't support the build complexity. This is hardly a solution.

### Native Interfaces ###

While I won't be investigating this one much, a similar solution would be to write as much of the code as possible in a native language (i.e. C++, C, etc.) and then call into that code using Java. Arguably this runs into similar issues as the Dynamic Classloading, but at least there's an API for it.

Alternatively, you can write Javascript for a WebView component and outsource most of your execution. The problem with this is you lose an incredible amount in terms of memory and performance. So while it would theoretically work, it would end up being painfully noticeable.

### Use fewer libraries ###

This unfortunately seems to be the most practical solution. I haven't done any benchmarking, but I'd be willing to guess the time it takes you to write the code yourself is less than the time it would take to implement a solution above. While it forces you to write lower-level code, it's not anywhere near the point of trying to use reflection to load a library at runtime.

Unfortunately you do become very limited in what external libraries you can use. There are a great number of productivity-boosting libraries and things available, but sacrifices have to be made.

Summary
-------

The 64k method limit in Dex is painful. And before you think that you'll never get anywhere near that limit, think about this:

* The Google Play Services library itself contains [20 thousand methods](http://jakewharton.com/play-services-is-a-monolith/). And you can't just use part of it. If you depend on Play services at all, one-third of your method count disappears

  * To be fair, people have invented [ways of stripping out](https://medium.com/@rotxed/dex-skys-the-limit-no-65k-methods-is-28e6cb40cf71) components you don't need. But this is still ridiculous.

* The Groovy language implementation for Android uses roughly [30 thousand methods](#groovy-method). A full half of your method count is immediately gone if you try and use Groovy.

So while both Play Services and Groovy provide some great features, it's basically one or the other. You can't have both. The 64k method limit is a big problem, and prevents people from using some of the great technology available. While I understand this is a technical nightmare to solve, there isn't really an "official" solution from Google to work around this. So, due to practical concerns, I won't be using Groovy. Much as I'd like to.

P.S.
----

If you're interested to see how many methods are used in your application, check out the script [over here](https://github.com/mihaip/dex-method-counts). It's a fantastic resource.

Finally, I've attached below a method count for a bare-bones app compiled to use Groovy. Enjoy!

<a name="groovy-method">Groovy Method Count</a>

```
Read in 31289 method IDs.

<root>: 31289
    android: 5
        app: 3
        view: 2
    com: 19
        example: 17
            bspeice: 17
                testgroovy: 17
        thoughtworks: 2
            xstream: 2
    groovy: 3483
        beans: 111
        grape: 430
        inspect: 21
        io: 59
        lang: 1288
        security: 2
        time: 112
        transform: 265
            builder: 64
            stc: 59
        ui: 30
        util: 1153
            logging: 45
        xml: 12
    groovyjarjarantlr: 3072
        ASdebug: 7
        actions: 357
            cpp: 81
            csharp: 81
            java: 80
            python: 115
        build: 22
        collections: 129
            impl: 89
        debug: 393
            misc: 23
        preprocessor: 172
    groovyjarjarasm: 1319
        asm: 1319
            commons: 451
            signature: 41
            tree: 254
            util: 229
    groovyjarjarcommonscli: 219
    groovyjarjarharmonybeans: 174
        editors: 88
        internal: 10
            nls: 10
    groovyjarjaropenbeans: 815
        beancontext: 208
    java: 1342
        applet: 2
        awt: 51
            event: 6
        io: 179
        lang: 562
            annotation: 12
            invoke: 44
            ref: 12
            reflect: 82
        math: 41
        net: 54
        nio: 4
            charset: 4
        security: 18
        sql: 5
        text: 10
        util: 416
            concurrent: 32
                atomic: 13
                locks: 10
            logging: 13
            prefs: 12
            regex: 17
    javax: 74
        swing: 71
            event: 1
            text: 2
            tree: 2
        xml: 3
            parsers: 3
    org: 20767
        apache: 4
            commons: 4
                cli: 4
        codehaus: 20749
            groovy: 20749
                antlr: 2013
                    java: 304
                    parser: 490
                    treewalker: 943
                ast: 2269
                    builder: 660
                    expr: 500
                    stmt: 150
                    tools: 156
                classgen: 1620
                    asm: 842
                        indy: 30
                        sc: 118
                cli: 7
                control: 928
                    customizers: 436
                        builder: 195
                    io: 29
                    messages: 24
                plugin: 2
                reflection: 381
                    android: 3
                    stdclasses: 69
                runtime: 8887
                    callsite: 309
                    dgmimpl: 413
                        arrays: 190
                    m12n: 31
                    memoize: 38
                    metaclass: 463
                    powerassert: 30
                    typehandling: 731
                    wrappers: 30
                syntax: 160
                tools: 678
                    ast: 150
                    gse: 24
                    javac: 104
                    shell: 53
                        util: 38
                transform: 3194
                    sc: 107
                        transformers: 61
                    stc: 585
                    tailrec: 1160
                    trait: 90
                util: 395
                vmplugin: 197
                    v5: 46
                    v6: 1
                    v7: 140
        fusesource: 7
            jansi: 7
        xml: 7
            sax: 7
                helpers: 1
```
