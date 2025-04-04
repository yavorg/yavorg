---
title: Building portable C# apps with Mobile Services
date: '2013-07-20T18:37:00-07:00'
tags:
- Azure Mobile Services
- Windows Phone
redirect_from: 
- /post/56006227192/
- /post/56006227192/portable-apps-mobile-services-pcl/
---
<p>I don&rsquo;t want to be one of those people who&rsquo;s always going on about their ViewModel, but I was pretty pleased with the way I was able to factor the <a href="https://github.com/yavorg/samples/tree/master/SlapChat">sample</a> from my <a href="{{ site.baseurl }}{% post_url tumblr/2013-07-01-build-mobile-services %}">Build talk</a>, so I thought I would share. My goal was to structure my Windows Phone 8 code in a way so I could reuse as much of it as possible if I decided to build a Windows Phone 7.5 or Windows Store version of the app.</p>
<p>I mostly followed standard MVVM best practices here, and I assume most of you are familiar with those. I leaned on <a href="http://mvvmlight.codeplex.com/">MvvmLight</a> as my framework of choice. <strong>The strategy here was to try to make all ViewModel and Model code easily portable. This was a natural fit with the <a href="http://msdn.microsoft.com/en-us/library/vstudio/gg597391(v=vs.110).aspx">Portable Class Library</a> (PCL) support introduced in VS 2012. </strong>This is the high-level project structure I devised:</p>
<img src="{{ site.baseurl }}/images/posts/tumblr/tumblr_inline_mq9dnsDqHD1qz4rgp.png"/>
<p>A lot of the app chrome is deeply platform-specific, so I needed platform-specific projects for those pieces: <strong>MyApp.Win8</strong> and <strong>MyAppWP8</strong>. The key was to separate the ViewModel and Model code, which can be made portable via their own PCL projects: <strong>MyApp.ViewModel</strong> and <strong>MyApp.Model</strong>. These projects can be referenced from a variety of C# platforms (.NET, Silverlight, Phone, Windows, even Xbox) without any modification, that&rsquo;s the beauty of this model.</p>
<p>There is one caveat with a PCL project: the way it works is by exposing a sort of lowest-common-denominator .NET surface area that is guaranteed to work across all the different platforms where that library is supposed to work. Clearly, the further back you go by adding more and more legacy platforms to the supported set, the .NET surface area your project can target shrinks. So you have to make a mindful choice given your business need, luckily VS makes that choice very easy for you. Here is what I picked for my PCL projects:</p>
<img alt="image" src="{{ site.baseurl }}/images/posts/tumblr/tumblr_inline_mq9e50amuB1qz4rgp.png"/>
<p>Let&rsquo;s talk about the two PCL projects in a little bit more detail.</p>
<h3>The ViewModel project</h3>
<p>This part is not specific to Mobile Services in any way, and has been covered well already by <a href="http://blog.tattoocoder.com/2013/01/portable-mvvm-light-move-your-view.html">other folks</a>. The key thing to realize is that the ViewModel code needs to depend on the MvvmLight framework. So someone needed to design a subset of MvvmLight that can be referenced from a PCL project. Luckily Laurent has already done the hard work here, so just reference the <a href="https://www.nuget.org/packages/Portable.MvvmLightLibs/">Portable.MvvmLightLibs</a> project from NuGet, and you&rsquo;re done.</p>
<h3>The Model project</h3>
<p>This was the slightly more interesting part, since this is the project that needs to talk to Mobile Services and load/save data for my application. Other than my model classes, which are pretty straightforward, I designed a few <strong>service interfaces</strong> that my ViewModel code could use to talk to Mobile Services. Here are those interfaces:</p>
~~~ csharp
public interface IChatService
{
    Task CreateUserAsync(User user);

    Task<ObservableCollection> ReadFriendsAsync(string userId);
    Task<ObservableCollection> CreateFriendsAsync(string userId, string emailAddresses);

    Task<ObservableCollection> ReadPhotoRecordsAsync(string userId);

    Task CreatePhotoRecordAsync(PhotoRecord record);

    Task<ObservableCollection> ReadPhotoContentAsync(string id);
    void DeletePhotoContent(string id);

    Task UploadPhotoAsync(Uri location, string secret, Stream photo);
    Stream ReadPhotoAsStream(Uri location);
    Uri ReadPhotoAsUri(Uri location);

}

