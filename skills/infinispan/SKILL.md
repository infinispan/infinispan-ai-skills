---
name: infinispan
description: Use when user asks about Infinispan, or code imports org.infinispan.*, or project has Infinispan Maven/Gradle dependencies. Covers configuration, API usage, troubleshooting, deployment, tuning, and migration.
---

# Infinispan Skill Suite

Infinispan is an open-source, in-memory distributed data grid and cache. This skill routes to specialized sub-skills based on user intent.

## Intent Classification

Read the user's request and route to the appropriate sub-skill:

```dot
digraph routing {
    "User request" [shape=doublecircle];
    "About config?" [shape=diamond];
    "About queries/search?" [shape=diamond];
    "About Spring?" [shape=diamond];
    "About Hibernate L2C?" [shape=diamond];
    "About security?" [shape=diamond];
    "About RESP/Redis?" [shape=diamond];
    "About CLI?" [shape=diamond];
    "About listeners?" [shape=diamond];
    "About API/code?" [shape=diamond];
    "About errors/perf?" [shape=diamond];
    "About tuning?" [shape=diamond];
    "About deployment?" [shape=diamond];
    "About upgrading?" [shape=diamond];
    "Ask clarifying question" [shape=box];

    "User request" -> "About config?";
    "About config?" -> "infinispan-config" [label="yes"];
    "About config?" -> "About queries/search?";
    "About queries/search?" -> "infinispan-query" [label="yes"];
    "About queries/search?" -> "About Spring?";
    "About Spring?" -> "infinispan-spring" [label="yes"];
    "About Spring?" -> "About Hibernate L2C?";
    "About Hibernate L2C?" -> "infinispan-hibernate" [label="yes"];
    "About Hibernate L2C?" -> "About security?";
    "About security?" -> "infinispan-security" [label="yes"];
    "About security?" -> "About RESP/Redis?";
    "About RESP/Redis?" -> "infinispan-resp" [label="yes"];
    "About RESP/Redis?" -> "About CLI?";
    "About CLI?" -> "infinispan-cli" [label="yes"];
    "About CLI?" -> "About listeners?";
    "About listeners?" -> "infinispan-listeners" [label="yes"];
    "About listeners?" -> "About API/code?";
    "About API/code?" -> "infinispan-api" [label="yes"];
    "About API/code?" -> "About errors/perf?";
    "About errors/perf?" -> "infinispan-troubleshoot" [label="yes"];
    "About errors/perf?" -> "About tuning?";
    "About tuning?" -> "infinispan-tuning" [label="yes"];
    "About tuning?" -> "About deployment?";
    "About deployment?" -> "infinispan-deploy" [label="yes"];
    "About deployment?" -> "About upgrading?";
    "About upgrading?" -> "infinispan-migration" [label="yes"];
    "About upgrading?" -> "Ask clarifying question" [label="unclear"];
}
```

## Routing Table

| Intent | Sub-skill | Signals |
|--------|-----------|---------|
| Configuration | `infinispan-config` | Cache modes, XML/YAML config, persistence, encoding, clustering, cross-site replication |
| Queries & Search | `infinispan-query` | Ickle queries, full-text search, vector search, spatial search, indexing, continuous queries |
| API & Code | `infinispan-api` | Embedded mode, Hot Rod client, REST client, counters, transactions |
| Spring Integration | `infinispan-spring` | Spring Boot starter, Spring Cache, Spring Session, reactive, per-cache config |
| Hibernate L2 Cache | `infinispan-hibernate` | Second-level cache, entity/collection/query cache, JPA @Cacheable |
| Security | `infinispan-security` | Authentication, authorization, TLS/SSL, security realms, SASL, Kerberos, audit logging |
| RESP / Redis | `infinispan-resp` | RESP endpoint, Redis compatibility, Jedis, Lettuce, Spring Data Redis, Redis migration |
| CLI | `infinispan-cli` | CLI commands, cache operations, backup/restore, user management, benchmarking, cross-site |
| Listeners & Events | `infinispan-listeners` | Cache listeners, client listeners, clustered listeners, CDI events, event filtering |
| Troubleshooting | `infinispan-troubleshoot` | Errors (ISPN/JGRP codes), exceptions, cluster problems, serialization failures, Hot Rod connectivity, log analysis |
| Tuning | `infinispan-tuning` | Performance, latency, throughput, GC pressure, heap sizing, capacity planning, monitoring metrics |
| Deployment | `infinispan-deploy` | Operator, Helm, Kubernetes, container images, scaling, monitoring |
| Migration | `infinispan-migration` | Version upgrades, deprecated features, config format conversion, store migration, embedded-to-server |

## Do NOT Trigger For

- Generic caching questions not about Infinispan (Redis standalone, Ehcache, Hazelcast)
- Unrelated code with no Infinispan references

## Cross-cutting

- Link to official docs at `https://infinispan.org/docs/` when relevant
- Default to latest Infinispan version (16.x) unless user specifies otherwise
- Suggest MCP server connection (`-Dorg.infinispan.feature.mcp=true`) when live inspection would help troubleshooting