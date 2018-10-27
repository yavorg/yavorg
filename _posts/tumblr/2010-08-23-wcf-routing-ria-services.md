---
title: Integrating WCF Routing with RIA Services
date: '2010-08-23T18:41:00-07:00'
tags:
- Silverlight
- WCF RIA Services
- WCF
tumblr_url: http://hashtagfail.com/post/1000967093/wcf-routing-ria-services
---
<p>Recently we have had a few customers ask us how WCF RIA Services integrates with the WCF Routing features shipped in .Net 4. I spent some time over the last week building a prototype and learning about the issues you encounter in this scenario.</p>
<p><a title="Sample source code" href="http://code.msdn.microsoft.com/Project/Download/FileDownload.aspx?ProjectName=RiaServices&amp;DownloadId=13630">The sample source code is available here for download.</a> Please keep in mind that in order for the sample works, you need the <a title="Sample database download" href="http://msftdbprodsamples.codeplex.com/">AdventureWorksLT</a> database deployed to your local SQL Server Express instance… and don’t forget to update the connection string in Web.config!</p>
<p>First, let’s go over the scenarios where a router may be needed:</p>
<ul><li><strong>Deployment in a DMZ:</strong> In this enterprise deployment scenario, a <a title="Wikipedia article on DMZs" href="http://en.wikipedia.org/wiki/DMZ_(computing)">DMZ</a> is used to separate the untrusted network (usually the Internet) from the application/data tier (where a domain service may live). In this case all the router does is pipe traffic from a public Internet address to a private address in the trusted subnet. There is no smartness required on the part of the router and at this point it is just doing HTTP in/HTTP out piping with very little regard as to the message payload. </li>
<li><strong>Content-based routing:</strong> In this case the router needs to open up the incoming message and make a decision, based on some information in that message, which endpoint to route the message to. This can be part of the DMZ scenario, but it is not required to enable it. </li>
</ul><p>In the DMZ scenario, the WCF router may be overkill (correct me if I’m wrong). The WCF router is content-based, which means can decode incoming messages and make decisions based on different filters. Unless you are using the content routing features, a simpler router such as IIS’ <a title="IIS Application Request Routing homepage" href="http://www.iis.net/download/ApplicationRequestRouting">Application Request Routing</a> will offer better performance and probably be easier to set up. Sandrino Di Mattia has a <a title="Making WCF RIA Services work in a DMZ/Multitier architecture using Application Request Routing" href="http://sandrinodimattia.net/blog/post/Making-WCF-RIA-Services-work-in-a-DMZMultitier-architecture-using-Application-Request-Routing.aspx">really good write-up</a> about how to do this.</p>
<p>With this disclaimer out of the way, here is how I got RIA Services working with the WCF content-based router.</p>
<h3>First hurdle: binary-encoded POX</h3>
<p>In order to offer optimal wire size, RIA Services uses WCF’s binary encoder combined with plain old XML (referred to as POX, in contrast to SOAP). In WCF terms that means the <a title="BinaryMessageEncodingBindingElement MSDN reference" href="http://msdn.microsoft.com/en-us/library/system.servicemodel.channels.binarymessageencodingbindingelement.aspx">BinaryMessageEncodingBindingElement</a> needs to be configured with <a title="BinaryMessageEncodingBindingElement.MessageVersion MSDN reference" href="http://msdn.microsoft.com/en-us/library/system.servicemodel.channels.binarymessageencodingbindingelement.messageversion.aspx">MessageVersion.None</a>. However if you try that yourself, you’ll notice that the current implementation of the encoder does not allow that. RIA Services uses a private version of the encoder. So in my sample, I provide my own implementation of the binary encoder, which supports plain XML. Here is the binding that the router needs to understand RIA Services traffic.</p>
~~~ csharp
Binding binaryPoxBinding = new CustomBinding( 
   new RiaBinaryMessageEncodingBindingElement(), 
   new HttpTransportBindingElement { 
      ManualAddressing = true, 
      MaxReceivedMessageSize = int.MaxValue
   } 
);
~~~
<h3>Second hurdle: switching to HTTP POST</h3>
<p>Domain services use HTTP GET requests by default on their queries. GET offers caching benefits over POST, which are particularly relevant in the case of domain services, where large data sets with static data may be retrieved frequently.</p>
<p>Unfortunately, I could not figure out a way to get the WCF router to successfully route a GET request. Speaking to the team who designed the router, it seems SOAP scenarios took precedence to REST scenarios when they did the work, and since SOAP does everything over POST, the other HTTP verbs all get turned to POSTs. This is unfortunate, however it is easy to tell the domain service to switch to using only POST:</p>
~~~ csharp
QueryAttribute(HasSideEffects = true)]
public IQueryable GetCustomers()
~~~
<h3>Final hurdle: URI query strings</h3>
<p>Along the same lines as the previous issue, the WCF router will ignore anything that comes in the URI after the endpoint name. This is unfortunate, as domain services expect the name of the query to come in the URI. WCF extensibility comes to the rescue, and by plugging in a simple message inspector, we can correct the address of outgoing messages.</p>
~~~ csharp
public object BeforeSendRequest(ref System.ServiceModel.Channels.Message request, System.ServiceModel.IClientChannel channel) 
{ 
   // This ensures that the router will correctly roundtrip URI query string parameters
   object via; 
   if (request.Properties.TryGetValue("Via", out via)) 
   { 
      string query = (via as Uri).AbsolutePath.Replace(_routerAddress.AbsolutePath, "");
      UriBuilder builder = new UriBuilder(_clientAddress);
      builder.Path += query;
      request.Headers.To = builder.Uri; 
   } 
   return null; 
}
~~~
<p>These three fixes are included in the sample project. Once the fixes are applied, we can start taking advantage of the capabilities of the router. By default we plug in a filter that matches all incoming messages.</p>
~~~ csharp
rc.FilterTable.Add(new MatchAllMessageFilter(), endpointList);
~~~
<p>However nothing prevents us from using some more sophisticated filters such as the <a title="XPathMessageFilter MSDN reference" href="http://msdn.microsoft.com/en-us/library/system.servicemodel.dispatcher.xpathmessagefilter.aspx">XPathMessageFilter</a>. Using that filter, we could look for specific elements inside the domain service message and route to different services based on that.</p>
<p>Hope this is useful to folks out there!<br/>-Yavor</p>
