---
layout: post
title: The PourOver Book
---

#### Prologue - Dependencies
[Underscore.js](http://underscorejs.org/)

#### Prologue 2 - Special Thanks
PourOver is very much indebted to [Backbone](http://backbone.js.org). In fact, it copies the Extend and Events modules from Backbone.
Furthermore, it is written in Backbone-ese and, indeed uses the Backbone.extend method to create its prototypes and Backbone.Events for its events. However, items in a PourOver collection are simple hashes/objects, not Backbone models. 

#### Preface 1 - What is PourOver
PourOver is a library for fast filtering and sorting of large collections -- think 100,000s of items -- in the browser. It's similar to [Crossfilter](http://square.github.com/crossfilter/) but supports aribitrary, chainable boolean combination of filters. It provides a simple interface for specifying queries such as: "Give me all the red or blue or green dresses that were not strapless and appeared in the 90s." You also get some cool features like collections that buffer their information in periodically, views that page and cache, fast sorting, and much, much more. I encourage you to skip down to "Chp 1. - The Philosophy of PourOver" to get more insight into the uses and rationale for PourOver.

#### Preface 2 - The Best Way to Learn PourOver
The best way to learn PourOver is by example:

- [Basic PourOver example]({{ site.baseurl }}/examples/examples_build/basic_pourover_ing.html)
- [Advanced views examples]({{ site.baseurl }}/examples/examples_build/advanced_views.html)
- [Advanced filters]({{ site.baseurl }}/examples/examples_build/advanced_filters.html)
- [Buffering]({{ site.baseurl }}/examples/examples_build/buffering.html)
- UI (Coming soon)
- Events (Coming soon)

#### Chp. 1 - The Philosophy of PourOver

So, what's it all for? Why do I need PourOver?

Let me describe for you how faceted search generally works,  alternatively, how apps work that allow you to filter, sort, select, etc. from some interface: The user performs some action, indicates some slice of the data that they want. Your app takes this request, sends it to a server, the server (that is processing these request for all your other users too) slices the data and returns the result. The app subs this data in and re-renders its view.

There are some pathological issues here. You are making the shared resource (the server) do all the work while the clients, more or less, sit idly by and just procress and create requests. You add a network roundtrip to every interaction with the page. That's [on average](http://www.igvita.com/2012/07/19/latency-the-new-web-performance-bottleneck/) adding 28-44 ms of latency to every action. 

What does this mean for the user? It means the difference between an app feeling like it's responsive to input and an app feeling like it's churning through a search, waiting to fetch data. 

What does this mean for the developer? It means that the faster a user changes filters, the harder they use your app the higher the load on your server. It means that paging through large, filtered sets of data is over-complicated. In fact, this was part of the reason PourOver was created. In 2012, for the Olympics we wrote an app, Imago, for moderating half a million wire photos of athletes. Despite frequent optimization -- and hand-coding a custom index selector -- the bottleneck always came when trying to page through a filtered subset of a changing database. Offset is a performance killer (at least with my meagre MySQL chops). 

See, with the filter -> query -> response pattern you are starting from scratch each time. Sure, your database has indices to speed things up. But, often, you've already done more work. Imagine you query for all the friends that are female. Your database looks around, returns 1000 friends. The app renders a list. Then, the user selects "under 25 years old". Now, the database looks again for all female friends and intersects that with all friends under 25. Return. Render. But you already know your female friends! The least amount of work would be just return the under 25 friends and intersect client-side. Paging adds an even greater level of complexity. 

PourOver is meant to make all this simpler, at least for our target use cases: large sets that are filterable and sortable by several attributes over small, finite domains. As described above, PourOver creates a cache for every possibility and then uses simple set algebra to make composite queries on the client. It's basically a client-side index. This means that when I query for all friends that are girls and under 25, the work that has to be done is just an intersection between the girl index and the under 25 index (called MatchSets in PourOver).

Furthermore, PourOver makes development simpler. You simply pull down all the data and then use boolean logic to compose queries that are automatically cached. You don't have to worry about optimizing database queries. You don't have to cripple user actions because they could possibly hose the database.

The challenge becomes: how do you get the large data set from server to client. At least the data that affects the filters (all the other information -- full text, descriptions, etc. -- can be buffered in). Arguably, this is a simpler, more one-dimensional problem. Data sets over small finite domains can be packed into tight, binary represenations: if there are 8 possibilities for each item, say, you can represent the value with 3 bits. Mixing in bit maps and, then, file compression, we have seen sets of 100k items pack into <100k.


######## Basic Concepts
**Collections**
A PourOver `Collection` is an array of items, indexed by collection ids (cids). Collections are accompanied by a set of filters and sorts that can be applied to retrieve sorted subsets of the collection. Collections know nothing about the state of these filters and sorts. The collection is merely responsible for adding, updating and removing items, filters, and sorts and for returning subsets of its items. The general pattern is that a filter will return a set of cids (from a cache) and then the collection will take those cids and return the corresponding items.

**Filters**
A PourOver `Filter` belongs to a collection and is associated with some way in which that collection may be filtered: an attribute, a function, etc. The filter caches which items correspond to its possibilities. Every filter has a hash of possibilities, each possibility has a list of `cids`. Filters can either be used statefully or non-statefully (we will show examples of this below in the filter section). 

**MatchSet**
The result of a query on a filter is a `MatchSet`. A MatchSet is like an array of cids but it remembers how it was created. Since a `MatchSet` may be the result of aribtrary unions, intersections, and differences, it is necessary to remember how it was composed so that individual operations may be undone or updated. For example: say you queried for red OR blue OR green dresses. The state of the color filter is now set to red OR blue OR green. But then, a new blue dress is added to the collection. The `MatchSet`'s memory -- called its `stack` -- knows automatically how to update itself based on the new dress addition. Without the user having to do anything, the color filter's current matchset state now contains the new dress.

**View**
A `View` stores a composite state of a collection, often what is meant to be rendered. There can be many views per collection. Views can be paged. Moreover, a view has a selection function which tells the view how to compose its various filters to produce the current set. For example, say you picked red OR blue OR green dresses that are strapless and from 1996. The view would have a selection function that intersects the color filter's `MatchSet` with that of the style filter and the year filter. However, the view could, alternatively, be told to union its filters together. Views also cache the items currently filtered/sorted by its state for fast re-renders.

**Sort** 
A `Sort` is pretty much what is sounds like. However, it doesn't cache the cids of the collection in order. Rather, it caches a *function* that can arrange any subset of the collection's cids in the sort order. Sorts, like filters, generally belong to collections. However, sorts can belong to views as well. The reason for this has to do with optimization but we will discuss this later.

#### Chp. 2 - Basic Usage

**Creating a collection**
Generally, the first thing you will want to do with PourOver is create a collection. 
	
	var data = [{name: "bob", eyes: 2, sex: "m"},{name: "margo", eyes: 1, sex: "f"},{name: "chuckles", eyes: 2, sex: "f"}],collection;
	
	collection = new PourOver.Collection(data);

Congratulations, you have created a collection. Now, say a message arrives: "Don't forget Amy. She only has one eye!" To add items to a PourOver collection:

	collection.addItems({name: "amy", eyes: 1, sex:"f"});

Excellent. Were there any filters or sort on this collection they would automatically be regenerated to accomodate this new item. However, there are no filters or sorts. 

`addItems` can take a single item or an array of items.

We can also update items:

	collection.updateItem(1,"name","margot");

Here, 1 is the cid. Currently, all collection operations are keyed off the item's cid. Like addItems, updateItem will regenerate all filters and rebuild all sorts.

Note: It is always more efficient to create a collection and then add all your items rather than to create a collection of empty items and then call `updateItem` many times in succession.

Removing of items is also supported. However, it is not a fast operation and should be avoided if possible. It is generally preferable to have a "visible" filter and simply hide items thusly rather than removing them.

	collection.removeItems(1);
	
`removeItem` either takes a single cid or an array of cids. It is much faster if these cids are sorted. If they are already sorted call `removeItems` with a second parameter "true"

	collection.removeItems([1,2],true);
	
**Adding filters**
Now, let's do something interesting with our collection. First, we need to make a filter. PourOver ships with a handful of filter defaults that should comprise most of the filtering you will be doing. More defaults will be added later. (We will cover how to create your own filter from scratch in the Advanced Usage section).

Say we want to make a filter for number of eyes, from our example above. The most common PourOver filter is the `exactFilter` which takes an array of possibilities, of which any object in the collection can satisfy only one. The filter defaults ship with constructor methods to simplify things even further

	var eye_filter = PourOver.makeExactFilter("eyes",[0,1,2]);

This says: "create a new exact filter for the attribute 'eyes'. An item may have 0, 1, or 2 as possible values for its `eyes` attribute." An item may have a value that is not included in the possibilities and this will not cause an error. It will simply not be findable on that attribute as it will not be added to any filter's `MatchSet`.

**NOTE: You must name your exact filter (the first argument) the same thing as the attribute that it is indexing, in this case "eyes"**

Now, we have to add this filter to our collection

	collection.addFilters(eye_filter);
	
Again, `addFilters` takes either a single filter or an array of filters. 

**NOTE: `addFilters` will clone the filter before adding it. Thus, you must refer to the new filter located at `collection.filters.eye` when using the filter in subsequent references.**

Now, let's use this filter to do some queries. Earlier, I mentioned that there are stateful and non-stateful ways to query filters. The former is useful if you have a UI that is tied to a filter, say a color selector. Then it's nice to be able to save the current query on the filter and any subsequent renders will get the right information. However, you might want to query a filter non-statefully  to, say, get the number of items satisfying a query. 

Statefully:

	collection.filters.eyes.query(1);
	var one_eye_cids = collection.filters.eyes.current_query.cids,
		one_eyed_people = collection.get(one_eye_cids);

Now, the eyes filter will remember the current_query and will return the same set until it is reset or cleared. Generally, you will not retrieve the `current_query` of a filter yourself, but a view's `selectionFn` will utilize the result. More on this in the "Creating Views" section below.

The same thing non-statefully:

	var one_eye_cids = collection.getFilteredItems("eyes",1).cids,
		one_eyed_people = collection.get(one_eye_cids);
	
	
Here, `getFilteredItems` will return a `MatchSet` containing the cids for the "margo" and "amy" objects from above. `get` then matches the cids back up to the original objects and returns them.


**Combining Filters**
The real power, though, becomes evident when we start combining filters. Let's first add a filter for the "sex" attribute

	var sex_filter = PourOver.makeExactFilter("sex",["m","f"]);
	collection.addFilters(sex_filter);
	
Now, say we wanted to query for all the two-eyed women in our collection. We simply get the `MatchSet` for each query & `and` them together.

	var two_eyeds = collection.getFilteredItems("eyes",2),
		women = collection.getFilteredItems("sex","f"),
		output_cids = two_eyeds.and(women).cids,
		two_eyed_women = collection.get(output_cids);
		
Remember, calls to `getFilteredItems` as well as its stateful cousin `current_query` return `MatchSet` responses. `MatchSet`s can be chained with `and` `or` or `not`, producing composite `MatchSet`s at each step. Finally, we pull the `cids` out of the composite match set and `get` them from our collection.

You can also use combination to construct compound filters from a single filter. If we wanted to get all one OR two-eyed people

	var two_eyeds = collection.getFilteredItems("eyes",2),
		one_eyeds = collection.getFilteredItems("eyes",1),	
		total_cids = two_eyeds.or(one_eyeds).cids,
		total_people = collection.get(total_cids);
		
The stateful version of this code is similar:
	
	collection.filters.eyes.query(1);
	collection.filters.eyes.unionQuery(2);
	var total_cids = collection.filters.eyes.current_query.cids,
		total_people = collection.get(total_cids);
		
PourOver filters support `unionQuery`, `intersectQuery`, and `subtractQuery` to statefully build up queries.

**Creating views**
Collections are interesting, but they aren't terribly well-suited for rendering. Collections can't store a meta-state of many filters combined, they don't have paging, they can't store a current sort. 

Fortunately, we have views. Views do all this. The main purpose of a View is to keep track of all the stateful things that have happened. This way, we can call `view.getCurrentItems()` and get an array of items sorted, filtered, and paged.

To create a view, we simply:

	view = new PourOver.View("default_view",collection)
	
This says create a new view named "default_view" for the collection `collection`.

However, just initializing a view like this wouldn't be very useful. Perhaps if we were just using the view to page or something, it would be. But, generally, we first extend the PourOver.View and then instantiate a new one of these extended views.

	MyView = PourOver.View.extend({
		template: JST.viewTemplate,
		render: function(){
			var items = this.getCurrentItems(),
				output = template({items:items});
			$("#container").html(output);
		},
		selectionFn: function(){
			var collection = this.collection,
				eyes_dimension = collection.getCurrrentFilteredItems("eyes"),
				sex_dimension = collection.getCurrentFilteredItems("sex");
			return eyes_dimension.and(sex_dimension);
		}
	});
	view = new MyView("default_view",collection)
	
Now, let's look at this bit of code in depth. If you have used Backbone before, you will recognize this style of creating new Views. However, these are not Backbone views. They are simply written in the same style.

First, we call `extend` on PourOver.View which simply just creates a new constructor object based on PourOver.View with some overridden and added methods in the prototype.

We specify a template to be used in rendering and then a render function. Generally, your view render function should call `this.getCurrentItems()` and use these items are the set to be rendered. As in most render functions, this one ends by replacing some HTML on our page `$("#container").html(output);`

The third method, `selectionFn` overrides the default `selectionFn` for the view. The `selectionFn` is used to cache the current filtered items on a view.   By default, all filters are intersected. This means that the `selectionFn` we have passed in achieves the same thing as the default. However, often you will need a more complicated combination of filters: unions, intersections and nots. 

**NOTE: `selectionFn` must return a `MatchSet` not an array of items. Remember, MatchSets are returned from `getCurrentFilteredItems` calls as well non-stateful queries over filters as well as from `and`, `or`, and `not` functions, chained off other MatchSets.**

The `selectionFn` is used as follows: All views have a method `setNaturalSelection`. This method calls `selectionFn` and then saves the result as `this.match_set`. Then, whenever `getCurrentItems` is called, it does not have to revaluate the filters. It simply pulls the `this.match_set` off the view and then applies any sorts or paging. This allows for fast switching of sorts and pages without filters having to be reassessed. `setNaturalSelection` is automatically called on a View whenever an item changes or a filter's query changes. However, you can call `setNaturalSelection` yourself if you need to force a refresh or a recache.

Other common attributes/methods to extend the default PourOver view with are:

- page_size: This sets a page size of the view. By default it is set to Infinite and returns the entire filtered and sorted set.
- current_sort: By default there is no sort on the view. However, if you want your view to start out sorted, here is where you specify that.
- current_page: The page to start the view on. By default, 0.
- initialze: The function to call after the view has been created. This function will be passed all the arguments passed into `new View(arguments)` and the `this` context will be the new view. This is a noop by default.

#Chp 2.5 - The Event Cycle
	
There are several basic types of events fired by PourOver objects:
	
- "change": Fired whenever items are added or removed in the collection.
- "change*:[attr]" : Fired whenever an item's [attr] is modified.
- "incrememental_change": Fired whenever an item is modified.
- "queryChange": Fired on a filter whenever a stateful query is made on a filter. Bubbles up to collectons.
- "selectionChange": Fired on a view whenever the view's match set is updated. This generally happens automatically when one of the filters are queried or the collection is changed.
- "sortChange": Fired on a view whenever the view's sort changes or is removed.
- "pageChange": Fired whenever a view's page changes.
- "update": Fired on a view whenever any change,queryChange,selectionChange, pageChange or sortChange happens on the view or its collection. This is likely the event you want to listen for to trigger a re-render
	
Whenever an item changes a specific attribute, "change:[attribute]" is fired (as well as a generic "change" event.) This event is what filters and sorts hook onto for optimized, incremental updating. This is done through "associated_attrs".

	var filter = new PourOver.Filter("eyes",[0,1,2],{associated_attrs: ["eyes"]})
	
This means that a new filter will be created, the will rebuild itself whenever an item in the collection changes an "eyes" attribute. Indeed when you make an exact filter, the name of the exact filter/ the attribute it is tied to is ALWAYS passed as an associated_attr. 

	var sort = new PourOver.Sort("eye-sort",{associated_attrs: ["eyes"]})

This will create a sort that will rebuild itself whenever the eyes attribute is updated. You must specify this when creating a sort on collection with attributes that may change while in a sorted state.

####Chp 3 - PourOver UI

PourOver comes bundled with a convenience interface for creating UIs for controlling the states of filters, think of a color picker or a list of possible options. This is called PourOver.UI. The main purpose of PourOver.UI is to translate between a filter's MatchSet and an easier to work with representation of the filter's current state.

For example, say you were working with an exact filter, the most common filter in PourOver. After a series of unionQuery calls you now have a pretty complicated stack object -- the stored state of your filter -- and it would be complicated or at least inconvenient to recurse through the stack to extract which filter possibilities had been selected. This would be useful if you wanted to render an element on the page that show the possibilities as either selected or deselected as for a UI control.

PourOver.UI comes with such an "element", `PourOver.UI.SimpleSelectElement`. 

All PourOver.UI Elements have a `getMatchSet` and `getFilterState` functions. The former should return whichever match set is associated with the UI element and the latter should translate that match set into some form that can be rendered, perhaps and array of selected values. By default, both of these functions are errored out and need to overridden in specific UI.Element instances. If you are using one of the preset elements such as PourOver.UI.SimpleSelectElement, you do not need to override `getMatchSet` and `getFilterState` as they are already defined for you.

Currently, the base UI.Element ships with a `getSimpleSelectState` and a `getSimpleRangeState`. These are very limited at the moment. `getSimpleSelectState` can only parse stacks of unioned exact filter queries and `getSimpleRangeState` can only parse a single range query. The idea is that as pattern of use in PourOver emerge we will add functionality to the base `Element` class.

To make an element representing a simple multi-selection or a ranged-selection, you can use the helper subclasses included.

	var simple_select = new PourOver.UI.SimpleSelectElement({filter: collection.filters.filter_to_render})
	var range_select = PourOver.UI.SimpleDVRangeElement({filter: collection.filters.filter_to_render})
	
**NOTE: The filter that you pass these constructors must be the actual filter that is added to your Collection. When you `addFilter` it clones the filter. This means you must use `collection.filters.foo` when referring to your filter as above.**
	
Even if you're not using one of these built-in classes, it's useful to organize your filter display around PourOver.UI.Element's abstract interface for easy debugging.

#### Chp 4 - Advanced Usage

####Chp 5 - FAQs

######## 1. What are the different filter types included with PourOver

- exactFilter: This is probably the filter you will use the most. It is for cases in which each item in your collection can have exactly one of a possible set of values. For example, a player can be on exactly one of several teams or a book can have exactly one of several genres. This is also the fastest filter to create and update. **Exact filters must always be named the same string as the attribute that they track**. Also, exact filters will automatically be given the attribute that the track as an associated_attr. This means that whenever, say, any item in the collection changes "team", the filter will be recached.
	
	To create a team filter, for example:

		var team = PourOver.makeExactFilter("team",["bulls","pistons","spurs"])
		team.query("bulls")
		
- rangeFilter and DVrangeFilter: These two are related and confusingly named. My apologies. A range filter is for cases in which collection members have a single numeric value for some attribute. You want to query that attribute by series of non-overlapping ranges. For example, every politician has a `money_raised` attribute. Your apps wants to display "Politicans with less than 1 million dollars","Pols with 1 - 5 million dollars","Pols with 5 million +". This is where you would use a rangeFilter:

		var moneyfilter = PourOver.makeRangeFilter("money_raised",[[0,1000000],[1000001,5000000],[5000001,1/0]])
		moneyfilter.query([0,1000000]);
		
	DVrangeFilters are similar. They also represent individual values that are queried by ranges. However, DVrangeFilters can support variable ranges (hence DV -- dynamic value). A good use case for this would for some incremental value, say color or number of children. Do not try to use a DVrangeFilter for a range with many values, say dollars between 1 and 10 million. Everything will crash. If you need this, we will bring in crossfilter to create this type of filter.
	
		var colorfilter = PourOver.makeDVrangeFilter("color",["red","orange","yellow","green","blue","indigo","violet"]);
		colorfilter.query(["red","blue"])
		// this will return all items with color between red and blue inclusive.
		
	**Range and DVRange filters must always be named the same string as the attribute that they track** Also, like exactFilters, range and dvrange filers will be associated to the attrs after which they are named.
	
- manualFilter: This filter, surpisingly, is a filter for manual queries. That is to say, rather than querying a manual filter with a desired value for some attribute, you pass `manualFilter` an array of cids and the matchset will be set to exactly those cids. This is useful when representing user-created slices of collections or editor-curated slices of collections. Say a user can move pictures into their personal collection. It is recommended that you do this by creating a manual filter to store the state of this collection, rather than creating some tag attribute.

		var edpicks = PourOver.makeManualFilter("edpicks")
		edpicks.query([3,7,19,25])

NOTE: when querying a manualFilter, the cids will always be sorted. If you need a specific order, to your manually-filtered set, use an explicitSort (covered later).
	
######## 2. What are the different sorts included with PourOver

Unfortunately, not many!

- explicitSort: This sort is for specifying a specific order for cids. This is useful in creating slideshows or featuring editorially ordered content.

		var collection_sort = PourOver.makeExplicitSort("collection-order",MOD.collection,"guid",[9,1,2],{associated_attrs:[]});
		
Here we create an explicit ordering on the guid attribute. Specifically, we are saying that this sort should put items in the order (based on guid) 9,1,2. All other items will sort to the end in insertion order. NOTE: It is important to specify associated_attrs as an empty array to prevent the sort from re-sorting when items are added or removed from the collection. Since, the sort is explicit, adding and removing items can't change the order.


		
#### Epilogue - The Philosophy of PourOver

----

Pourover is distributed under the [Apache 2.0 License](https://github.com/NYTimes/pourover/blob/master/LICENSE.txt).

<img src="{{site.baseurl}}/public/opennews-logo.png" alt="OpenNews logo" width="120" style="margin: 0" />
<a href="http://opennews.org/code.html" style="font-size:14px;">Released for OpenNews Code Convening, April 2014</a>
