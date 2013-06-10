---
layout: post
title: Darting in and out
---

Complex browser applications have interested me for some time, one of my first
jobs after completing university was to develop a WYSIWYG editor that rendered 
from XML. Following that I produced a vehicle tracking system that used google 
maps to show live and historic data.

Some of the issues that I've had with designing large, javascript driven 
applications are

* Lack of classes.
* Dealing with multiple source files.
* Finding a decent IDE with code-completion.

<!--break-->

Fortunately [firebug](http://getfirebug.com), and in more recent years 
[chromium](http://www.chromium.org), have made debugging complex javascript 
far easier. There are also a huge number of javascript libraries available that help with 
[glossing](http://jquery.com) over the differences between browsers, adding 
special [effects](http://moofx.mad4milk.net) and even 
[coercing](http://mootools.net/docs/core/Class/Class) the prototypal objects 
into class-like entities.

But what I've really wanted for some time are native classes, the ability to
structure a project sensibly with many source files and a simple IDE with 
code-completion that works across project and third-party code alike, not just the 
core libraries.

Enter [Dart...](http://www.dartlang.org)


### Dart

Dart is an open-source Google-driven project that aims to ease the development 
of large scale browser applications. The language borrows features from a number
of other languages and therefore feels very familiar to work with. It also provides
a core set of libraries that give you a substantial amount of functionality
along with a simple to use third-party [packaging](http://pub.dartlang.org/) 
system.

Some of the features I like:

* Class-based.
* Optional typing.
* A simple IDE with code-completion.
* Integrated debugging using the Dart VM.
* Source map generation to allow debugging of the generated javascript.
* Compiles to efficient javascript which is important since only Dartium, supplied
with the Dart SDK, actually contains a Dart VM.

There are plenty of examples, blog posts and documentation so I'm not going to
discuss the language here but talk about using it instead.


### Packages

Mostly as a learning exercise I produced a couple of Dart packages. 


#### [css_animation](http://pub.dartlang.org/packages/css_animation)

My first foray into producing a library for Dart, this is a simple helper for 
building CSS3 animation rules. 


#### [model_map](http://pub.dartlang.org/packages/model_map)

The mirrors system in Dart intrigued me, and although it could not compile
to javascript at the time I wanted to see if it could be used for object
serialisation. Turns out it could, although it was not straightforward since
most of the mirror system relies on futures and therefore complex objects
require cascades of futures.


### Applications

The Dart core is approaching [stability](http://news.dartlang.org/2013/04/core-libraries-stabilize-with-darts-new.html) 
and the dart2js compiler is producing some impressive [results](http://www.dartlang.org/performance).

In fact we decided that it was stable enough when we recently had need to quickly 
knock together an administration application (Nodal) which would permit nodes on a 
local network to securely connect to a server.


#### Nodal

<a class="lux" href="/images/nodal.png" title="Nodal administration system" >
	<img src="/images/nodal-small.png" />
</a>

A Nodal server broadcasts over the local network so that it can be discovered by nodes. 
These nodes then announce themselves to the server via a REST interface (thanks to the wonderful 
[ServiceStack](http://www.servicestack.net) libraries) and include their public SSH keys. The web 
interface on the server displays all of the nodes and allows valid ones to be enabled, 
at which point the node can SSH to the server and gain access to any services required 
for the current application through the use of port tunneling.

This method means that Nodal doesn't care what application the nodes are for, it 
simply provides a way of connecting them together in a secure manner. It was initially 
developed for use in the [Qui](http://www.psi-ltd.com/biometrics) project.


### Conclusion

Dart is a very promising project which can greatly improve the development experience
as browser based applications become increasingly complex. I just hope that, with
the dart2js compiler supporting all modern browsers, it will be sticking around for 
a while.
