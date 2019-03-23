---
layout: post
title: RPC and MQTT
published: true
---

IoT protocol nowadays is much about MQTT. Let's face it. AWS IoT is about MQTT (over TCP and WebSocket). Google Cloud IoT is also MQTT and HTTP. Azure IoT is MQTT, HTTP, and well AMQP.

The good is that the IoT protocol know is converging. The frontend is MQTT with a lot of library for Linux, Windows, and RTOS. The backend may have more choices AMQP, Kafka, or other queues.

The bad is that MQTT may not be the best protocol for IoT at all. Given this common case in IoT: you control light. This is what will happen with RPC:

Handle:

    rpc.provide(name: string, callback)

Request:

    rpc.make(name: string, arg{}, return: function())

The RPC would travel across the cloud if they are not on the same local network

    requester --- { cloud } --- handler

A good RPC would return an error if there is no handler, or the handler just went offline. Other factors that need to take into account is timeout, load balancing.

# RPC with MQTT

Back to MQTT, how could we archive it? With v3.1.1, there is no request-response mechanism. We have to publish a topic and wait to receive at another one.

Request:

    client.publish(requestTopic, arg{})
    client.subscribe(responseTopic, callback)

Handle:

    client.subcribe(requestTopic, arg{}, callback)

Here come a few problems:

- requester and handler have to prior-agree on request, response topic. Or the response topic is included in the request argument. One of our projects uses the later approaching.
- the timeout is at the application layer.
- no load balancing. Well, in the "turn on/off a light example", we surely do not need a load balancing. An RPC from device to cloud will make more sense. User case: a Zwave gateway makes a call to cloud handler to report a new thermostat added to Zwave network. For high availability, more than one workers will be at the task. With MQTT, all workers handle the same request, hence duplicate the resource.

With new MQTT v5, the problem partly solved. v5 define request/response method at the MQTT level. Here is the code at requester:

    client.publish(requestTopic, arg{expectedResponseTopic, ...}})
    client.subscribe(expectedResponseTopic, callback)

v5 adding one thing: expected response topic embedded in the request. Again we are missing the load balancing and timeout.

TL;DR: If we really need RPC, just do not use MQTT.