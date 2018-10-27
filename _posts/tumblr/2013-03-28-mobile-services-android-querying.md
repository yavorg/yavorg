---
title: 'Episode III: Exploring the richness of the Mobile Services Android client
  query model'
date: '2013-03-28T00:57:00-07:00'
tags:
- Azure Mobile Services
- Android
tumblr_url: http://hashtagfail.com/post/46493261719/mobile-services-android-querying
---
<p>In today&rsquo;s post we continue our Android blog series for Mobile Services. Here are the other posts in the series.</p>
<ul><li><a href="{{ site.baseurl }}{% post_url tumblr/2013-03-04-mobile-services-android-typed-untyped %}">Working with typed and untyped data</a></li>
<li><a href="{{ site.baseurl }}{% post_url tumblr/2013-03-04-mobile-services-android-serialization-gson %}">Customizing serialization using the gson library</a></li>
<li><a href="{{ site.baseurl }}{% post_url tumblr/2013-03-28-mobile-services-android-querying %}">Exploring the richness of the query model</a></li>
<li>Implementing authentication correctly</li>
<li>Intercepting HTTP traffic for fun and profit</li>
<li>Unit testing your client code</li>
<li>Advanced push scenarios</li>
</ul><p>Today we will talk about the query model in the Android client. In our C# client we were lucky to be able to lean on LINQ, but after some research we found that there is no out-of-the-box equivalent in the Android platform. I&rsquo;m glad to stand corrected here: please let me know in the comments if we missed something.</p>
<p>Given that, we decided to implement our own fluent query API that supports the same richness that you get with LINQ, so it deserves a thorough writeup. We&rsquo;ll assume we&rsquo;re working with our same reference type we used in our last tutorial, but we&rsquo;ve added a field or two to make things more interesting to query against:</p>
~~~ java
public class Droid {
    
    @com.google.gson.annotations.SerializedName("id")
    private Integer mId;
    @com.google.gson.annotations.SerializedName("name")    
    private String mName;
    @com.google.gson.annotations.SerializedName("weight")    
    private Float mWeight;
    @com.google.gson.annotations.SerializedName("numberOfLegs")	
    private Integer mNumberOfLegs;
    @com.google.gson.annotations.SerializedName("numberOfWheels")	
    private Integer mNumberOfWheels;
    @com.google.gson.annotations.SerializedName("manufactureDate")	
    private Date mManufactureDate;
    @com.google.gson.annotations.SerializedName("canTalk")	
    private Boolean mCanTalk;
	
    public Integer getId() { return mId; }
    public final void setId(Integer id) { mId = id; }
	
    public String getName() { return mName; }
    public final void setName(String name) { mName = name; }
	
    public Float getWeight() { return mWeight; }
    public final void setWeight(Float weight) { mWeight = weight; }
	
    public Integer getNumberOfLegs () { return mNumberOfLegs; }
    public final void setNumberOfLegs(Integer numberOfLegs) { 
        mNumberOfLegs = numberOfLegs; 
        }
	
    public Integer getNumberOfWheels() { return mNumberOfWheels; }
    public final void setNumberOfWheels(Integer numberOfWheels) { 
        mNumberOfWheels = numberOfWheels; 
        }

    public Date getManufactureDate() { return mManufactureDate; }
    public final void setManufactureDate(Date date) {
        mManufactureDate = date;
    }
	
    public Boolean getCanTalk() { return mCanTalk; }
    public final void setCanTalk(Boolean canTalk) { 
        mCanTalk = canTalk; 
    }

}
~~~
<p>Let&rsquo;s assume we have a few of these instances in a mobile services table. We will also grab a reference to the typed and JSON-based table, so we can show the differences in the query programming model.</p>
~~~ java
MobileServiceTable&lt;Droid&gt; table = client.getTable(Droid.class);
MobileServiceJsonTable untypedTable = client.getTable("droid");
~~~
<p>If these don&rsquo;t seem familiar, please review the <a href="/posts/2013/03/04/mobile-services-android-typed-untyped">first post in my series</a>. </p>
<h3>Sorting</h3>
<p>Let&rsquo;s start with sorting, here we show a simple ascending/descending sort using both the typed and JSON-based model.</p>
~~~ java
table.orderBy("name", QueryOrder.Ascending).execute(new TableQueryCallback&lt;Droid&gt;() {
    public void onCompleted(List&lt;Droid&gt; result, int count, Exception exception,
            ServiceFilterResponse response) {
        if(exception == null){
            for (Droid d : result) {
                Log.i(TAG, "Read object with ID " + d.getId());    
            }
        }	
    }
});

