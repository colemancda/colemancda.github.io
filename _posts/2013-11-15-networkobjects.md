---
layout: post
title:  "NetworkObjects"
date:   2013-11-15
---
![Picture]({{ site.url }}/images/networkobjects.png)

I am currently in my Senior year in High School and hope to graduate soon. I have a lots of stuff going on so its hard for me to find time for programming. Luckily I’m volunteering to make a iOS for a NPO for community service hours so I’ll I’m able to do some programming. I’m trying to develop a Distributed Core Data Framework called NetworkObjects. I wanted to use Objective-C and Apple’s awesome OS X to power a server. I realized that most Internet Service APIs use REST, JSON, user authentication, etc. So I did a lot of research and wondered how Apple powers its iTunes Store. So I looked into WebObjects. WebObjects was originally written in Objective-C but was later ported to Java and is now a Java framework. I realized that their EOF was a primitive Core Data framework and that I could recreate WebObjects in Objective-C using Core Data, GCD and CFNetwork. Fortunately I found CocoaHTTPServer on Github and I built my framework on top of it. NetworkObjects is a distributed (or networked) object graph powered by Core Data. It can accelerate the development cycle of an app (almost all iOS apps connect to an internet service) by providing Cocoa server and client classes. You could use this for the backend of an iOS app or for LAN communications in games. The Open Source project is hosted in [GitHub][NetworkObjects].

[NetworkObjects]: http://github.com/colemancda/networkobjects