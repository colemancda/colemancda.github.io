---
layout: post
title:  "Universal iOS / OS X Frameworks"
date:   2015-02-11
categories: Programming
---

With iOS 8, Apple has given us the ability create dynamic frameworks for iOS, something that has always been availible for OS X. Apart from the fact that your app and extensions can share the same code, Apple is encouraging developers to share code between iOS and OS X. Apple's describes an example of how to do so in its WWDC 2014 Session 223 video. The problem with the described steps is that it build 2 separate targets, which normally wouldn't be a big deal, except for the following inconviences:

- 2 separate framework targets always have to keep their compiled sources and dependencies in sync

- 2 separate testing unit bundles are required and also need to keep their sources and depesdencies in sync

- Code that will use your framework that will also build for iOS and OS X will require import statements like:

{% highlight swift %}
#if os(iOS)
    import MyFramework
#else
    import MyFrameworkOSX
#endif
{% endhighlight js %}

Becuase I found that import syntax hideous, through a lot of trial and error I discovered how to make a dynamic framework compile for iOS and OS X, as well as have a single universal test unit bundle. I mainly developed this process for [NetworkObjects][NetworkObjects], but since then I've been making merge requests for frameworks I use like [ExSwift][ExSwiftMergeRequest] and [CocoaLumberjack][CocoaLumberjackMergeRequest].

[NetworkObjects]: https://github.com/colemancda/NetworkObjects
[ExSwiftMergeRequest]:https://github.com/pNre/ExSwift/pull/76
[CocoaLumberjackMergeRequest]: https://github.com/CocoaLumberjack/CocoaLumberjack/pull/341

###1. Change the project's build architechures and supported platforms.

This should change your framework's and test unit's build architechures and supported platforms as well. If not, then manually change them to inherit from the project's build settings.
![Step One]({{ site.url }}/images/universalframeworkstep1.png)

- Base SDK: I recommend OS X, but it will work with iOS too. Note that with with iOS as the base SDK, "My Mac" target is separated into 3 different targets.

- Supported Platforms: ```macosx iphoneos iphonesimulator```

- Valid Architectures: ```arm64 armv7 armv7s i386 x86_64```


