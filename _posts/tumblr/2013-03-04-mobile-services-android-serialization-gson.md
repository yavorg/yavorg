---
title: 'Episode II: Customizing serialization using the gson library in the Mobile
  Services Android client'
date: '2013-03-04T22:57:00-08:00'
tags:
- Azure Mobile Services
- Android
redirect_from:
- /post/44606137082/
- /post/44606137082/mobile-services-android-serialization-gson/
---
<p>Welcome to part two of my blog series about the new Mobile Services Android client SDK. Here are the other posts in the series:</p>
<ul><li><a href="{{ site.baseurl }}{% post_url tumblr/2013-03-04-mobile-services-android-typed-untyped %}">Working with typed and untyped data</a></li>
<li><a href="{{ site.baseurl }}{% post_url tumblr/2013-03-04-mobile-services-android-serialization-gson %}">Customizing serialization using the gson library</a></li>
<li><a href="{{ site.baseurl }}{% post_url tumblr/2013-03-28-mobile-services-android-querying %}">Exploring the richness of the query model</a></li>
<li>Implementing authentication correctly</li>
<li>Intercepting HTTP traffic for fun and profit</li>
<li>Unit testing your client code</li>
<li>Advanced push scenarios</li>
</ul><p>Today let&rsquo;s focus on customizing object serialization using the <a href="https://code.google.com/p/google-gson/">gson library</a>, which the Android client library uses under the covers to serialize objects to JSON data. In the previous episode of this series we named our Java object in lowercase, so all the properties would get serialized that way on the wire:</p>
~~~ java
public class droid {
    public Integer id;
    public String name;
    public Float weight;
    public Integer numberOfLegs;
    public Integer numberOfWheels;
}
~~~
<p>For all of you seasoned Java developers out there, this immediately sticks out as bad programming style: we&rsquo;re not capitalizing things correctly and also we&rsquo;re not using getters and setters. This next class uses gson&rsquo;s serialization attributes to produce the exact same wire representation as above.</p>
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
}
~~~
<p>There is another catch here&hellip; our type name is <strong>Droid</strong> (uppercase), but maybe we want our table name to be lowercase. It&rsquo;s just a matter of which <strong>getTable</strong> overload we use. </p>
~~~ java
// Assumes table name is "Droid"
MobileServiceTable table = client.getTable(Droid.class);
// Allows you to use table named "droid" - use this one!
MobileServiceTable table = client.getTable("droid", Droid.class);
~~~
<p>This was a relatively simple serialization trick. To do more we need to whip out the big gun: <strong>GsonBuilder</strong>. To successfully talk to your mobile service, the client needs to do some customizations of its own, so we we need to get a pre-cofigured instance from the the <strong>MobileServiceClient.</strong><strong>createMobileServiceGsonBuilder()</strong> static method. We can then add our own modifications and pass the instance back to the client. The below code does a lot in just a few lines:</p>
<ul><li>It enables pretty-printing for our output JSON</li>
<li>It turns on automatic property lower-casing, so we can remove the <strong>@com.google.gson.annotations.SerializedName</strong> annotations from our type definition above. All that copy pasting was making me feel like an attack of the clones (sorry couldn&rsquo;t come up with a better pun here&hellip;)</li>
</ul>
~~~ java
client.setGsonBuilder(
    MobileServiceClient
    .createMobileServiceGsonBuilder()
    .setFieldNamingStrategy(new FieldNamingStrategy() {
        public String translateName(Field field) {
            String name = field.getName();
            return Character.toLowerCase(name.charAt(1))
                + name.substring(2);
            }
        })
        .setPrettyPrinting());
~~~
<p>Of course we need to add the above code before the calls to any of the methods on the client. </p>
<p>To really put the cherry on the cake, let&rsquo;s tackle a very challenging problem: complex types. So far all of our properties have neatly serialized to JSON primitives. Imagine we want to add the following property on our <strong>Droid</strong> object:</p>
~~~ java
private ArrayList&lt;String&gt; mCatchphrases;
    
