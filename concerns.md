# Concerns

## List

* data validation
* logging
* data access
* external systems access
* configuration management
  * service discovery
  * configuration stores \(e.g., etcd, zookeeper\)
* instrumentation
* monitoring
* availability
  * terms
    * Mean Time To Recover \(MTTR\)
    * Mean Time Between Failures \(MTBF\)
    * Mean Time To Detect \(MTTD\)
    * Availability = MTBF / MTBF + MTTR
  * circuit breaker pattern
* scalability
* performance
  * load balancing
  * metrics collection
* exception management
* security and access management
* documentation
* build
* deployment
  * blue/green
* testing
  * A/B testing
* release practices
  * blue/green deployment
* SLAs
* SLOs
* KPIs

## Reading vs Writing

| Writing | Reading |
| :--- | :--- |
| Enforcing data integrity | Performing efficient searches and lookups |
| Providing atomic updates / transactions | Calculate derived values \(sums, ...\) |
| Facilitating version control \(optimistic concurrency locking\) | Providing multiple views of data |
| Enforcing write permissions | Enforcing row and column level permissions |



