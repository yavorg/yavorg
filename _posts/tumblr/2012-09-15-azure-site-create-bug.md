---
title: Site creation in "azure" tool temporarily unavailable
date: '2012-09-15T21:14:00-07:00'
tags:
- Windows Azure
- Node.js
redirect_from:
- /post/31636752900/
- /post/31636752900/azure-site-create-bug/
---
<p><em><strong>Update: It appears that azure site start/stop are also affected. We are working on a fix for both issues.</strong></em></p>
<p>We are experiencing a temporary bug in the &ldquo;azure&rdquo; cross-platform tool for Mac and Linux. Some recent changes in the Windows Azure Websites API have broken some of the site creation functionality supported by the tool.</p>
<p>If you create a site using the site create command similar to what is shown below,  you will experience an &ldquo;Access is denied.&rdquo; message.</p>
~~~ shell
azure site create yavortest --git
info:    Executing command site create
+ Enumerating locations
+ Enumerating sites
+ Retrieving user information
error:   Access is denied.
error:   site create command failed
~~~
<p>We are sorry for this temporary inconvenience. We will push a fix to the Azure Websites API within the upcoming few weeks. In the mean time, please use the portal to create your sites and configure Git publishing.</p>
