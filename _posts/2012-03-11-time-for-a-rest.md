---
layout: post
title: Time for a REST - How ServiceStack solved our mono woes
---

We were about to abandon our Linux based project due to performance problems and memory leaks 
with [mono](http://www.mono-project.com) ASP.NET and then we discovered [ServiceStack](http://www.servicestack.net).

We're a start-up company of two and prefer developing in Linux using open-source software and libraries. 
This is also cost effective for the business since we have fewer licences to purchase, both on our own 
machines and any systems we sell. Additionally, remote management through SSH is a major advantage.

<!--break-->

### The short version

We have been developing an instrumentation system with a web application front-end. After spending a 
large amount of time battling memory leaks, stability and performance issues with the mono ASP.NET 
implementation we finally switched to ServiceStack which has resolved all of those problems.


### A generic instrumentation system written in C#

A while back [Justen](http://minimalreadership.blogspot.com) came up with the idea of developing a generic 
instrumentation system. Many industrial instruments are provided with their own specific software containing 
poorly designed interfaces or in some cases no software at all.

Our application consists of two parts:

1. A daemon that is responsible for talking to the instruments.
2. A browser interface that runs on a kiosk but is also available over the network, thereby allowing 
engineers to monitor and control individual instruments remotely if required.

The browser interface is mostly implemented in javascript with a C# server-side web application to provide 
the instrument readings, status and configuration information.
The daemon application is also implemented in C#, since there are libraries available for communicating with 
serial ports and USB devices, meaning that we could share code with the web application.


### The original design: MVC and NHibernate

A database would be used to provide the configuration of channels and instruments, it would also contain 
information about the instrument specific variables. These variables would be populated with data by the 
daemon and read by the web application which would also be able to "post" modifications back to the 
daemon where appropriate (variables that represent instrument settings, for example).
The technologies used in the first design were as follows:

* [Mono](http://www.mono-project.com) (since we wanted to deploy on a Linux based kiosk/server).
* [MySQL](http://www.mysql.com) database.
* [Fluent + NHibernate](http://fluentnhibernate.org) libraries to simplify database access.
* [MVC2](http://www.asp.net/mvc) as the web application framework.
* [MooTools](http://mootools.net) javascript library.

This design had many issues but probably the biggest flaw at the time was the use of the database to store
the transient variable values read from the instruments. Not only was this a major bottleneck but it also 
caused constant disk access.


### The second attempt: MVC3 and memory tables

When it came to the second version we foolishly thought that the way to solve the issue was to use memory 
tables in the database. This version also involved a major redesign of the javascript engine:

* Instead of multiple views in the MVC there would be a single view to load the javascript application which 
would then handle the menu options and various displays itself. The controllers in the MVC would still provide
the JSON responses for the various requests made from javascript.

* We did not fancy having to implement specific javascript classes for each instrument added in the future 
and so we came up with the crazy idea of describing instrument panels with XML that could then be parsed 
and rendered appropriately by the javascript (this gives us the opportunity to create the panels in other 
applications from the same XML).

We moved the web application to MVC3 and after a lot of work produced components capable of rendering 
themselves from XML (instrumentation specific since it involves binding to instrument variables amongst 
other things).


### Discovering problems by polling ... a lot

The web application is rather unconventional as it involves a long running javascript engine polling 
the server at regular intervals (every 500ms or so) for both status information and variable readings. 
This uncovered some rather major memory leaks, especially when left running for days on end. For an 
instrumentation server this is no good, the application has to be able to run indefinitely without 
causing the system to eventually grind to a halt!

And thus began the tedious search for leaks...


### Memory leaks and stability issues with System.Web under mono

We spent some time tracing the cause of the leaks, although that was nothing compared to the amount 
of time we've spent trying to prevent them, since they did not appear to originate in our code. The major 
culprit was traced back to System.Web and we submitted a [bug](https://bugzilla.xamarin.com/show_bug.cgi?id=381) 
report, but that has been open for over 6 months now. 

A partial workaround was to completely disable sessions but we wanted to add authentication to parts of the application 
later on and so we also experimented with the MVC3 SessionState attribute which should have allowed us to disable sessions
for most of the controllers. However, it would seem that there is some supporting functionality missing in mono so 
this attribute has no effect. Another [bug](https://bugzilla.xamarin.com/show_bug.cgi?id=3012) report was submitted, 
but at the time of writing this post there had been no response.

We solved the authentication problem at the time by switching to a simple method of our own that stores tokens
in a memory table which are referenced by cookies.

Even with a variety of workarounds we were still leaking when running through XSP and using the Boehm 
garbage collector, but with SGen the leaking seemed to stop. Even with SGen enabled, it still leaked 
when running through apache and mod_mono, therefore we started running our application through XSP 
and used mod_proxy to provide access to it.

Unfortunately that wasn't the end of it since the application would randomly crash when using SGen, 
so we had the choice of a gradual leak with one GC or random crashing with the other. We also suspected
that something in NHibernate was leaking, but it was hard to pinpoint. On top of all this, this web 
application would intermittently not start up properly with errors regarding duplicate keys in the role 
manager or object null exceptions.

The daemon, on the other hand, shared a lot of code with the web application and yet it seemed stable, 
was perfectly happy running under Boehm and did not leak memory.

Last year, when we first started discovering all of these problems Justen wrote about [them and some of the interim 
solutions we found](http://minimalreadership.blogspot.com/2011/07/why-is-aspnet-on-mono-like-new-pet-that.html). 

After a lot of time and effort we were about ready to either ditch the project entirely or resort to running 
it under the .NET framework. This would have been difficult for us to support since we develop in Linux, plus 
Windows licences would have been required for any kiosk/server units and all our work in setting up a kiosk 
image would have gone to waste. We would also lose the ability to simply SSH into the units remotely to 
configure or troubleshoot them when at our distributors.

### The silver lining - ServiceStack

The switch to memory tables for the variable readings had stopped the constant disk access but had actually 
made the performance bottlenecks worse (it seems memory tables have some serious limitations). The solution 
was, of course, to use a memory caching system like [memcached](http://memcached.org) which would allow both 
our daemon and web application to access the information.

At this point we stumbled across [ServiceStack](http://www.servicestack.net) and realised that we had 
inadvertently developed a RESTful application since the controllers in the MVC were simply acting as 
services for the javascript engine. Even though we could not really afford to spend more time on the 
project, we decided to take a week or two to attempt porting our application to ServiceStack and see 
if that resolved any of our issues.

As soon as we began porting to ServiceStack, things started looking up:

* ServiceStack provides a caching interface with memcached being one of the available providers.
* We can run our services through a console application and then simply host our javascript, css and images with apache.
* Our session-less authentication was simple to port to ServiceStack and has been updated to use the caching system.
* Start up time of the application is much faster.
* Very little boilerplate code or configuration is required.
* Simple to use with lots of helpful examples available.
* Source code is straightforward to follow if you want to find out how something works.
* It encourages the use of forward-compatible data transfer objects (DTO).
* Helpful and friendly support from the developers. 

The only problem we found during the port was nothing to do with ServiceStack but an issue with how browsers 
handle [asynchronous GET requests](http://minimalreadership.blogspot.com/2012/02/asynchronous-http-gets-being-slow.html).


### Major performance improvements and increased stability

Switching from memory tables to memcached for the storage of variable readings completely removed the 
bottleneck, thereby providing us with an order of magnitude increase in speed for those requests.

The entire application was running much faster, mostly thanks to the extremely efficient JSON serializer 
used by ServiceStack, and the memory leaks were massively reduced but not entirely eliminated. This may 
confirm our earlier suspicion that NHibernate might be leaking.

Fortunately ServiceStack also provides a very fast and extremely light-weight library called 
[OrmLite](https://github.com/ServiceStack/ServiceStack.OrmLite). Having had some success with switching 
our application to ServiceStack we decided to take the plunge and rip out NHibernate, replacing it with OrmLite.

Now, I'm not keen on writing SQL and managed to avoid it completely by using NHibernate, but our database 
is not very complex and therefore the transition to OrmLite has not been too difficult. In fact, there are
certain queries that I've been able to optimise very nicely compared to their NHibernate counterparts, 
further improving the speed of our service requests.


### Leaky results?

Having only just completed the port there is still a fair amount of testing to do, but the results so far
are promising. Both the daemon and REST application have been running for a few days now and neither appear
to be showing signs of leaking memory!


### Conclusions

Mono is a fantastic project bringing C# to all platforms, however we have been very disappointed with the lack
of interest in solving what seems to be a rather critical bug in System.Web. Combined with the instability 
issues and lack of packages for debian/ubuntu we were considering giving up on mono altogether. 

However, the excellent ServiceStack came to the rescue and has provided us with the means to continue developing
our applications under mono on the platform we prefer. It's just a shame we didn't discover it earlier!
