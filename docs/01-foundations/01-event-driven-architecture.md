# Event-Driven Architecture (EDA)

> **Goal of this document:** Give you enough background in event-driven architecture to understand why AsyncAPI and this parser exist. No prior EDA knowledge needed.

---

## The Problem with Request/Response

Most developers first learn web development through **REST APIs**. In REST, a client asks for something and the server answers immediately:

```
Client                    Server
  |                          |
  |--- GET /users/42 ------->|
  |<-- { name: "Alice" } ----|
  |                          |
```

This works well when:
- The client needs data right now
- The operation completes quickly
- There is only one consumer of the data

But many real-world systems do not fit this model:

| Scenario | Why REST/RPC struggles |
|----------|----------------------|
| A payment processor notifying 10 downstream services that an order was paid | You'd need to call all 10 services synchronously; if one is slow, everything waits |
| A GPS tracker sending location every second to potentially thousands of subscribers | You can't poll millions of trackers per second |
| A stock ticker broadcasting price changes the instant they happen | Clients polling every second still miss sub-second updates |
| An IoT sensor sending temperature readings continuously | Sensors shouldn't need to know who is listening |

The common theme: the data **producer** (sensor, payment system, stock exchange) should not need to know about, or wait for, every **consumer** (analytics service, dashboard, notification engine).

---

## The Event-Driven Model

In event-driven architecture, components communicate by **publishing events** and **subscribing to events**. No direct calls between services.

```
Producer                 Broker (Message Bus)          Consumers
   |                           |                           |
   |--- event: "order.paid"--->|                           |
   |                           |--- "order.paid" -------> Service A (send email)
   |                           |--- "order.paid" -------> Service B (update inventory)
   |                           |--- "order.paid" -------> Service C (trigger analytics)
   |                           |
```

### Key Concepts

**Event**
An immutable record of something that happened. "User 42 placed order #999 at 14:30 UTC." An event describes the past — it is a fact.

**Producer (Publisher)**
The component that emits an event. It fires and forgets — it does not know who, if anyone, is listening.

**Consumer (Subscriber)**
A component that receives and processes events. It declares interest in certain types of events.

**Broker (Message Bus / Message Queue)**
The infrastructure that sits between producers and consumers. It receives events from producers and delivers them to interested consumers. Examples: Apache Kafka, RabbitMQ, AWS SNS/SQS, MQTT broker (Mosquitto), NATS, Redis Pub/Sub.

**Channel (Topic / Queue)**
A named pipe through which a specific type of event flows. A broker can have thousands of channels. Example: `user.registered`, `payment.completed`, `sensor.temperature.reading`.

**Message**
The data packet sent through a channel. It has a **payload** (the actual data) and optionally **headers** (metadata).

**Schema**
The contract that defines the shape and type of a message's payload. "This payload is a JSON object with fields `userId` (string) and `amount` (number)."

**Subscription**
A consumer's declaration that it wants to receive messages from a specific channel.

---

## A Concrete Example: Streetlights

This example is used throughout the AsyncAPI specification itself, so you will see it referenced everywhere in this documentation.

Imagine a smart city system that controls streetlights:

- Each streetlight has a sensor that measures ambient light.
- A central control service can send commands to turn lights on or off.
- A monitoring service subscribes to all sensor readings.

```
Streetlight Sensor                   MQTT Broker              Control Service
       |                                  |                          |
       |--- "smartylighting/streetlights/ |                          |
       |     europex/1/berlin/main/       |                          |
       |     status/0" (lux reading) ---->|                          |
       |                                  |                          |
       |                                  |<-- "turn/on" ------------|
       |<-- command: turn on -------------|                          |

Monitoring Service                   MQTT Broker
       |                                  |
       |--- subscribe: "*/status/*" ------>|
       |<-- lux reading from all lights ---|
```

In this system:
- The **streetlight** is a producer (publishes readings).
- The **control service** is both a producer (commands) and consumer (might listen for status).
- The **monitoring service** is a consumer.
- The **MQTT broker** is the message bus.
- `smartylighting/streetlights/europex/1/berlin/main/status/0` is a channel name.

---

## Why Async Matters

| Property | Request/Response (REST) | Event-Driven |
|----------|------------------------|--------------|
| **Coupling** | Tight — caller must know receiver's address | Loose — producer doesn't know consumers |
| **Availability** | Both sides must be up simultaneously | Consumer can be offline; broker buffers messages |
| **Scalability** | Linear — add load balancers | Natural fan-out — add consumers, broker handles distribution |
| **Latency** | Bounded by slowest downstream call | Fire-and-forget from producer's perspective |
| **Resilience** | Cascade failures | Isolated — one consumer failing doesn't affect others |

---

## Common Protocols and Brokers

You will encounter these in AsyncAPI documents:

| Protocol | Use Case | Examples |
|----------|----------|---------|
| **AMQP** | Enterprise messaging, reliable delivery | RabbitMQ, Azure Service Bus |
| **MQTT** | IoT, low-bandwidth devices | Mosquitto, AWS IoT, HiveMQ |
| **Kafka** | High-throughput event streaming, replay | Apache Kafka, Confluent |
| **WebSocket** | Real-time browser/server communication | Any WebSocket server |
| **STOMP** | Simple text messaging | ActiveMQ |
| **HTTP** | Webhooks, server-sent events | Any HTTP server |
| **NATS** | Cloud-native lightweight messaging | NATS.io |

---

## EDA vs Microservices

EDA and microservices are different concepts that are often used together:

- **Microservices** is an architectural style about how to decompose an application into small, independently deployable services.
- **EDA** is a communication pattern — how those services talk to each other.

You can have microservices that communicate via REST (tight coupling) or via events (loose coupling). EDA is the pattern; microservices is the decomposition strategy. Many modern systems use both.

---

## Vocabulary Cheat Sheet

| Term | Meaning |
|------|---------|
| Event | An immutable record of something that happened |
| Producer | Publishes events to a channel |
| Consumer | Subscribes to events from a channel |
| Broker | Infrastructure routing events from producers to consumers |
| Channel / Topic | Named pipe for a specific type of event |
| Message | The data packet sent through a channel |
| Payload | The body of a message (the actual data) |
| Header | Metadata attached to a message (not the payload) |
| Schema | Contract defining the shape of a payload |
| Subscription | A consumer's declaration to receive from a channel |
| Fan-out | One event delivered to many consumers |
| Queue | A channel where each message is consumed by exactly one consumer |
| Topic | A channel where each message is delivered to all subscribers |
| Dead letter queue | Where messages go when they cannot be delivered |

---

## Next Step

Now that you understand event-driven architecture, read [02-what-is-asyncapi.md](./02-what-is-asyncapi.md) to learn how AsyncAPI provides a machine-readable contract for these systems.
