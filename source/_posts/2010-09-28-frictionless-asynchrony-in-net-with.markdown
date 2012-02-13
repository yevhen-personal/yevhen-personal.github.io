---
layout: post
title: Frictionless asynchrony in .NET with Coroutines
date: 2010-09-28 00:15
comments: true
categories: [Silverlight, WCF]
---

In my last article I've mentioned that the approach to asynchronous programming presented in the article could be used in any other non-Silverlight contexts. Since that time I've did a lot of small improvements to the codebase and introduced some new classes to support other common use-cases. Time to show.

## What's new ##

- Added ability to asynchronously invoke any synchronous operation
- Added ability to transparently invoke APM version of synchronous method
- Switched dispatching to use common synchronization context

I've also included two sample applications: one is Windows Forms and another is Silverlight. In Windows Forms application I have sample code which demonstrates asynchronous polling and background downloading of web page. The former shows how to transparently call normal synchronous method asynchronously and the latter shows how to turn any non-standard (non-APM, EAP or proprietary) asynchronous operation to be compatible with the Coroutines infrastructure.

## Asynchronous execution of synchronous methods ##

The sample application shows the scrolling ticker with the latest news from the Reuters feed. The class which fetches news is very simple (thanks to Linq to Xml):

{% codeblock lang:csharp %}

public class NewsFeed
{
   public string Download(int top)
   {
       string timestamp = DateTime.Now.ToString("HH:mm:ss");        

       XElement rss = XElement.Load("http://mobile.reuters.com/reuters/rss/TOP.xml");
       IEnumerable<XElement> titles = rss.Descendants("item").Descendants("title");

       return timestamp + ":   " +  string.Join("      ",
              titles.Take(top).Select(x => (string)x).ToArray());
   }
}

{% endcodeblock %}

Here, we have normal synchronous method and we need to invoke it asynchronously. AsyncCommand / AsyncQuery<TResult> pair of classes is your friend in this situation.

{% codeblock lang:csharp %}

readonly NewsFeed news = new NewsFeed();

IEnumerable<IAction> UpdateNews()
{
    while (true)
    {
        var query = new AsyncQuery<string>(() => news.Download(5));
        yield return query;

        if (!query.Failed)
            ticker.ScrollText = query.Result;

        yield return Sleep.Timeout(TimeSpan.FromSeconds(120));
    }
}

{% endcodeblock %}

Internally, ThreadPool's `QueueUserWorkItem()` method is used for asynchronous execution of synchronous calls.

## Transparent invocation of APM method pair ##

In many places in .NET framework, in addition to synchronous methods, APIs do provide their APM counterpart (`BeginXXX\EndXX` method pair). In most cases (that I've inspected) they are nothing more than just an asynchronous invocation of the very same synchronous method. In such cases you can safely ignore them and just use `AsyncCommand / AsyncQuery<TResult>` classes to asynchronously invoke synchronous methods. But there some (very) rare cases when APM method pair may provide some internal optimizations. For such cases, for invocation you still specify the synchronous method but the APM method pair will be used instead.

{% codeblock lang:csharp %}

var command = new AsyncCommand(() => instance.Method(parameter), apm: true);

{% endcodeblock %}

Sometimes, all you have is only the APM method pair. As an example see `System.Net.WebRequest`. This class provides only `BeginGetResponse\EndGetResponse` APM methods but no GetResponse synchronous method. Support for such scenarios is coming. It will be something similar to `Observable.FromAsyncPattern` from Reactive Extensions.

## Custom asynchronous patterns ##

You can make any other non-standard (non-APM) or proprietary asynchronous API to be compatible with the Coroutines infrastructure provided by Servelat Pieces. For this you'll need to use `CustomAsyncCommand / CustomAsyncQuery<TResult>` classes and you'll also need to implement a custom `AsyncCallConductor`. Call conductor is mediator that has a known interface, recognized by the Coroutines infrastructure.

Let's see how we could turn EAP-based model to be compliant with our infrastructure. As an example I'll use `System.Net.WebClient` class. The WebClient's API so far:

{% codeblock lang:csharp %}

public class WebClient : Component
{
    public void DownloadStringAsync(Uri address);
    public void DownloadFileAsync(Uri address, string fileName);

    public event DownloadStringCompletedEventHandler DownloadStringCompleted;
    public event AsyncCompletedEventHandler DownloadFileCompleted;
}

{% endcodeblock %}
	
Remember, for synchronous WCF service interface in Silverlight we were required to create its asynchronous twin. Here we need an opposite :)

{% codeblock lang:csharp %}

public interface IWebClient
{
    string DownloadString(Uri address);
    void DownloadFile(Uri address, string fileName);
}
	
{% endcodeblock %}

That was the fist step. The next step is to provide a special implementation of this interface derived from `AsyncCallConductor` class.

{% codeblock lang:csharp %}

public class WebClientConductor : AsyncCallConductor, IWebClient
{
    readonly WebClient client;

    public WebClientConductor(WebClient client)
    {
        this.client = client;
    }

