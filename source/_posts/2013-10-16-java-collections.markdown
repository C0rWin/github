---
layout: post
title: "Java Collections, Guava, reducing boilerplate code"
date: 2013-10-16 15:11
comments: true
categories: java guava collections
---

Guava is a framework developed by Google to bridge the gap in Java SDK Collections API's. It introduces quite a lot of different techniques and development patterns which one was expecting to find in regular Collections API, but apparently wasn't able. I will try to show a few different techniques which helps me to reduce boilerplate code in my projects.
<!-- more -->

###1. Getting element at position i from Set.
How often you've seen something similar to following code?
{% codeblock lang:java %}
Set<String> names = new HashSet<String>( );
names.add( "John D."); 
\\ ...
\\ and then you realize you need to pull first item from your collection
Iterator<String> it = names.iterator( );
String firstName = it.next( ); 
{% endcodeblock %}
 Or it might be even worse:
 
 {% codeblock lang:java %}
 Set<String> names = new HashSet<String>( );
 names.add( "John D.");
 // adding more names skipped.
 // and now we need 5th element from the set.
 Iterator<String> it = names.iterator( );
  
 for( int idx = 0; it.hasNext( ) && idx < 4; idx++,it.next( ) ) {
   	// Iterate over
 
 String fithElement = null;
 if ( it.hasNext( )) { // We stoped because idx reached 4, hence skipped only 4 items.
 	fithElement = it.hext( );
 } else {
 	fithElement = null;
 }
 {% endcodeblock %}
 
 To avoid writing again and again these lines of code Guava, introduces FluentIterable interface and util class Iterables which enables us to re-write lines from above as follows:
 
 {% codeblock lang:java %}
 // Same initialization as previous
 String firstElement = Iterables.get( names, 0);
 // And
 String fithElement = Iterables.get( names, 4);
 {% endcodeblock %}
 
 Or with edge cases of first or last element it might looks like:
 
 {% codeblock lang:java %}
 String firstElement = Iterables.getFirst( names, "defaultValue"); // return first element from set of names
 String lastElement = Iterables.getLast( names); // return the last element
 {% endcodeblock %}
 
###2. Finding element in collection of complex POJO.
For now suppose we have a famous Person pojo, which looks like:

{% codeblock lang:java %}
class Person {

	private String firstName;
	
	private String lastName;
	
	private String email;
	
	public Person( final String firstName, final String lastName, final String email) {
		this.firstName = firstName;
		this.lastName = lastName;
		this.email = email;
	}
	
	public String getFirstName ( ) {
		return firstName;
	}
	
	public String getLastName( ) {
		return lastName;
	}
	
	public String getEmail( ) {
		return email;
	}
	
	// omitting setters since not relevant for current demo/example.
}
{% endcodeblock %}

Now we have a collection of persons say:

{% codeblock lang:java %}
List<Person> people = new ArrayList<Person>( );

people.add( new Person( "John", "Doe", "john@doe.com"));
people.add( new Person( "William", "McLoren", "william@mcloren.com"));
//...
people.add(  new Person( "Bruce", "Jonson", "bruce@jonson.com"));
{% endcodeblock %}

Now we need to lookup from this collection all persons with name "John". With plain Java Collections SDK it might look like:
{% codeblock lang:java %}
Person result = null;
for ( Person each: people) {
	if ( "John".equals( each.getFirstName( ))) {
		result = each; 
		break;
	}
}	
{% endcodeblock %}

Guava Iterables utils class introduce find method which get two parameters an intance of Iterable interface and Predicate, method iterates over all elements and returns back on those which muches the predicate (e.g. predicate return true for them). Now it became:

{% codeblock lang:java %}
class FirstNamePredicate implements Predicate<Person> {

	private String firstName;
	
	public FirstNamePredicate( final String firstName) {
		this.firstName = firstName;
	}
		
	@Overide
	public boolean apply( Person person) {
		return firstName.equals( person.getFirstName( ));
	}
}

// and now we can use it to seach element we would like from given collection.

Iterable<Person> result = Iterables.find( people, new FirstNamePredicate( "John"));

{% endcodeblock %}

Basically one line will do the work, while previously we have had to use for each loop. Now one could possibly argue and say "Wait a moment! Instead of simple for each you have written done a whole class!". And my answer for that yes of course I did! Now I do able to reuse my code, now I then I've lookup logic decoupled I can easily test it. Moreover this is just one simple example, but as you can imagine it could be much more complicated then this specific one.

Now, we have to pay attention on the fact that find method will result in run-time exception in case element is not found which will lead to following code:

{% codeblock lang:java %}
try {
	Iterable<Person> result = Iterables.find( people, new FIrstNamePredicate( "John"));
catch ( Exception e) {
	// And here need to take care of cases where element is not found.
}
{% endcodeblock %}

A much nicer way will be to use recently introduced new method of __Iterables__ which is __tryFind__, as a result it will return instance of __Optional__ and in case there is no such element in collection instead of having to handle exceptional case now you can simply :

{% codeblock lang:java %}
Optional<Person> result = Iterables.tryFind( people, new FIrstNamePredicate( "John"));
// and now you can check whenever value of object is presented
if ( result.isPresent( )) {
	handleResult( result.get( ));
} else {
	handleNoResult( result);
}
{% endcodeblock %}

Or it could be even:

{% codeblock lang:java %}
Optional<Person> result = Iterables.tryFind( people, new FIrstNamePredicate( "John")).or( new Person( "UFO","Alien"));
// and it will always will return you optional result with some value
// hence no need to check for presence you can simple call get method
// of course you will have to handle default value differently.
Person person = result.get( );
handleResult( person);
{% endcodeblock %}

###3. Building map and reverse map.

Sometimes you have to build up some dictionary on your Java code, e.g. you have set of relations between two entity types and could be even between entity and Java primitives. Say you have key of type __A__ and value of type __B__. You probably will finish with writing something like:

{% codeblock lang:java%}
Map<A, B> dictionary = new HashMap<A, B>( );
// now the initialization loop
 while ( it.hasMore( )) {
 	B val = it.next( );
 	A key = computeKey( val);
 	dictionary.put( key, val);
 }
{% endcodeblock %}

But now, suppose that relation between __A__ and __B__ is one to one and you would like to have possibility to make look up for values in both directions, so the code now will look like:

{% codeblock lang:java%}
Map<A, B> dictionary = new HashMap<A, B>( );
Map<B, A> reverseDictionary = new HashMap<B, A>( )
// now the initialization loop
 while ( it.hasMore( )) {
 	B val = it.next( );
 	A key = computeKey( val);
 	dictionary.put( key, val);
 	reverseDictionary( val, key);
 }
 
 // now we able to make lookup in both directions
 // say having key we can get value:
 B val = dictionary.get( key);
 
 // or having value we can get a key
 A key = reverseDictionary( val);
{% endcodeblock %}

Guava introduce new collection type called __BiMap__, which is bidirectional map, designed to answer the question similar to current example, so instead of having code above we can simply write:

{% codeblock lang:java %}
BiMap<A, B> dictionary = new HashBiMap<A, B>( );
// now the initialization loop
 while ( it.hasMore( )) {
 	B val = it.next( );
 	A key = computeKey( val);
 	dictionary.put( key, val);
 }
 
 // now in order to get key for a value we have to do:
 A key = dictionary.inverse( ).get( val);
{% endcodeblock %}

There is much more things and useful functionality provided by Guava, I'll continue to describe uses cases and method which will allow to remove boiler plate and redundant code.
