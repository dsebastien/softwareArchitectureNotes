# Event-Driven Architecture

## Event Notification System

* goals & benefits
  * reverse dependencies
  * decouple systems /modules
  * turn the _change_ into a first-class element
* events vs commands
  * events: "Something has happened; I don't know or care what should/needs to happen next."
  * commands: "Something needs to happen, go do it!"
* issue

  * no global view of what's happening within the system

  * only solution: watch the events and see what happens, looking at the flow of messages through the system

## Event-Carried State Transfer

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

## Event Sourcing

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
  * versioning