    public void DownloadFile(Uri address, string fileName)
    {
        client.DownloadFileCompleted += (sender, e) =>
        {
            Exception = e.Error;
            Complete();
        };

        client.DownloadFileAsync(address, fileName);
    }

    public string DownloadString(Uri address)
    {
        client.DownloadStringCompleted += (sender, e) =>
        {
            Exception = e.Error;
            Result = e.Result;
            Complete();
        };

        client.DownloadStringAsync(address);

        return null;
     }
}

{% endcodeblock %}

In this implementation class every interface operation is converted to the actual call. When the call is completed the implementation should signal about that, set operation result and handle any errors related to the execution. Basically it just conducts an asynchronous operation. Now you can use `CustomAsyncQuery` which knows how to conduct the call using your `AsyncCallConductor` implementation.

{% codeblock lang:csharp %}

static IEnumerable<IAction> BackgroundDownload()
{
    var conductor = new WebClientConductor(new WebClient());

    var uri = new Uri("http://www.codeproject.com/");
    var query = new CustomAsyncQuery<WebClientConductor, IWebClient, string>
       (conductor, client => client.DownloadString(uri));

    yield return query;

    File.WriteAllText(@"C:\Temp\1.html", query.Result);
}

{% endcodeblock %}

Looks like a lot of work. Well, you'll need to implement your custom call conductor only once and than you can reuse it everywhere. For some of the frequently used .NET classes like `WebRequest` or `WebClient` I'll possibly provide conductor implementations right in the Servelat Pieces library. And as always we can hide some dirty construction code behind the Builder interface.

{% codeblock lang:csharp %}

public class WebClientCallBuilder
{
    readonly Func<WebClient> factory;

    public WebClientCallBuilder(Func<WebClient> factory)
    {
        this.factory = factory;
    }

    public CustomAsyncCommand<WebClientConductor, IWebClient> Command(
        Expression<Action<IWebClient>> expression)
    {
        return new CustomAsyncCommand<WebClientConductor, IWebClient>(
            new WebClientConductor(factory()), expression);
    }

    public CustomAsyncQuery<WebClientConductor, IWebClient, TResult> Query<TResult>(
        Expression<Func<IWebClient, TResult>> expression)
    {
        return new CustomAsyncQuery<WebClientConductor, IWebClient, TResult>(
            new WebClientConductor(factory()), expression);
    }
}

{% endcodeblock %}

This will simplify client code down to this:

{% codeblock lang:csharp %}

static IEnumerable<IAction> BackgroundDownload()
{
    var build = new WebClientCallBuilder(() => new WebClient());
    var uri = new Uri("http://www.codeproject.com/");

    var query = build.Query(client => client.DownloadString(uri));
    var cmd = build.Command(client => client.DownloadFile(uri, @"C:\Temp\2.html"));

    yield return Parallel.Actions(query, cmd);

    if (!query.Failed)
       File.WriteAllText(@"C:\Temp\1.html", query.Result);
}

{% endcodeblock %}

## Common synchronization context ##

Previously dispatching of call completion was done through the special Dispatcher class and it was required to implement specific dispatcher for each environment (WinForms, Silverlight, ASP.NET etc). But thanks to the recent Jeffrey Richter's article it is not required anymore. The Coroutines infrastructure provided by Servelat Pieces project should now correctly work in any environment.

According to Jeffrey:

> For GUI applications such as Windows Forms, Windows Presentation Foundation, and Silverlight, the threading model dictates that all UI components must be updated by the thread that created them. So, when a GUI thread calls SynchronizationContext.Current, a reference to an object that knows how to invoke the delegate on the calling GUI thread is returned.
>    
> For ASP.NET applications, calling SynchronizationContext.Current returns a reference to an object that knows how to associates the original client request's identity and culture with the calling thread before invoking the delegate.
>   
> Console and NT service applications have no threading model and therefore do not require synchronization. 

All async actions which require dispatching inherits from the base `DispatchAction` class. As every action is (should be) created on the main thread I simply grab the synchronization context on initialization and then use it to synchronize call completion.

{% codeblock lang:csharp %}

public abstract class DispatchAction : IAction
{
  public event EventHandler Completed;

  readonly SynchronizationContext dispatcher = SynchronizationContext.Current;

  protected internal void SignalCompleted()
  {
      EventHandler handler = Completed;

      if (handler == null)
          return;

      if (dispatcher == null)
      {
          handler(this, EventArgs.Empty);
          return;
      }

      dispatcher.Post(x => handler(this, EventArgs.Empty), null);
  }

  public abstract void Execute();
}

{% endcodeblock %}

That were all news for today. I'm currently busy working on the third article in the series which is about unit testing of asynchronous code (and how Coroutines and transparent asynchrony changes the game) and hope to publish it next week. I would love to hear feedback from you on Servelat Pieces project. So if you have something to say, you're welcome. In particular I'd love to hear what do you think about its usefulness, applicability to the real-world problems, usability, whether it sucks or not and so on. Cheers!