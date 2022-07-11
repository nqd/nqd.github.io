---
layout: post
title: Don't use Faktory if you need high reliability
description: Don't use Faktory if you need high reliability
published: true
---

Just in case one of my projects needs a background job, and my company is using 
[Faktory](https://github.com/contribsys/faktory) here and there, I spend some time studying this background job engine.

Faktory is a new product of the same company that created 
[sidekiq](https://sidekiq.org/), a famous background job in the Ruby community.
I have never got a chance to work with Ruby as well as sidekiq, I was so excited to look at Faktory.

Well, my conclusion is that though Faktory is a nice little server, if you cannot tolerate losing the job, you should not use this.

## Faktory background

Faktory runs in the client, server model. There is a Faktory server and Faktory clients. Clients are part of your programs which generate jobs and consume them.

![Faktory-Client-Server](/images/2022-07-11-dont-use-faktory-seriously/faktory-model.drawio.svg)
{:width="800px" :class="img-responsive"}

Picture: Faktory runs in the client-server model.

Communication between Faktory clients and the server is via a TCP connection. Right, using a bare-bone TCP connection with text-based message. Take a heartbeat message, for example, the client must send a BEAT every N seconds as proof of liveness

```
BEAT {"wid":"4qpc2443vpvai","rss_kb":1234567}
```

Any clients that understand the protocol can work with Faktory. And that is the basics of programming languages independent. We are having a Go, Ruby as official libs. There are community libs for different language as Node, Python, Elixir.

My first impression is like well the guys are bold enough to build their message via TCP. Why don't simply use gRPC for communication? I don't have the answer right now, guessing the author loves Redis (which is the only storage for Faktory so far) so much and wants to go [the same text-based protocol](https://github.com/contribsys/faktory/wiki/Worker-Lifecycle#network-connection) like that of Redis.

The whole purpose of Faktory, as well as any background job server, is to answer the `Can I do this work later?` question. To be specific, it should be able to schedule a job in one of the following cases:
a. Run this job as soon as possible,
b. Run this job in the next 30 mins, and
c. Run this job at 9:00 AM every Monday.

As we will see later, Faktory has different APIs for those cases, and also different implementations.

Each job is stored in Faktory storage as a JSON object. This job struct could help us understand how the system works.

The following is an example of the Faktory [Jobs struct in Go](https://github.com/contribsys/faktory/blob/main/client/job.go#L33)

```
{
  "jid": "123861239abnadsa",
  "jobtype": "some-type-name",
  "queue": "some-queue-name", // default is default
  "args": [1, 2, "hello"],
  "reserve_for": 600, // optional, min 60 sec
  "at": "2022-02-20T15:30:17.111222333Z", // optional
  "retry": 3, // optional, default 25, exponential backoff
  "custom": {
    "locale": "fr",
    "user_id": 1234567,
    "request_id": "5359948e-6475-47cd-b3bb-3903002a28ca"
  }
}
```

When a worker fails to process a job, the server waits for `15 + count ^ 4 + (rand(30) * (count + 1))` before retrying. After retrying for `retry` times (3 in the above example), the server moves the job to a Dead Set which is similar to a dead letter queue in traditional queueing systems.

## What could go wrong?

Faktory is a server-client, language-independent job processing system. Some benchmark shows it could handle up to 1000s jobs/second/node. What could Faktory go wrong and in which case?

### One instance of Faktory server

Much to my surprise, we could only run one instance of Faktory server. Yes, there is no redundancy, like that of RabbitMQ or Kafka.

Faktory server is trully SPOF.

### Redis as storage

Redis is very fast, reliable, and supports rich data structures that are suitable for scheduling purposes. Not only Faktory take advantage of Redis, but many different systems like [Bull](https://github.com/OptimalBits/bull), [Kue](https://github.com/Automattic/kue).

Faktory uses a local copy of Redis to maintain and persist job data.

For the OSS version, we cannot expose it via TCP meaning we cannot replicate the data to a different host. In different words, we will lose entered data when the Faktory server storage is corrupted.

As the old wise saying Something can go wrong will go wrong. Local Redis in the OSS version is another SPOF.

For the Enterprise version, we can configure to open a TCP port for [replication purpose](https://github.com/contribsys/faktory/wiki/Ent-Redis-Gateway#configuring-a-replica) or connect the server to a [remote Redis cluster](https://github.com/contribsys/faktory/wiki/Ent-Remote-Redis). In both cases, Redis primary node copies data to replica noes in an async way, a.k.a the job could be lost if the primary node dies before syncing to replicas.

### Moving btw Redis queues are not atomic

For the jobs that are set ASAP, they are put in a Redis FIFO queue with LPush and RPop commands. You got the ideas.

For the jobs to run later (or at a specific time in the future), they are put in a Redis ZSET queue (sorted set). There is one single thread that manually polls the job from this ZSET queue to the normal FIFO queue for execution. Does the move actions (delete and add) atomic? No. Look at the [enqueue function](https://github.com/contribsys/faktory/blob/3d23fca667d9d459ead0a445cd9b8ab45b25cb08/storage/redis.go#L392), we see that if the program crash just right after removing from ZSET and before adding to FIFO queue, we lost the job.

There is a chance that we lost messages when Faktory code itself crashes.

Conclusion: We may not want to use Faktory when we cannot tolerate losing jobs. Faktory is not reliable, it is a SPOF. 