---
title: JsonValue guts and glory
date: '2010-10-29T08:41:00-07:00'
tags:
- JavaScript
tumblr_url: http://hashtagfail.com/post/1432485895/jsonvalue-json-walkthrough
---
<p>As part of the new set of WCF features at <a href="http://wcf.codeplex.com">http://wcf.codeplex.com</a>, we&rsquo;re porting a feature that has existed in Silverlight to the framework. <strong>JsonValue </strong>is the base abstract class for a set of types that you can use to work with JSON data in a weakly-typed way: <strong>JsonPrimitive</strong>, <strong>JsonArray</strong>, and <strong>JsonObject</strong>. The idea here is similar to how you use XElement to work with XML data: you don&rsquo;t need to pre-generate a strong type to deserialize into.</p>
<h3>Basics</h3>
<p>We start with a JSON string, and we parse it easily in one line.</p>
~~~ javascript 
string customers = @"
[
    {   "ID" : "538a868a-c575-4fc9-9a3e-e1e1e68c70c5",
        "Name" : "Yavor",
        "DOB" : "1984-01-17",
        "OrderAmount" : 1e+4,
        "Friends" : [
            "007cf155-7fb4-4070-9d78-ade638df44c7",
            "91c50a40-7ade-4c37-a88f-3b7e066644dc"
        ]
    },
    {   "ID" : "007cf155-7fb4-4070-9d78-ade638df44c7",
        "Name" : "Joe",
        "DOB" : "1983-02-18",
        "OrderAmount" : 50000,
        "Friends" : [
            "91c50a40-7ade-4c37-a88f-3b7e066644dc"
        ]
    },
    {   "ID" : "91c50a40-7ade-4c37-a88f-3b7e066644dc",
        "Name" : "Miguel",
        "DOB" : "1982-03-19",
        "OrderAmount" : 25.3e3,
        "Friends" : [
            "007cf155-7fb4-4070-9d78-ade638df44c7"
        ]
    }
]";

