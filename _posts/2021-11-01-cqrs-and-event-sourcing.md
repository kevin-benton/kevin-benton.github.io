---
layout: post
title:  "CQRS and Event Sourcing with Azure Functions and Cosmos DB"
date:   2021-11-01 10:00:00 -0500
categories: blog
---

Early in my development career, I learned how to create a basic REST API with the 
[Slim PHP framework][slim-link]. This basic API defined resources and allowed a developer to 
perform basic CRUD operations against thouse resources. As I grew, I learned that direct 
database manipulation isn't always the best approach to designing an API. Resource constraints 
and scaling led to a need for something more robust. This is where I found out about CQRS and 
event sourcing.
<!--more-->

Now CQRS and event sourcing are 2 different API design patterns; however, they fit very well 
together. In this article, I will briefly introduce CQRS and event sourcing and show an example 
of how this can be implemented on the Azure platform using Azure Functions and Cosmos DB. While 
there are other ways to implement these 2 patterns, the bindings of Azure Functions and the 
change feed of Cosmos DB allows for this to be a very easy task. However, there are some tradeoffs.

## CQRS

CQRS stands for command and query responsibility segregation. This separates the read and write 
operations of a database. While this doesn't make it abundantly clear of why this pattern is useful, 
it is easier to imagine a large system like Twitter. While this is probably not how they implement 
their system, you can image this pattern being used for a user's timeline. When I tweet something, a 
command is created and written to the write store - a very fast operation. This triggers an async 
process that will add my tweet to the timelines of all my followers in the read store. Now when one 
of my followers queries the read store for their timeline, the read is optimized.

Other benefits of CQRS include:
1. Independent Scalability - Writes and reads can be independently scaled so that you can keep reads 
   cheap and fast.
2. Optimized Read Schemas - One write command can update multiple read models to create simpler queries 
   for clients.
3. Security - Only allow writes on certain domains.

However, there are also tradeoffs when implementing CQRS over basic CRUD.

1. Complexity - It is an extra layer of design and implementation in the system.
2. Messaging - Although not directly required, CQRS implementations usually also introduce some form 
   of async messaging to keep endpoints even faster and in order.
3. Eventual Consistency - You lose the strong consistency with direct database updates as now a process 
   has to run to update the read models.

## Event Sourcing

Event sourcing is a pattern that instead of keeping a database model of your resource updated with the 
latest state, all events on an aggregate are stored in a database and then the state of the aggregate 
can be rebuilt at any time. This can be thought of as an immutable ledger. Once an event is added to 
the database, it cannot be changed. When you need to know the state of the object, you pull all the 
events from the database sorted by created time, and apply those events to build the state.

This fits well with CQRS because a command can trigger multiple events to be created for an aggregate. 
These events make up the write side of the database. Those events can be read directly as the read 
store to return the current state or read models can be created as part of an async process that can 
then be queried.

Like CQRS, event sourcing also has its benefits and tradeoffs. Benefits include a built in audit log 
in the form of events, traceability of change with the addition of a user/system property in the
commands and events, and extensibility to be able to replay events at any time to build new read 
models. Tradeoffs though again introduce complexity, messaging, and eventual consistency as detailed 
in the CQRS section.

## Example

In my latest [repo][issue-tracker], I have started an issue tracker project to better learn these 2 
patterns. I have started with the backend API before moving to a front end implementation. So what 
advantages does implementing these 2 patterns add to my solution?

First, I wanted each issue to have a history attached. This includes changes in properties such as 
state and priority. With the user ID added to each command and event, event sourcing adds a built in 
audit log for each issue.

Second, I wanted the API to be very responsive and not worry about dirty writes when multiple users 
are potentially updating the same issue. CQRS allows me to create commands for data manipulation and 
use a queue based system to apply those changes. Any oddities can be explained in the audit log. With 
this, all endpoints will be lightning fast as they either write a simple object or perform a point 
read on an object.

To build this, I used Azure Functions and Cosmos DB to utilize the bindings and change feed for async 
messaging. Let me illustrate by describing the creation of an issue through the API.

1. Call to Create Issue endpoint with title, description, and priority.
2. Command is created in command table with passed in data and added ID.
3. Background change feed process takes the command and generates a created event in the events table.
4. Another background change feed process takes the event and updates a read model with the issue data.

It is not until after step 4 that the issue is available in the read endpoints of the API. This is the 
eventual consistency that is described in the CQRS and event sourcing sections.

A more detailed write up and illustration of the data flow and architecture can be found in the 
[software architecture document][sad-link].

There are some tradeoffs of using the change feed that I would like to point out. If using CQRS and 
event sourcing in a true microservice architecture, it is not good practice to let other services 
directly create commands in this service's database. Therefore other services would either have to use 
the APIs provided by the service (a synchronous operation) or gRPC (also synchronous) or have access 
to a message bus to place the command on and not worry about it being processed. The last way is the 
most scalable method. But if there is a message bus in use, such as service bus, the API endpoints 
should also use that message bus for commands to not duplicate effort or code.

Likewise, if another service needs issue events, it makes sense to write events to a pub/sub topic so 
that multiple services can describe and create/update their own read models. Since a pub/sub topic is 
already created, this service can jsut subscribe to that topic to update its own read model to prevent 
adding one more method of getting events such as using the change feed.

Now that neither the commands nor events are using the change feed, is Cosmos DB still the best choice 
for the system? Maybe not. This pattern can be used with any database - however be prepared to tackle 
some other problems such as writing the command to a database and processing queue at the same time. 
What if one fails? You will need to look at other patterns such as the outbox pattern. Maybe the 
change feed can be used in addition to the message bus to provide the outbox pattern. I say all of 
this vaguely so that you are aware that there are added complexities to consider when adding CQRS 
and/or event sourcing.

## Conclusion

Overall, CQRS and event sourcing add complexity to your system in a tradeoff for scalability and 
responsiveness. It will be up to you to decide if this complexity is necessary or required for your 
system. I hope that this article gives you an overview as to some of the things you need to decide if 
you are considering these patterns.

[slim-link]: https://www.slimframework.com
[issue-tracker]: https://github.com/kevin-benton/issue-tracker
[sad-link]: https://github.com/kevin-benton/issue-tracker/blob/main/docs/sad.md