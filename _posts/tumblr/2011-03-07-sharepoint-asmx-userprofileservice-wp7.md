---
title: Accessing SharePoint UserProfileService from Windows Phone 7
date: '2011-03-07T12:05:00-08:00'
tags:
- WCF
- Silverlight
- Windows Phone
redirect_from:
- /post/3706541868/
- /post/3706541868/sharepoint-asmx-userprofileservice-wp7/
---
<p><em><strong>UPDATE: &hellip; and it wasn&rsquo;t long before Matthew McDermott went ahead and implemented ths as an actual sample, which you can get <a title="Matthew's sample showing how to do this" href="http://www.ableblue.com/blog/archive/2011/03/18/silverlight-and-sharepoint-user-profile-service-guids.aspx">here</a>.</strong></em></p>
<p>A while back I <a title="Workaround ASMX services using char and guid for Silverlight 4" href="http://blogs.msdn.com/b/silverlightws/archive/2010/05/26/workaround-for-accessing-some-asmx-services-from-silverlight-4.aspx">blogged a workaround</a>for accessing some ASMX services from Silverlight 4. The problem was that the <strong>guid</strong> and <strong>char </strong>types that those services return are not recognized by Silverlight and you end up with the exception below. One of the important services affected by this is SharePoint&rsquo;s UserProfileService, which I realize is pretty important to a lot of developers.</p>
~~~
System.ServiceModel.Dispatcher.NetDispatcherFaultException was unhandled by user code. The formatter threw an exception while trying to deserialize the message: There was an error while trying to deserialize parameter http://tempuri.org/:HelloWorldResponse. The InnerException message was 'Error in line 1 position 268. Element 'http://tempuri.org/:HelloWorldResult' contains data of the 'http://microsoft.com/wsdl/types/:guid' data contract. The deserializer has no knowledge of any type that maps to this contract. Add the type corresponding to 'guid' to the list of known types - for example, by using the KnownTypeAttribute attribute or by adding it to the list of known types passed to DataContractSerializer.'.  Please see InnerException for more details.
~~~
<p>(Same goes for <a href="http://microsoft.com/wsdl/types/:char">http://microsoft.com/wsdl/types/:char</a>)</p>
<p>We have already fixed this issue and you will not need the workaround anymore in the next version of Silverlight.</p>
<p>In implementing the workaround, I used an <a title="IClientMessageInspector over at MSDN" href="http://msdn.microsoft.com/en-us/library/system.servicemodel.dispatcher.iclientmessageinspector(v=VS.95).aspx">IClientMessageInspector</a>, which unfortunately is only available starting in Silverlight 4. So all of our developers using Silverlight 3 (and in particular folks writing apps for Windows Phone 7) cannot use the workaround.</p>
<p>Fortunately, there is a way to &ldquo;fake&rdquo; an IClientMessageInspector on older versions of Silverlight, and I wrote a sample for that back in the Silverlight 2 days. So combining these two samples together, you can be on your way:</p>
<ol><li>Create a &ldquo;fake&rdquo; IClientMessageInspector like in <a title="Fake IClientMessageInspector sample for Silverlight 2" href="http://code.msdn.microsoft.com/Project/Download/FileDownload.aspx?ProjectName=silverlightws&amp;DownloadId=3473">this sample</a></li>
<li>Use the IClientMessageInspector implementation that treats the <strong>guid</strong> and <strong>char</strong> types emitted by ASMX from <a title="ASMX workaround sample for Silverlight 4" href="http://code.msdn.microsoft.com/Project/Download/FileDownload.aspx?ProjectName=silverlightws&amp;DownloadId=11647">this sample</a>.</li>
</ol><p>Hopefully this will unblock folks out there, if there is significant interest here I can create a combined sample that shows the whole thing end-to-end.</p>
