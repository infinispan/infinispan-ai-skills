# Connecting AI Assistants to Infinispan via MCP

Infinispan Server provides a built-in [Model Context Protocol](https://modelcontextprotocol.io) (MCP) endpoint. This lets AI assistants like Claude Desktop or Claude Code interact directly with your running Infinispan server — querying caches, reading logs, and inspecting cluster state.

## 1. Enable MCP on Your Infinispan Server

MCP is an experimental feature, disabled by default. Enable it with a system property:

```bash
# Linux/macOS
bin/server.sh -Dorg.infinispan.feature.mcp=true

# Windows
bin\server.bat -Dorg.infinispan.feature.mcp=true

# Container
podman run -p 11222:11222 \
  -e USER="admin" \
  -e PASS="changeme" \
  -e JAVA_OPTIONS="-Dorg.infinispan.feature.mcp=true" \
  quay.io/infinispan/server:16.0
```

The MCP endpoint is available at `http://localhost:11222/v3/mcp`.

## 2. Connect Claude Desktop

Add the following to your Claude Desktop configuration file:

**macOS:** `~/Library/Application Support/Claude/claude_desktop_config.json`
**Windows:** `%APPDATA%\Claude\claude_desktop_config.json`

```json
{
  "mcpServers": {
    "infinispan": {
      "url": "http://localhost:11222/v3/mcp",
      "headers": {
        "Authorization": "Basic YWRtaW46Y2hhbmdlbWU="
      }
    }
  }
}
```

The `Authorization` header is Base64-encoded `admin:changeme`. Replace with your actual credentials.

To generate the Base64 value:
```bash
echo -n 'admin:yourpassword' | base64
```

## 3. Connect Claude Code

Add the MCP server to your Claude Code project settings (`.claude/settings.local.json`):

```json
{
  "mcpServers": {
    "infinispan": {
      "type": "url",
      "url": "http://localhost:11222/v3/mcp",
      "headers": {
        "Authorization": "Basic YWRtaW46Y2hhbmdlbWU="
      }
    }
  }
}
```

## What You Can Do with MCP

Once connected, your AI assistant can interact with your Infinispan server using these capabilities:

### Tools (Actions)

| Tool | Description |
|------|-------------|
| `getCacheNames` | List all available caches |
| `createCache` | Create a new cache |
| `getCacheEntry` | Retrieve a specific cache entry |
| `setCacheEntry` | Insert or update a cache entry |
| `deleteCacheEntry` | Delete a cache entry |
| `queryCache` | Run Ickle queries against a cache |
| `getCacheConfiguration` | Get a cache's full configuration |
| `getCacheStats` | Get cache statistics (hits, misses, entries, latency) |
| `getClusterHealth` | Get cluster status, members, and node count |
| `getSchemas` | List registered Protobuf schemas |
| `getCounterNames` | List all distributed counters |
| `getCounter` | Get a counter's current value |
| `increment` / `decrement` | Modify a counter's value |

### Resources (Read-Only Context)

| Resource | Description |
|----------|-------------|
| `infinispan+logs://server` | Server log (errors, cluster events) |
| `infinispan+logs://audit` | Audit log (security events) |
| `infinispan+logs://rest-access` | REST API access log |
| `infinispan+logs://hotrod-access` | Hot Rod protocol access log |
| `infinispan+logs://memcached-access` | Memcached protocol access log |
| `infinispan+logs://resp-access` | RESP (Redis) protocol access log |
| `infinispan+logs://gc` | JVM garbage collection log |

### Prompts (Guided Workflows)

| Prompt | Description |
|--------|-------------|
| `find-documentation` | Find relevant Infinispan documentation for a topic |
| `configure-cache` | Guided cache configuration with mode selection and features |
| `diagnose-issue` | Structured troubleshooting based on symptoms |
| `setup-client` | Client setup help (Hot Rod, REST, embedded) |
| `setup-cross-site` | Cross-site replication configuration guidance |

## Example Interactions

Once connected, you can ask your AI assistant things like:

> "Check the server logs for any errors in the last 200 lines"

> "What caches exist on the server and what are their stats?"

> "Query the 'users' cache for all entries where status = 'active'"

> "Show me the cluster health — how many nodes are up?"

> "The cache seems slow. Check the stats and server logs to diagnose."

The AI assistant will use the MCP tools automatically to answer these questions by interacting with your live server.

## Security Notes

- The MCP endpoint respects Infinispan's authentication and authorization.
- Log access requires ADMIN permission.
- Cache operations respect cache-level authorization roles.
- Use TLS in production — configure the MCP URL with `https://`.
- Never commit credentials to version control. Use environment variables or secrets management.
