---
layout: post
title: Sliding session expirations in ServiceStack
---

At this time [ServiceStack](http://www.servicestack.net) does not support sliding session expirations. 
It does, however, contain very simple to use authentication modules that support a variety of storage 
options. In an earlier project, before the authentication modules existed, we implemented our own 
authentication system, but for new projects it makes sense to use the ones provided since they require 
very little boilerplate code.

<!--break-->

When asked about sliding expiration, [Demis](https://twitter.com/demisbellot) (the creator of ServiceStack) posted a 
[very helpful](http://stackoverflow.com/questions/14857921/how-to-advance-the-session-timeout-in-servicestack) answer. 
However, this solution involves a global response filter that will slide the expiry for all authenticated requests 
but many of our projects revolve around a single-page app that is often 
[polling the server-side](http://teadriven.me.uk/2012/03/11/time-for-a-rest) for data. 
This polling, especially on a kiosked system, would essentially extend the session lifetime indefinitely, 
what we really want is to extend the session on specific service requests (ones that correspond to user interaction).

It turns out the response filter attributes in ServiceStack are powerful and extremely easy to extend. What follows 
is a simple class that can be added to any ServiceStack project and provides an attribute that applies to both 
service classes and methods.


### Implementation

{% highlight csharp %}
[AttributeUsage(AttributeTargets.Class | AttributeTargets.Method, Inherited = true, AllowMultiple = false)]
public class SlideExpirationAttribute : ResponseFilterAttribute
{
	public TimeSpan? Expiry { get; set; }


	public SlideExpirationAttribute(int expiry)
	{
		this.Expiry = (expiry < 0) ? (TimeSpan?)null : TimeSpan.FromSeconds(expiry);
	}

	
	public SlideExpirationAttribute() : this(-1) 
	{
	}


	public override void Execute(IHttpRequest req, IHttpResponse res, object requestDto)
	{
		var session = req.GetSession();
		if (session != null) req.SaveSession(session, this.Expiry);
	}
}
{% endhighlight %}

This supports modification of the expiry time in seconds, which thereby allows each service/method
to adjust the expiration independently.
If no expiration is defined then the default one will be used (set via the SessionExpiry property 
in the corresponding AuthProvider).


### Usage

When applied to a service class it affects all requests:

{% highlight csharp %}
[Route("/item")]
public class Item
{
	public int Id 		{ get; set; }
	public string Name	{ get; set; }
	public int Value 	{ get; set; }
}

[SlideExpiration]
class ItemService : Service
{
	public Item Get(Item request)
	{
		return request;
	}


	public Item Put(Item request)
	{
		return request;
	}
}
{% endhighlight %}

But it can also be applied to individual request methods:

{% highlight csharp %}
[Route("/poll")]
public class Poll
{
	public int Id { get; set; }
}


class PollService : Service
{
	public Poll Get(Poll request)
	{
		return request;
	}


	[SlideExpiration(60)]
	public Poll Put(Poll request)
	{
		return request;
	}
}
{% endhighlight %}

In this case the session expiration is only modified on a PUT request, but a polled GET will 
not affect the session timeout.
