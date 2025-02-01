---
title: Getting the WebOperationContext of a HTTP response in Silverlight
date: '2010-10-20T12:25:00-07:00'
tags:
- Silverlight
- WCF
redirect_from:
- /post/1360386757/
- /post/1360386757/weboperationcontext-silverlight-response/
---
<p>This came up as a question from a customer today: how do you get details of the HTTP response message that a WCF proxy in Silverlight received? If you thought of  <a title="System.ServiceModel.OperationContext" href="http://msdn.microsoft.com/en-us/library/system.servicemodel.operationcontext(v=VS.95).aspx">OperationContext</a> and <a title="System.ServiceModel.Web.WebOperationContext" href="http://msdn.microsoft.com/en-us/library/system.servicemodel.web.weboperationcontext(v=VS.95).aspx">WebOperationContext</a>, you&rsquo;re on the right track, but you have only half of the story.</p>
<p>In Silverlight, in order to get to these context objects, you have to switch from the event-based async pattern to the more complex Begin/End-based async pattern. Within that pattern, you need to instantiate an OperationContextScope and call the End* method inside that scope, before you can access the context objects themselves. Check out this code snippet:</p>
~~~ csharp
public MainPage()
{
    Service1 proxy = new Service1Client() as Service1;
    proxy.BeginDoWork(new AsyncCallback(Callback), proxy);
}

void Callback(IAsyncResult result)
{
    Service1 proxy = result.AsyncState as Service1;

    using(new OperationContextScope((proxy as Service1Client).InnerChannel))
    {
        string foo = proxy.EndDoWork(result);
        // Don't forget to reference System.ServiceModel.Web.Extensions.dll 
        // from the Silverlight SDK, otherwise WebOperationContext won't be
        // recognized

        // Get the HTTP status code or anything else WebOperationContext gives you
        HttpStatusCode status = WebOperationContext.Current.IncomingResponse.StatusCode;
    }
}
~~~
<p>This is about as complex as WCF can ever get, so I thought I&rsquo;d share :)</p>