JsonArray ja = JsonValue.Parse(customers) as JsonArray;
~~~
<p>Note some interesting aspects of this JSON: guids and dates are hidden inside strings.</p>
<p>You can easily construct the same object programmatically.</p>
~~~ javascript
JsonArray ja = new JsonArray {
    new JsonObject {
        {"ID", new Guid("538a868a-c575-4fc9-9a3e-e1e1e68c70c5")},
        {"Name", "Yavor"},
        {"DOB", new DateTime(1984, 01, 17)},
        {"OrderAmount", 10000},
        {"Friends", new JsonArray{
            new Guid("007cf155-7fb4-4070-9d78-ade638df44c7"),
            new Guid("91c50a40-7ade-4c37-a88f-3b7e066644dc")
        }}
    },
    new JsonObject {
        {"ID", new Guid("007cf155-7fb4-4070-9d78-ade638df44c7")},
        {"Name", "Joe"},
        {"DOB", new DateTime(1983, 02, 18)},
        {"OrderAmount", 50000},
        {"Friends", new JsonArray{
            new Guid("91c50a40-7ade-4c37-a88f-3b7e066644dc")
        }}
    },
    new JsonObject {
        {"ID", new Guid("91c50a40-7ade-4c37-a88f-3b7e066644dc")},
        {"Name", "Miguel"},
        {"DOB", new DateTime(1982, 03, 19)},
        {"OrderAmount", 25300},
        {"Friends", new JsonArray{
            new Guid("007cf155-7fb4-4070-9d78-ade638df44c7")
        }}
    }
};
~~~
<p>Getting a value works easily using an indexer and a cast. You can cast the value into any type you like (if the cast is meaningful). The ReadAs generic method allows you a more fluent way to cast, but does exactly the same thing as the cast itself.</p>
~~~ javascript
// Get a value
string name = (string)ja[0]["Name"];
name = ja[0]["Name"].ReadAs<string>();
~~~
<p>Setting a value is also straightforward. There are implicit casts defined from all the common CLR primitives into JsonPrimitive, so you don&rsquo;t have to worry about creating the JsonPrimitive yourself.</p>
~~~ javascript
// Set a value
ja[0]["Name"] = "Yavor Georgiev";
ja[1]["ID"] = new Guid();
ja[2]["OrderAmount"] = 30000;
~~~
<p>To make the type even more useful, we allow you to parse out any values hidden inside strings (such as guids and dates in this example) by using the same cast/ReadAs metaphor you use already.</p>
~~~ javascript
Guid id = ja[0]["ID"].ReadAs<Guid>(); 
DateTime dob = ja[0]["DOB"].ReadAs<DateTime>(); 
float orderAmount = ja[0]["OrderAmount"].ReadAs<float>();
~~~
<p>This concludes what you need to know in order to use the types the following contains information about some advanced capabilities that might not be needed by everyone.</p>
<h3>Advanced: &ldquo;safe&rdquo; casts</h3>
<p>The cast/ReadAs method works well, however if the given cast/parse operation cannot be accomplished, it will throw an exception. This might make it difficult to write safe code that uses the type. You could do something like this, but it is very ugly.</p>
~~~ javascript
Guid id;
ja[0]["ID"].TryReadAs<Guid>(out id);
DateTime dob;
ja[0]["DOB"].TryReadAs<DateTime>(out dob);
float orderAmount;
ja[0]["OrderAmount"].TryReadAs<float>(out orderAmount);
~~~
<p>We&rsquo;ve addressed this by adding the notion of a fallback to the ReadAs method: if the method itself fails, it will return the fallback value provided.</p>
~~~ javascript
Guid id = ja[0]["ID"].ReadAs<Guid>(new Guid());
DateTime dob = ja[0]["DOB"].ReadAs<DateTime>(DateTime.Now);
float orderAmount = ja[0]["OrderAmount"].ReadAs<float>(0);
~~~
<p>We call this a &ldquo;safe&rdquo; cast.</p>
<h3>Advanced: &ldquo;safe&rdquo; indexers</h3>
<p>Just like an unsafe cast might cause your code to throw an exception, indexing into a nonexistent member might also cause an exception to be thrown.</p>
~~~ javascript
Guid firstFriend = ja[0]["Friends"][0].ReadAs<Guid>(new Guid());
~~~
<p>You might try working around this by checking bounds as you go, but the code is a mess.</p>
~~~ javascript
JsonValue friends; 
if (ja.Count > 0 && (ja[0] as JsonObject).TryGetValue("Friends", out friends))
{
    if (friends.Count > 0)
    { 
        Guid firstFriend = (friends as JsonArray)[0].ReadAs<Guid>(new Guid()); 
    } 
}
~~~
<p>To make this easier, we have provided the notion of &ldquo;safe&rdquo; indexers that will not throw if they encounter a missing or null member through the ValueOrDefault method. The expression will fail at the very last moment when you attempt to do a cast/ReadAs. If you use ReadAs with a fallback, your expression is guaranteed not to throw&hellip; ever.</p>
~~~ javascript
Guid firstFriend = ja.ValueOrDefault(0).ValueOrDefault("Friends").ValueOrDefault(0).ReadAs<Guid>(new Guid());
~~~
<h3>Advanced: dynamic support</h3>
<p>Some folks, especially those of you coming from a JavaScript background, prefer the ability to &ldquo;dot&rdquo; into a type when accessing one of its nested keys, instead of using indexers. To address this, we&rsquo;ve implemented IDynamicMetaObjectProvider in JsonValue, so you can write something like this.</p>
~~~ javascript
Guid firstFriend = ja.AsDynamic()[0].Friends[0];
~~~
<p>Note that the AsDynamic() method is just more fluent shorthand for (dynamic)JsonValue.</p>
<p>Because we think this syntax will be more attractive to folks coming from JavaScript, we&rsquo;ve also ensured that the &ldquo;safe&rdquo; indexers and &ldquo;safe&rdquo; casts are used by default when working with the dynamic version of a JsonValue.</p>
<p>This is it!</p>
<p>The set of types as they ship in Silverlight are documented <a title="System.Json" href="http://msdn.microsoft.com/en-us/library/cc626400(v=VS.95).aspx">here</a>. The new features described above are documented in-depth <a href="http://wcf.codeplex.com/wikipage?title=JsonValue">here</a>. To download the JsonValue source code or binaries, head on over to <a href="http://wcf.codeplex.com">http://wcf.codeplex.com</a>. Please use the site&rsquo;s issue tracker to let us know if you find any issues.</p>