untypedTable.orderBy("name", QueryOrder.Descending).execute(new TableJsonQueryCallback() {
    public void onCompleted(JsonElement result, int count, Exception exception,
            ServiceFilterResponse response) {
        if(exception == null){
            JsonArray results = result.getAsJsonArray();
            for(JsonElement item : results){
                Log.i(TAG, "Read object with ID " + 
            item.getAsJsonObject().getAsJsonPrimitive("id").getAsInt());
            }
        }
    }
});
~~~
<p>Note that the type returned by <strong>orderBy</strong> is <strong>MobileServiceQuery&lt;T&gt;</strong>. That is the entry point into this fluent query model. By adding do that query, you could compose sorting, paging, and filtering in any arbitrary order, as we will see further down. To terminate the query, just use the <strong>execute</strong> method. You will notice that the query stays constant regardless of whether you used the typed or JSON model, the only thing that changes is whether the execute method gets back a TableQueryCallback (typed) or TableJsonQueryCallback (JSON). So all the examples that follow will work in either case.</p>
<h3>Paging</h3>
<p>Paging is also simple:</p>
~~~ java
table.top(10).skip(10).execute(/* callback */ );
~~~
<p>You can easily compose this with sorting (or filtering as we will see below):</p>
~~~ java
table.top(10).skip(10).orderBy("name", QueryOrder.Ascending)
    .execute(/* callback */);
~~~
<h3>Filtering</h3>
<p>This is by far most interesting part of the model. To build an filter expression you have to somehow get a <strong>MobileServiceQuery&lt;T&gt;</strong> instance from the table you&rsquo;re about to query, so you can start tacking predicates onto it. We already saw that the <strong>top</strong>, <strong>skip</strong>, and <strong>orderBy</strong> methods already provide that entry point. If you don&rsquo;t want to bother with sorting and paging, we provide another &ldquo;vanilla&rdquo; entry point into the model, simply by calling <strong>where</strong>. You can use either of these to start building a filter. In the examples below I&rsquo;ll simply show how to build the query part, it&rsquo;s your job to remember to tack on <strong>execute</strong> at the end.</p>
<h4>Comparators</h4>
<p>Consider the following examples for some basic comparison operators to fetch droid objects from our table given their number of legs.</p>
~~~ java
// droid.numberOfLegs &gt; 3
table.where().field("numberOfLegs").gt(3);

// droid.numberOfLegs &gt;= 3
table.where().field("numberOfLegs").ge(3);

// droid.numberOfLegs &lt; 3
table.where().field("numberOfLegs").lt(3);

// droid.numberOfLegs &lt;= 3
table.where().field("numberOfLegs").le(3);

// droid.numberOfLegs === 3
table.where().field("numberOfLegs").eq(3);

