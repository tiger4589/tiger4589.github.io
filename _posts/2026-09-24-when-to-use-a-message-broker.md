---
layout: post
title: "When should your application use a message broker?"
date: 2026-09-24
tags: message-broker messaging microservices asynchronous-communication software-architecture event-driven-architecture distributed-system eventual-consistency reliability
---

When building a software, one of the first decisions we make is to let different parts of the system communicate through HTTP.  
If service A needs something from service B we'll just call the service B API:

```csharp
var response = await client.PostAsJsonAsync("api/endpoint", class_instance);
```

This works fine, and usually all we need, until it doesn't.

As systems grow, we will encounter situations where synchronous HTTP communication makes things awkward.  
The receiving service might be temporarily unavailable, or maybe more than one service needs to consume the same event. The process might be slow and take several minutes, or maybe you don't care when the work happens, as long as eventually it will happen.

That's when a message broker becomes interesting. However, that doesn't mean every application needs RabbitMQ, a service bus, or another messaging platform.

So when should you actually use one?

<!--more-->

## One failure breaks everything

What's better than the usual e-commerce application for demonstration? So, let's imagine you're working on such.

A customer creates an order, and your API receives a `POST /orders`, the request gets validated, stores the order, and returns a `201 Created`.

Let's assume that after creating an order you'll also need to send a confirmation email, update stock inventory, process payment, generate an invoice, and notify some other internal systems such as analytics.

You can probably implement all of that directly:

```csharp
public async Task CreateOrder(Order order)
{
    await orderService.Execute(order);
    await paymentService.Validate(order);
    await invoiceService.Generate(order);
    await emailService.SendConfirmation(order);
    await inventoryService.UpdateStock(order);
    await dispatchService.Dispatch(order);
}
```

This might work, but creating an order is now coupled to multiple different systems.

Suppose your mail provider was unavailable at the time of order creation, should the client order fail, just because you couldn't send an email? Probably not. However, with synchronous communication, this is bound to happen.

Any service failure in this case would fail the request completely, even if unrelated to the order creation itself.

A message broker in this case lets you separate those responsibilities, and allow the order creation to move forward regardless of what other systems are doing.

![Message Broker Flow](/assets/message-broker/broker_flow.png)

Now the order can be created, and it won't be directly affected by other services being unavailable. The order API will create the order and store it, and publish an event saying that a new order has been created. The message will remain in the broker until consumers process it. A single point of failure in communication is a strong sign where messaging may be useful.

## You don't need an immediate response

HTTP works well when you need an immediate answer. If your customer was looking into his order history, a broker doesn't make much sense. The information is needed at the moment of request.

But, considering our previous scenario, invoice generation is not usually the priority of a customer creating an order. This might take a few seconds to generate, more time to upload, and sending it via email will add more external dependencies. Does the customer really need to wait for the invoice? Not really.

This is another problem that a message broker can fix by making the necessary process asynchronous.

## Multiple systems need the same information

While still looking at the same order scenario, we can easily see that multiple systems beside the order service need the same information: an order has been created.

Without messaging, the order service has to know about all of those different services, and for every new system introduced means modifying the Order service again.

However by using a message broker, the producer doesn't need to know who consumes the event, it simply publishes it for other systems to consume. Introducing a new service is as simple as plugging another charger in the same power socket. If it needs to listen to new orders created, it will subscribe to that type of message and do the necessary when it gets one.

The difference between telling each service what to do and publishing an event is important.

## Temporary downtime shouldn't lose work

Imagine your warehouse system going down for more than ten minutes. If you're using a normal HTTP call, then it will fail during that time. Now you'll need to decide what happens next:

- Retry?
- How many times?
- Where to store the failed request?
- When to fetch and execute the failed request?

A broker already provides this infrastructure for you, and it will take care of storing and delivering failed messages. This doesn't make failure disappear, since you still need to think about retries, dead-letter queues, and idempotency. But, the architecture and infrastructure of a broker is designed around failure instead of assuming everything will be ok, and it's easier to use it than coming up with your own resilience system. Keep in mind that although the broker helps with durability and delivery, they don't solve idempotency or business correctness by themselves.

