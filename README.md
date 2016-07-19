# Building Distributed Systems with Akka.NET clustering

This file contains course notes

## Course Overview

- Scalability and fault tolerance
- Seed nodes and gossip
- How to arrange your system
- Step-by-step examples

## Introducing Clusters and What They Can Do for You

- All we need to do is bring extra nodes online and they automatically form a cluster helping us scale.
- We cannot vertically scale forever, Akka lets us add more machines to horizontally scale.
- Anything that can go wrong, will go wrong. If a node goes down, the other nodes will decide that it is down and not send anything to it. If it comes back online, the other nodes will welcome it back into the cluster.
- There is no single point of failure in Akka. Contrast this with a message bus architecture where if the bus goes down, the whole thing does.

## Bringing Systems Together to Form Clusters

- If we have a canonical Akka.NET app which simply creates an actor system form a blank HOCON file.
 - First add the Akka.Cluster NuGet package. It depends on Akka.Remote, which does the heavy lifting.
 - Then switch providers in the HOCON file, and enable Helios as the transport protocol (a loose port of Netty from Java). Need to set the hostname otherwise it will default to 0.0.0.0 which binds to all addresses on your machine, port 0 tells the OS to assign any free port.


- A seed node is a node with a well-known address that is persistent and that all other nodes can connect to.
 - This is a single point of failure, but we can have multiple seed nodes.
 - Any actor system can be a seed node but it is recommended to have a dedicated one.
 - When a seed node goes down the cluster will continue to work, but new nodes cannot join.
- Petabridge have created a lightweight dedicated seed node called [Lighthouse](https://github.com/petabridge/lighthouse).


- To finish enabling clustering we add the cluster section to our config.
 - Whilst Helios supports UDP, Akka.NET it only supports Helios in TCP node
 - All nodes that form a cluster must all have the same actor system name


- To set up the cluster, download Lighthouse and run it as a console app.
 - Lighthouse also integrates with Topshelf to allow deployment as a windows service.
 - We need to change the actor system name and verify the listening port.
 - We need to include ourself as a listed seed node. Lighthouse will do this if we forget but it is good practice.


- **NOTE:** I had several problems with HOCON spelling mistakes and capitalisation here.

## Interacting with the Cluster
