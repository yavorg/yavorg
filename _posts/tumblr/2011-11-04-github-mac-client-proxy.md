---
title: 'Configuring GitHub Mac client proxy'
date: '2011-11-04T12:03:00-07:00'
tags: []
redirect_from:
- /post/12333547822/
- /post/12333547822/github-mac-client-proxy/
---
<img src="{{ site.baseurl }}/images/posts/tumblr/tumblr_lu5gxjcrXl1qd717fo1_640.png" style="margin-left: auto; margin-right: auto"/>
<p>Trying to clone a repo using the GitHub for Mac client at work today was failing due to our proxy server. All my repos would show up, but they would fail when cloning, and trying to synchronize wouldn&rsquo;t work either. I know I am sitting behind a HTTP proxy at work, but I assumed the GitHub client would just inherit the proxy settings already defined in the OSX preferences. Apparently not&hellip; so I hunted around the app to try and find dedicated proxy settings. Again, no luck here, but after looking around forums I discovered you could use the command line to set the proxy:</p>
~~~ shell
git config --global http.proxy http://proxy:port
~~~
<p>I restarted the GitHub client, and now it would happily clone and sync my repos.</p>