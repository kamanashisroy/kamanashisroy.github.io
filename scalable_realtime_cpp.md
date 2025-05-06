
## Real-time cloud architecture with thread affinity.

Recently I worked on cloud-native CPP application where we developed real-time services. These services are running on distributed nodes and highly scalable. 

Let us discuss my work by dividing it into topics on these keywords.

### Cloud-native service

These services are running in cloud servers(VMs) and communicating with each other. 


#### Communication

Talking about communication, there are open-standards for 3GPP, SIP and WebRTC. That being said, we can develop a persistent service that has **ability to change** using,

- open standard like ASN.1.
- gRPC/proto that generates file in different languages.
- Thrift

If we think **deeply**, the way we allow things change or make them **soft** is making each of these members in messages **optional**.

With that, I would caution that I do not love the gRPC generated C code . I think modern C++ 17 has better ability , we can even do C++ ORM(object-relational-mapping) now a days.


### Real-time nature

We can allow thousands of **objects** in CPP in some array/map or under some kind of object-tree. We can also use ORM to write them in memory or SQLite/DB for persistence.

Now to process these objects in a work-flow, in CPP we have **state-machines** that process incoming messages and send/post messages to other services.


#### Event loop and reactors

We call these event-loops, because in one end we are receiving messages for-ever in a loop.
Note that other than this message-queue synchronizations, these workers are totally independent of each other.

![Event Loop](EventLoopWithContext.svg)

#### Event-driven

Our state-machines change states based on events/messages . 

#### How is it realtime?

- It is realtime because these workers do not wait for each-other.
    - That means whenever there is message in *message-queue*, it handles them immediately.
- There is no *context-switch* between workers because a group of workers reside in same thread.
- And a single thread has affinity to same processor, so the it is **cache coherent**.
- The real-IO using TCP/IP are non-blocking.

We can also call these event-loops as [reactors](https://en.wikipedia.org/wiki/Reactor_pattern).

Let us see these multi-reactor per-thread layout.

![Thread layout](RealTimeCloudArchitectureCpp.svg)

This layout gives us good vertical scalablity as the number of thread increases.

### How is it better than go-routinge.

Go-routines also called grean-threads are good for workflow that has multiple-tasks done one after another.

For example, I remember in asterisk-PBX we had lua dialplan. These dialplans will prompt user for an extension and take action interactively. These lua dialplans were written in lua-coroutines. 

Go is similar, but different than lua in a sense that it is compiled. Now to configure such workflow (or dialplan) we can write some JSON data and we can automatically load them as go object-tree.

So like the CPP diagram, we have the object-tree and we have an workflow. Now to make it better, we also have a timeout. And even better we can run multiple *tasks* in parallel with some fork and join pattern without using any state-machine.

TODO examples in go and C++.


### How to scale?

Now these reactor modules are a group of services with coherence. For scalability or bottle-neck we can move these reactor-modes into different nodes based on our need.




