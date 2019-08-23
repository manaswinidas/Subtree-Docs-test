# Kogito Data Index Service

This service aims for capturing and indexing data produced by one more Kogito runtime services.

Overall goals include:
* Focus on domain data ( Orders, Travel, etc ) 
* Flexible data structure
* Infinispan as first-class persistence service
* Distributable and cloud-ready
* Messaging based communication with Kogito runtime ( Kafka - Cloud Events )
* Powerful querying API using GraphQL

It is also important to note that this service is not intended to be used as permanent storage or audit log information. Focus is to make business domain data easily accessible for processes that are currently in progress.

## Technical Overview
From a technical perspective, the Data Index is a Quarkus application based on VertX and reactive messaging that exposes a GraphQL endpoint, allowing client applications to easily access business domain-specific data as well as technical detailed information about running process instance.

In its current version, it uses Kakfa messaging to consume [CloudEvents](https://cloudevents.io) based messages from Kogito runtimes, process and index the information for later consumption via GraphQL queries.
These events contain information about units of work executed for a process. Visit [process-runtime-events](https://github.com/kiegroup/kogito-runtimes/wiki/Configuration#process-runtime-events) for more details about the payload of the messages.
This data is then parsed and pushed into different Infinispan caches.
These caches are structured as follows:
 - Domain cache: This cache is a generic cache, one per process id, where the process instance variables are pushed as the root content. This cache also includes
some process instance metadata, which allows correlating data between domain and process instances. Data is transferred as JSON format to Infinispan server as a concrete Java type is not available. 
 - Process instance cache: Each process instance is pushed here containing all information, not only metadata, that includes extra information such as nodes executed.

Storage is provided by [Infinispan](https://infinispan.org/) which enables a cloud-ready and scalable persistence, as well as [Lucene](https://lucene.apache.org/) based indexing. Communication between the Index Service and Infinispan is handled via [Protocol Buffers](https://developers.google.com/protocol-buffers/).

Once the data is indexed and stored into the cache, the Data Index service inspects the process model in order to update the [GraphQL](https://graphql.org) schema, allowing a type-checked query system for consumer clients.

