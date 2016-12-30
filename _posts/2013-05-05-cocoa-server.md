---
layout: post
title:  "Cocoa Server"
date:   2013-05-05
---
![Picture]({{ site.url }}/images/colemanserver.png)

I’m interested in backend technology. As a front-end iOS and Mac developer I don't wanna learn HTML nor PHP. I’m trying very hard to make my personal website a ‘web app’, but written in Objective-C. How can I do that? Well first lets look at what I need. Most web apps have a backend API, a web front-end (their website) and a mobile front-end (iOS app).
Currently, with my Objective-C knowledge I can only make the iOS app, so how do I make the others in Objective-C? Well I am currently making the backend as an OS X app. I’m using Core Data to internally handle a SQL DB and CocoaHTTPServer to serve JSON objects. I plan on having a another OS X app remotely connect to the API in order to add new blog entries. For the web front-end I plan on using Cappucino’s Objective-J, a Javascript superset almost identical to Objective-C. Besides the wonderful C and SmallTalk syntax ported to the Web, Objective-J comes with its own AppKit and Foundation frameworks as well. Hopefully I’ll be able to finish this project soon, since I am currently employed at as a Junior iOS Analyst at Tecla Labs and lack the time needed to work on the project.