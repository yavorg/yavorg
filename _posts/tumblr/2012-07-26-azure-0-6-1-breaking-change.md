---
title: Whoops, sorry we broke you
date: '2012-07-26T16:34:00-07:00'
tags:
- Windows Azure
- Node.js
featured_image: '/images/posts/tumblr/tumblr_m7sjtoJTKd1qccglw.jpg'
tumblr_url: http://hashtagfail.com/post/28086921700/azure-0-6-1-breaking-change
---
<a href="http://www.flickr.com/photos/runran/3069844856/"><em>Image credit: runran on Flickr</em></a>
<p>We made a breaking change between versions 0.6.0 and 0.6.1 of the <a href="https://github.com/WindowsAzure/azure-sdk-for-node">Windows Azure command-line tool for Mac and Linux</a>. You can get the tool from <a href="http://windowsazure.com">http://windowsazure.com</a> or via the <strong>azure</strong> package on <strong>npm</strong>. The tool lets you create and manage Azure websites and Virtual Machines from any platform.</p>
<p>If you imported your publish settings file using version 0.6.0 and then updated the tool, you might be greeted by the following message:</p>
~~~ powershell 
PS C:\Users\yavorg&gt; azure site list
info:    Executing command site list
+ Enumerating locations
error:   Host is not reachable. This may be due to lost internet connectivity. Please check your connection.
info:    Error information has been recorded to azure_error
~~~
<p>This is because we made a change to the way we internally store your publish settings information, and the old format doesn&rsquo;t work with the updated tool. Fortunately here is an easy fix: just clear your old publish settings and import them again:</p>
~~~ powershell
PS C:\Users\yavorg&gt; azure account clear
PS C:\Users\yavorg&gt; azure account import '.\node.publishsettings'
~~~
<p>Thanks, and sorry for the inconvenience!</p>
