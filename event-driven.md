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

## Main concepts

* events
  * represents a change in state that is relevant in the system
  * should include or reference enough context and metadata so that subscribers receiving the events 
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

## Event design

With such an architecture, events are front and center. They must be carefully designed

## High level approaches

### Event Notification System

* goals & benefits
  * decouple systems /modules & reverse dependencies
  * turn the _change_ into a first-class element
* events vs commands
  * events: "Something has happened; I don't know or care what should/needs to happen next."
  * commands: "Something needs to happen, go do it!"
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

* events created for everything then processed
* events persisted in a data store
* application state = current state
* the application state can be kept in memory only and never persisted
  * the application state can be rebuilt from the log
* analogy
  * version control system
  * ledge in a financial system
* benefits
  * audit
    * the log contains everything that's ever happened
  * debugging
    * take a copy of the system, feed it with events and observe what happens
    * time travel debugging
  * alternative state
  * memory image
    * keep the application state in memory
    * if the system crashes, quickly rebuild from the log \(or from snapshots\)
* drawbacks
  * unfamiliar
  * external systems
    * responses of external systems must also be put into events so that the full history is kept
  * event schema
  * identifiers
  * asynchrony
    * not necessarily needed with an event source system
    * can be nice because it improved responsiveness, but adds complexity
  * versioning
    * can get complicated
    * if the application state schema changes, can the log still be fully replayed?
    * snapshots can help because then it's only needed to reapply events that came after that snapshot \(i.e., shorter period of time\)
    * one advice: avoid business logic between events and their storage in the log \(otherwise versioning becomes an issue\)
* what to store?
  * input event
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

### CQRS \(Command Query Responsibility Segregation\)

* often problematic
* separate components that read and write to the permanent store
* two separate models \(actually separate components\)
  * one to deal with writes \(updates\)
  * one to deal with reads
* commands either produces events, throws error or nothing happens

* queries only return data, free from side effects

* separated storage, optimized for each side

* event updates consumed by all consumers

* event handled only once: command

## Design idea for a platform using Event Sourcing

Goal: design a highly available, horizontally scalable with loose coupling between the components.

MUST: leverage functional reactive programming \(FRP\). Java/Kotin: Reactor, JS/TS: RxJS...

### Components

#### Back-end

Microservice architecture with at least the following components:

* a platform-wide \(i.e., global\) highly available message broker _with_ streaming support \(e.g., Kafka\)
  * all relevant platform events transit through it
* 1-n microservices
  * bounded context \(e.g., payments subsystem, user profile subsystem, ...\)
* a "server event mediator" embedded in each microservice
  * dispatches events within the microservice for consumption/reaction
* a "server event publisher" embedded in each microservice
  * publishes events to the message broker
* a platform-wide "back-end event mediator"
  * responsible for orchestration between event publishers and subscribers
* a platform-wide "back-end event manager"
  * responsible for long-term storage \(event sourcing and creation & management of snapshots\)

Easy to add analytics

* real time: subscribe to topics
* delayed: leverage the long-term storage & snapshots

#### Clients

* a client event mediator embedded in each client
  * dispatches events within the client for consumption/reaction
* an event publisher embedded in each client
  * e.g., GraphQL calls or WebSocket

### Client Layers

If we take the example of a Web or node.js based application, the structure could be as follows.

* UI: dumb + smart components and their controllers
* Services
  * hold business logic
  * interact with lower layers
* Repositories
  * REST and/or GraphQL client
  * leverage WebSockets and/or Server-Sent Events

### Platform microservices layers

As stated, the whole platform will consist of 1-n microservices.

Let's consider the layers of a typical Java microservice:

* GraphQL and/or REST layer
* Services
  * hold business logic
  * take care of transaction management
  * take care of authorization
  * ...
* Repositories
  * handle interactions with data sources/stores

### Flow of events between components

#### Clients

Communicate directly with 1-n back-ends through REST and/or GraphQL calls \(using queries & mutations\).

When doing so they trigger \(directly or indirectly\) the creation of events on the back-end platform.

In some cases clients will directly create the events and publish them. Each contacted back-end hit by calls decides when to generate events for the rest of the platform \(cfr next section\).

When clients receive events for topics they've subscribed to, their Client Event Mediator will take care of reactively handling the event.

Flow in that case: Client Event Mediator -&gt; Service Layer \| other internal subscribers

#### Platform microservices

Activity of microservices may be triggered by different means: client requests \(e.g., GraphQL queries and/or REST calls\), scheduled activities \(in-app triggers, external triggers, ...\), event notifications, ...

During activities \(e.g., client has submitted new data\), the microservice's service layer decides if events need to be created. When it is the case, the event \(value-object\) is created and passed to the Server Event Publisher which takes care of sending the event to the message broker \(e.g., using the Kafka client\).

Flow in this case: Service Layer -&gt; Server Event Publisher -&gt; Message Broker

Microservices may also subscribe to some topics. When it receives events, its Server Event Mediator will take care of reactively handling these \(i.e., letting them flow for consumption\).

Flow in this case: Server Event Mediator -&gt; Service Layer -&gt; ...

### Open Questions / TODO

* add a chaos monkey component: killer in the house
* design of the events hierarchy?
* design of the events structure?
* storage of the events?
* evolution of the events and associated payloads?
* overlap/separation between platform events and events clients may/should know about?
* microservices: send events to kafka directly or go through the platform-wide event mediator?
* clients: send events to the platform-wide event mediator or to another specific microservice instead?
  * ideally clients should have a single interlocutor \(graphql idea\) but that's creating a spof
    * related question about subscriptions
* how to handle subscriptions to events?
  * from microservices
  * from clients
* how to handle up-to-dateness of clients?
  * numbering of all events to be able to clearly reconstruct a story and know exactly at which point in time each client currently is
* leverage blockchains? tamper-proofness / auditing / ...
* auditing: what information to keep for audit trails?

# Links

* [https://en.wikipedia.org/wiki/Publish–subscribe\_pattern](https://en.wikipedia.org/wiki/Publish–subscribe_pattern)



