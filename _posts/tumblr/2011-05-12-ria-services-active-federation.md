---
title: Active Federation using RIA Services and WIF
date: '2011-05-12T21:06:00-07:00'
tags:
- WCF RIA Services
- Silverlight
redirect_from:
- /post/5441929717/
- /post/5441929717/ria-services-active-federation/
---
<p><strong><em>UPDATE: A coworker here at Microsoft has brought up some interesting thoughts regarding this approach, and I&rsquo;ve added a brief discussion below the second diagram.</em></strong></p>
<p>Recently I decided to experiment with RIA Services in federated authentication scenarios. For those of you that aren&rsquo;t familiar with federation, let&rsquo;s consider the following scenario: my company Contoso builds an expense reporting application that I host in the cloud. One day I get Fabrikam to sign up as a client, and now I have to onboard all Fabrikam employees onto my system. It is true that I could create a login for every employee, but that is impractical because Fabrikam employees have to remember that extra login, and also I would need to keep my list of logins in sync when employees join or leave Fabrikam.</p>
<p>The solution for this is <strong>federated authentication</strong>, where my app can use a third-party (Fabrikam&rsquo;s) authentication provider (also known as a Security Token Service or STS) to authenticate users. I won&rsquo;t try to explain federation from scratch, because Microsoft&rsquo;s patterns &amp; practices has a <a title="patterns &amp; practices introduction to federation" href="http://msdn.microsoft.com/en-us/library/ff359101.aspx">great write-up</a>. You can squint and ignore some of the claims stuff as it&rsquo;s not super relevant to what I&rsquo;m describing here.</p>
<p>To build an app with federated auth using RIA Services, most folks resort to <strong>passive federation</strong>, which you might be familiar with from using things like your LiveID login. The passive approach is browser-friendly and uses a combination of cookies, URIs and redirects to log you in to the site you&rsquo;re trying to access.  The downside in the case of Silverlight is that you cannot complete the login inside your app: you have to navigate away from the Silverlight control, type in your credentials into a HTML (or ASPX) page, and then be bounced back by your browser to reload the control. Another downside is that you need to host a HTML page for the user to type in their credentials, in addition to the Silverlight app and the web services, which might not be desirable.</p>
<p>The following diagram has a more detailed explanation of the exact flow that happens during the passive scenario:</p>
<img alt="Passive federation with RIA Services" src="{{ site.baseurl }}/images/posts/tumblr/tumblr_ll3zirBuBj1qccglw.png" style="margin-left: auto; margin-right: auto"/>
<p>From what I&rsquo;ve seen, most folks use this passive pattern with RIA today. Eugenio Pace&rsquo;s blog <a title="Passive federation with RIA Services" href="http://blogs.msdn.com/search/searchresults.aspx?q=RIA%20Services%20&amp;sections=3815">has a few great write-ups and demos that you can try out</a>. Some are a bit dated, but you should be able to find your way around.</p>
<p>What I wanted to try out was the <strong>active federation</strong> case, in which the Silverlight app talks dirctly to the identity provider. This has the benefit of having the user type their credentials directly into the Silverlight app, without having to leave the page, which results in a more polished and faster user experience. Eventually, as the crypto support in Silverlight improves, it may end up being a more secure solution as well, if we are able to encrypt and sign the tokens that are being exchanged. The following diagram demonstrates the slightly different message flow in the active case. Note that the app itself is doing most of the work, not the browser.</p>
<img alt="Active federation with RIA Services" src="{{ site.baseurl }}/images/posts/tumblr/tumblr_ll414g0h4u1qccglw.png" style="margin-left: auto; margin-right: auto"/>
<p>Some interesting considerations emerged in conversation with Robert O'Brien, an architect here at Microsoft. Please consider these before building your own app:</p>
<ul><li>One of the things we gain in this approach over a passive federation approach is the ability to log the user in from inside our UI. That provides for a better user experience, but it also puts the burden on our app to expose a credentials UI and pass the token over to the service. Potentially this could be a security risk for our app. Some folks believe that this goes against the spirit of federated authentication since it takes control away from the identity provider.</li>
<li>The implementation shown here relies on WS-Trust as the protocol for the token. Unfortunately in today&rsquo;s federated web, it seems like only ADFS and maybe OpenID servers honor that protocol. If you are trying to use WS-Trust and use Facebook/Yahoo/Google (either directly or through Azure&rsquo;s Access Control Service) as the identity provider, you will find that WS-Trust is probably not supported. So outside the enterprise environment, this approach might not work.</li>
</ul><p>Now some specifics of the sample. It&rsquo;s a simple master/detail application that allows you to display and edit data. There are two registered users <strong>fabrikam\yavor</strong> and <strong>fabrikam\test</strong> and the password for both of them is <strong>12345</strong>. If you don&rsquo;t log in and try to display the data, you will get an error because of the <strong>EnableClientAccess </strong>attribute.</p>
~~~ csharp
[RequiresAuthentication]
[EnableClientAccess()]
public class CustomersService : LinqToEntitiesDomainService<AdventureWorksLT2008R2Entities>;
~~~
<p>All authenticated users can view the data, but only users in the Editor role can make changes to it. Only the user <strong>fabrikam\yavor </strong>is in that role.</p>
~~~ csharp
[RequiresRole("Editors")]
public void UpdateCustomer(Customer currentCustomer)
~~~
<p>The bottom line here is that even though we have a STS with external user authentication, we kept authorization local to our app (we use ASP.NET roles). That&rsquo;s a choice you can make as a developer - you can also externalize the user roles (or claims) as part of the STS, or keep them local.</p>
<p>There are lots of extra details in the app, which you&rsquo;ll see by exploring the source.</p>
<p><strong><a title="Sample download" href="http://code.msdn.microsoft.com/Active-Federation-using-bfaaea97">The code is available here.</a></strong> Note that you need the following installed on your machine for it to work:</p>
<ul><li>Visual Studio 2010</li>
<li>SQL Server 2008 R2 Express</li>
<li>SQL Server 2008 R2 Management Studio Express </li>
<li>WCF RIA Services V1 SP1/SP2</li>
<li><a title="WIF download" href="http://msdn.microsoft.com/en-us/security/aa570351">Windows Identity Foundation Runtime</a></li>
<li><a title="AdventureWorks LT database sample" href="http://msftdbprodsamples.codeplex.com/releases/view/55926">The AdventureWorks LT 2008R2 database for SQL Server Express 2008 R2</a></li>
</ul><p>Here are some additional notes on the structure of the sample:</p>
<ul><li>First, please run the included <strong>SetupCertificates.cmd</strong> script inside the <strong>Scripts</strong> folder</li>
<li>The solution will create two applications in your local IIS instance: <strong>AddressBookFederatedAuth </strong>and <strong>IdentityProviderAndSts</strong>. Make sure you launch Visual Studio as <strong>Administrator</strong>, so it has permission to create those. Also make sure you enable <strong>HTTPS</strong> so both of the applications in the <strong>IIS Manager</strong></li>
<li>Change the <strong>Application pool </strong>under which those applications are running to use the <strong>Network Service</strong> account. </li>
<li>Use SQL Server Management Studio, make sure <strong>Network Service</strong> is added as a login to your SQL instance. When you add the account, make sure you give him some sane Server Roles (for example <strong>sysadmin</strong>). Let me know in the comments if you find any of these steps difficult and I can provide more detailed instructions.</li>
</ul><p>I want to credit <a title="Eugenio's blog" href="http://blogs.msdn.com/b/eugeniop/">Eugenio Pace</a>, <a title="Kyle's blog" href="http://blogs.msdn.com/b/kylemc/">Kyle McClellan</a>, and the folks behind the <a title="Download the Identity Developer Training Kit" href="http://go.microsoft.com/fwlink/?LinkId=148795">Identity Developer Training Kit</a>, from where I shamelessly stole some code.</p>
