---
title: Using custom connection strings with the mobile services .NET backend
date: '2014-05-06T15:58:00-07:00'
tags:
- Azure Mobile Services
redirect_from:
- /post/84964727000/
- /post/84964727000/mobile-services-net-backend-connection-strings/
---
<p><em><strong>UPDATE: The below content is now outdated&hellip; we have enabled a first-class connection string picker on the Configure tab of your mobile service. Specify the connection string there, and then you can easily use it exactly the same way as if it were defined in Web.config.</strong></em> </p>
<p>Since we introduced the .NET backend in Mobile Services, with rich support for connecting to existing databases, we have received a lot of questions around using custom connection strings. By default, Mobile Services creates an Azure SQL Database for you and the connection string used to refer to that is named <strong>MS_TableConnectionString.</strong> Our quickstart and Visual Studio project template uses that connection string by default. </p>
<p><strong>So what do you do if you want to use your own connection string? </strong>It depends on what kind of database yo have.</p>
<ul><li><strong>If your database is a Azure SQL database</strong>, just use the nifty Change Database button on the Configure tab of your mobile service. This will directly update the <strong>MS_TableConnectionString</strong>.<br/><img alt="image" src="{{ site.baseurl }}/images/posts/tumblr/mobile-services-net-backend-connection-strings-1.png"/></li>
<li><strong>If you are bringing your own database</strong> (for example SQL Server on IaaS), you cannot currently modify the <strong>MS_TableConnectionString </strong>connection string. We are working on enabling that and adding first-class support for connection string management right in the portal, but for now we have a simple workaround. Read on.</li>
</ul><p>The first thing you need to do is find the <strong>App Settings</strong> section on the <strong>Configure</strong> tab of your mobile service. Create a new setting called for example <strong>onPremisesDatabase</strong> and set the value as your custom connection string.</p>
<img alt="image" src="{{ site.baseurl }}/images/posts/tumblr/mobile-services-net-backend-connection-strings-2.png"/>
<p>Inside your app, find the <strong>DbContext</strong> that your .NET mobile service is using. Modify it as shown below. You might need to add an assembly reference to <strong>System.Configuration</strong> to your project.</p>
~~~ csharp
public class customDatabase1Context : DbContext
{
    public customDatabase1Context()
        : base(ConfigurationManager.AppSettings["onPremisesDatabase"])
    {
    }
}
~~~
<p>Then simply publish your changes and your service will now use your custom connection string. You are welcome to go and update the app setting in the portal if you need to and you don&rsquo;t need to re-publish your app for the change to take effect.</p>
<p>Hope this helps!</p>
