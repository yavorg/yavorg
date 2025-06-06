---
title: Ski weather app with the new Mobile Services scheduler
date: '2012-12-21T13:01:00-08:00'
tags:
- Azure Mobile Services
redirect_from:
- /post/38488024433/
- /post/38488024433/mobile-services-scheduler/
---
<p><strong><em>Update:</em></strong><em> Corrected the server script code snippet.<br/><strong>Update 2:</strong>The example below will unfortunately no longer work due to Twitter API changes&hellip; I need to update it. In the mean time, our official documentation shows a very similar scenario, so <a href="http://www.windowsazure.com/en-us/develop/mobile/tutorials/schedule-backend-tasks/">check that out</a>.<br/></em></p>
<p>As <a href="http://weblogs.asp.net/scottgu/archive/2012/12/21/great-updates-to-windows-azure-mobile-services-web-sites-sql-data-sync-acs-media-more.aspx">Scott announced earlier today</a>, Mobile Services is celebrating the holiday season by bringing you some cool new features, including the ability to execute scheduled jobs (or &ldquo;cron jobs&rdquo;). </p>
<blockquote>
<p>&quot;Windows Azure Mobile Services now supports the ability to easily schedule background jobs (aka CRON jobs) that execute on a pre-set timer interval, and which can run independently of a device accessing the service (ensuring that you don’t block or slow-down any requests from your users).  This job scheduler functionality allows you to perform a variety of useful scenarios without having to create or manage a separate VM.&quot;</p></blockquote>
<p>Some of the scenarios you can enable with it include:</p>
<ul><li>Periodically purging old/duplicate data from your tables.</li>
<li>Periodically querying and aggregating data from external web service (tweets, RSS entries, location information) and cache it in your tables for subsequent use.</li>
<li>Periodically processing/resizing images submitted by users of your service.</li>
<li>Scheduling the sending of push notifications or SMS messages to your customers to ensure they arrive at the right time of the day.</li>
</ul>
<p>Here I want to give a more detailed demonstration of the feature and celebrate another winter tradition: skiing. Let’s walk through a simple scenario where we will use a scheduled job to aggregate and analyze tweets related to ski conditions near Seattle. The source code is available <a href="https://github.com/yavorg/samples/tree/master/SeattleSkiBuddy">here</a>.</p>
<p><img alt="Finished SeattleSkiBuddy app screenshot" src="{{ site.baseurl }}/images/posts/tumblr/mobile-services-scheduler-1.png"/></p>
<p>Let’s assume we already have a mobile service called <strong>skibuddy</strong>. We can navigate to the new Scheduler tab in the Windows Azure portal to create a scheduled job. </p>
<p><img alt="Scheduler tab in Azure portal" src="{{ site.baseurl }}/images/posts/tumblr/mobile-services-scheduler-2.png"/></p>
<p>Let’s go ahead and create a job called <strong>getUpdates</strong>. We will use the default interval of <strong>15 minutes</strong> that is already selected in the dialog.</p>
<p><img alt="Create new scheduled job dialog" src="{{ site.baseurl }}/images/posts/tumblr/mobile-services-scheduler-3.png"/></p>
<p>Once the job is created we will now see it in the summary view. You will note that the job starts off as disabled, so let’s click the job name to add our script.</p>
<p><img alt="Scheduler tab in Azure portal" src="{{ site.baseurl }}/images/posts/tumblr/mobile-services-scheduler-4.png"/></p>
<p>On the job details page, we see information about our job and also we also have an opportunity to change the schedule. For now let’s leave everything as-is and navigate over to the <strong>Script</strong> tab.</p>
<p><img alt="Scheduled job detail page" src="{{ site.baseurl }}/images/posts/tumblr/mobile-services-scheduler-5.png"/></p>
<p>Here we see the script editor window that has been pre-populated with a default empty script.</p>
<p><img alt="Scheduled job detail page with empty script" src="{{ site.baseurl }}/images/posts/tumblr/mobile-services-scheduler-6.png"/></p>
<p>Let’s replace the default script with the following one.</p>
~~~ javascript
var updatesTable = tables.getTable('updates');
var request = require('request');

function getUpdates() {   
    // Check what is the last tweet we stored when the job last ran
    // and ask Twitter to only give us more recent tweets
    appendLastTweetId(
        'http://search.twitter.com/search.json?q=ski%20weather&amp;result_type=recent&amp;geocode=47.593199,-122.14119,200mi&amp;rpp=100', 
        function twitterUrlReady(url){
            request(url, function tweetsLoaded (error, response, body) {
                if (!error &amp;&amp; response.statusCode == 200) {
                    var results = JSON.parse(body).results;
                    if(results){
                        console.log('Fetched new results from Twitter');
                        results.forEach(function visitResult(tweet){
                            if(!filterOutTweet(tweet)){
                                var update = {
                                    twitterId: tweet.id,
                                    text: tweet.text,
                                    author: tweet.from_user,
                                    location: tweet.location || "Unknown location",
                                    date: tweet.created_at
                                };
                                updatesTable.insert(update);
                            }
                        });
                    }            
                } else { 
                    console.error('Could not contact Twitter');
                }
            });
            
        });
}

