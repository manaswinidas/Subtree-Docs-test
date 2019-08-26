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

![](https://github.com/kiegroup/kogito-runtimes/blob/master/docsimg/data-index-architecture.jpg)

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

# Instructions for developers

Data Index service is a Quarkus based application that aims to consume [CloudEvents](https://cloudevents.io) based messages from Kogito runtimes, process and index the information for later consumption via GraphQL queries.
These events contain information about units of work executed for one or multiple processes. Visit [process-runtime-events](https://github.com/kiegroup/kogito-runtimes/wiki/Configuration#process-runtime-events) for more details about the payload of the messages.
This data is then parsed and pushed into different Infinispan caches.
These caches are structured as follows:
 - Domain cache: This cache process type specific cache, one per process id, where the process instance variables are pushed as the root content. This cache also includes
some process instance metadata, which allows correlating data between domain and process instances. Data is transferred as JSON format to Infinispan server as a concrete Java type is not available. 
 - Process instance cache: Each process instance is pushed here containing all information, not only metadata, that includes extra information such as nodes executed.

Storage is provided by [Infinispan](https://infinispan.org/) and integrated using `quarkus-infinispan-client`. Communication between the Index Service and Infinispan is handled via [Protocol Buffers](https://developers.google.com/protocol-buffers/) which requires creating
`.proto` files and marshallers to read and write bytes. For more information, visit the [Quarkus Infinispan Client Guide](https://quarkus.io/guides/infinispan-client-guide).

In order to enable indexing of custom process models, the Data Index Service consumes proto files generated for a given process, see [process-instance-variables](https://github.com/kiegroup/kogito-runtimes/wiki/Persistence#process-instance-variables).
That can be added by simply making the proto files available in a folder for the service to read once starting.
Once the Protocol Buffer model is parsed, a respective [GraphQL](https://graphql.org) type is generated to allow clients to query the information. Again, the process data is available
via the two caches, where users can query based on the technical aspects (Process Instances) or domain-specific (proto defined type).

The Data Index Service is also a [Vert.X](https://vertx.io) based application for more details, visit [using-vertx](https://quarkus.io/guides/using-vertx). An important aspect
provided by Vert.X is the native integration with [Java GraphQL](https://www.graphql-java.com/) provided by the [vertx-web-graphql](https://vertx.io/docs/vertx-web-graphql/java/).

Messaging integration is also provided by Quarkus using `quarkus-smallrye-reactive-messaging-kafka`. For more details and configuration details, visit [kafka-guide](https://quarkus.io/guides/kafka-guide) and [smallrye-reactive-messaging](https://smallrye.io/smallrye-reactive-messaging/).

## Getting started

You will need:
 - Infinispan server:
   Download and run https://downloads.jboss.org/infinispan/10.0.0.Beta4/infinispan-server-10.0.0.Beta4.zip.
 Service will be available on port 11222. This should match the setting `quarkus.infinispan-client.server-list=localhost:11222` on `application.properties`.
   To enable our simplified demo setup, go to /server/conf/infinispan.xml and remove the security domain from the endpoints definition:
```xml
<endpoints socket-binding="default">
```

 - Kafka messaging server:
   The best way to get started is to use the Docker image that already contains all the necessary bits for running Kafka locally.
   For more details visit: https://hub.docker.com/r/spotify/kafka/ 
   ```
   docker run -p 2181:2181 -p 9092:9092 --env ADVERTISED_HOST=localhost --env ADVERTISED_PORT=9092 spotify/kafka
   ``` 
   For a compreensive list of options for setting up the Data Index Kafka consumer please visit: [configuring-the-kafka-connector](https://quarkus.io/guides/kafka-guide#configuring-the-kafka-connector) and [Kafka consumer configuration](https://kafka.apache.org/documentation/#consumerconfigs)
   
### Running

As a Quarkus based application, running the service for development takes full benefit of the dev mode support with live code reloading.
For that simply run:
```
mvn clean compile quarkus:dev
```

Once the service is up and running, it will start consuming messages from the  Kafka topic named: `kogito-processinstances-events`.
For more details on how to enable a Kogito runtime to produce events, please visit [publishing-events](https://github.com/kiegroup/kogito-runtimes/wiki/Configuration#publishing-events).

When running on dev mode, the [GraphiQL](https://github.com/graphql/graphiql) UI is available, on `http://localhost:8180/`, which allows exploring and querying the available data model. Alternatively, it is also possible to use a GraphQL client API to communicate with the exposed endpoint at `http://localhost:8180/graphql`.

In there you can explore the current types available using the Docs section on the top right and execute queries on the model.
Some examples:

#### Querying the technical cache
```graphql
{
  ProcessInstances {
    id
    processId
    state
    parentProcessInstanceId
    rootProcessId
    rootProcessInstanceId
    variables
    nodes {
      id
      name
      type
    }
  }
}
```  

##### Filtering 
The provided GraphQL schema also allows for further filtering of the results. A filter attribute is optional and allows multiple combinations. A few examples:

```graphql
{
  ProcessInstances(filter: {state: ACTIVE, limit: 10, processId: "travels_sub"}) {
    id
    processId
    state
    variables
  }
}
``` 

```graphql
{
  ProcessInstances(filter: {id: ["abb9f626-8a54-444d-943a-25b969a2cd1c", "446fcdb8-aa79-4044-baf9-6c471e0fbe1b"]}) {
    id
    processId
    state
    variables
  }
}
``` 

#### Querying the domain cache
Assuming a Travels model is deployed
```graphql
{
  Travels {
    visaApplication {
      duration
    }
    flight {
      flightNumber
      gate
    }
    hotel {
      name
      address {
        city
        country
      }
    }
    traveller {
      firstName
      lastName
      nationality
      email
    }
  }
}
``` 
##### Filtering
Filtering the domain-specific cache is, at this stage, open to a simple string, which is based in Infinispan query API. Please note, that we do plan to replace this for a GraphQL typed search, similar to what is provided by the technical cache. This open search allows for some interesting searches such as:

* List all Travels for travellers that first name starts with `ma`.
```graphql
{
  Travels(query: "from org.acme.travels.travels.Travels t where t.traveller.firstName:'ma*'") {
    flight {
      flightNumber
      arrival
      departure
    }
    hotel {
      name
    }
    traveller {
      firstName
    }
    processInstances {
      id
      processId
      rootProcessId
      rootProcessInstanceId
      parentProcessInstanceId
    }
  }
}
``` 
* List the flight details related to a specific process instance.
```graphql
{
  Travels(query: "from org.acme.travels.travels.Travels t where t.processInstances.id:'abb9f626-8a54-444d-943a-25b969a2cd1c'") {
    flight {
      flightNumber
      arrival
      departure
    }
  }
}
``` 

#### Loading proto files from the file system
To bootstrap the service using a set of proto files from a folder, simply pass the following property `kogito.protobuf.folder` and any .proto file contained in the folder will automatically be loaded when the application starts.
You can also set up the service to reload/load any changes to files during the service execution by setting `kogito.protobuf.watch=true`.

```
mvn clean compile quarkus:dev -Dkogito.protobuf.folder=/home/git/kogito-runtimes/data-index/data-index-service/src/test/resources -Dkogito.protobuf.watch=true
```

### Building

To generate the final build artifact, sim run:

```
mvn clean install
```

The Maven build will generate a uber jar at `target/data-index-service-${version}-runner.jar`, that can be executed via:
```
java -jar target/data-index-service-8.0.0-SNAPSHOT-runner.jar
```
 
### Infinispan Indexing

Infinispan embeds a Lucene engine in order to provide indexing of models in the cache. To be able to hint the engine about which attributes should be indexed,
it needs to annotate the proto file attributes using Hibernate Search annotations  `@Indexed` and `@Field`.
For more details, visit [indexing_of_protobuf_encoded_entries](https://infinispan.org/docs/dev/user_guide/user_guide.html#indexing_of_protobuf_encoded_entries).  

Sample indexed model:
```
/* @Indexed */
message ProcessInstanceMeta {
    /* @Field(store = Store.YES, analyze = Analyze.YES, analyzer = @Analyzer(definition = "keyword")) */
    optional string id = 1;
}
```