public interface IAuthenticationService
{
    Task LoginAsync(string token);
}
~~~
<p>My particular scenario revolved around a picture messaging service (ala SnapChat), which hopefully explains the method names; it also supported authentication. </p>
<p>Once I factored out all the networking via these interfaces, it was super easy to make my app testable. I could write a local mock implementation of the interfaces and my app would work client-side without any networking. I could then write the real implementation and swap these out very easily when I wanted to build the actual working version of the app. The key thing was to use MvvmLight&rsquo;s support for <a href="http://en.wikipedia.org/wiki/Inversion_of_control">IoC</a>, so I could easily swap the implementations. The basic principle here is to use a <strong>locator</strong> every time you need a reference to the implementation of a given service interface. It&rsquo;s the locator&rsquo;s responsibility to supply an implementation of the interface. Usually you configure that in one central location, which gives you the ability to switch between implementations with a one-line code change or configuration change. Here&rsquo;s what the code looks like. In the ViewModel, always reference your services like so: </p>
~~~ csharp
IChatService chatService = ServiceLocator.Current.GetInstance<IChatService>();
~~~
<p>Then in the ViewModelLocator class, you can switch between implementations:</p>
~~~ csharp
SimpleIoc.Default.Register<IChatService, ConnectedChatService>();
SimpleIoc.Default.Register<IAuthenticationService, ConnectedAuthService>();

// For testing
// SimpleIoc.Default.Register<IChatService, MockChatService>();
// SimpleIoc.Default.Register<IAuthenticationService, MockAuthService>();
~~~
<p>This gives us nice flexibility, but we still haven&rsquo;t addressed the implementation of <strong>ConnectedChatService</strong> and <strong>ConnectedAuthService:</strong> how you actually use Mobile Services from a PCL project.</p>
<p>Implementing ConnectedChatService is very straightforward with the new <a href="https://www.nuget.org/packages/WindowsAzure.MobileServices/">WindowsAzure.MobileServices</a> SDK on NuGet. One of its least-advertised features is that it fully supports PCL, so you can painlessly reference it from your project and use Mobile Services as you normally would.</p>
<p>Implementing ConnectedAuthService is slightly trickier. The issue here is that <strong><a href="http://msdn.microsoft.com/en-us/library/windowsazure/microsoft.windowsazure.mobileservices.mobileserviceclient.aspx">MobileServiceClient</a></strong> has two interesting login methods that act slightly differently.</p>
<ul><li><a href="http://msdn.microsoft.com/en-us/library/windowsazure/microsoft.windowsazure.mobileservices.mobileserviceclientextensions.loginwithmicrosoftaccountasync.aspx"><strong>LoginWithMicrosoftAccountAsync(string authenticationToken)</strong><br/></a>Assumes you are using the <a href="http://msdn.microsoft.com/en-us/live/ff519582.aspx">Live Connect SDK</a> to authenticate out-of-band and just pass Mobile Services the token. No UI required, so it can be used from a PCL project.</li>
<li><strong><a href="http://msdn.microsoft.com/en-us/library/windowsazure/microsoft.windowsazure.mobileservices.mobileserviceclientextensions.loginasync.aspx">LoginAsync(MobileServiceAuthenticationProvider provider</a></strong><strong><a href="http://msdn.microsoft.com/en-us/library/windowsazure/microsoft.windowsazure.mobileservices.mobileserviceclientextensions.loginasync.aspx">)<br/></a></strong>Invokes web view/oAuth-based UI to obtain token, no Live SDK  required. Uses platform-specific UI, so can&rsquo;t use from a PCL project.</li>
</ul><p>If we are using the Live SDK to log in the user, then you must use the LoginWithMicrosoftAccountAsync method to provide the auth token obtained from Live to Mobile Services. You can use that method from the PCL project so if this is the scenario you&rsquo;re using, your ConnectedAuthService implementation can live in the PCL project.</p>
<p>If you don&rsquo;t want to bother with the Live SDK and just want the simplest possible auth experience, you are likely using the LoginAsync method. That method needs platform-specific UI, so it is not available when you use the Mobile Services SDK from the PCL project. But fear not, our project structure allows us to solve this cleanly:</p>
<ol><li>Add a reference to the Mobile Services NuGet package to your platform-specific project (MyApp.Win8 or MyApp.WP8 in the above example). </li>
<li>Implement ConnectedAuthService in that project</li>
<li>You can now reference LoginAsync.</li>
</ol><p>If you think about it, this actually makes sense. The code for hosting a web view is different on every platform, so the Mobile Services SDK can&rsquo;t magically make that work in the PCL project. There needs to be some platform-specific code in the SDK that does the right thing per platform, and that&rsquo;s why the LoginAsync method is only available in the platform-specific project.</p>
<p>Hope this was useful, for more details see the <a href="https://github.com/yavorg/samples/tree/master/SlapChat">sample code</a> on GitHub.</p>
