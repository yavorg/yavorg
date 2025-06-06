---
title: WCF and JavaScript clients at PDC 2010
date: '2010-10-26T18:48:00-07:00'
tags:
- WCF
- JavaScript
redirect_from:
- /post/1411309356/
- /post/1411309356/wcf-http-jquery-pdc10/
---
<p>As you <a title="Glenn Block's post about his PDC talk" href="http://blogs.msdn.com/b/gblock/archive/2010/10/26/taking-http-support-in-wcf-to-the-next-level.aspx">may have heard</a> from <a title="Glenn Block Twitter page" href="http://twitter.com/gblock">@gblock</a> WCF is making some significant new investments around HTTP to make sure HTTP-based services are first-class within our stack. As part of this effort, we are renewing our focus on JavaScript clients and <a title="jQuery homepage" href="http://jquery.com/">jQuery</a> in particular. To learn more, check out Glenn&rsquo;s PDC10 talk:</p>
<hr/>
<p><strong>Building Web APIs for the Highly Connected Web</strong><br/>Friday 10/26/10, 9:00 AM-10:00 AM (GMT-7) <br/>In person: Kodiak Room / Microsoft Campus Redmond, WA<br/>Live stream: <a href="http://player.microsoftpdc.com/Session/17a9e09f-4af1-4ef3-8a5a-ebf1e9bd9c8e">http://player.microsoftpdc.com/Session/17a9e09f-4af1-4ef3-8a5a-ebf1e9bd9c8e</a> </p>
<p>And to leave you with a little teaser&hellip; join us for the talk to find out more!</p>
~~~ csharp
WebClient client = new WebClient();
string result = client.DownloadString("http://search.twitter.com/search.json?q=%23PDC10");

JsonObject jo = (JsonObject) JsonValue.Parse(result);

Console.WriteLine("Latest PDC tweet is by {0} whose image is {1}: {2}",
    jo["results"][0]["from_user"].ReadAs<string>(),
    jo["results"][0]["profile_image_url"].ReadAs<Uri>(),
    jo["results"][0]["text"].ReadAs<string>());
~~~
