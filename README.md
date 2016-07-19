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


- **NOTE:** I had several problems with HOCON spelling mistakes and capitalisation here. In the end I pasted their completed config in and it worked, still got no idea what had changed.


- Gossip: special messages that spread throughout the cluster and spread the state of other nodes in the system, eventually giving all the nodes a consistent state (zombie apocalypse!). They contain their view of all other nodes at any one time.


- Cluster lifecycle:
 - We have a seed node A and other node B.
 - C wants to join. It knows about A and sends it a message asking it to join. It is knew about multiple seeds it would send it to all.
 - A responds with an acknowledgement, then C will send a message to the fastest responded to say it is joining.
 - The next gossip message from A will contain information about C, and so B will know that C is 'Joining.
 - Once all nodes know that C is joining, A will promote it to 'UP' and spread this via gossip.
 - Leaving is similar, C would declare itself as 'Leaving' via gossip and it is promoted to 'Exiting', at which point is shuts down, and C is removed from gossip.

## Interacting with the Cluster

- Look at the original Globomantics solution.The API is the top-level actor. It can handle `LoginEvent` and `VideoWatchedEvent` messages.
- The `VideoWatchedEvent` message is triggered when a user watches a video and it contains the `videoID` and `userID`. It sends this to the `VideosWatchedStore`, a persistent actor that uses `Akka.Persistence` which persists the state to disk.
 - This actor sits behind a pull router which contains multiple stores. Event messages are broadcast to all stores, requests are round-robined.
 - This means is one actor crashes, the others than pick up the slack until it recovers.
- When it received a `LoginEvent`, it spawns a dedicated `RecommendationWorkflow` actor to do the recommender algorithm. This algorithm requires details of videos we have seen which come from the `VideosWatchEventStore` store, and details of unwatched ones which we get from the `VideosDetailsFetcher`.
 - The fetcher requests information from a remote API, and is gated to only allow one request at a time to stop us swamping our network connection.
 - We can control how many concurrent requests we allow by controlling the size of the `VideoRequests` pool, and only start the job when we have at least one routee for each type of router.
 - The actor is terminated once its work is finished.
- The important bit is the message flow, sending messages to pools of watched and unwatched video fetchers. These pools can be scaled independently.


- Now let's convert this to clustering in the same way in the previous module.
- We also modify the HOCON further, specifically the `router` config sections and their `cluster` subsections which make them cluster-aware.
- The `roles` attribute means we only deploy to modes with the `API` role, which stops us deploying actors to our `Lighthouse` seed node.
- If we fire up two nodes and then a client, we can see how the messages are processed.


- At this stage things have become quite complicated, which is where microservices come in.
- Looking at the solution, we can split out the core API and workflow, roles that are stateless (fetchers) and those with state (stores) into separate projects.
 - These can run as separate nodes in the cluster with different roles, and we can reason about individual roles rather than the system as a whole.
 - They can be deployed separately which is very useful when considering state.
 - Nodes in different roles may have different requirements e.g. internal access, persistence, so we can use different machine characteristics.
 - We can thus scale out individual roles.
- We model stateful and stateless nodes to start with no actors onto which other nodes can remote deploy actors.
- Fire up one of each service and then use the client.


- There are a few fallacies of distributed computing to help us develop with these in mind:
 - The network is not reliable. If we are waiting for a response, use a timeout.
 - Latency is not zero. If we receive a response after the timeout due to high latency, we need to be able to handle it properly and ensure the state isn't corrupted
 - Topology is not static. We can move actors from local to remote without changing code, so be aware that we don't know to where the actors are deployed.
 - Transport cost is not zero. Distributing will almost definitely slow the system under low load scenarios due to the network overhead.
 - The network is not homogeneous. Some remote nodes may be in different data centres and might have difference specs. For example round robin might not work if one node is stronger than the other so we can use other routing strategies or have the actors ask for work.
