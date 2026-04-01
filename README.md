# Infinispan AI Skills

AI-powered skills for [Infinispan](https://infinispan.org) — the open-source, in-memory distributed data grid. These skills help AI coding assistants provide accurate, context-aware guidance for Infinispan configuration, API usage, troubleshooting, and deployment.

## What's Included

| Skill | Description |
|-------|-------------|
| **infinispan** | Dispatcher — detects your intent and routes to the right sub-skill |
| **infinispan-config** | Cache modes, persistence, indexing, encoding, security, clustering, cross-site replication |
| **infinispan-api** | Embedded mode, Hot Rod, REST, Ickle queries, counters, transactions, Spring/Quarkus |
| **infinispan-troubleshoot** | Structured diagnosis for cluster, serialization, performance, and persistence issues |
| **infinispan-deploy** | Infinispan Operator, Helm charts, container images, scaling, upgrades, monitoring |

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

**Troubleshooting:**
> "My cluster nodes aren't discovering each other on Kubernetes"

**Deployment:**
> "Help me deploy Infinispan on OpenShift with the Operator"

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
