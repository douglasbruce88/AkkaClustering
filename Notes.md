# Building Distributed Systems with Akka.NET clustering

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
- Then switch providers in the HOCON file, and enable Helios as the transport protocol (a loose port of Netty from Java).

## Interacting with the Cluster
