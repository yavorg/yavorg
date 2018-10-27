---
title: 'Episode I: Working with typed and untyped data in the Mobile Services Android
  client'
date: '2013-03-04T22:55:00-08:00'
tags:
- Azure Mobile Services
- Android
tumblr_url: http://hashtagfail.com/post/44606054459/mobile-services-android-typed-untyped
---
<p>To celebrate the Mobile Services Android launch, I&rsquo;ve decided to put together a blog series over the next week to highlight some of the cool new features in our Android SDK. Here is the list of subjects I plan to cover:</p>
<ul><li><a href="{{ site.baseurl }}{% post_url tumblr/2013-03-04-mobile-services-android-typed-untyped %}">Working with typed and untyped data</a></li>
<li><a href="{{ site.baseurl }}{% post_url tumblr/2013-03-04-mobile-services-android-serialization-gson %}">Customizing serialization using the gson library</a></li>
<li><a href="{{ site.baseurl }}{% post_url tumblr/2013-03-28-mobile-services-android-querying %}">Exploring the richness of the query model</a></li>
<li>Implementing authentication correctly</li>
<li>Intercepting HTTP traffic for fun and profit</li>
<li>Unit testing your client code</li>
<li>Advanced push scenarios</li>
</ul><p>Today, let&rsquo;s focus on the data access aspect of the client, which lets us work with server entities in a typed or untyped way. Let&rsquo;s start with a simple class that we want to store in a server-side table. We&rsquo;re naming our class and properties to resemble the serialized JSON, we will cover how to customize the mapping between the properties and the serialized representation in a later post.</p>
~~~ java
public class droid {
    public int id;
    public String name;
    public Float weight;
    public Integer numberOfLegs;
    public Integer numberOfWheels;
}
~~~
<p>Note that a phantom menace lurks here: the class needs to contain a property called <strong>Id, ID,</strong> or <strong>id</strong> which will be populated by the server when it stores the entity.</p>
<p>To store instances of the class inside Mobile Services, we first need to create our Mobile Service in the Azure portal and then create a table called <strong>droid</strong>. Capitalization is important here. We then use the URL and application key provided in the portal to instantiate the <strong>MobileServiceClient</strong>.</p>
~~~ java
MobileServiceClient client = new MobileServiceClient(
    "https://droid.azure-mobile.net/",
    "RrBQdeuaScZCIUsDSNnHPUyKowDDdW89",
    this);
~~~
<p>The third parameter we pass here is the Activity or Service where the client is instantiated. We use that reference to grab the application context, which we use to store the installation ID for this application and also to display our authentication UI dialog.</p>
<p>Now we need to get a reference to the table object so we can insert data into it. Let&rsquo;s look at the different overloads of the <strong>getTable</strong> method. </p>
~~~ java
public class MobileServiceClient {
    public MobileServiceJsonTable getTable(String name);
    public <E> MobileServiceTable<E> getTable(String name, Class<E> clazz);
    public <E> MobileServiceTable<E> getTable(Class<E> clazz);
}
~~~
<p>The first overload is for working with our untyped JSON-based programming model, we will discuss that in a second. The second overload lets you work with a table if its name is different from the type name. The last overload assumes the class name and the table name are the same; this is the simplest one and that&rsquo;s what we&rsquo;ll use here.</p>
~~~ java
MobileServiceTable<droid> table = client.getTable(droid.class);
~~~
<p>We can then instantiate a new instance of the <strong>droid</strong> class and insert it into our server-side table.</p>
~~~ java
droid r2d2 = new droid();
r2d2.name = "R2D2";
r2d2.numberOfLegs = 0;
r2d2.numberOfWheels = 2;
r2d2.weight = 432.21f;

table.insert(r2d2, new TableOperationCallback<droid>() {
    public void onCompleted(droid entity, Exception exception,
            ServiceFilterResponse response){
        if(exception == null){
            Log.i(TAG, "Object inserted with ID " + entity.id);
        }
    }
});
~~~
<p>A few things about the above snippet:</p>
<ul><li>You will notice the use of the <strong>onCompleted</strong> method of the <strong>TableOperationCallback&lt;E&gt; </strong>class. This is how we implement the asynchronous pattern that lets us make network requests without blocking the UI thread. If you want code to run &ldquo;after&rdquo; the results are inserted on the server, place the code inside that method.</li>
<li>Always check the <strong>exception</strong> property to make sure the operation was successful before you attempt to use the <strong>entity</strong> property.</li>
<li>The <strong>entity</strong> property will contain the newly inserted object. Note that the <strong>id</strong> property has been populated on the server.</li>
<li>The <strong>insert</strong> and <strong>update</strong> methods <em>mutate</em> the original object instance. In other words <strong>entity == r2d2</strong> in the above example.</li>
<li>The <strong>response</strong> property gives you direct access to the HTTP response that came back on the server. More on why that&rsquo;s useful in a subsequent post.</li>
</ul><p>Note that by default mobile services have dynamic schematization turned on. That means that the objects inserted determine the columns that end up being created in the database. This is a handy feature to speed up development but you probably want to turn it off in production.</p>
<p>Updating an object is straightforward. </p>
~~~ java
r2d2.weight = 500f;
table.update(r2d2, new TableOperationCallback<droid>() {
    public void onCompleted(droid entity, Exception exception,
			ServiceFilterResponse response) {
		if(exception == null){
			Log.i(TAG, "Updated object with ID " + entity.id);
		}
	}
});
~~~
<p>So is deleting. Notice the shape of the <strong>onCompleted</strong> method on the <strong>TableDeleteCallback</strong> class is slightly different because there is no object to return. </p>
~~~ java
table.delete(r2d2, new TableDeleteCallback() {
    public void onCompleted(Exception exception, 
            ServiceFilterResponse response) {
        if(exception == null){
            Log.i(TAG, "Object deleted");
        }
    }
});