// droid.numberOfLegs !== 3
table.where().field("numberOfLegs").ne(3);
~~~
<p>You&rsquo;ll note the use of the <strong>field</strong> method to pick the property on the server that we want to test against. One interesting case that comes up here is how to test if a value is set on the server (is not undefined). This can be a little awkward:</p>
~~~ java
// droid.numberOfLegs === undefined
table.where().field("numberOfLegs").ne((Number)null);
~~~
<h4>Values</h4>
<p>You might get curious about what are the kinds of values that you can use on the right-hand side of your operators. You will notice that all of the above operators take a <strong>Number, boolean, String, </strong>or <strong>Date</strong>. Each of these has its own specific characteristics, let&rsquo;s look at <strong>Date</strong> first.</p>
~~~ java
// droid.manufactureDate == new Date(Date.UTC(2210, 6, 30));
table.where().field("manufactureDate").eq(getUTCDate(2210, 6, 30, 0, 0, 0);

private static Date getUTCDate(int year, int month, int day, int hour, int minute, int second) {
    GregorianCalendar calendar = new GregorianCalendar(TimeZone.getTimeZone("utc"));
    int dateMonth = month - 1;
    calendar.set(year, dateMonth, day, hour, minute, second);
    calendar.set(Calendar.MILLISECOND, 0);

    return calendar.getTime();
}
~~~
<p>Note that it is no so straightforward to work with UTC dates in Android, so the above example provides a convenience method to do that.</p>
<p>This is great if you want to compare the whole date, but what if you want to compare individual parts of the date (year, month, day). We have a solution for that as well:</p>
~~~ java
// droid.manufactureDate.getUTCFullYear() === 2210
table.where().year(field("manufactureDate")).eq(2210); 

// droid.manufactureDate.getUTCMonth() === 6
table.where().month(field("manufactureDate")).eq(6);

// droid.manufactureDate.getUTCDay() === 30
table.where().day(field("manufactureDate")).eq(30);

// droid.manufactureDate.getUTCHours() === 10
table.where().hour(field("manufactureDate")).eq(10);

// droid.manufactureDate.getUTCMinutes() === 15
table.where().minute(field("manufactureDate")).eq(15);

// droid.manufactureDate.getUTCSeconds() === 11
table.where().second(field("manufactureDate")).eq(11);
~~~
<p>One thing to remember here is that these are server queries. What&rsquo;s actually happening under the covers is that the left-hand and right-hand side of the query are being serialized on the wire and executed on the server. Just in case you were wondering why you can&rsquo;t simply express this as a function: we need to have a way to easily parse the expression and send it to the server.</p>
<p>Next let&rsquo;s look at <strong>Number</strong>. There are a few interesting things we may want to do to the left-hand side on the server: addition, subtraction, multiplication, division, modulus, as well as things like floor, celiling, and rounding for floats.</p>
~~~ java
// droid.numberOfLegs + 2 === 18
table.where().field("numberOfLegs").add(2).eq(18);

// droid.numberOfLegs - 2 === 16
table.where().field("numberOfLegs").sub(2).eq(16);

// droid.numberOfLegs * 2 === 16
table.where().field("numberOfLegs").mul(2).eq(16);

// droid.numberOfLegs / 2 === 8
table.where().field("numberOfLegs").div(2).eq(8);

// droid.weight % 2 === 1
table.where().field("weight").mod(2).eq(1);

// Math.floor(droid.weight) &gt; 10
table.where().floor(field("weight")).gt(10);

// Math.ceil(droid.weight) &gt; 10
table.where().ceiling(field("weight")).gt(10);

// Math.round(droid.weight) &gt; 5
table.where().round(field("weight")).gt(5);
~~~
<p>Lastly let&rsquo;s look at the things we can do with <strong>String</strong> on the server:</p>
~~~ java
// droid.name.indexOf("c3") === 0
table.where().startsWith("name", "c3");

// showing how to do this in JavaScript is counterproductive in this case
table.where().endsWith("name", "po");

// droid.name.search("c3") !== -1
table.where().subStringOf("c3", "name");

// droid.name.concat(" rocks") === "c3po rocks"
table.where().concat(field("name"), val(" rocks")).eq("c3po rocks");

// droid.name.indexOf("3p") === 1
table.where().indexOf("name", "3p").eq(1);

// droid.name.substr(2) === "po"
table.where().subString("name", 2).eq("po");

// droid.name.substr(2, 3) === "po"
table.where().subString("name", 2, 1).eq("po");

// droid.name.replace("c3po", "r2d2) === "r2d2"
table.where().replace("name", "c3po", "r2d2").eq("r2d2");

// droid.name.toLowerCase() === "c3po"
table.where().toLower("name").eq("c3po");

// droid.name.toUpperCase() === "C3PO"
table.where().toUpper("name").eq("C3PO");

// droid.name.trim() === "c3po"
table.where().trim("name").eq("c3po");

// droid.name.length === 4
table.where().length("name").eq(4);
~~~
<p>As you play around with these APIs you&rsquo;ll notice that all of them take not just <strong>String</strong> types, but you can also specify <strong>field(&ldquo;name&rdquo;)</strong> and <strong>val(5)</strong>. This is to enable you to interchange client and server values in your query for maximum flexibility. For example, if you want to check if one server column is a substring of another, simply use <strong>subStringOf(field(&ldquo;column1&rdquo;), field(&ldquo;column2&rdquo;))</strong>. Supercool!</p>
<h4>Logical operators</h4>
<p>We&rsquo;ve shown how to do comparisons, now let&rsquo;s look at how we can string those comparisons together into logical expressions. Here are some simple examples of and, or, and not.</p>
~~~ java
// droid.numberOfLegs &lt; 3 &amp;&amp; droid.numberOfWheels &gt; 3
table.where().field("numberOfLegs").lt(3).and().field("numberOfWheels").gt(3);

// droid.numberOfLegs &lt; 3 || droid.numberOfWheels &gt; 3
table.where().field("numberOfLegs").lt(3).or().field("numberOfWheels").gt(3);

// droid.numberOfLegs !== 3
table.where().not().field("numberOfLegs").eq(3);
~~~
<p>These are great simple cases, but what if we want to support grouping (nesting) of the logical operators? For example if we want to select any droids that are named &ldquo;c3po&rdquo; or are similarly humanoid and have two legs and no wheels. Here we go:</p>
~~~ java
// droid.name === "c3po" || 
// (droid.numberOfLegs === 2 &amp;&amp; droid.numberOfWheels === 0)
table.where().field("name").eq("c3po")
    .or(field("numberOfLegs").eq(2).and().field("numberOfWheels").eq(0));
~~~
<p>What you see in this example is that the logical operators can also take parameters as a way to do grouping. Neat!</p>
<h3>Projection</h3>
<p>Last but not least, let&rsquo;s talk about projection. There are two interesting scenarios here:</p>
<ol><li>Fetch only selected properties of a large object from the server</li>
<li>Deserialize your server object into a different type (for example a derived type)</li>
</ol><p>We can implement the first case simply by using the <strong>select</strong> method as part of our query:</p>
~~~ java
// Without a query
table.select("name", "weight").execute(/* callback */);

// As part of a query
table.where().startsWith("name", "c").select("name", "numberOfLegs").execute(/* callback */);
~~~
<p>Here is how to do the second case. Imagine we have a subclass of <strong>Droid</strong> called <strong>ProtocolDroid</strong> which contains only a subset of the properties we want to fetch:</p>
~~~ java
MobileServiceTable subclassedTable = client.getTable("droid", ProtocolDroid.class);
subclassedTable.select("name", "numberOfLegs").execute(/* callback */);
~~~
<p>This writes up today&rsquo;s monster post! In the next post&hellip; auth.</p>