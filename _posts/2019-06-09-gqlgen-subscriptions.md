---
layout: post
title: gqlgen, subscriptions, and NATS
description: gqlgen, subscriptions, and NATS
published: true
---

GraphQL has been being talked a lot recently, thanks to many cool features like strong type, no more over/under fetching like REST.

For me, another interesting feature of GraphQL is subscription to provide realtime updating. I have been working in IoT for a few years, realtime status updating is an always wanted feature: most recent status of lights, thermostats has to be displayed in the user dashboard, current upload/download bandwidth of a router.

Just use WebSocket, you may say, as people have been doing for a decade. Agree. GraphQL uses WebSocket under the hood also. The most valuable asset is a set of [client libraries](https://www.apollographql.com/docs/react/) that people can share and use. Lib for react, vue, and native mobile apps also. Standard library and method save our dev and maintenance efforts.

Why [gqlgen](https://gqlgen.com/)? It's based in Golang, and strong type by design. And easy to work with.

## Work with one handler, let's make it simple

My first attempt to use the default todo app, with a subscription to newly created todo. Defining a struct to keep all todos

```{golang}
type resolver struct {
	sync.Mutex
	todos     []*Todo
	observers map[string]chan *Todo
}
```

The `observer` keep track of all subscribers.

Following is the resolver for subsciption. The resolver function returns a todo chan (`event`) for each subscription request. A goroutine waits for a close signal (ctx.Done()) to clean up the event registration as well.

```{golang}
func (s *subscriptionResolver) TodoAdded(ctx context.Context) (<-chan *Todo, error) {
	id := randString(8)
	event := make(chan *Todo, 1)

	go func() {
		<-ctx.Done()
		s.Lock()
		delete(s.observers, id)
		s.Unlock()
	}()

	s.Lock()
	s.observers[id] = event
	s.Unlock()

	return event, nil
}
```

Then when a new todo created, we should send this new item to list of clients via event channels.

```{golang}
func (r *mutationResolver) CreateTodo(ctx context.Context, input NewTodo) (*Todo, error) {
	todo := &Todo{
		Text:   input.Text,
		ID:     fmt.Sprintf("T%d", rand.Int()),
		...
	}
	r.todos = append(r.todos, todo)

	r.Lock()
	for _, observer := range r.observers {
		observer <- todo
	}
	r.Unlock()

	return todo, nil
}
```

And here is the result.

[Subscription with NATS](/images/2019-06-09-gqlgen-subscriptions/sub-local.gif){:width="400px" :class="img-responsive"}

## Scale-up boy, we need to work with multiple handlers

Previous setup just works well as long as we only have one worker to handle. In modern day, we expect our service to be always on; hence we throw several workers behind a load balancer to get high availability and handle more traffic.

Apollo comes with [Pubsub implementation](https://www.apollographql.com/docs/apollo-server/features/subscriptions/) running on one worker just like we did before. To scale up, they suggest the community implementation relies on buses like Redis, RabbitMQ, Kafka.

We can do the same, with [NATS](https://nats.io/). If you never heard of NATS, it's a simple open source messaging system. NATS is fast, [very fast](https://nats-io.github.io/docs/nats_tools/natsbench.html). NATS is simple to use, you will see :).

First, we define to todos struct with NATS connection.

```{golang}
type resolver struct {
	todos   []*Todo
	nClient *nats.EncodedConn
}

func New() Config {
	nc, err := nats.Connect(nats.DefaultURL)
	nClient, _ := nats.NewEncodedConn(nc, nats.JSON_ENCODER)
	if err != nil {
		log.Fatalln(err)
	}
	return Config{
		Resolvers: &resolver{
			nClient: nClient,
		},
	}
}
```

With anew subscription request, we just sub a related topic with NATS. I use `todotopic` as the topic for new todo.

```{golang}
func (s *subscriptionResolver) TodoAdded(ctx context.Context) (<-chan *Todo, error) {
	event := make(chan *Todo, 1)

	sub, err := s.nClient.Subscribe("todotopic", func(t *Todo) {
		event <- t
	})

	if err != nil {
		return nil, err
	}

	go func() {
		<-ctx.Done()
		sub.Unsubscribe()
	}()

	return event, nil
}
```

And when the new todo created, just publish to `todotopic` topic.

```{golang}
func (r *mutationResolver) CreateTodo(ctx context.Context, input NewTodo) (*Todo, error) {
	todo := &Todo{
		Text:   input.Text,
		ID:     fmt.Sprintf("T%d", rand.Int()),
		...
	}
	r.todos = append(r.todos, todo)

	r.nClient.Publish("todotopic", todo)

	return todo, nil
}
```

Testing by running a local NATS server, and two workers at port 8081, 8080. One client subscribe at :8081, one client create new todo at :8080

[Subscription with NATS](/images/2019-06-09-gqlgen-subscriptions/sub-nats.gif){:width="400px" :class="img-responsive"}

NATS takes a lot of complexity under the hood to provide us a "scalable" subcription with gqlgen.
I also have a simple prototype with Redis.

Maybe will release a `mq` package to make the subscription more easy. Stay tune.
