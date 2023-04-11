# Chapter 5 - Replication


## Table of Contents
1. [Mind Map](#mind-map)

## Mind Map
![mindmap](/DDIA-notes/chapter5/DDIA%20Chapter%205.jpg)

Replication means keeping a copy of the same data on multiple machines, so that we can:

- keep the data geographically close to the users worldwide
- to increase availability even with some failures
- scale out the number of machines to serve more users concurrently

## Leaders & Followers

How do we make sure data ends up on all the replicas? active/passive or master/slave replication

### Synchronous vs Asynchronous Replication
- one replica is the leader that handles all **writes*
- the leader then sends a replication log or change stream to all followers, either push or pull
    - synchronously or asynchronously - when does the leader respond ok?
    - consistency vs availability(the leader will block all writes if one follower is not responding)
    - semi-synchronous
- **reads** can go to either leader or follower

### Setting Up New Followers
- Take a consistent snapshot of the leader's data at some point in time T0
- Copy the snapshot to the new follower
- New followers then connect to the leader and request all changes since time T0 based on the replicate log
- caught up

### Handling Node Outages 
- Follower failure: each follower will keep a log of the data it has received from the leader. If the follower comes back from a temporary interruption, it can request all the changes during that time it was disconnected.
- Leader failure: Failover - One of the followers needs to be **promoted** to be the new leader. 
    - heartbeat: nodes frequently bounce messages back and forth between each other, if a node does not respond within a time range, it's considered as dead. 
    - deciding the right timeout threshold is tricky, longer timeout means a longer time to recover and a shorter timeout means unnecessary failover which is a burden to any high-load system.
    - election: the best candidate for leadership is usually the follower with the most up-to-date data.
    - re-config the system using the new leader, even if the old leader comes back (split brain). 
    - in an asynchronous replication system, it's possible the new leader might have incomplete/conflicting data and we can only discard them - think carefully because this will violate data durability. 


### Implementation of Replication Logs
- Statement-based replication: the leader logs every write request (INSERT, UPDATE, DELETE) and sends it to its followers. It can break if the request is **nondeterministic(RAND())** or if there are concurrent transactions where the order is different. The solution is the leader can replace any nondeterministic value with a fixed return value and pass it down.
- Write Ahead Log (WAL): it is an append-only sequence of *bytes* containing all writes to the database. WAL describes the data on a very **low level** - which bytes were changed in which disk blocks.
 - Logic(row-based) log replication: 

 
