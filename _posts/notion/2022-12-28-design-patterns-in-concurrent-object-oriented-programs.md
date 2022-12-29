---
layout: post
comments: false
date: 2022-12-28
title: Design Patterns in Concurrent Object-Oriented Programs
tags:
  - concurrency
  - design-patterns

---

Concurrent Programming is not much about threads or locks, it is more about the state. In Object Oriented programming, the state is typically encapsulated as an Object. So in this context, the state means the `Object's state`, which is generally stored in `state variables` or `static fields`.  Write safe concurrent programs or thread-safe programs is more about managing access to state, more particularly access to shared, mutable state. Here shared means, it can be accessed by multiple threads. Whether an object needs to be defined as thread-safe is dependent on how the object is used rather than what the object does.


When we have a shared mutable state being accessed by multiple threads, often multiple threads `race` through the actions performed on the state and result in an unpredictable resultant state.


So the access needs to be coordinated using `synchronization`. This is often done using various OS and language-level semantics such as Locks, Conditional Variables, Mutexes, Atomic Variables, etc.


However, when we have a huge codebase with a few tens to hundreds or an unpredictable number of threads (unpredictable at compile time) accessing a shared mutable state, it is very hard to write correct synchronizing or concurrent code that does correctly what is designed to do. So, we need better abstractions that are specifically designed for programs involving shared mutable states. We need better abstractions, we need better object-oriented design patterns. This paper outlines a few of them, which were first produced in the book “Pattern Languages of Program Design” over various editions.


---


## Event Handling Patterns


Focuses on how to initiate, receive, demultiplex, dispatch, and process events in networked systems.

1. Reactor Design Pattern
2. Proactor Design Pattern

## Reactor Design Pattern


Also known as Dispatcher, Notifier. Another pattern that is very closely related to this is Proactor Pattern.


This pattern is categorized as an “Event handling Pattern”.


> 💡 “Reactor design pattern allows event-driven applications to demultiplex and dispatch service requests that are delivered to an application from one or more clients.”


This uses the “Hollywood principle” of ‘don't call us, we will call you’, where a component called reactor waits for the events to arrive synchronously, demultiplex them to associated event handlers (event handlers that are registered to handle this type of event) responsible to process these events and dispatches them to appropriate hook method on event handler.


This way, event handlers don’t call the application for events, instead, the reactor demultiplexes and calls the appropriate event handler. This inverts the flow within an application.


This is very useful for an application developer because he can focus on just writing event handlers and re-use the reactor’s demultiplexing and dispatching mechanisms.


However, the reactor pattern is relatively straightforward to program, and use and also has some practical constraints on its applicability. It doesn’t scale to a large number of simultaneous service requests, and causes congestion when there are long-running service requests. This is where `Proactor design pattern` comes into play to alleviate all these issues.


However, it is very useful to learn reactor design patterns, to better understand proactor design pattern.


Let's consider a distributed logging service, where there are multiple clients trying to record information about their status within a distributed system by sending ‘service requests’ to logging server.


