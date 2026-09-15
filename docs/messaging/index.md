# Messaging

Asynchronous, decoupled communication between services: message queues,
Kafka's log-based architecture, delivery semantics, and the retry/
dead-letter patterns that make consumers resilient to failure.

This section is framework/broker-agnostic where possible; broker-specific
configuration (RabbitMQ, Kafka, SQS) is illustrated with representative
examples rather than a full operations guide for any one system.

## Topics

1. [Message Queues and Messaging Patterns](01-message-queues-and-messaging-patterns.md)
2. [Kafka Fundamentals](02-kafka-fundamentals.md)
3. [Delivery Semantics and Idempotent Consumers](03-delivery-semantics-and-idempotency.md)
4. [Retry Strategies and Dead-Letter Queues](04-retry-strategies-and-dead-letter-queues.md)