// You can also use the ID directly
// table.delete(r2d2.id, ...);
~~~
<p>We will look at reading in a subsequent post, but for now let&rsquo;s stick to a very simple example. This first snippet reads an object by ID.</p>
~~~ java
table.lookUp(1, new TableOperationCallback<droid>() {
    public void onCompleted(droid entity, Exception exception,
            ServiceFilterResponse response) {
        if(exception == null){
            Log.i(TAG, "Read object with ID " + entity.id);    
        }
    }
});
~~~
<p>This second example will load up all server objects (a &ldquo;select *&rdquo; query). We will take a look at the details of the query model in a subsequent post.</p>
~~~ java
table.execute(new TableQueryCallback<droid>(){
    public void onCompleted(List<droid> result, int count, Exception exception,
            ServiceFilterResponse response) {
        if(exception == null){
            for (droid d : result) {
                Log.i(TAG, "Read object with ID " + d.id);	
            }
        }	
    }
});
~~~
<p>Now let&rsquo;s take a look at the untyped model. Frequently having a client-side type to represent server-side data is very handy for debugging and data-binding purposes. But sometimes creating types is just cumbersome and you want to simply treat your entities as dictionaries on the client. </p>
<p>The key here is to get an instance of the untyped <strong>MobileServiceJsonTable</strong> by calling the right <strong>getTable</strong> overload on <strong>MobileServiceClient</strong>.</p>
~~~ java
MobileServiceJsonTable untypedTable = client.getTable("droid");
~~~
<p>Note that you need to specify the table name here since we are not supplying a type to deserialize into, so the client has no way of knowing what table name to expect.</p>
<p>Our untyped model relies on the <a href="https://code.google.com/p/google-gson/">gson</a> serialization library, so we will use its <strong><a href="http://google-gson.googlecode.com/svn/trunk/gson/docs/javadocs/com/google/gson/JsonObject.html">JsonObject</a></strong> type to build up our object.The following snippet covers the same scenarios as shown above in the typed case. One difference in behavior here is that the original objects are not mutated in the insert/update case.</p>
~~~ java
JsonObject c3po = new JsonObject();
c3po.addProperty("name", "C3PO");
c3po.addProperty("numberOfLegs", 2);
c3po.addProperty("numberOfWheels", 0);
c3po.addProperty("weight", 300f);

untypedTable.insert(c3po, new TableJsonOperationCallback() {
    public void onCompleted(JsonObject jsonObject, Exception exception,
            ServiceFilterResponse response) {
        if(exception == null){
            Log.i(TAG, "Object inserted with ID " + 
        jsonObject.getAsJsonPrimitive("id").getAsInt());
        }
    }
});

// In the JSON case, we don't touch the original object, so the
// id does not get added and we need to add it ourselves
c3po.addProperty("id", 8);
c3po.addProperty("weight", 200f);
untypedTable.update(c3po, new TableJsonOperationCallback() {
    public void onCompleted(JsonObject jsonObject, Exception exception,
            ServiceFilterResponse response) {
        if(exception == null){
            Log.i(TAG, "Updated object with ID " + 
        jsonObject.getAsJsonPrimitive("id").getAsInt());
        }
    }
});

untypedTable.delete(c3po, new TableDeleteCallback() {
    public void onCompleted(Exception exception, ServiceFilterResponse response) {
        if(exception == null){
            Log.i(TAG, "Object deleted");
        }
    }
});

// You can also use the ID directly
// untypedTable.delete(c3po.getAsJsonPrimitive("id").getAsInt(), ...)

untypedTable.lookUp(1, new TableJsonOperationCallback() {
    public void onCompleted(JsonObject jsonObject, Exception exception,
            ServiceFilterResponse response) {	
        if(exception == null){
            Log.i(TAG, "Read object with ID " + 
        jsonObject.getAsJsonPrimitive("id").getAsInt());	
        }
    }
});

untypedTable.execute(new TableJsonQueryCallback() {
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
<p>That is all for today. <a href="{{ site.baseurl }}{% post_url tumblr/2013-03-04-mobile-services-android-serialization-gson %}">In the next episode, we look at how to customize serialization with the power of the gson library.</a></p>
