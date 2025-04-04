---
title: Debugging SGEN LoaderExceptions errors
date: '2011-05-06T16:33:00-07:00'
tags:
- WCF
redirect_from:
- /post/5255977780/
- /post/5255977780/sgen-loaderexceptions-error/
---
<p>Recently we were contacted by a customer who was building a Release version of their assembly in Visual Studio and encountered the following error:</p>
<img src="{{ site.baseurl }}/images/posts/tumblr/tumblr_lkspj38nwO1qccglw.png" alt="SGEN error in Visual Studio" style="margin-left: auto; margin-right: auto"/>
<p>You could get the same error if you attempt to run the sgen.exe tool on the built assembly:</p>
~~~ 
Microsoft ® Xml Serialization support utility
[Microsoft ® .NET Framework, Version 4.0.30319.1]
Copyright © Microsoft Corporation. All rights reserved.
Error: Unable to load one or more of the requested types. Retrieve the LoaderExceptions property for more information.

If you would like more help, please type "sgen /?".</p>
~~~
<p>The reason why this happens is that in Release builds, Visual Studio attempts to generate a serialization assembly containing the types in your solution, to improve XmlSerializer serialization performance, should you choose to serialize your types. <strong>This can be disabled by going to the Build tab of the project properties and setting &ldquo;Generate serialization assembly&rdquo; to &ldquo;Off&rdquo;.</strong></p>
<p>However if you legitimately want to take advantage of the serialization assembly, this error can be very frustrating to diagnose. The exception text leads you to believe Fusion is having a problem loading one of the dependent assemblies that your assembly is referencing. That may be the case, and you can easily diagnose that problem by launching the <strong>Fusion Log Viewer</strong>, then enabling assembly logging, and then rebuilding your project (or rerunning sgen.exe). <a title="Hanselman on Fusion LoaderErros" href="http://www.hanselman.com/blog/BackToBasicsUsingFusionLogViewerToDebugObscureLoaderErrors.aspx">Scott Hanselman has a good write-up on how to do this.</a></p>
<p>Sometimes after inspecting the Fusion log, you&rsquo;ll see that all assemblies were infact loaded correctly, but you&rsquo;re still getting this error. In recent investigation we found that sometimes sgen.exe will show this error, and it won&rsquo;t have to do with an assembly loading problem. Here is how to get to the bottom of that case.</p>
<ol><li>In Visual Studio create a dummy <strong>Console Application</strong>. We won&rsquo;t actually use this, it&rsquo;s just a way for us to start sgen.exe in the debugger.</li>
<li>Go to the Debug tab of the project properties and set 
<ul><li><strong>Start external program</strong>= C:\Program Files (x86)\Microsoft SDKs\Windows\v7.0A\Bin\NETFX 4.0 Tools\sgen.exe</li>
<li><strong>Command line arguments</strong>= /f /a:MyAssembly.dll </li>
<li><strong>Working directory</strong>= C:\FolderContainingMyAssembly</li>
</ul></li>
<li>Configure the debugger to break on all exceptions by going to the <strong>Debug</strong> menu and selecting <strong>Exceptions</strong> and then checking <strong>Thrown</strong> next to <strong>Common Language Runtime Exceptions</strong></li>
<li>Also ensure that if you go to <strong>Tools</strong> &gt; <strong>Options</strong> &gt; <strong>Debugging</strong> that <strong>Enable Just My Code</strong> is unchecked.</li>
<li>Then you are ready to press <strong>F5</strong>. If everything works correctly, Vistual Studio will break at an exception. Right click on <strong>sgen.exe </strong>anywhere in the <strong>Call Stack </strong>window and load the symbols from the <strong>Microsoft Symbol Servers</strong>.<br/> <img src="{{ site.baseurl }}/images/posts/tumblr/tumblr_lksrk1Zm0P1qccglw.png" alt="Loading sgen.exe symbols" style="margin-left: auto; margin-right: auto"/></li>
<li>Once the symbols have loaded, press F5 until you get to a <strong>ReflectionTypeLoadException</strong>. At that point you can select <strong>View Details</strong> and inspect the <strong>LoaderExceptions</strong> property yourself. This property will usually reveal the real source of the problem:<br/><img src="{{ site.baseurl }}/images/posts/tumblr/tumblr_lksru3zrry1qccglw.png" alt="ReflectionTypeLoadException details" style="margin-left: auto; margin-right: auto"/></li>
</ol><p>Hope this is useful!</p>
