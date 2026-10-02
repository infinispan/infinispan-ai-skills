# Infinispan AI Skills

AI-powered skills for [Infinispan](https://infinispan.org) — the open-source, in-memory distributed data grid. These skills help AI coding assistants provide accurate, context-aware guidance for Infinispan configuration, API usage, troubleshooting, tuning, deployment, and migration.

## What's Included

| Skill | Description |
|-------|-------------|
| **infinispan** | Dispatcher — detects your intent and routes to the right sub-skill |
| **infinispan-config** | Cache modes, persistence, indexing, encoding, security, clustering, cross-site replication |
| **infinispan-api** | Embedded mode, Hot Rod, REST, counters, transactions |
| **infinispan-query** | Ickle queries, full-text search, vector search, spatial search, indexing, continuous queries |
| **infinispan-spring** | Spring Boot starter, Spring Cache, Spring Session, reactive support, per-cache configuration |
| **infinispan-hibernate** | Hibernate second-level cache, entity/collection/query cache, JPA integration |
| **infinispan-security** | Authentication, authorization, TLS/SSL, security realms, SASL, Kerberos, audit logging |
| **infinispan-resp** | RESP (Redis-compatible) endpoint, Jedis/Lettuce/Spring Data Redis, Redis migration |
| **infinispan-cli** | CLI commands, cache operations, backup/restore, user management, benchmarking |
| **infinispan-listeners** | Cache listeners, client listeners, clustered listeners, CDI events, event filtering |
| **infinispan-troubleshoot** | Structured diagnosis with error code index for cluster, serialization, performance, persistence, and Hot Rod issues |
| **infinispan-tuning** | JVM sizing, GC selection, cache/persistence/network tuning, monitoring metrics and thresholds |
| **infinispan-deploy** | Infinispan Operator, Helm charts, container images, scaling, monitoring |
| **infinispan-migration** | Version upgrade guides, config format conversion, store migration, deprecation tracking |

## Installation

### Claude Code

```bash
claude /install-plugin https://github.com/infinispan/infinispan-ai-skills
```

Or manually copy the `skills/` directory into your project's `.claude/skills/` directory.

### Manual Installation (any AI tool)

1. Clone this repository:
   ```bash
   git clone https://github.com/infinispan/infinispan-ai-skills.git
   ```
2. Copy the `skills/` directory into your project's AI configuration directory
3. Your AI assistant will automatically discover the skills

## Usage

Once installed, the skills activate automatically when you:

- Ask about Infinispan (e.g., *"How do I configure a distributed cache?"*)
- Work with code that imports `org.infinispan.*`
- Have Infinispan as a Maven or Gradle dependency

### Examples

**Configuration:**
> "Help me configure a distributed cache with JDBC persistence and cross-site replication"

**API usage:**
> "Show me how to set up a Hot Rod client with ProtoStream marshalling"

**Queries & Search:**
> "How do I do vector search with kNN in Infinispan?"

**Spring:**
> "Set up Spring Boot with Infinispan for caching and session externalization"

**Hibernate:**
> "Configure Infinispan as Hibernate second-level cache"

**Security:**
> "How do I set up TLS and LDAP authentication for Infinispan?"

**RESP / Redis:**
> "Can I use Jedis or Spring Data Redis with Infinispan?"

**CLI:**
> "How do I back up and restore an Infinispan cluster?"

**Listeners:**
> "How do I listen for cache entry changes in Hot Rod?"

**Troubleshooting:**
> "My cluster nodes aren't discovering each other on Kubernetes"

**Tuning:**
> "My cache writes are slow — help me diagnose and tune performance"

**Deployment:**
> "Help me deploy Infinispan on OpenShift with the Operator"

**Migration:**
> "We're upgrading from Infinispan 15 to 16 — what configuration changes do we need?"

## MCP Server Integration

Infinispan Server includes a built-in [MCP (Model Context Protocol)](https://modelcontextprotocol.io) endpoint that AI assistants can connect to for **live server interaction** — querying caches, reading logs, inspecting cluster health, and more.

See [docs/mcp-setup.md](docs/mcp-setup.md) for setup instructions.

## Requirements

- Infinispan 16.x (skills default to latest version)
- Java 17+
- For MCP integration: a running Infinispan Server with MCP enabled

## Contributing

Contributions are welcome! If you'd like to improve the skills or add new ones:

1. Fork this repository
2. Create a feature branch
3. Submit a pull request

## License

Apache License 2.0 — same as Infinispan.
