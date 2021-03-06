# Rachis - RavenDB's Raft Implementation

Definition of Rachis: Spinal column, also the distal part of the shaft of a feather that bears the web.   
![Figure 1. Clustering. Rachis.](images/cluster-rachis.png)

Rachis is the name we picked for RavenDB's Raft implementation. Raft is an easy-to-understand consensus protocol. 
You can read all about it [here](https://raft.github.io/), and there's also a nice visualization.   

###Why Raft?   
Raft is a distributed consensus protocol. It allows you to reach an order set of operations across your entire cluster. 
This means that you can apply a set of operations on a state machine, and have the same final state machine in all 
nodes in the cluster. It is also drastically simpler to understand than Paxos, which is the more known alternative.   

###What is Rachis?   
It is a Raft implementation. To be rather more exact, it is a Raft implementation with the following features:   
*The ability to manage a distributed set of state machine and to reliably commit updates to said state machines.   
*Dynamic topology (nodes can join and leave the cluster on the fly, including state sync).   
*Large state machines (snapshots, efficient transfers, non voting members).   
*ACID local log using Voron Storage Engine.   
*Support for in memory and persistent state machines.   
*Support for voting & non voting members.   
*A lot of small tweaks for best behavior in strange situations (forced step down, leader timeout and many more).   

###Where can I look at it?   
In the [RavenDB repository](https://github.com/ayende/ravendb). Note that to get it working, you'll have to pull 
Voron Storage Engine as well.   

###How does it work?
You can read the [Raft paper](http://web.stanford.edu/~ouster/cgi-bin/papers/raft-atc14) and the full thesis, of 
course, but there are some subtleties, so it is worth going into a bit more detail.   

Clusters are typically composed of odd number of servers (3,5 or 7), which can communicate freely with one another. 
The startup process for a cluster requires us to designate a single server as the seed. This is the only server 
that can become the cluster leader during the cluster bootstrap process. It is done to make sure that during startup, 
before we had the chance to tell the servers about the cluster topology, they won't consider themselves as single 
node clusters and start accepting requests before we add them to the correct cluster.   

Only the seed server will become the leader and the others will wait for instructions. We can then let the seed 
server know about the other nodes in the cluster. It will initiate a join operation which will reach to the other 
nodes and setup the appropriate cluster topology. At that point, all the other servers are on equal footing, and 
there is no longer any meaningful distinction between them. The notion of a seed node is relevant **only** for 
cluster bootstrap, once that is done, all servers have the same configuration, and there is no difference between 
them.   

###Dynamically adding and removing nodes from the cluster   
Removing a node from the cluster is a simple process. All we need to do is update the cluster topology, and we are 
done. The removed server will get a notification to let it know that it has been disconnected from the cluster, 
and will move itself to a passive state (note that it's okay if it doesn't get the notification, we are just being 
nice about it).   

The process of adding a server is a bit more complex. Not only are we having to add a new node, we also need to make 
sure that it has the same state as all the other nodes in the cluster. In order to do that, we handle it in multiple 
stages. A node added to the cluster can be in one of three states: Voting (full member of the cluster, able to 
become a leader), Non-Voting (just listening to what is going on, can't be a leader) and Promotable (getting up to 
speed with the cluster state).   

Non voting members are a unique case, they are there to enable some advanced scenarios (cross data center 
communication, as we currently envision it).   

The promotable category is a lot more interesting. Adding a node to an existing cluster can be a long process, 
especially if we are managing a lot of data. In order to handle that, we're adding a server to the promotable category, 
in which case we are starting to send it the state - it needs to catch up with the rest of the cluster. Once it has 
caught up with the cluster (it has all the committed entries in the cluster), we will automatically move it to the 
voting category.   

Note that it is fine for crashes to happen throughout this process. The cluster leader can crash during this, and 
we'll recover and handle this properly.   

###Normal operations
During normal operations, there is going to be a leader that is going to be accepting all the requests for the cluster, 
and handle committing them cluster wide. During those operations, you can spread reads across members in the cluster, 
for better performance.    

###Low probability scenario
If a majority of the cluster is unable to connect to the leader node, a new node will be selected as the leader. 
The old leader will notice that it can't talk to the majority of the nodes and will step down.   
However, there is a short period of time in which the leader is unable to talk to the majority of the cluster, 
but is yet unaware of that. In that period of time, it may accept writes (since it assumes it is the leader). 
Concurrently, the new leader is also accepting writes, and those writes may conflict.   
This is a low probability scenario, and it is already handled via RavenDB Replication Conflicts mechanism. 

## Related articles

- [Server: Scaling-Out: Clustering: Overview](./clustering-overview)
- [Studio: Manage Your Server : Clustering](../../../studio/management/cluster)   
