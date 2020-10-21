---
layout: default
title: Events Executor
abstract:
  This article describes the design of an events based executor as an alternative to the waitsets approach.
published: true
author: '[Mauro Passerino - iRobot](https://github.com/mauropasse) Lenny Story - iRobot [Alberto Soragna - iRobot](https://github.com/alsora)'
---

{:toc}

# {{ page.title }}

<div class="abstract" markdown="1">
{{ page.abstract }}
</div>

Original Author: {{ page.author }}

## Motivations

Many articles have discussed the CPU overhead of the default ROS 2 executor the `SingleThreadedExecutor` and of its improved version the `StaticSingleThreadedExecutor`.
 - [Reducing ROS 2 CPU overhead by simplifying the ROS 2 layers](https://discourse.ros.org/t/reducing-ros-2-cpu-overhead-by-simplifying-the-ros-2-layers/13808)
 - [SingleThreadedExecutor creates a high CPU overhead in ROS 2](https://discourse.ros.org/t/singlethreadedexecutor-creates-a-high-cpu-overhead-in-ros-2/10077)
 - [ROS 2 Middleware Change Proposal](https://discourse.ros.org/t/ros2-middleware-change-proposal/15863)

We can identify some major contributors to this overhead:
 - The cost of modifying and updating waitsets.
   Currently this operation happens multiple times for every iteration of the executor, even if the majority of robotics systems are mostly static, i.e. the number of ROS 2 entities is fixed and known at startup.
   A detailed walk-through of the waitset management process can be found [here](https://discourse.ros.org/t/ros2-middleware-change-proposal/15863/6).
 - Inefficiency of the timers management.
   This is currently done in the rcl layer, where at each iteration the full list of timers associated with an executor is checked twice.
 - The presence of several abstraction layers that do not simply forward data, but rather perform non trivial operations, between the application and the underlying middleware.
 
The CPU overhead of ROS 2 with respect to other pub/sub systems (as the same DDS middlewares that it uses under the hood), makes it difficult to integrate ROS 2 on embedded platforms with constrained resources, as a not negligible amount of CPU is needed just for the executor to identify entities with work to do.
 
## Requirements
 
The following requirements have been taken account for the design of a new ROS 2 executor:
 - Good performance: the executor shall not be the CPU bottleneck of a ROS 2 system, thus allowing to obtain performance comparable to the ones of the underlying middleware that is being used.
 - Scalability: The executor must perform well independently of the system size, e.g. the latency for processing a fixed amount of messages must be independent from the number of entities.
   This can flow into the "You don't pay for what you don't use" paradigm.
 - Extensibility: The executor should set a base from which it is straight-forward to add more functionalities without impacting performances or requiring heavy modifications of the ROS2 core libraries.
 - Ordering of events: The executor should process events in the same order they have occurred.
 - High precision and accurate timers: The executor should provide mechanisms to ensure that timers are executed with the minimum possible latency.
 
## The Events Executor

In order to address the aforementioned requirements, we designed the so-called `EventsExecutor`.
The `EventsExecutor` inherits from the base `Executor` class.
The main difference between this and the other executor implementations is that the `EventsExecutor` does not use the concept of a waitset.
The `EventsExecutor` is characterized by two main components, i.e. the `EventsQueue` and the `TimersManager`.

### Brief Overview

In order to execute generic ROS 2 entities (i.e. everything except timers), the `EventsExecutor` relies on an event queue.
Whenever a ROS 2 entity that is associated to the executor has work to do, it will push an event into the executor's queue.
Then the executor can process these events in a FIFO manner, without need for expensive entities look-ups.
Processing an event results in different operations depending on the entity that generated it.

Timers are executed in a separate task by a timers manager, where they are kept in a priority queue sorted according to their expiration time. The task then has to monitor only the first element of the queue and can execute its callback as soon as it expires.
 

### EventsQueue

For what concerns entities other than ROS 2 timers, the `EventsExecutor` requires that as soon as these entities have some work to do, they push an event into the executor's `EventsQueue`.
Note that events are produced in the RMW layer, i.e. where the underlying middleware notifies the ROS 2 entity that there is new work to do, on the other hand, events are executed in the rclcpp layer by the `EventsExecutor`.
The event data structure that is  pushed into the `EventsQueue`, must contain all what is needed for the executor in order to be able to process that particular event.
This includes the type of the entity that generated the event and an handle to its corresponding rclcpp object.

Considering the entities currently available in ROS 2, the content of the event data-structure can be any of the following:
 - `ExecutorEventType::SUBSCRIPTION_EVENT` and an identifier for a `rclcpp::SubscriptionBase` object.
 - `ExecutorEventType::SERVICE_EVENT` and an identifier for a `rclcpp::ServiceBase` object.
 - `ExecutorEventType::CLIENT_EVENT` and an identifier for a `rclcpp::ClientBase` object.
 - `ExecutorEventType::WAITABLE_EVENT` and an identifier for a `rclcpp::Waitable` object.
     
Let's consider as an example how ROS 2 subscription are handled in the current implementation of the `EventsExecutor`.
The underlying middleware will notify the rmw_subscription object whenever a new message arrives. The rmw_subscription object will have to push an event data-structure into the `EventsExecutor`'s queue.
This event data-structure will contain the `ExecutorEventType::SUBSCRIPTION_EVENT` value (to denote that this event comes from a subscription) and a raw pointer to the `rclcpp::SubscriptionBase` object which will have to execute it.    

It is responsibility of the `EventsExecutor` to setup (and eventually also to clean up) entities such that they can push such events.

An application can add ROS 2 nodes or callback groups to an `EventsExecutor` using the corresponding APIs (`add_node()` and `add_callback_group()`).
Added nodes and callback groups are inspected in order to find all the existing ROS 2 entities that have to be associated with the executor.
Whenever new entities are associated to the `EventsExecutor` an initialization procedure takes place, with the purpose of provisioning those entities with a way for pushing events as soon as they have some work to do.
This consists in passing the following objects from the rclcpp layer to the rmw layer:
 - The `EventsExecutorCallback`, a function pointer that can be used to push events.
 - The identifier of the rclcpp entity.
 - A pointer to the EventsExecutor itself.

The initialization procedure presents some small variations depending on the type of the entity.

##### Client/Server/Subscription

These entities have a 1-to-1 correspondence between objects in the rclcpp and the rmw layer.
The initialization is straight forward and consists in having the `EventsExecutor` to pass the aforementioned objects to the rclcpp subscription, which then will forward them to rcl and finally to rmw.

##### Waitable

`Waitables` are a concept that exists only in the rclcpp layer.
Each class that inherits from `Waitable`, will have to define its own function for forwarding objects to the rmw layer.
For example, a `SubscriptionIntraProcess` relies on a guard condition in the rmw layer, so it will forward to it the tools for pushing events.
On the other hand, a `QOSEventHandler`, will forward the objects to the underlying rmw QoSEvent implementation.

Note that `Waitables` can be used as a way to implement generic custom events, as it will be described in the next sections.

##### Timer

Timers are not required to push events into the `EventsExecutor`'s `EventsQueue`, so new timer entities are simply redirected to the `TimersManager` object.

The aforementioned entities are initialized as soon as their node or callback group is added to the `EventsExecutor`.
However, there are also other entities which can push events and that are not related to any node.

##### EventsExecutorNotifyWaitable

The `EventsExecutorNotifyWaitable` derives from `Waitable` and is used by the `EventsExecutor` to receive an event whenever any of the context interrupt guard condition (e.g. ctrl-c) or the own executor interrupt guard condition are triggered.
This can be achieved by having this class to implement a function that forwards the tools for pushing events to two distinct rmw guard condition objects.

##### EventsExecutorEntitiesCollector

The `EventsExecutorEntitiesCollector` derives from `Waitable` and it is used by the `EventsExecutor` to setup entities that are added to nodes or callback groups while it is spinning.
This class will forward the tools for pushing events to the notify guard condition of each of the nodes associated with the executor.


### TimersManager

The `TimersManager` is a class that allows to monitor and execute multiple timers.
It should respect the following specification:
 - Timers are dynamic and they can be added or removed while the `TimersManager` is running.
 - The `TimersManager` should support both periodic as well as one-shot timers (with the second currently not available in ROS 2).
 - The `TimersManager` need to support all the modes of a ROS 2 executor (i.e. `spin()`, `spin_some()`, etc).
 - Users should be able to extend the `TimersManager` to improve its performance according to their specific use-case.
 
The current implementation of the `TimersManager` implements the above design in the following way.

In order to use the `TimersManager` within a blocking `spin()` call, a `TimersManager` task is started. This task will continuously execute timers and sleep until the next timer is ready as long as the executor is still spinning.
This is performed through the following loop:
 1. Get the time before the first timer expires.
 2. Sleep for the required amount of time.
 3. Execute all ready timers.
 4. Repeat

Creating a new task provides the most efficient way for offloading timers and it ensures that timers are executed in a timely manner, without having to wait for other entities to be processed.
However, this may not be compatible with the non-blocking variants of `spin()`.
To handle them, the following APIs are exposed:
 - `get_head_timeout()` which returns the time before the first timer expires.
 - `execute_head_timer()` which execute the first timer if it's ready.
 - `execute_ready_timers()` which execute all the ready timers if any.

By composing these APIs, it is possible to implement the non-blocking variants of `spin()` without the need of additional threads for monitoring the timers.
This has the advantage to give to the executor a more fine-grained control on which and how many timers are executed and saves the overhead of continuously starting and killing a thread if the `spin()` variants is called within a loop.

The current implementation of the `TimersManager` uses an heap priority queue to keep the timers sorted.
Whenever a timer is added or removed from the `TimersManager`, the queue is heapified again.
Whenever a timer is executed, its expire time will be automatically updated, so it's necessary to provide an efficient operation for updating the root element, while the rest of the queue is still a valid heap.
This is currently done using the pop_heap function from std library, that has 2 log(n) complexity

Moving the timers management into its own class allows to isolate this feature from the rest of the system, allowing to develop it independently. 
Moreover, by separating timers execution from sorting, it is possible to extend the `TImersManager` object to use different algorithms, such as timer wheels, or to take advantage of the knowledge of which OS will run your application, by using low level OS timers, without having to modify the executor.


### Events Execution

The simplest usage of the `EventsExecutor` consists in calling the `spin()` method which results in having the `EventsExecutor` to start the `TimersManager` task, while at the same time continuously waiting for events to be pushed into the `EventsQueue`.
Events are executed in a FIFO manner, following the order in which they are pushed into the `EventsQueue`.

Executing an event is done by calling the corresponding API on the rclcpp object identified by the event.
Let's consider a ROS 2 subscription as an example.
The event will contain the `ExecutorEventType::SUBSCRIPTION_EVENT` value, so the executor will know that it has to get the `rclcpp::SubscriptionBase` object that generated this event and call the `take_and_do_error_handling()` API.

`Waitables` implement their own `execute()` API, which can be called directly.

The `EventsExecutor` can allow entities to not be blocked while events are being processed by implementing the `EventsQueue` through two separate queues, one for storage and accumulation of events and the other for their execution.

In order to implement the non-blocking variants of `spin()` without requiring the creation of an additional task, which in these cases it would come with complex synchronization requirements, it is sufficient to query the `TimersManager` for the time before the first timer expires and then use this value as a timeout for the conditional wait for new events.


### Cleanup and ownership

The `EventsExecutor` should not keep ownership of the nodes and of the entities that are associated to it.

TODO DESCRIBE ENTITIES DESTRUCTION
