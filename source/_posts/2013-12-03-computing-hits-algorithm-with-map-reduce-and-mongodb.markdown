---
layout: post
title: "Computing HITS algorithm with Map-Reduce and MongoDB."
date: 2013-12-03 01:27
comments: true
categories: Map-Reduce HITS MongoDB Computation Algorithms PageRank
---

Recently I've tackled the goal to compute HITS algorithm on large graph which represent different types of links among documents. HITS algorithm was first designed and developed by [Jon Kleinberg][kleinberg_home] from Cornell University, you can read more details on [wiki][hits_wiki] or in his original paper ["Authoritative Sources in a Hyperlinked Environment"][hits_paper]. 

Briefly the main idea is, given directed graph $G=(V,E)$, we define $B_{in}(v)=\\{u | \, (u,v) \in E\\}$, all vertices which links to node $v$. Similarly we will define $B_{out}(v)=\\{ u | (v,u) \in E\\}$, all vertices which node $v$ points to. Having these definitions now we can define recursive relationships to compute authority and hub scores, iteration $n+1$ we be equals to:

*  The "authority" score defined as $a_{n+1}(v)=\frac{1}{ \sqrt{ \sum_{u \in B_{in}(v)}h_n(u)^2}}\sum_{u \in B_{in}(v)}h_{n}(u)$
* And hub score is $h_{n+1}(v)=\frac{1}{\sqrt{ \sum_{u \in B_{out}(v)}a_n(u)^2}}\sum_{u \in B_{out}}a_{n}(u)$

(we have to normalize weights in order for process to converge)

Now, once we are talking of huge graph, very first thing which came in my mind is to setup Hadoop cluster, model the graph and run computation on it. The fact that HITS computation easily fits model of map-reduce makes it even more preferable approach.

One of my goals was to learn/view the results almost immediately without any additional tooling or code writing, being able to change the model and different parts of computation process. Well, after few hours I've spend reading Hadoop manuals I understood that task going to be not that easy as expected, since class which represent graph entity has to implement [Writable][writable] interface to enable Hadoop framework to serialize or deserialize object into/from HDFS (HaDoop File System). It became even worse when I understood that there is no easy way to view results of execution, since I've to write my own viewer of the results to view serialized results in HDFS (I'm not saying this is too complicated or hard, just wanted to make it as easy as possible with writing less code). At this point I've realised that MongoDB has its own [map-reduce][map_reduce] engine, so decided to spent few more minutes to understand how it works and whenever I can leverage from it. After several minutes of reading I've understood that this is exactly that I was looking for.

* I can setup cluster of MongoDB, to be able to process large dataset.
* Writing map-reduce functionality is simple as just writing two javascript functions.
* Can use mongo client to connect to database and execute queries in order to see the results.
* In case I need to adjust something in algorithm or add something there is no need to recompile code (like in case of Hadoop), just rewrite map or reduce function and execute it immediately on MongoDB client.

Here I'll try to explain that exactly I did and how it works.
<!-- more -->

###1. Produce initial collection.

First step I've dumped my graph into MongoDB collection, where each entry looks like:

{% codeblock lang:javascript %}
{ 	
label : "nodeName", 
	score : 1.0, 	
	links : [
		{ 
			label : "childNodeName1",
			score : 1.0
		},
			...
	]
}	
{% endcodeblock %}

for tutorial purpose I'll assume collection name is "graph".

Meaning I've model the graph with adjacency list, e.g. each node has it's own record with list of others node it connects to. Scores for everyone initialised to 1 (doesn't really matter it will converge anyway, but much easier for debugging, and not sure if I can say something wise about convergence speed).
Here we can pay attention that using this collection is very intuitive to compute hub score for each node, since we have a list of all children. All we need is to iterate over list of items, sum points for each item, therefore receiving the hub score.

Next step I splitted computation of authority and hubs between two collections, first one to handle nodes and outlinks and second one is to keep nodes and inlinks. Therefore one collection became output for map reduce action on another. So next stage is to initialise them and provide map reduce functions, to be able to run the computation.

###2. The initialization.

First step I'll write map function to initialize hubs collections and then reduce function.

{% codeblock map lang:javascript %}
var map = function( ) {
	var that = this;
	this.links.forEach( function( item) {
		emit( { label: that.label, score: that.score}, item);
	});
}
{% endcodeblock %}

The purpose why this mapper function looks like that is to make collection to look like output of map reduce on MongoDB.

Next we need to write reducer which will has to answer all requirements of MongoDB [map reduce][map_reduce] framework:

{% codeblock reduce lang:javascript %}
var reduce = function( key, values) {
	var result = new Array( );
	values.forEach( function(item) {
		if ( item.links)  { // to handle consequence invocations of reducer
			item.links.forEach( function( item ) {
				result.push(item);
			})
		} else {
			result.push( item);
		}
	});
	return { links : result};
}
{% endcodeblock %}

And finally in order to unify output of the computation I'll provide finalize function:

{% codeblock finalizer lang:javascript %}
var finalizer = function( key, val) {
	if ( val.list)
		return val;
	return { list : [val]};		
}
{% endcodeblock %}

And here we go to produce first collection to start with calculations:

{% codeblock %}
db.graph.mapReduce( map, reduce, { out : "graph_hubs", finalize : finalizer});
{% endcodeblock %}

This will produce startup point and collection called graph_hubs, which we going to use to compute authority scores and create second collection. Output will look like:

{% codeblock lang:javascript %}
{
	"_id" : {
		label : "nodeName",
		score : 1,
	}
	value : {
		list : [
			{ 
				label : "childNodeName",
				score : 1,
			},
			...
		]
	}
}
{% endcodeblock %}

###3. Universal mapper function.

I've the initialization step, since I'd like to have both authority and hubs collections look the same, so now I can provide some generic function which during mapping values will also compute authority or hub score (depends on which collection we execute the code).

{% codeblock lang:javascript %}
var universMap = function( ) {
	var score = 0;
	var weight = 0;
	this.value.links.forEach( function( item) {
		score += item.score;
		weight += Math.pow( item.score, 2);
	});
	var that = this;
	this.value.links.forEach( function( item) {
		emit( item, { label : that._id.label, score : score/Math.sqrt( weight)});
	});
}
{% endcodeblock %}

###4. Computation.

Ok, we setup and ready to go with calculation process, for sake of simplicity I'm going to perform iterations without really checking whenever process indeed converges given some thresthold.

{% codeblock lang:javascript %}
for ( i = 0; i < 20; ++i) {
	db.graph_hubs.mapReduce( universMap, reduce, { out : "graph_auths", finalize : finalizer});
	db.graph_auths.mapReduce( universMap, reduce, { out : "graph_hubs", finalize : finalizer});

}
{% endcodeblock %}

###5. Getting results.

{% codeblock lang:javascript %}
	db.graph_auth.find( {}, { _id : 1}).sort( { "\_id.score": -1}).limit(20);
{% endcodeblock %}

Command above will return us top-20 nodes with highest authority score (similar code will return the top-20 for hubs score).

We can use same [map reduce][map_reduce] framework to join results from both collections to have them in one place. Now the next step I'm working on is to compute page rank score using same technique.

[kleinberg_home]:http://www.cs.cornell.edu/home/kleinber/
[hits_wiki]:http://en.wikipedia.org/wiki/HITS_algorithm
[hits_paper]:http://www.cs.cornell.edu/home/kleinber/auth.pdf
[writable]:http://hadoop.apache.org/docs/current/api/org/apache/hadoop/io/Writable.html
[map_reduce]:http://docs.mongodb.org/manual/core/map-reduce/