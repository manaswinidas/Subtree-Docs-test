# 0.2.0

## Release date - mid July

### Main themes of the release 

* message start and end events that allow smooth integration with Apache Kafka and possibly other messaging
* multi instance characteristic for service nodes and reusable subprocesses
* Unit of Work support to allow finer control of execution and grouping related operations
* refactor service discovery when running in Kubernetes based environments

# 0.3.0

## Release date - mid August

### Main themes of the release 

* runtime persistence based on Infinispan
* data index service initial implementation to enable management and human task centric use cases
* events for runtime based on CloudEvents - this is an integration between runtime services and data index service
* enable domain specific metrics - mainly data driven to be available for dashboards
* operator to provision SSO and inject system tokens into containers so they can interact smoothly
* open tracing for any service interaction from within a process or decision

# 0.4.0

## Release date - mid September

### Main themes of the release 

* variable tagging to allow adding labels to variables to describe their relevance
* configurable human tasks life cycle to allow control over what is needed to work on user assigned tasks
* data index service searching capabilities based on domain data
* VSCode extension to provide additional capabilities for developers - test, debug, deploy etc
* operator provision data index and runtime storage on demand
* operator discover and automatically register metrics endpoints in OpenShift provisioned Prometheus

# 0.5.0

## Release date - mid October

### Main themes of the release 

* task inbox with base functionality to work on tasks as task user and task admin
* timer and job service + timers in the process at least intermediate and boundary timers
* operator to provision timer service on demand, meaning only then runtime service uses timers and de-provision when no longer needed

# 0.6.0

## Release date - mid November

### Main themes of the release 

* management console that allows administrators of the business automation to have insights into running services and historical data
* operator to provision complete environment including sso, management console, task inbox, timer service and runtime services
* multi tenant task inbox

# 1.0.0

## Release date - mid December
