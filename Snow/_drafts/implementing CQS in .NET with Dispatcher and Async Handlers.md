
## Audience

If most of these terms look unfamiliar then this post is probably not for you: CQRS, Repository, Aggregate (of DDD), NuGet, Unit Testing, Dependency Injection, Document DB. Due to the scope of this post, I won’t be able to go in details in any of these topics as this is a direct implementation.

Greg Young did a few paragraphs intro on CQRS here: CQRS, Task Based UIs, Event Sourcing agh, if you want a simple intro I recommend at least reading the first half of Greg’s post (before the Event Sourcing).

In this post, I mean the plain-CQRS pattern and not the whole patterns that are associated with CQRS. Plain CQRS opens the door for other patterns such as Event Sourcing and ES is usually associated with CQRS. This is the plain CQRS with no other associated pattern.

We use the term Request for a data object containing parameters that you want to dispatch to a handler. 

First, to explain the terms I use and to what I refer in this article.

- **CommandRequest** - Command, Query  
I treat the `Commands` and `Queries` as `Request` objects. Every read or write operation represents a `Request`. 
- **CommandResponse** - Command Response, Query Response  
Every object returned from handler execution is a `Response` object. `Request` and `Response` classes come together as a pair.
- **CommandHandler** - class that executes the `Request` and as a result returns suitable `Response`. Could be a CommandHandler or a QueryHandler, in thus code I don't make a difference.
- **CommandDispatcher** - class that maintains list of handlers. For each dispatched request, it finds it's handler and executes it. It's goal is to decouple the handler executions from the caller.

- **Event** - event request
- **EventHandler** - handles events
- **EventPublisher** - publishes events

Some `Request` examples: GetCustomerInfo, ChangeCustomerAddress, PayBill, ListBillsPaged etc.

Finally we want to give it a request context - a data structure passed to each handler in the chain, with global information including a property bag.

<br /><br /><br /><br /><br /><br /><br /><br /><br /><br /><br /><br /><br /><br /><br /><br /><br /><br /><br /><br /><br /><br />

Implementing Command and Query in .NET with Dispatcher and Async Handlers


The request is independent of its processing and the application
dynamically links the command to the command handlers.
Changing logic now is entirelly foccused in one class. So changes will happen only there. What a relief?

- flexibility
- sharing
- intention revealing names

Benefits?
- Removes the Fat Controller problem
- User intent
- naming
Drawbacks?

Here, we can send the request as a parametar object to the handler
Command Dispatcher

- https://cuttingedge.it/blogs/steven/pivot/entry.php?id=92


Transitioning from large Application Service objects to small and focused Usecase objects (Request + Response + Handler + Validation)

Past days in order to abstrack away from Fat controllers, people started pulling orchestration code out of controllers and putting them to so called application service objects. But, as seen - that way just the cheese has been moved with little benefit gain. Error handling strategy still was same. Classes were not focussed. They were too big to be handled. Many code repetitions were done. But on contrary, now we were able to do good integration and unit tests. Reduced code duplication. Made the ability for code reuse for different deployment environments.
In this article I will show you the next natural step in refactoring these application services. It is not something revolutionary or new, but certinally is evolunionary in application services terms.
We have single point of contact for anything that we might need regarding operation's execution.

In the past I had the chance to work with many applications that made use of application services and I was thrilled how good it is. Next, when I started architecting systems for our client I made use of those app services which seemed good. But as the time goes, the experience increseas and than your mind finds some patterns, sees how your team works with the system, commits code, changes files, traverses the solution directory when building a feature - and you start to understand that it's not so natural to work like this.

Command Dispatcher is often used with a hierarchical architecture to avoid the Fat Controller problem and allow us to decouple from the caller.
https://iancooper.github.io/Paramore/FatController.html
https://iancooper.github.io/Paramore/WhyCommandProcessor.html
https://iancooper.github.io/Paramore/CommandsCommandDispatcherandProcessor.html

# Functional over Technical code structure and organization


# Configuration management with Bootstrapper class

# Feature folder



http://blog.ploeh.dk/2014/05/19/di-friendly-framework/