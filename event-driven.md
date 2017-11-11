# Event-Driven Architecture \(EDA\)

## About

EDA is an architecture paradigm centered around reacting to event notifications.

Can help create scalable and highly response applications but is more complex and requires a different mindset during development.

Example use cases:

* chat sessions
* social interactions
* games
* stock tickers
* monitoring
* dynamic data visualizations

Define interactions and handles state changes through the production and reaction to events using publishers, subscribers and event mediators.

With EDA we consider that the events are the main source of truth \(thus are first-class citizen in the system\). The analogy can easily be made with the real world:

* people try to make things happen by sending orders around them
  * issue a command like: "Order a pizza"
* other people and the world in general evaluates those commands and either accepts or rejects those
  * * an event is created: "Pizza ordered" containing all the details \(e.g., large, 4 cheese types, to deliver to ...\)
    * if it were rejected we could have "Pizza order rejected" ...
* once the order is accepted, the pizza can be prepared and delivered
  * in our system then we get additional events like "Pizza prepared", "Pizza delivered", ...

With EDA, events are kept preciously \(cfr Event Sourcing below\) as they represent everything that has ever actually happened in the system.

EDA heavily relies on Domain-Driven Design \(DDD\) principles, thus read this first: [https://www.gitbook.com/book/dsebastien/domain-driven-design-notes](https://www.gitbook.com/book/dsebastien/domain-driven-design-notes)

## Main concepts

* commands

  * request to perform something in the system
    * commands either produce events, throw error or nothing happens
  * immutable \(a request is a request, it shouldn't be changed\)
  * may be rejected or accepted
  * need validation

* command handler

  * validates commands
  * when allowed/accepted, commands modify the system state and 0..n events are be generated \(1 is common\)
  * command handlers contain logic
    * validation of structure, values, domain/business rules
    * security, auditing, ...

* events

  * facts
  * represent changes _that have already occurred_ in the system
  * should include or reference enough context and metadata so that subscribers receiving the events
  * immutable \(just like the past, you can't change it\)
  * cannot be deleted

* event publisher \(aka generators, sources, emitters\)

  * detects state changes
  * gathers the necessary information to describe the event
  * transfers the event to the event mediator
  * should have no knowledge, dependencies or expectations on event subscribers

* event subscriber \(aka handlers, sinks, consumers\)

  * register with an event mediator to receive an alert when the mediator receives a particular event type \(aka event "topic"\)
    * i.e., please push values to me regarding "x"
  * execute the necessary business logic and actions for the rest of the system to react to the event
  * have no dependencies or expectations on the event sources

* event mediator

  * provide the mechanism to transfer events from the publishers to the subscribers
  * allows subscribers to register for event types
  * receives events from publishers and alerts the relevant subscribers
    * asynchronously
  * may act as a simple pass-through or play a more active role
    * aggregate events
    * add security, compliance, etc
    * provide optimizations & load balancing

* saga \(aka process manager\)

  * component that reacts to domain events in a cross-aggregate, eventually consistent manner \(time can also be a trigger\)
  * sagas are sometimes purely reactive and sometime represent workflows
  * a saga is a state machine driven forward by incoming events \(which may come from different aggregates\)
    * some states will have side-effects \(e.g., sending commands, talking to external Web Services, sending e-mails, ...\)
  * sagas are doing things that individual aggregates can't do

## High level approaches

### Event Notification System

* goals & benefits
  * decouple systems /modules & reverse dependencies
  * turn the _change_ into a first-class element
* events vs commands
  * events: "Something has happened; I don't know or care what should/needs to happen next."
  * commands: "Something needs to happen, go do it!"
  * events should NOT be treated as commands: otherwise it will result in a poor implementation of EDA and an unpredictable system
  * commands should not be stored
    * command sourcing is often a bad practice
    * there's not guarantee that replaying commands will result in the same events!
* issue

  * no global view of what's happening within the system

  * only solution: watch the events and see what happens, looking at the flow of messages through the system

### Event-Carried State Transfer

* less often used
* events carry all the data that downstream systems might need
* downstream systems keep copies of the data in the events
* benefits
  * decoupling
  * reduce load on upstream systems
  * improved availability
* drawbacks
  * duplication of data
  * eventual consistency

### Event Sourcing

* when all changes to a system are caused by events
* events created for everything then processed
* events persisted in a data store
* application state = current state
* the application state can be kept in memory only and never persisted
  * the application state can be rebuilt from the log
* analogy
  * version control system
  * ledge in a financial system
* benefits
  * simple
  * flexible \(easy data migrations\)
  * performant \(caching, scaling, ...\)
  * audit trail
    * the log contains everything that's ever happened in the system \(true history\)
    * provides natural audit and traceability
  * intent trail
  * debugging
    * take a copy of the system, feed it with events and observe what happens
    * time travel debugging
  * alternative state
  * memory image
    * load the events at startup
    * keep the application state in memory
    * if the system crashes, quickly rebuild from the log \(or from snapshots\)
  * ease to extend the system naturally
    * since all events are kept, it's it's easy to exploit them
  * persistence of events is very easy, straightforward and efficient \(append-only\)
  * testing is clear: use exclusively the commands, events and exceptions
  * extra business value: BI, ML, temporal queries
* drawbacks
  * unfamiliar
  * external systems
    * responses of external systems must also be put into events so that the full history is kept
  * design needed for domain events
    * event schema is a must
  * identifiers
  * asynchrony
    * not necessarily needed with an event source system
    * can be nice because it improved responsiveness, but adds complexity
  * not trivial to query: build a snapshot to be able to query
  * uses more space
  * versioning
    * can get complicated
    * if the application state schema changes, can the log still be fully replayed?
    * snapshots can help because then it's only needed to reapply events that came after that snapshot \(i.e., shorter period of time\)
    * one advice: avoid business logic between events and their storage in the log \(otherwise versioning becomes an issue\)
    * upgrading events can be done by both the write and read sides which can upgrade events
      * if an event can't be upgraded it probably means that it's a completely different event
    * very important to design & document the event taxonomy \(event catalog\)
* what to store?
  * commands \(aka input event\)
    * buy 15 widgets \(captures business semantics\)
  * internal event \(captures change in records\)
    * buy 15 widgets
    * price: $30
    * shipping: $5
    * total: $33
  * output event
    * 15 widgets total $33 \(captures observable change\)
  * what to store then?
    * not storing the input events mean we lose the initial intention
    * not storing the internal event also loses information \(how the system reacted to the input event\)
    * maybe store all events?
  * commands can be logged but don't belong in the event store
* snapshots

  * optimization where a snapshot of the aggregate's state is saved in the event queue every so often so that the system can start from the snapshot instead of from scratch \(can speed things up; e.g., reduce time to recover\)

  * usually better to start without snapshots and adding it later if needed

* interoperability & compression

  * events may be compressed using libraries like Google's Protocol Buffers or Apache Avro

  * using Protocol Buffers or Avro also help with interoperability

### CQRS \(Command Query Responsibility Segregation\)

* CQRS separates commands \(performing actions\) from queries that return data
  * separates components that read and write to the permanent store
  * separates models \(actually separate components\)
    * one to deal with writes \(updates\)
    * one to deal with reads \(only return data, free from side-effects\)
  * separates storage, optimized for each side
* CQRS's separation of concerns aims to
  * simplify the system \(complex means "braided together", hence decoupling simplifies things :p\)
  * allow each to scale independently: very useful to scale the read side more
* read side

  * listens to events published from the write side, projects those events down as changes \(i.e., commands!\) to the local model and allow queries to be made on that model
  * make the cost of correlating model data \(i.e., JOIN\) from being per-read to being per-write
  * a query on a read side is just a straight SELECT, because data is already in the shape the client wants

* event updates consumed by all consumers

* event handled only once: command

* there's always a contract

## Event design and catalog

With such an architecture, events are front and center. They must be carefully designed and put in a detailed _event catalog_.

The catalog should allow to answer the following questions for each event type:

* what is the unique name of the event?
* what is the meaning/utility of the event?
* what is the name of the corresponding aggregate/aggregate root?
* what is the name of the corresponding topic?
* which components publish that event and when/why?
* what components may require the ability to subscribe/react to the event?
* what data/metadata must be included with this event type?
  * what are the data formats to use?
* how to detect state changes for this event type?
  * ensure there's a unique identifier associated with each event \(uuid\)
  * ensure there's a unique sequential number associated with each event \(for ordering\)
* security information
  * who can publish that event \(roles / attributes / constraints\)

At runtime the following information is also needed for each event:

* eventId
* aggregateId
* sequenceNumber: sequence number in the history of a particular stream
* creationTimestamp: transaction start time; the same for all events committed in one transaction
* correlationId
* eventData

With the above, it's easy to have an interface for going through the event store:

```
public interface EventStoreReader {
    List<Event> getEventsForStream(UUID streamId, long afterSequence, int limit);
    List<Event> getEventsForAllStreams(long afterEventId, int limit);
    Optional<Long> getLastEventId();
}
```

## Command design and catalog

* subjectId
* userId
* creationTimestamp
* commandId
* commandData

## Runtime concerns

* eventual consistency \(!\)
  * avoid dependencies on which order subscribers execute
  * identify events that must happen in a specific sequence and ensure to have an ordering mechanism in place for those
  * tolerate inconsistencies, knowing that eventually the system will be consistent
* error-handling with decoupled events
  * cannot rollback events \(immutable!\)
  * create "failure" events and react to that to fix issues
* offline subscribers
  * take advantage of event sourcing \(i.e., reapply event history / snapshots\) to put back disconnected/offline clients to an up-to-date / correct state
* event mediation

## Design ideas \(WIP\)

Goal: design a highly available, horizontally scalable system with loose coupling between its components.

MUST: leverage functional reactive programming \(FRP\). Java/Kotin: Reactor, JS/TS: RxJS...

### Components

#### Back-end platform

Micro-service architecture with at least the following components.

##### Platform-wide message broker \(aka event mediator\)

Example: [Apache Kafka](/apache-kafka.md)

All relevant "events" transit through it.

##### A platform-wide "client gateway"

All clients interact with it: queries, mutations, subscriptions

Exposes REST/GraphQL APIs

Handles subscriptions \(e.g., client interested in X\) \(**TODO**\)

##### A platform-wide "subscription manager" \(TODO\)

...

##### 1-n micro-services

One per bounded context \(e.g., payments subsystem, user profile subsystem, ...\)

##### A platform-wide Chaos Monkey

Killer in the house.

Spawned randomly and breaks things.

##### Analytics micro-service

Collect information about the platform

### Clients & Client Layers

The structure could be as follows.

* UI: dumb + smart components and their controllers

* Services

  * hold business logic
  * interact with lower layers

* Repositories \(interact with the back-end\)

  * e.g., REST and/or GraphQL client
  * create command objects based on requests
  * leverage WebSockets and/or Server-Sent Events

* Event Handlers

  * contains the event mediator: dispatches events within the client for consumption/reaction
  * contains the event publisher: handles REST/GraphQL or WebSocket calls

### Platform command microservices layers

As stated, the whole platform will consist of 1-n microservices.

Let's consider the layers of a typical Java microservice:

* optional: GraphQL and/or REST layer
  * validates input then delegates to the service layer
* Server Event/Command Mediator \(aka event handler\)
  * listens to specific topics and pushes incoming events/commands to the service layer
  * publishes events to the message broker
* Aggregate / Aggregate Root
  * handle commands and generate events
  * apply events
  * take care of
    * authorization
    * transaction management
    * ...
* Services

  * responsible for handling requests
  * either create a new aggregate
  * or update an existing one

* Domain model

* Repositories

  * handle interactions with data sources/stores
  * handle snapshots of the application state

### Flow of events between components

#### Clients

Communicate directly with

* option 1: 1-n back-ends
* option 2: only the platform's client gateway

In both cases, communication occurs through REST and/or GraphQL calls \(using queries & mutations\)

When doing so they trigger \(directly or indirectly\) the creation of events on the back-end platform.

When clients receive events for topics they've subscribed to, their Client Event Mediator will take care of reactively handling the event.

Flow in that case: Client Event Mediator -&gt; Service Layer \| other internal subscribers

#### Platform microservices

Activity of micro-services may be triggered by different means: client requests \(e.g., GraphQL queries and/or REST calls\), scheduled activities \(in-app triggers, external triggers, ...\), event notifications, ...

During activities \(e.g., client has submitted new data\), the service layer decides if a new aggregate needs to be created or if an existing one should be updated.

Afterwards, the aggregate decides if events need to be created. When it is the case, the event is created and passed to the Server Event Publisher which takes care of sending the event to the message broker \(e.g., using the Kafka client\).

Flow in this case: Aggregate -&gt; Server Event Publisher -&gt; Message Broker

Microservices may also subscribe to some topics. When it receives events, its Server Event Mediator will take care of reactively handling these \(i.e., letting them flow for consumption\).

Flow in this case: Server Event Mediator -&gt; Service Layer -&gt; ...

### State Machines

When clients interact with the back-end \(whatever micro-service\), they issue "commands".

When a command is received, if accepted, it triggers the creation of an event. That event is stored and processed.

When we process each event, the concerned parts of the system may change their "state" we can represent all the possible state of some sub-system as a Finite State Machine \(FSM\). Basically, events are what allows the FSM to go from a state to another.

The benefit if we use FSMs is that once an event occurs, we can pass it to the FSM and see if it changed its state. Once the state changes, new events are generated and dispatched. Another benefit is that FSMs are easy to describe, encapsulate the logic, ...

### Last-Versioned pattern

A useful pattern with Event Sourcing is to hold events twice:

* once in a retention-based topic
* once in a compacted topic

The retention-based topic will be larger as it holds the version history of your data.

The compacted topic is just the "latest" view and will be smaller, so it's faster to load into a Memory Image or state store \(e.g., DB\).

#### Apache Samza can help

With Samza you write jobs that consume the events in a log and build cached views of the data. When a job first starts up, it can build up its state by consuming all the events in the log. Then, on an ongoing basis, whenever a new event appears in the stream, it can update the view accordingly. The view can be any existing database or index. Samza just provides the framework for processing the stream.

Anyone who wants to read data can now query the materialized views created and maintained by Samza. Those views are just databases, indexes or caches and you can send read-only requests to them in the usual way.

To write, you write to the event log and there is an explicit transformation process which takes the data on the log and applies it to the materialized views.

### Open Questions / TODO

* construction of the read model
  * redis?
  * mysql or other?
* snapshots handling
* subscriptions handling
  * option 1: platform's client gateway is stateful
    * it delegates data fetching to the subscription manager but is the one having a long-standing open connection with the subscribed client\(s\)
    * plays with commands and listens to specific subscription-data events on a dedicated topic \(e.g., observe subscription-data events that match known clients\)
    * problem: clean-up of inactive clients, stop data generation, etc
  * option 2: platform's client gateway is stateless
    * point the client to an instance of the subscription manager and let that one handle things itself

# Links

* [https://martinfowler.com/eaaDev/EventSourcing.html](https://martinfowler.com/eaaDev/EventSourcing.html)

* [http://eventuate.io/](http://eventuate.io/)
* [http://blog.xebia.fr/2017/01/16/event-sourcing-comprendre-les-bases-dun-systeme-evenementiel/](http://blog.xebia.fr/2017/01/16/event-sourcing-comprendre-les-bases-dun-systeme-evenementiel/)

* [https://github.com/cer/event-sourcing-examples](https://github.com/cer/event-sourcing-examples)

* [https://en.wikipedia.org/wiki/Publish–subscribe\_pattern](https://www.gitbook.com/book/dsebastien/software-architecture-notes/edit#)



