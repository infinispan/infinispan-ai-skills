---
name: infinispan
description: Use when user asks about Infinispan, or code imports org.infinispan.*, or project has Infinispan Maven/Gradle dependencies. Covers configuration, API usage, troubleshooting, and deployment.
---

# Infinispan Skill Suite

Infinispan is an open-source, in-memory distributed data grid and cache. This skill routes to specialized sub-skills based on user intent.

## Intent Classification

Read the user's request and route to the appropriate sub-skill:

```dot
digraph routing {
    "User request" [shape=doublecircle];
    "About config?" [shape=diamond];
    "About API/code?" [shape=diamond];
    "About errors/perf?" [shape=diamond];
    "About deployment?" [shape=diamond];
    "Ask clarifying question" [shape=box];

    "User request" -> "About config?";
    "About config?" -> "infinispan-config" [label="yes"];
    "About config?" -> "About API/code?";
    "About API/code?" -> "infinispan-api" [label="yes"];
    "About API/code?" -> "About errors/perf?";
    "About errors/perf?" -> "infinispan-troubleshoot" [label="yes"];
    "About errors/perf?" -> "About deployment?";
    "About deployment?" -> "infinispan-deploy" [label="yes"];
    "About deployment?" -> "Ask clarifying question" [label="unclear"];
}
```

## Routing Table

| Intent | Sub-skill | Signals |
|--------|-----------|---------|
| Configuration | `infinispan-config` | Cache modes, XML/YAML config, persistence, indexing, encoding, security, clustering, cross-site replication |
| API & Code | `infinispan-api` | Embedded mode, Hot Rod client, REST client, Ickle queries, counters, transactions, Spring/Quarkus |
| Troubleshooting | `infinispan-troubleshoot` | Errors, exceptions, performance issues, cluster problems, serialization failures, log analysis |
| Deployment | `infinispan-deploy` | Operator, Helm, Kubernetes, container images, scaling, upgrades, monitoring |

## Do NOT Trigger For

- Generic caching questions not about Infinispan (Redis standalone, Ehcache, Hazelcast)
- Unrelated code with no Infinispan references

## Cross-cutting

- Link to official docs at `https://infinispan.org/docs/` when relevant
- Default to latest Infinispan version (16.x) unless user specifies otherwise
- Suggest MCP server connection (`-Dorg.infinispan.feature.mcp=true`) when live inspection would help troubleshooting