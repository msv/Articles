## The Basics of NSIncrementalStore

In this article I will introduce NSIncrementalStore, an abstract class provided by Apple that allows custom implementations to define the logic behind basic Core Data operations.  In layman's terms, it allows you to hook in your own custom backend to Core Data.   

Core Data, Apple's object graph and data persistence framework, is a powerful feature that has the ability to provide apps with a sand-boxed local SQLite database.  Apple has taken the time to design a framework which abstracts all the database logic away from the developer while emphasizing read and write performance, scalability and low memory usage.  While the concepts and language used by Core Data presents a slight learning curve for some developers, the benefit of using it is enormous.  Out of the box you are provided with such features as a solid user interface for creating your app's object graph and support for relationships with delete rules.  There is also superb query support, with the ability to write a query in string format and have values passed in and converted during run time.

While almost all apps have some use for a local database, these days the concept of data storage is being shifted to the cloud. Now more than ever, applications are being designed such that there is a need to have data centralized, so it can be accessed by multiple devices across multiple platforms at the same time. Apple introduced Core Data support for iCloud a while back, but that doesn't solve the cross platform problem, and in reality the support is limited. Instead, many developers either roll their own backends or use services like StackMob, an end-to-end platform for building, deploying and growing mobile applications. The problem we want to solve, however, is how we integrate our remote backend into Core Data so we get the best of both worlds.

Meet NSIncrementalStore.  Introduced in iOS 5.0, and somewhat of a hidden gem, this class allows developers to define the logic that takes place at certain points during Core Data operations.  For example, when you execute a fetch request, somewhere in the performance management logic Core Data will call your custom implementation for fetching.  The puesdocode is simple: convert the fetch request into a REST-based request, execute it on your remote backend, and return the results back to Core Data.  The implementation is left open to the developer. These hooks are implemented during saves, fetches, and faults i.e. all interactions with the external persistent store.  You can route the requests to wherever your heart desires, as long as you return the results in the correct format so Core Data can continue executing the operation properly.

Before diving into the methods that need to be implemented for an incremental store, lets take a look at the difference between an Atomic Store and an Incremental Store.  It's actually pretty simple: an atomic store is used to read and write entire sets of data in XML form, whereas an incremental store works similarly to Core Data's default behavior - by bringing data from the external store into memory in as small as increments as possible.  One of the biggest patterns Core Data uses is the idea of bringing object data into memory "incrementally."  This extreme lazy-load approach keeps the amount of in-memory data as low as possible.  By default, when you fetch objects, they are returned as object references, called faults.  It's not until you actually do something with the object's data that it's brought into memory.  Even so, if you made a call to read an attribute, only the object's attribute values would be brought into memory, leaving behind relationship data until you specifically asked for it.  To match the fast performance of bringing fetched object data into memory from a local store, many incremental store implementations will keep a cache of fetched objects to use to fill faults.

NSIncrementalStore, which inherits from NSPersistentStore, only requires 5 methods to fully implement.  Let's take a brief look at each method and its role:

### loadMetadata:

A simple method where you can define metadata for your custom incremental store, such as the base URL for where the external persistent store lives.

### executeRequest:withContext:error:

This is where all primary Core Data operations start.  When this method is called, it is passed a request type (save or fetch), the managed object context that initiated the request, and a reference to store any error that occurs during the process.

From here it is up to the developer to write the logic which handles the request, be it a save (create, update, delete) or fetch (read).

The bulk of the incremental store logic stems from this method, as it requires translating the request appropriately, sending it to the persistent store, handling the response, and returning the results in the correct format.

### newValuesForObjectWithID:withContext:error:

This method is called sporadically when Core Data needs the values of a particular object, either to fill a fault (bring the values into memory) or compare to in-memory values for merge purposes before a save.  It is passed the managed object ID, the managed object context, and a reference to an error.

Like I mentioned earlier, you could either make a network request to the persistent store for the values, or use some kind of cache where the object's values were stored at the time of the original fetch.

The return type of this method is a dictionary-like data structure called an NSIncrementalStoreNode, which represents a managed object and its data.   The data should include the values for all the attributes and optionally to-one relationships.

### newValueForRelationship:forObjectWithID:withContext:error:

This method exclusively asks for the value of a relationship, be it to-one or to-many.  The method gets passed the relationship name, the parent object ID, the managed object context and a reference to an error.  Whether a cache is used or the developer makes a request to the persistent store, this method returns the managed object IDs of the related objects for the given relationship.

### obtainPermanentIDsForObjects:error:

You may notice that when you first create a managed object in memory, it has a managed object ID whose last component is "t" plus some random string.  The "t" is for temporary.  The role of this method is to assign permanent IDs to newly inserted objects before executeRequest:withContext:error: is called. To create the managed object ID, NSIncrementalStore provides a method newObjectIDForEntity:referenceObject:.  This allows the developer to choose the reference object, which ends up being the random string at the end of the ID, and comes in handy so the Core Data ID can be related to the primary key of the object in the external persistent store.  After the save, you'll notice the "t" becomes a "p", for permanent.

Once you have implemented the 5 methods your incremental store will be ready to go!  Basic reads and writes are pretty simple to get up and running.  You will probably see yourself spending more time converting different queries or supporting unique features your backend provides.  The customization is endless, and Core Data seems to have purposely left the method implementations open ended.  If enough developers actively use incremental store we will hopefully see more methods in the future to allow deeper access into the full save/fetch pipeline.

In the meantime, there are a few open-sourced examples of incremental store implementations:

StackMob, https://github.com/stackmob/stackmob-ios-sdk, built their entire iOS SDK around a custom incremental store implementation, so developers could use Core Data directly to interact with their service.  This is an example of a full fledged incremental store implementation that, while customized to work with StackMob, shows off many examples of request conversions and even a concurrency API to run Core Data operations in an asynchronous fashion.

AFIncrementalStore, https://github.com/AFNetworking/AFIncrementalStore, lays out the foundation of incremental store for developers to build off of.

A Chris Eidof post which details building an incremental store from scratch: https://gist.github.com/chriseidhof/1860108.

A Sealed Abstract article on what NSIncrementalStore means for developers in the future: http://sealedabstract.com/code/nsincrementalstore-the-future-of-web-services-in-ios-mac-os-x/.



