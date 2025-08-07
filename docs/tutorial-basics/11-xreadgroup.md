---
sidebar_position: 11
---

# XREADGROUP
Multiple coordinated consumers (consumer groups)

## Use when
- You’re building a web app, microservice, or distributed system.
- You want to scale out consumers (load balancing).
- You want reliable processing, acknowledgment, retrying, or at-least-once delivery.
- You need fault tolerance or consumer crash recovery.

## Behavior:
- Works with consumer groups: Redis tracks who read what.
- Each entry is delivered to only one consumer in the group.
- Supports acknowledgment (XACK) — Redis knows when a message is processed.
- Supports pending message management and retries (via XPENDING, XCLAIM).

## Analogy:

Like a message queue — multiple workers pull from the same queue, and each message is only handled once.

## When to use? 
| Use Case                                 | Use          | Notes                                                         |
| ---------------------------------------- | ------------ | ------------------------------------------------------------- |
| Web app or microservices needing scaling | `XREADGROUP` | Enables multiple consumers, load balancing, fault tolerance   |
| Reliable stream processing               | `XREADGROUP` | Supports acknowledgments and pending message management       |
| Distributed system with crash recovery   | `XREADGROUP` | Crucial for ensuring messages aren’t lost or double-processed |
