# system-design-gotchas

![A developer at a laptop reaches out in alarm as a tornado of pink light tears servers and databases out of a miniature city](images/banner.png)

This repo is a collection of gotcha questions about system design, the kind where the obvious answer turns out to be wrong. Each one is a moment from running a real service: a deploy, a failover, a spike in traffic.

Pick what you think happens, then open every answer, the ones you ruled out too. Each wrong one says why it seemed right and what happens instead.

## What's in it for you

- **New to backend work?** Start from the top. Each topic explains its terms before the first question.
- **Already on call?** You'll recognise the scenes, and find out which of your assumptions hold when traffic grows or a node dies.
- **Preparing to talk through a design?** Each question gives you a scene and a few plausible answers. Practise saying why each wrong one fails.
- **Reviewing a design?** Every right answer ends with what to do, and the last topic covers what breaks where the parts meet.
- **How deep does it go?** The topics run from latency and DNS to replication, queues, releases and whole systems. Each starts with definitions and goes down to failures seen in production. No question depends on a particular product.

## Topics

From the network up to whole systems, in a suggested order:

1. [Performance and capacity](topics/performance-and-capacity/README.md): what latency, throughput and tail percentiles tell you, and what a queue does as load climbs.
2. [DNS, TCP and TLS](topics/dns-tcp-and-tls/README.md): what happens before a request reaches a server: names and their caches, connections, lost packets and certificates.
3. [HTTP caching and CDNs](topics/http-caching-and-cdns/README.md): who may store a response, for how long and for whom, and what a purge reaches.
4. [Load balancers and proxies](topics/load-balancers-and-proxies/README.md): how traffic is split, how a backend leaves and rejoins, and what a proxy does to connections and headers.
5. [Rate limiting and load shedding](topics/rate-limiting-and-load-shedding/README.md): how a limit counts calls across many servers, and what to refuse when demand passes what you can serve.
6. [Timeouts, retries and circuit breakers](topics/timeouts-retries-and-circuit-breakers/README.md): what a caller does when no answer comes: deadlines, backoff, idempotency keys and breakers.
7. [Consistency, consensus and clocks](topics/consistency-consensus-and-clocks/README.md): what a read may return when copies disagree, how a group agrees, and what a timestamp can prove.
8. [Distributed locks and IDs](topics/distributed-locks-and-ids/README.md): what a lock or a lease guarantees across machines, and how IDs stay unique and in order.
9. [Replication](topics/replication/README.md): what read replicas, lag, failover and writes in several regions do to your data.
10. [Sharding and partitioning](topics/sharding-and-partitioning/README.md): choosing a shard key, hot keys, moving data to new nodes and queries across shards.
11. [Caching](topics/caching/README.md): how a cache in front of a store fills, goes stale and fails, from invalidation to stampedes.
12. [Choosing a data store](topics/choosing-a-data-store/README.md): what each kind of store gives up, and which conditions settle the choice.
13. [Message queues and event streams](topics/message-queues-and-event-streams/README.md): queues and logs, delivery guarantees, ordering, consumer lag and poison messages.
14. [Event-driven architecture](topics/event-driven-architecture/README.md): events between services: the outbox, change data capture, sagas and changing an event's schema.
15. [WebSockets and real-time delivery](topics/websockets-and-real-time-delivery/README.md): long-lived connections on many nodes: reconnects, rollouts, fan-out and slow clients.
16. [Microservices](topics/microservices/README.md): service boundaries, shared databases, chatty calls, discovery and gateways.
17. [Service-to-service security](topics/service-to-service-security/README.md): what mTLS and tokens prove between services, and who may act for whom.
18. [Containers and orchestration](topics/containers-and-orchestration/README.md): the life of a running copy: probes, shutdown, resource limits and autoscaling.
19. [Releasing changes](topics/releasing-changes/README.md): rolling, blue-green and canary releases, feature flags, schema changes and what a rollback can undo.
20. [Observability](topics/observability/README.md): logs, metrics and traces across services, and what each can and cannot show.
21. [Load testing](topics/load-testing/README.md): how a test makes its load, what it measures and what passing means.
22. [Reliability and disaster recovery](topics/reliability-and-disaster-recovery/README.md): SLOs and error budgets, backups and restores, RPO and RTO, and running in several regions.
23. [System design](topics/system-design/README.md): the parts together, the failures that appear only where they meet, and whole systems such as a feed, a chat or a booking site.

## License

MIT. See [LICENSE](LICENSE).
