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
 String firstElement = Iterables.getFirst( names); // return first element from set of names
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


