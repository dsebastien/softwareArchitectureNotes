# GraphQL

From GraphQL specs: "GraphQL is unapologetically driven by the requirements of views and the front-end engineers that write them."

## What?

* Interface Definition Language
* language agnostic
* public specification

## Principles

* product-centric
* client-driven
* strongly typed
* introspective

## Motivations

* release cadence
  * server can be released at will
  * client app takes hours to days to release \(e.g., Android clients\)
  * many app versions in the wild concurrently
    * some clients won't want or won't be able to upgrade
* latency
  * reduce the number of network requests
  * worst case: make many dependent requests and wait
* bandwidth
  * get ONLY what you need

## Why not rest?

* REST resources aligned with the exact needs of a view means NOK with REST \(back to ad-hoc RPC\)
* time spent maintaining many versioned endpoints
* client defining the response \(i.e., content negociation\) is rare in practice
* REST is loosely specified
* comparison / analysis: [https://apihandyman.io/and-graphql-for-all-a-few-things-to-think-about-before-blindly-dumping-rest-for-graphql/](https://apihandyman.io/and-graphql-for-all-a-few-things-to-think-about-before-blindly-dumping-rest-for-graphql/)

## Benefits

* make multiple queries at once
* one API in hiding others if needed
  * useful analogy: "If you want to go and buy things from 3 different stores\)
    * with REST: make 3 different calls \(1 towards each store's API\)
    * with GraphQL: make 1 call to the GraphQL server and let it take care of everything
* strongly-typed schema with documentation

  * built-in discovery for GraphQL APIs!
    * query the \_\_schema meta-field
    * ```
      query {
          __schema {
              types {
                  name
              }
          }
      }
      ```
  * hides the details of the back-end architecture from the front-end

  * the schema can be written in GraphQL schema language

* clients easily choose what they data get

  * they define the response
    * each call fetches exactly what is needed, no less, no more
  * including what they get after a mutation \(i.e., create, update or delete operation\)
  * responsibility swapped to the client side

## Operation types

* query
* mutation
* subscription

## Field Types

* String
* Int
* Float
* ID: same as String
* Boolean
* Enum
* Object

By default, all fields are nullable

## Best practices

* naming matters: self-documenting names
* search

  * separate the data away from the feature
  * design search as a type in the graph

    * ```
      type Search {
          id: ID
          title: String
          results(first: Int, after: ID): [Node]
          suggestedSearches(first: Int, after: ID): [Search]
      }

      type Query {
          search(query: String): Search
      }
      ```

      allows to later go further into that graph \(e.g., get the suggestedSearches, ...\)

* create a single, cohesive graph

* describe the data, not the view

  * don't model for the view we're going to support

* version-less API

  * think about how the product will evolve in the future \(i.e., will my API still make sense in the next iteration?\)
  * no need to version the endpoint
  * clients use types in different ways
  * clients can define their own fragments \(i.e., subsets of specific types\)
  * adding fields to types is cheap
  * breaking changes
    * cannot change the high level types!

* thin interface

  * don't try to handle everything with GraphQL \(e.g., authentication, authorization, caching, session management, DB access, ...\)
  * do it OUTSIDE!

* hide implementation details

  * what if the implementation changes?
    * does the API continue to make sense when that happens?

* basic questions

  * would a new engineer understand this?
  * what might v2 look like?
  * will this work for all future clients?
  * what if the implementation changes?

* principles & lessons

  * solve the most important real problem that you have
    * problems drive priorities
  * think like the client
  * yagni
  * second time around problem

## Subscriptions

With event-based subscriptions, the client tells the server that is cares about or one more events. Whenever these events trigger, the server notifies the client. This model requires the server to identify a set of well-known events and how to raise/propagate them, ahead of time.

Subscriptions require server-side state, such as which subscriptions are active, what events they are listening to and the mapping of client connections to subscriptions.

Types:

* fixed-payload subscriptions: payloads are defined by the server-side ahead of time \(fixed\)
* zero-payload subscriptions: sends empty events to the client where it triggers a data fetch
* data-transformation pipelines: for cases where data payloads differ between subscribers

Alternative: live queries. The client issues a standard query. Whenever the answer to the query changes, the server pushes the new data to the client. The different is that live queries do not depend on the notion of events.

With GraphQL subscriptions, the client sends the server a GraphQL query and query variables. The server maps these inputs to an event stream and executes the query when the events trigger.

GraphQL subscriptions produces privacy-aware, right-sized payloads without pushing business logic to the event/messaging layer.



