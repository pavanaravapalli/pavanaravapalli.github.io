---
layout: post
title: "The Fix Was Simple. The Architecture Wasn’t"
date: 2019-09-28 19:45:00 +0530
categories: [architecture, design]
---

In 2019 I tried to add one small feature to a clustered Java application: when one node changes a setting, tell the other nodes to drop their cached copy. A few hours of design. Maybe a day of code.

The application would not start. Spring reported a circular reference between two components, and I never shipped the feature.

For a long time I told this story as "I hit a bug". That is not accurate. The bug was not in my code. It was in how the pieces depended on each other.

> **About this post.** This article draws on challenges I’ve encountered in my work and the lessons they taught me about designing products. The examples and component names are illustrative and have been generalized.


## The problem was simple

Several nodes, one database, and a local cache on every node. Local caches are fast, but they do not know about each other. One node updates a setting, and the others keep serving the old copy. Same setting, different answer, depending on which node took the request.

The fix is not clever. Keep the local caches and add a cluster-wide message. When a setting is created, updated or deleted on one node, that node broadcasts an event over the node-to-node channel the application already had. Each node updates its own cache based on the event type. Any engineer would sketch it on a whiteboard in ten minutes.

## Then startup said no

The cache needed the broadcaster, so it could announce changes. The cluster side needed configuration values to start, and those came through the cache. As far as I remember the shape of it, each side waited for the other, and the container could not build either one.

```
cache  --->  broadcaster (cluster side)
  ^                |
  |                v
  +------  configuration lookup
```

Nothing in my change was wrong. The structure had no place for a dependency in that direction.

## Why this was a design problem

The two parts were designed apart, at different times. The cache started as a local convenience, and the cluster code started as a way for nodes to coordinate. Neither was written expecting the other to need it.

Clustering, high availability and cache consistency cut across layers. They need to touch several parts of a system at once, and a design that assumes dependencies only flow one way has no room for that. Every new requirement then either bends the structure or fights it.

I don't blame the people who wrote it. A design is a bet on the requirements of the day, and requirements keep moving.

## The part that stays with me

With a clean structure, my change was a small fix. With the structure we had, it was a refactoring project before I could even begin. The cost of the feature was not the feature. It was untangling what came before it.

That is the price of a rigid structure. You do not pay it at design time. You pay it years later, one blocked feature at a time.

## What I would do today

**1. Draw the arrows first.** Before writing any code for a new feature, draw which part depends on which, including the new arrow your feature adds. If the new arrow closes a loop, stop and fix the structure first. Five minutes on paper beats days of startup errors.

**2. Invert the dependency with a small contract.** The cache should not know that a cluster exists. It should depend on one tiny interface, and the cluster side should implement it.



```java
public interface InvalidationPublisher {
    void publish(String cacheName, String key);
}
```

```
cache  --->  InvalidationPublisher  <---  cluster side
```

Both sides depend on the small shared contract, so neither depends on the other, and the loop is gone.

**3. Connect them at startup, not at construction.** The cluster side registers itself as the cache's publisher once everything is up. Spring can sometimes get around a loop with setter injection, but it cannot resolve one between constructor-injected components, and recent Spring Boot versions reject circular references by default. Even when a workaround runs, it hides the design problem.

**4. Send small events.** Broadcast the key and the event type, not the new value. Receivers evict the entry and reload from the database on the next read. An eviction is safe to do twice, so a node reacting to its own event, or a repeated event, does no harm.


<img src="/assets/cache_cluster_circular_dependency_before_after.svg" alt="Approach before and after the fix" style="max-width: 100%;">


## What I would keep an eye on

A clean dependency graph does not make delivery reliable. Messages can be late or lost, for example when a node restarts, so I would still give every entry an expiry time as a safety net. Also, after a restart or a deploy every node starts with an empty cache and reads from the single database at once, which is a separate failure worth planning for.

## The takeaway

Do not treat your architecture diagram as final. Treat it as the best guess for today's requirements, and review it when they change. If a small feature is hard to add, the feature is probably not the problem.

*Have you been blocked by a design that was right once and wrong later? I would like to hear how you dealt with it.*

---

*Written by Pavan Aravapalli, based on hands-on development experience with clustered Java applications. Generic overview; no employer, product or customer details.*