The central logging server then writes the records to various output streams or devices (some database or message queue or object storage like s3


![](https://s3.us-west-2.amazonaws.com/secure.notion-static.com/0a4a3b44-fe94-4745-b0ed-69e067869891/Untitled.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45EIPT3X45%2F20221229%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20221229T181125Z&X-Amz-Expires=3600&X-Amz-Signature=d9fe7e646eb5b416391392cd5627ef14c4c222c855c8f3b99d036b96b102ffa4&X-Amz-SignedHeaders=host&x-id=GetObject)


The most intuitive way to develop a concurrent logging server is to use multiple threads that can process multiple client concurrently. This way, we can synchronously accept network connections and spawn a thread-per-connection to handle client record.


However, it has its own disadvantages

1. Threading may be inefficient and non-scalable due to context switching
2. May require complex use of concurrency semantics
3. It may be better to align the number of threads to spawn on the number of CPUs available rather than the number of clients.

So, a multi-threaded, a thread-per-connection is not a correct and efficient approach for this kind of problem.


Typically in event-driven applications, the application should be able to handle multiple service requests simultaneously, even though those requests are processed later (even serially within application).


So, below is the functional requirements we are looking to solve for this logging server.

1. Neither should not block any single event  nor exclude event sources
2. Avoid unnecessary context switching and data movement among CPUs.
3. Adding new or improved services (event handlers) should require minimal effort
4. The application code should be abstracted or free from the complexity of multi-threaded or synchronization semantics.

### Solution


For each service application offer, an event handler is introduced that processes certain types of events from certain event sources. Event handlers registers with a rector. The reactor uses a synchronous event demultiplexer to wait for an event to occur on event sources.


Upon an event, the demultiplexer notifier reactor, which synchronously dispatches the event to appropriate event handlers and the event is performed.


Key components

1. Handle
	1. Provided as OS (such as sockets), these are event sources, clients connect to these and send the events
	2. Kernel methods such as ‘accept` and ‘read’ can be used on handle to wait for an event to occur without blocking calling thread
	3. Each connected client typically gets an handle
	4. Set of handles are referred as ‘Handle Set’, we can also wait on a handle set.
2. Synchronous Event Demultiplexer
	1. A function which waits on a handle set.
	2. Kernel methods such as ‘select’ can be used to wait for an event on the handle set. This method blocks the callee thread until there is an event on the handle set.
3. Event Handler
	1. Defines an interface for processing events that occur or received on a handle. Consists of one or more hook methods
4. Concrete Event Handler
	1. Implements Event Handler, defines an application service
5. Reactor
	1. Defines an interface that allows concrete event handlers to register or remove event handlers on associated handles
	2. Runs application’s event loop
	3. Uses synchronous demultiplexer to wait for events on its handle set
	4. Dispatches an event to appropriate hook method on registered handlers to process the event

![](https://s3.us-west-2.amazonaws.com/secure.notion-static.com/8d268d3d-cb3d-4dc1-b3e1-05dd21a757d9/Untitled.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45EIPT3X45%2F20221229%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20221229T181125Z&X-Amz-Expires=3600&X-Amz-Signature=0fdcad06621e2b448ba06c70e414fcd4d191fde03b15cb59134fedc257fb8aa7&X-Amz-SignedHeaders=host&x-id=GetObject)


### Benefits

1. Separation of Concerns
	1. decouples demultiplexing, dispatching mechanisms from the application or hook method.
2. Modularity, reusability, and configurability
	1. Adding a new concrete event handler is easy and straight forward
3. Portability
	1. select, accept and read are supported by both UNIX and windows kernels.
4. Coarse-grained concurrency control
	1. Events are serialized at the event demultiplexer level, so there is no need for complicated synchronization within a concrete event handler.

### Liabilities

1. OS should support synchronous event demultiplexing on handle set, else this model doesn’t fit
2. Non-pre-emptive, Since we execute events synchronously, long-running event handlers such as blocking IO or sending image over network , etc can block the entire process and reactor might not be able to respond to events from other handlers.
3. Due to its inverted flow of control, it is hard to debug and testing.

# Proactor Design Pattern


In the applications like web servers, synchronously executing events or requests is not desired when there are long-running events or requests. If the OS supports asynchronous IO operations, it is efficient to take advantage of that. OS which support async IO operations typically provide IO completion port, which is a queue managed my OS and results of completed io operations are queue here. And threads of control can pull the results from this completion port and dispath those results/events and take next actions. Async opertions represent potentially long-duration operations that are used in implementation of services (example: A get method that reads from table on database or disk and returns as list)


However, there are numerous challenges when programming for async io operations because of seperation in time and space, where operations are invoked and completion occur.


Proactor, is a pattern that is designed to handle these kind of async io operations i.e. to efficiently demultiplex IO operations.


Key Components

1. Async Operation
2. Completion Handler
	1. interface that consists of one or more hook methods
	2. Represents set of operations avaiable for processing completion result of async operations
3. Concrete Completion Handler
	1. Implementations of Completion handler.
4. Async Operation Processor
	1. Executes async operations on a particular handle
	2. Completed result is queue to IO Completion port (queue)
	3. Often implemented by OS
5. Completion Event Queue
	1. Or IO Completion Port
	2. Buffers completed async operation results
6. Asynchronous event demultiplexer
	1. Function that waits for completion results or event to be inserted into to Completion Event Queue
7. Proactor
	1. Provides event loop
	2. Calls an asynch event demultiplexer to wait for completion events to occur
	3. Upon event and return of event demultiplexer, proactor then dipatches the result to its associated completion handler’s hook method.
8. Intiator
	1. Invokes async operations on async operation processor
	2. With each call, it associates a appropriate completion handler

![](https://s3.us-west-2.amazonaws.com/secure.notion-static.com/c6428628-75ac-430a-a36f-1f33fd9d7ad3/Untitled.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45EIPT3X45%2F20221229%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20221229T181125Z&X-Amz-Expires=3600&X-Amz-Signature=7fc7667c2389804ba0bfd10990aa3530fba554899d401900ae16037dd6cce9dc&X-Amz-SignedHeaders=host&x-id=GetObject)


### Benefits

1. Seperation of concerns
	1. decouples application-independent async mechanism from application-specific logic
	2. Example: JsonResponseCompletionHandler, can send the hashmap (result of completed async operation) as json response to the associated request
2. Portability
	1. Works on asny OS that supports async I/O. Since this logic is abstracted from application logic
3. Encapsulation of concurrency semantics
	1. Application logic is abstracted from required concurrency semantics of proactor model
4. Performance
	1. No context switching, since proactor activate threads which have a completion result events to process
5. Abstraction of application synchronization
	1. As long as concrete completion handlers dont spawn additional threads, no synchronization semantics needed in application-specific logic.

### Liabilities

1. Restricted applicability
	1. if the OS doesn’t support async IO, then this pattern cannot be applied
2. Hard to debug, test
3. Once the async operation is submitted, Initiator will be unable to control or cancel the running operation.

---


## Concurrency Patterns


Focuses on how to design for sharing resources amount multiple threads or processors.


## Active Object Design Pattern


Also know as **Concurrent Object**


Decouples method execution from the method invocation. Enhances concurrency by simplifying access to object’s method that reside in their own thread of control.


> Active object execute its method in a different thread than its clients.


Lets assume, we have a lot of threads trying to communicate with each other, in OOP, communication is typically done by invoking methods on the objects, if the invokation takes more time to execute, this should not block other threads trying to communicate to same object or the program. And synchronization access to these objects should be straightforward or abstracted. We can typically see this kind of problem in Gateway design, which routes messages from one or more suppliers to one or more consumers.


The proposed Active Object Design decouples method invocation on the object from method execution. Typically in passive objects, the method execution happens in the same thread of control of the method invocation. In Active or Concurrent objects, the method execution will occur in a seperate thread and method invocation in the client’s thread of control, so both are different threads.


![](https://s3.us-west-2.amazonaws.com/secure.notion-static.com/6626d90e-81e2-410a-a412-68c293cb9ad8/Untitled.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45EIPT3X45%2F20221229%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20221229T181125Z&X-Amz-Expires=3600&X-Amz-Signature=3d1f0d1161e3c3c541b065a8e82907edeba162dffa449fb84e5bf6d3cf882475&X-Amz-SignedHeaders=host&x-id=GetObject)

- When client program invokes a method on proxy, the proxy constructs a method request (parameters, context and some other information as a object) and inserts it into the Activation List (some kind of bounded buffer) and proxy returns a “Future” object to the client program, using which the result of the actual method execution can be fetched.
- Activation List lies in the Active Object’s Thread of control.
- Scheduler waits and fetches available method requests based on various criteria.
- A Servant is the actual implementation of Active Object (defines behaviour and state). Proxy provides a strongly typed interface to the servant, this is preferred over loosely-typed messages between threads. Proxy also defines guards when a particular method can be executed.
- Once the method request is fetched, Servant executes on its thread of control and the result is filled in the corresponding future.
- Typically, Activation list, scheduler and servant can be combine to a single object in the implementation.

### Benefits

1. Enhances application concurrency and simplifies synchronization complexity
2. Method execution order is different from invocation order
	1. this is to make sure the invocations adheres to the guards and scheduling policies of the active object
	2. thus, this decoupling improves performance and flexiblity (in the sense the client is not required to be aware of above)

### Liabilities

1. There can be performance overhead, if the objects are fine-grained, since this can result in more context switching, synchronization and data movement.
2. Hard to debug and test, due to non-determinsm of scheduler and OS thread scheduler.

# Monitor Object Design Pattern


With Active Object pattern, we have seen how to sycnhronize access to object’s method running in its own thread of control (because of the problem, which demands a thread per active object).


What if, we want multiple threads to cooperatively schedule their executions on a object, so that only one thread is executing at any given time within the object, here the method invocation and execution is done on the client’s thread.


Let’s call these objects as Monitor Objects or Monitors and any access by clients should be through synchronized. To serialize access to object’s state monitors contain monitor lock. There are one or more monitor condition (condition variables) to determine when a synchronized method can suspend or resume a execution.


Threads can schedule their execution sequences cooperatively by waiting and notifying each other via monitor conditions associated with monitor object.


Synchronized method use thread-safe method exported by monitor object (acquire, release, wait, notify)


Each monitor object contains its own monitor lock. These are generally mutex locks provided by OS. Synchronized methods use this lock to serialize method invocation, each method should acquire and release the lock before entering and before exiting the method, since the thread which acquire the lock can only release the lock, this ensure there is only thread executing any method on object at any given time.


Java uses similar concept, where every object is a monitor object. marking a method `synchronized` makes the implicity acquire and release the monitor lock associated with the object. We can also perform `wait` and `nofify` and `notifyall` on the monitor object.


![](https://s3.us-west-2.amazonaws.com/secure.notion-static.com/b3b7f29b-3a1f-47dc-9699-fcf047758c5a/Untitled.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45EIPT3X45%2F20221229%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20221229T181125Z&X-Amz-Expires=3600&X-Amz-Signature=2ab8251a37e853f020dd69b438261651bd97c29b5077178a3cf350c0b1d58137&X-Amz-SignedHeaders=host&x-id=GetObject)


### Benefits

1. Simplication of concurrency control
2. Simplification of scheduling method execution

### Liabilities

1. Nested monitor lockout
	1. similar to nested monitor lock in java
	2. two nested objects in java cannot share a monitor lock
	3. so when we call inner.wait(), inner monitor lock is released, but outer monitor lock is retained
	4. so it is now not possible to set the inner condition variable, since the only way we can access any inner is via outer and its monitor lock is acquired indefinetely.
2. Monitor object callees dont have control over the scheduling of its method invocation on the monitor object methods.
