# GraphQL

From GraphQL specs: "GraphQL is unapologetically driven by the requirements of views and the front-end engineers that write them."

## Benefits

* make multiple queries at once and 
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
* clients easily choose what they data get
  * including what they get after a mutation \(i.e., create, update or delete operation\)
  * responsibility swapped to the client side

## Operation types

* query
* mutation
* subscription

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



