---
layout: post
title: Gobench supports gRPC
description: Gobench supports gRPC
published: true
---

What is news: Gobench is supporting
[gRPC](https://github.com/gobench-io/gobench/blob/master/clients/gbGrpc/grpc.go),
both unary RPC and streaming. Now you can use Gobench to benchmark servers with
HTTP, MQTT, NATs, and gRPC.

See Gobench project at [https://github.com/gobench-io/gobench](https://github.com/gobench-io/gobench)

To test the gRPC benchmark, we will we will use the examples provided by
[grpc-go](https://github.com/grpc/grpc-go) project.

```
git clone https://github.com/grpc/grpc-go
cd grpc-go/examples
```

## Unary gRPC

Run the greeter server:

```
go run helloworld/greeter_server/main.go
```

Then create a new Gobench scenario on this [link](greeter_client/main.go). This
program generates 5 virtual users each sends `helloworld.Greeter/SayHello` RPC
in 2 minutes. The dashboard records the number of responses and the request
latency as depicted in the following picture.

![Unary-RPC](/images/2021-07-18-gobench-grpc-support/gobench-unary-rpc.png)
{:width="800px" :class="img-responsive"}

Picture: Metrics recorded for SayHello unary gRPC.

## gRPC stream

Run the route_guide streaming server:

```
go run route_guide/server/server.go
```

Then create a new Gobench scenario on this [link](./route_guide/main.go). This
program generates 5 virtual users each sends
`routeguide.RouteGuide/ListFeatures` RPC in 2 minutes. The dashboard records
number of new streaming created, latency, the number of the streaming requests
sent (which will be equal to the number of new streaming created), the streaming
request latency, and the number of the total messages received in all streams
with their latency.

![Unary-RPC](/images/2021-07-18-gobench-grpc-support/gobench-streaming-rpc.png)
{:width="800px" :class="img-responsive"}

Picture: Metrics recorded for ListFeatures gRPC streamming.