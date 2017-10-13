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

## Design ideas with Kafka

event publishers \(n applications\) -&gt; send events -&gt; event mediators \(e.g., kafka\) \(listens/pushes to topics\) -&gt; push to subscribers

* client-side
  * smart components &lt;-&gt; service layer &lt;-&gt; web workers &lt;-&gt; Client Event Mediator &lt;-&gt; WebSocket
    * subscribe to event sources through the event mediator
    * react to received events: WebSocket -&gt; Client Event Mediator -&gt; services
    * publish events: services -&gt; Client Event Mediator -&gt; WebSocket
* back-end platform
  * microservice A
    * \(client request \| scheduler &lt;-&gt; API &lt;-&gt;\) service layer &lt;-&gt; Kafka client &lt;-&gt; Server Event Mediator 
      * subscribe to event sources through the event mediator
      * react to received events: Kafka client -&gt; service layer \(listeners\)
      * publish events: services -&gt; Kafka client -&gt; Server Event Mediator\)

  * microservice B
    * ...
  * Server Event Mediator
    * also a microservice
    * subscribe to topics

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

## 



