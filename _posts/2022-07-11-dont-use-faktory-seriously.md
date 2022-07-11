---
layout: post
title: Don't use Faktory if you need high reliability
description: Don't use Faktory if you need high reliability
published: false
---

Just in the case one of my project needs a background job, and my company is using 
[Faktory](https://github.com/contribsys/faktory) here and there, I spended sometimes to study this background job engine.

Faktory is a new product of the same company who created 
[sidekiq](https://sidekiq.org/), a famous background job in Ruby community.
I have never got a chance working with Ruby as well as sidekiq, I was so excited to look at Faktory.

Well, my conclusion is that though Faktory is a nice little server, if you cannot tolarate to lost the job, you should not use this.

## Faktory background

Faktory runs in client, server model. There is a Faktory server, and Faktory clients. Clients are part of your programs which generate job and consume them.

![Faktory-Client-Server](/images/2022-07-11-dont-use-faktory-seriously/faktory-model.drawio.svg)
{:width="800px" :class="img-responsive"}

Picture: Faktory runs in client - server model.

Communication between Faktory clients and server are via a TCP connection. Right, using bare-bone TCP connection with text based message. Take heartbeat message for example, the client must send a BEAT every N seconds as proof of liveness

```
BEAT {"wid":"4qpc2443vpvai","rss_kb":1234567}
```

Any clients that understand the protocol can work with Faktory. And that is the basic of programming languages indedepent. We are having Go, Ruby as official libs. There are community libs for different language as Node, Python, Elixir.

My first impression is like well the guys are bold enough to build their own message via TCP. Why don't simply use like gRPC for the communication? I don't have the answer right now, guessing the author love Redis (which is the only story for Faktory so far) so much and want to go [the same text base protocol](https://github.com/contribsys/faktory/wiki/Worker-Lifecycle#network-connection) like that of Redis.

The whole purpose of Faktory, as well as any background job server, is to answer the `Can I do this work later?` question. To be specific, it should be able to schedule a job in one of the following cases:
a. Run this job as soon as possible,
b. Run this job in next 30 mins, and
c. Run this job at 9:00 AM on every Monday.

As we will see later, Faktory has different APIs for those cases, and also different implementations.