## Traffic spikes

Suppose your system normally gets around 10 orders per second. A promotion starts and now you're receiving 200 orders per second. Assuming your order service is capable of handling the traffic, but the invoicing service is not capable of doing so and is only able to handle 50 invoice per second, then you're in trouble.

Introducing a broker in this case will act as a buffer. Your orders service can still publish those 200 orders created events, and the invoice system can consume them at the rate of 50 per second without crashing.

Different services can operate and will probably need to operate in different speeds. Introducing the broker as a buffer is another sign.

# Don't add a broker just because you can

## Messaging introduces complexity

Instead of service A calling service B, now you'll have a broker in between. This forces you to think about:

- message contracts
- serialization
- retries
- dead-letter queues
- duplicate messages and idempotency
- ordering
- observability
- deployment
- versioning
- eventual consistency
- request/response patterns
- queue vs topic
- exactly-once vs at-least-once semantics

Debugging also becomes a little bit harder. With HTTP, you can follow a single request from beginning to end, but with asynchronous messaging, an action continue several seconds, or minutes, after the original request has finished.

A message broker solves problems, but it also gives you a new category of problems to manage.

### Transactional Outbox Pattern

One of the complexities and pitfalls using a message broker is dismissing the idea that everything might fail at one point.

If your API is doing something like this:

```csharp
await dbContext.SaveChangesAsync();

await publishEndpoint.Publish(created_order);
```

then you'll eventually get into troubles.  
What happens if the database commit succeeds, but publishing the message fails?  
What happens if the message is published, but the database commit failed?

This is a very important and critical pattern that you should think about, and I will write more about that in another post because I believe it merits its own explanation.

## Commands or Events

A distinction should be made between these two. A command asks for something to happen:

- GenerateInvoice
- SendEmail
- UpdateInventory

while an event describes something that has already happened:

- OrderCreated
- OrderPaid
- OrderCancelled

Commands usually have one intended receiver, while an event can have from zero to many consumers. Choosing between commands and events needs careful thoughts, because messaging isn't simply replacing HTTP calls with queues.

## Eventual consistency

Once a message broker and asynchronous messaging is introduced, your system might no longer be always consistent.

If an order is created, then an `OrderCreated` event was published, and your inventory service is listening to that event, there might be a short period of time where the order exists, but the inventory system hasn't processed it yet.

This might be acceptable, or completely unacceptable. Consider a bank system for example, transferring money from an account consecutively can't be asynchronous because it will introduce a huge risk. You probably need an immediate answer if the account has enough funds to transfer. On the other hand, updating a dashboard can handle a few seconds delay.

The business requirement matters more than the technology in this case. Ask yourself, is eventual consistency acceptable or not?

# HTTP or Message broker?

A useful rule of thumb is the following:

Use HTTP when the caller needs an immediate response.  
Consider messaging when the caller primarily needs to publish that something happened.

For example:

| Scenario                                | HTTP | Messaging |
| --------------------------------------- | ---- | --------- |
| Retrieve details                        | Yes  | No        |
| Validate discount code                  | Yes  | No        |
| Send confirmation email                 | No   | Yes       |
| Generate invoice asynchronously         | No   | Yes       |
| Query current inventory                 | Yes  | No        |
| Notify different systems about an event | No   | Yes       |

These aren't rules, but they're a useful starting point.

# Final thoughts

You probably won't need a message broker when:

- You need immediate answers
- The workflow must be strongly consistent in one transaction
- Your system is small and simple
- You can tolerate simple failures with retries
- You cannot handle eventual consistency
- Ordering and exactly-once are strict

A broker starts becoming useful when one or more of these are true:

- Work does not need to happen directly
- Services should be loosely coupled
- Multiple system needs to react to same event
- Temporary downtime of different services should not lose work
- A long-running process
- You want independent scaling of producers and consumers

But if your application is small, synchronous, and simple, adding messaging will give you more operational complexity than benefits.

**Start with the simplest architecture that satisfies your requirements, and solve the problems that you have, and not the problems that you might have in the future.**

Sometimes using a normal HTTP call is all you need.
