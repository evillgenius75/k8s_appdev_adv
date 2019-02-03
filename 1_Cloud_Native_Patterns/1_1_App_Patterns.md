# Application Patterns for Containers in Kubernetes

## Sidecar Pattern
Sidecar design pattern is a form of single-node, multiple containers application patterns. Sidecar pattern advocates the usage of additional container for extending or enhancing the main container. Let's use an example of a web-server container that is based on legacy code and orginally served non-secure HTTP traffic. To add SSL capabilities a sidecar container can be added to extend the functionality of the original service. Some of the benefits of this pattern include the following:
* Separation of concerns: Let each container take care of their core concern. Web server container would be serving web pages while sidecar container would be processing server logs. This would be helpful during issues resolution stages when issues related to one concern should not conflict with other concern.
* Single responsibility principle: A container has primarily got one reason for change. In other words, each container do one thing well. Based on this principle, one can have separate teams work on different containers as their concerns are segregated well enough.
* Cohesiveness/Reusability: With sidecar containers for processing logs, this container can as well be reused at other places in the application.

## Ambassador Pattern
Ambassador pattern is another form of single-node, multiple containers application patterns. This pattern advocates usage of additional containers as proxies to the external group/cluster of servers. The primary goal is to simplify the access of external servers using the ambassador container. This container can be grouped as a logical atomic unit such that the application container can invoke this ambassador/proxy container which can, then, invoke the cluster of servers. The following is primary advantage of using ambassador container:

* Let ambassador container takes care of connecting to cluster of servers; Application container connects the ambassador container using “localhost” paradigm.
* Above enables using standalone server (and not cluster of servers) in the development environment. Application container opens the connection to the server on “localhost” and find the proxy without any service discovery.

## Adapter Pattern
Adapter pattern is yet another form of single-node, multiple containers application patterns. It advocates usage of additional containers to present a simplified, homogenized view of an application running within a container. These containers can be called as adapter container. The main container communicates with the adapter container through localhost or a shared local volume. This is unlike ambassador container which simply has access to external servers.

A concrete example of the adapter pattern is adapters that ensure all containers in a system have the same monitoring interface. 