public ArrayList&lt;String&gt; getCatchphrases() {
    if(mCatchphrases == null){
        mCatchphrases = new ArrayList&lt;String&gt;();
    }
    return mCatchphrases;
}
private final void setCatchphrases(ArrayList&lt;String&gt; catchphrases){
    mCatchphrases = catchphrases;
}
~~~
<p>Having objects and arrays as properties is a rather advanced feature mostly because we now have to a make a call about how those are represented on the server. Mobile Services doesn&rsquo;t support that out-of-the-box and you would get an error similar to this one:</p>
~~~ json
{
    "code":400,
    "error":"Error: The value of property 'catchphrases' is of type 'object' which is not a supported type."
}
~~~
<p>Let&rsquo;s roll our own custom solution. On serialization we will store the JSON-serialized representation of our complex properties inside JSON strings. On the server, we will simply store the strings inside a string column. On deserialization, we will parse those strings as JSON and deserialize them back into typed objects. A tall order!</p>
<p>Our strategy here is to register a custom serializer for <strong>ArrayList&lt;T&gt;</strong> types. Here is an implementation I put together:</p>
~~~ java
import java.lang.reflect.ParameterizedType;
import java.lang.reflect.Type;
import java.util.Collection;

import com.google.gson.JsonArray;
import com.google.gson.JsonDeserializationContext;
import com.google.gson.JsonDeserializer;
import com.google.gson.JsonElement;
import com.google.gson.JsonParseException;
import com.google.gson.JsonParser;
import com.google.gson.JsonPrimitive;
import com.google.gson.JsonSerializationContext;
import com.google.gson.JsonSerializer;

public class CollectionSerializer&lt;E&gt; implements 
JsonSerializer&lt;Collection&lt;E&gt;&gt;, JsonDeserializer&lt;Collection&lt;E&gt;&gt;{

    public JsonElement serialize(Collection&lt;E&gt; collection, Type type,
            JsonSerializationContext context) {
        JsonArray result = new JsonArray();
        for(E item : collection){
            result.add(context.serialize(item));
        }
        return new JsonPrimitive(result.toString());
    }

	
    @SuppressWarnings("unchecked")
    public Collection&lt;E&gt; deserialize(JsonElement element, Type type,
            JsonDeserializationContext context) throws JsonParseException {
        JsonArray items = (JsonArray) new JsonParser().parse(element.getAsString());
        ParameterizedType deserializationCollection = ((ParameterizedType) type);
        Type collectionItemType = deserializationCollection.getActualTypeArguments()[0];
        Collection&lt;E&gt; list = null;
		
        try {
            list = (Collection&lt;E&gt;)((Class&lt;?&gt;) deserializationCollection.getRawType()).newInstance();
            for(JsonElement e : items){
                list.add((E)context.deserialize(e, collectionItemType));
            }
        } catch (InstantiationException e) {
            throw new JsonParseException(e);
        } catch (IllegalAccessException e) {
            throw new JsonParseException(e);
        }
		
        return list;
    }
}
~~~
<p>This code is only slightly magical, read through it if you want to stretch your understanding of Java generics. This only covers collections, I&rsquo;ve left doing the same for classes as an exercise to the reader.</p>
<p>We now need to register the serializer. <strong>MobileServiceClient</strong> provides a convenience method for you here:</p>
~~~ java
client.registerSerializer(ArrayList.class, new CollectionSerializer&lt;Object&gt;());
~~~
<p>All that does is add it to your <strong>GsonBuilder</strong> instance using the identically named method.</p>
<p>That&rsquo;s it for today, in the <a href="{{ site.baseurl }}{% post_url tumblr/2013-03-28-mobile-services-android-querying %}">next post we&rsquo;ll dive into the query model</a>.</p>