// Find the largest (most recent) tweet ID we have already stored
// (if we have stored any) and ask Twitter to only return more
// recent ones
function appendLastTweetId(url, callback){
    updatesTable
    .orderByDescending('twitterId')
    .read({success: function readUpdates(updates){
        if(updates.length){
            callback(url + '&amp;since_id=' + (updates[0].twitterId + 1));
        } else {
            callback(url);
        }
    }});
}

function filterOutTweet(tweet){
    // Remove retweets and replies
    return (tweet.text.indexOf('RT') === 0 || tweet.to_user_id);
}
~~~
<p>Here is what the script does:</p>
<ul><li>First, the script goes out to the Twitter API and collects tweets from Seattle’s geographical area. Because we don’t want to get the same tweets over again, we check if a given tweet has already been fetched and only ask Twitter for newer tweets.</li>
<li>Once we collect the tweets we process each tweet to remove retweets and replies. You can imagine this logic being a lot more complex and adding deeper analysis.</li>
<li>Lastly, we store the results in a table called <strong>updates</strong>, which we have pre-created. You can easily create the table yourself by going back to the main page for your mobile service in the portal and using the <strong>Data</strong> tab. You don’t have to use tables in your scheduled job. You could just as easily send a push notification to users of your app alerting them of new updates from Twitter. For more information, see <a href="https://www.windowsazure.com/en-us/develop/mobile/tutorials/push-notifications-to-users-dotnet/" title="Push notification tutorial">this tutorial</a>.</li>
</ul><p>Now that we&rsquo;ve entered the script, we can press the <strong>Save</strong> button and then select <strong>Run once</strong> to execute a trial run. The Run once functionality lets you easily test out your job script before you enable the job for recurring execution.</p>
<p><img alt="Scheduled job detail page with finished script" src="{{ site.baseurl }}/images/posts/tumblr/mobile-services-scheduler-7.png"/></p>
<p>Once you run the script, head over to the <strong>Data</strong> tab on the main page for your mobile service and select the <strong>updates</strong> table.</p>
<p><img alt="Data tab in Azure portal" src="{{ site.baseurl }}/images/posts/tumblr/mobile-services-scheduler-8.png"/></p>
<p>You should see it has now been populated with the tweet data we fetched in our job. Our mobile clients can now read the data from that table and display it to their users.</p>
<p>Next, open the <strong>Logs</strong> tab from the main portal page for your mobile service. </p>
<p><img alt="Logs tab in Azure portal" src="{{ site.baseurl }}/images/posts/tumblr/mobile-services-scheduler-9.png"/></p>
<p>You should see an entry generated by our scheduled job script. Any calls to the <strong>console</strong> API from your job script and all errors generated by that script will be logged here. This makes the <strong>Logs</strong> table very useful while debugging and monitoring your scheduled job.</p>
<p>Now that we have made sure the job is working as expected, we are ready to enable it. We can go back to the <strong>Scheduler</strong> tab, select our job, and click <strong>Enable</strong>.</p>
<div><img alt="Scheduler tab in Azure portal" src="{{ site.baseurl }}/images/posts/tumblr/mobile-services-scheduler-10.png"/></div>
<p>Once the job has run for a while, we can check the <strong>updates</strong> table and also the logs to ensure it is executing. If we return to the <strong>Scheduler</strong> tab, it conveniently provides last run and next run data for the job.</p>
<p><img alt="Scheduler tab in Azure portal" src="{{ site.baseurl }}/images/posts/tumblr/mobile-services-scheduler-11.png"/></p>
<p>This was a quick overview of the new scheduled jobs feature in Mobile Services. I&rsquo;ll skip the rest of the app implementation as it has been covered <a href="https://www.windowsazure.com/en-us/develop/mobile/tutorials/get-started-with-data-dotnet/">many times</a>, you can find the source at the GitHub link above. </p>
<p>Here are some extra details on the feature that we didn’t get to cover:</p>
<ul><li>If you select <strong>On demand</strong> for the job schedule, the job will never automatically run and will always stay disabled. You can manually run the job via the <strong>Run once</strong> button. This is useful for implementing one-time setup and cleanup scripts for your service, especially in development.</li>
<li>Different schedules are supported, including minutes, hours, days, and months. </li>
<li>By default mobile services users in free mode are allowed to create 1 scheduled job. If more jobs are needed for your scenario, upgrade your mobile services to reserved mode via the <strong>Scale</strong> tab, and you will now be able to create up to 10 jobs.</li>
</ul><div></div>
