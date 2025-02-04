---
title: Silverlight Firestarter demo
date: '2010-12-09T00:00:00-08:00'
tags:
- Silverlight
- WCF
redirect_from:
- /post/15441242450/
- /post/15441242450/silverlight-firestarter-demo/
---
<p>In my <a href="{{ site.baseurl }}{% post_url tumblr/2010-12-08-silverlight-firestarter-2010-talk-and-demo %}" title="Talk slides and video">previous post</a> I linked to the video and slides from my talk. <a href="http://code.msdn.microsoft.com/Project/Download/FileDownload.aspx?ProjectName=silverlightws&amp;DownloadId=14622" title="BookShelf app">This point contains the code sample, which is available here.</a></p>
<h3>Setting up your machine</h3>
<p>Here is what you need on the machine before you get started:</p>
<ul><li>Windows 7</li>
<li>Visual Studio 2010</li>
<li>Silverlight 4 tools for Visual Studio 2010</li>
<li>SQL Server 2008 Express (which I&rsquo;m pretty sure comes with Visual Studio 2010 by default)</li>
<li>IIS7</li>
</ul><p>Before you proceed, make sure the user account under which SQL Server Express is running is <strong>NETWORK SERVICE</strong>. If you don&rsquo;t remember how this was installed, open up <strong>Task Manager</strong> (make sure you select &ldquo;Show processes from all users&rdquo;) and look for the sqlsevr.exe process. If you find that you need to change the user, <a href="http://msdn.microsoft.com/en-us/library/ms345578.aspx" title="How to: Change the Service Startup Account for SQL Server">this topic on MSDN should come in handy</a>.</p>
<p>The other tricky part is installing/configuring IIS. Go to the search box in the Start menu and type &ldquo;Turn Windows features on or off&rdquo;, then select top result. In the box shown, make sure the following are checked.</p>
<img src="{{ site.baseurl }}/images/posts/tumblr/tumblr_ld54puktoj1qccglw.png" style="margin-right: auto; margin-left: auto"/>
<h3>Deploying the basic app</h3>
<ol><li>Download the zip file and unzip the content in a folder such as C:\Firestarter. It is actually <u>very important</u> to download this in a location other than your C:\Users\YourUser folder, due to <a href="http://support.microsoft.com/kb/2002980" title="Problems with SQL Server Express user instancing and ASP.NET Web Application Projects">an apparent bug with SQL Server Express</a>.</li>
<li>Open Visual Studio as <strong>Administrator</strong> and open the solution file. You will be prompted to create a new website in IIS, and you should agree. If this succeeds you should see the solution open with both projects loaded correctly. But we&rsquo;re not quite done yet.</li>
<li>We need to make sure the account under which IIS is running has access to SQL Express. Go into the IIS Manager (type <strong>inetmgr</strong> into the Start menu search box) and click on <strong>Application Pools</strong>. Find the <strong>ASP.NET v4.0</strong> application pool and then select <strong>Advanced Settings</strong> on the right side. Find the <strong>Identity</strong> row and change that to <strong>NetworkService</strong>.<br/><img src="{{ site.baseurl }}/images/posts/tumblr/tumblr_ld5688WyHC1qccglw.png" style="margin-right: auto; margin-left: auto"/></li>
<li>We also need to make sure that the <strong>NETWORK SERVICE</strong> user has access to the location where we put our files. Let&rsquo;s assume you used C:\Firestarter as I recommended above. Go to the the folder properties, then go to the <strong>Security</strong> tab. <strong>NETWORK SERVICE </strong>is part of the <strong>Users</strong> group on any machine, so we can just make sure that the <strong>Users</strong> group has read and write  permission. If you don&rsquo;t see  the <strong>Users</strong> group go ahead and add the <strong>NETWORK SERVICE </strong>account explicitly here and give it read and write permission. Make sure the permissions are inherited all the way down, especially to the <strong>App_Data</strong> folder and the <strong>.mdf</strong> files inside that folder. You should double-check that every <strong>.mdf</strong> file allows the <strong>Users</strong> group or the <strong>NETWORK SERVICE</strong> account read and write permission. Otherwise you will get a stream of inexplicable errors as you attempt to follow the rest of this post.</li>
</ol><p>You should now be able to hit F5 in Visual Studio, the application should pop up and you should be able to see the book categories and click on individual books.</p>
<p>At this point you can also try and use the demo to enable the debugging features I cover in the video.</p>
<h3>Setting up Windows Authentication</h3>
<p>Needless to say, before we can demonstrate this scenario, we need to make sure the machine we are using is joined to a domain.</p>
<p>We need to do the IIS configuration step that I refer to in the video, but I don&rsquo;t show. We can open <strong>IIS Manager</strong> and find our web application, then double-click the <strong>Authentication</strong> icon and make sure <strong>Windows Authentication</strong> is enabled.</p>
<img src="{{ site.baseurl }}/images/posts/tumblr/tumblr_ld56ejv6tg1qccglw.png" style="margin-right: auto; margin-left: auto"/>
<p>We can then follow the rest of the steps as shown in the video. One exception is that instead of my personal domain credential, you will need to specify your own domain credential in the <strong>BookService.svc.cs </strong>code behind file. In the download I&rsquo;ve changed it to domain\user.</p>
<h3>Setting up Forms Authentication</h3>
<p>Before we can get this to work, we need to first configure our website for access over HTTPS. Like I mention in the video, this authentication scheme requires HTTPS.</p>
<ol><li>Back in <strong>IIS Manager</strong> we need to enable HTTPS on the Default Web Site bindings list. You can follow <a href="http://weblogs.asp.net/scottgu/archive/2007/04/06/tip-trick-enabling-ssl-on-iis7-using-self-signed-certificates.aspx" title="Setting up HTTPS in IIS using self-signed certs">this tutorial by ScottGu</a>. </li>
<li>Once the HTTPS binding is created for the parent website, we need to enable the HTTPS protocol for our <strong>BookService.Web</strong> site. We can do that by going to <strong>Advanced Settings </strong>and adding <strong>https </strong>in the list of <strong>Enabled Protocols</strong>. The values are comma-separated in case you were wondering.<br/><img src="{{ site.baseurl }}/images/posts/tumblr/tumblr_ld56ykc01s1qccglw.png" style="margin-right: auto; margin-left: auto"/></li>
<li>If you end up using a self-signed certificate, realize the IE will display an error page warning you that the cert is not signed. That means that any calls that Silverlight tries to make to that page will be blocked automatically. The easiest way to fix this is to go over to<strong> Internet Options </strong>&gt; <strong>Advanced</strong> and in the Security section deselect &ldquo;Warn about certificate address mismatch&rdquo;. Don&rsquo;t forget to restart IE afterwards, and most importantly, don&rsquo;t forget to reenable this option when you are done playing with the demo.</li>
<li>The last thing we need to realize is that we will be accessing the SL application over HTTP, but internally it will be making calls to HTTPS services. This is cross-scheme access and is normally disabled, even for same-domain calls. To address this we need a special <strong>clientaccesspolicy.xml</strong> file, which we place at the root of our IIS install, in my case that is C:\inetpub\wwwroot. The file looks like this:</li>
</ol>
~~~ xml 
<?xml version="1.0" encoding="utf-8"?>
<access-policy>
   <cross-domain-access>
      <policy>
         <allow-from http-request-headers="SOAPAction">
            <domain uri="http://localhost"/>
         </allow-from>
         <grant-to>
            <resource path="/BookShelf.Web" include-subpaths="true"/>
         </grant-to>
      </policy>
   </cross-domain-access>
</access-policy>
~~~
<p>Once all of this is done, we can go ahead and follow the steps in the video. Please look carefully at both <strong>Web.config</strong> and <strong>ServiceReferences.ClientConfig</strong> to make sure you comment out all sections that refer to ASP.NET authentication.</p>
<p>That is it, please let me know if you hit any issues in the comments.</p>
<p>Cheers,<br/>-Yavor</p>
