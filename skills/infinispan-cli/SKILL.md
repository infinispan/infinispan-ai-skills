---
name: infinispan-cli
description: Use when user asks about the Infinispan CLI - connecting, cache operations, counters, backup/restore, batch mode, benchmarking, user management, cross-site management, server reports.
---

# Infinispan CLI Guide

Help users interact with Infinispan Server through the command-line interface (CLI). The CLI provides an interactive shell and batch mode for server administration, cache operations, and diagnostics.

## Workflow

1. Determine what the user wants to do (connect, manage caches, administer users, etc.)
2. Provide the exact CLI command with flags and examples
3. Explain the expected output or behavior
4. Warn about common pitfalls

## Installing the CLI

The CLI ships with Infinispan Server. A standalone native CLI is also available.

```bash
# Homebrew (macOS)
brew tap infinispan/tap && brew install infinispan-cli

# RPM (Fedora/RHEL) — add repo first
sudo curl -o /etc/yum.repos.d/infinispan.repo https://download.jboss.org/infinispan/packages/yum/infinispan.repo
sudo dnf install infinispan-cli

# DEB (Debian/Ubuntu) — add repo and signing key first
curl -fsSL https://download.jboss.org/infinispan/packages/apt/infinispan.gpg | sudo gpg --dearmor -o /usr/share/keyrings/infinispan-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/infinispan-archive-keyring.gpg] https://download.jboss.org/infinispan/packages/apt stable main" | sudo tee /etc/apt/sources.list.d/infinispan.list
sudo apt update && sudo apt install infinispan-cli

# Scoop (Windows)
scoop bucket add infinispan https://github.com/infinispan/scoop-bucket && scoop install infinispan-cli

# From server distribution
bin/cli.sh   # Linux/macOS
bin\cli.bat  # Windows
```

## Connecting to Infinispan Server

### Local Connection

```bash
# Start CLI then connect (prompts for credentials)
bin/cli.sh
connect

# Connect to non-default port (e.g., offset 100)
connect 127.0.0.1:11322
```

### Auto-Connect

```bash
# Set auto-connect URL so CLI connects on startup
bin/cli.sh config set autoconnect-url http://admin:password@localhost:11222

# With basic auth in URL
config set autoconnect-url http://user:pass@hostname:11222

# With OAuth token
config set autoconnect-url http://TOKEN@hostname:11222
```

### TLS/SSL Connection

```bash
# Configure truststore for server certificate validation
bin/cli.sh config set truststore /path/to/truststore.jks
bin/cli.sh config set truststore-password secret

# Configure client keystore for mutual TLS
bin/cli.sh config set keystore /path/to/keystore.p12
bin/cli.sh config set keystore-password secret

# Verify TLS settings
bin/cli.sh config get truststore
bin/cli.sh config get truststore-password
```

### Container Connection

```bash
# Run CLI in a container on the host network
podman run -it --net=host infinispan/cli
# Then: connect (prompts for credentials)

# Or with docker
docker run -it --net=host infinispan/cli
```

### CLI Configuration

```bash
# Custom CLI storage directory
bin/cli.sh -Dcli.dir=/path/to/cli/storage
# Or via environment variable
export ISPN_CLI_DIR=/path/to/cli/storage

# Get/set config properties
bin/cli.sh config set <property> <value>
bin/cli.sh config get <property>
```

## Navigation

The CLI uses a hierarchical resource tree. Navigate with `cd`, list with `ls`, inspect with `describe`.

```bash
# Resource tree structure
[//containers/default]> ls
caches, counters, configurations, schemas, tasks

# Navigate into caches
[//containers/default]> cd caches
[//containers/default/caches]> ls

# Navigate into a specific cache
cd caches/mycache

# View resource details
describe
describe caches/mycache

# Use tab for auto-completion
# Use -h on any command for help
```

**Top-level resources:** `containers`, `cluster`, `server`

## Cache Operations

### Creating Caches

```bash
# From a built-in template
create cache --template=org.infinispan.DIST_SYNC mycache

# From an XML file
create cache --file=mycache.xml mycache

# Inline XML
create cache distcache "<distributed-cache />"

# Inline JSON
create cache jsoncache '{"distributed-cache":{"mode":"SYNC"}}'

# Verify
ls caches
describe caches/mycache
```

### Adding and Retrieving Entries

```bash
# Put with --cache flag
put --cache=mycache hello world

# Or navigate to cache context first
cd caches/mycache
[//containers/default/caches/mycache]> put hello world

# Get an entry
get hello
# Returns: world

# Put with encoding and from file
put --cache=people --encoding=application/json --file=person.json personOne

# Compare-and-swap
cas <key> <old-value> <new-value>
```

### Removing Entries and Caches

```bash
# Remove a specific entry
remove --cache=mycache hello

# Clear all entries from a cache
clearcache mycache

# Drop (delete) an entire cache
drop cache mycache
```

### Modifying Cache Configuration

```bash
# Alter cache from file
alter --file=mycache.xml mycache

# Alter a single attribute
alter --attribute=<attr> --value=<val> mycache
```

### Rebalancing

```bash
# Disable rebalancing for all caches
rebalance disable

# Enable rebalancing for all caches
rebalance enable

# Enable for a specific cache
rebalance enable caches/mycache
```

### Topology

```bash
# Set current topology as stable
topology set-stable mycache

# Force (when missing nodes >= numOwners)
topology set-stable mycache -f
```

### Querying

```bash
# Upload a Protobuf schema
schema upload --file=person.proto person.proto

# Run an Ickle query
query "from org.infinispan.example.Person p WHERE p.name='John' ORDER BY p.age ASC"

# Query with pagination
query --cache=people --max-results=20 --offset=0 "from Person"
```

## Statistics

```bash
# Container-level statistics
stats

# Cache-level statistics
stats /containers/default/caches/mycache

# From cache context
cd caches/mycache
stats
```

Output is JSON containing hit/miss counts, average read/write times, number of entries, etc.

## Counter Operations

### Creating Counters

```bash
# Strong counter (consistent, supports CAS)
create counter --initial-value=3 --storage=PERSISTENT --type=strong my-strong-counter

# Weak counter (faster, eventually consistent)
create counter --concurrency-level=1 --initial-value=5 --storage=PERSISTENT --type=weak my-weak-counter
```

**Flags:**
- `--type`: `strong` or `weak`
- `--initial-value`: starting value
- `--storage`: `PERSISTENT` (survives restart) or `VOLATILE`
- `--concurrency-level`: concurrency hint (weak counters only)

### Counter Operations

```bash
# List counters
ls counters

# Describe a counter
describe counters/my-weak-counter

# Select a counter context
counter my-weak-counter

# Add a delta (increment)
add --delta=2

# Subtract (decrement)
add --delta=-4

# Suppress return value (strong counters)
add --delta=3 --quiet=true

# Reset to initial value
reset

# Drop a counter
drop counter my-strong-counter
```

## Backup and Restore

### Creating Backups

```bash
# Auto-generated backup name
backup create

# Named backup
backup create -n my-backup

# Custom directory on server
backup create -d /some/server/dir

# Backup only caches and templates
backup create --caches=* --templates=*

# Backup specific proto schemas
backup create --proto-schemas=schema1,schema2

# List available backups
backup ls

# Download a backup archive
backup get my-backup

# Delete a backup from server
backup delete my-backup
```

### Restoring Backups

```bash
# Restore from server path
backup restore /path/on/server/backup.zip

# Upload and restore from local path
backup restore -u /local/path/backup.zip

# Restore only caches
backup restore /path/on/server/backup.zip --caches=*
```

**Important:** The target container name must match the container name in the backup archive.

## Batch Mode

### File-Based Batch

```bash
# Run a batch file
bin/cli.sh -f batch.cli

# Multiple batch files
bin/cli.sh -f batch1.cli -f batch2.cli
```

Example batch file (`batch.cli`):
```
connect --username=admin --password=changeme localhost:11222
create cache --template=org.infinispan.DIST_SYNC mybatch
put --cache=mybatch hello world
put --cache=mybatch hola mundo
ls caches/mybatch
disconnect
```

Batch files support `${property}` system property expansion.

### Interactive Batch (stdin)

```bash
# Pipe commands via stdin
bin/cli.sh -c localhost:11222 -f -
```

## User Management

### Creating and Managing Users

```bash
# Create a user with group membership
user create admin -p changeme --groups=admin

# List users
user ls

# List groups
user ls --groups

# Describe a user
user describe admin

# Remove a user
user remove admin

# Add user to group
user groups john --groups=developers
```

### Role Management

```bash
# Grant roles to a user
user roles grant --roles=deployer katie

# Revoke roles from a user
user roles deny --roles=temprole username

# List roles for a user
user roles ls katie

# List all roles
user roles ls

# Create a custom role
user roles create --permissions=ALL_WRITE,LISTEN myrole
user roles create --permissions=ALL_READ,ALL_WRITE simple

# Describe a role
user roles describe simple

# Remove a role
user remove roles temprole
```

### Flushing Security Caches

```bash
# Flush ACL caches cluster-wide after property realm changes
server aclcache flush
```

**Important:** Property realm credentials are local to each node. Changes to users, groups, or roles are not automatically distributed across nodes.

## Benchmarking

```bash
# Basic benchmark against local server
benchmark hotrod://localhost:11222

# Benchmark with custom value size and specific cache
benchmark --value-size=10000 --cache=largecache hotrod://localhost:11222

# Full benchmark with all modes and 20 threads
benchmark --mode=All --threads=20 https://user:password@server:11222

# Using a saved bookmark name
benchmark prod
```

**Supported protocols:** `http`, `https`, `hotrod`, `hotrods`, `redis`, `rediss`

**Key flags:**

| Flag | Default | Description |
|------|---------|-------------|
| `--threads` | 10 | Number of concurrent threads |
| `--cache` | benchmark | Cache name to benchmark |
| `--key-size` | 16 | Key size in bytes |
| `--value-size` | 1000 | Value size in bytes |
| `--keyset-size` | 1000 | Number of distinct keys |
| `--count` | 5 | Number of measurement iterations |
| `--time` | 10s | Duration of each iteration |
| `--warmup-count` | 5 | Number of warmup iterations |
| `--warmup-time` | 1s | Duration of each warmup iteration |
| `--mode` | Throughput | Throughput, AverageTime, SampleTime, SingleShotTime, All |
| `--verbosity` | NORMAL | SILENT, NORMAL, EXTRA |
| `--time-unit` | MICROSECONDS | NANOSECONDS, MICROSECONDS, MILLISECONDS, SECONDS |

The recommended metric is **throughput**.

## Server Reports and Diagnostics

### Diagnostic Report

```bash
# Download a diagnostic report
server report
```

Downloads a `tar.gz` archive (`infinispan-<hostname>-<timestamp>-report.tar.gz`) containing:
- **Host info:** CPU, memory, disk, OS release
- **Network info:** IP addresses, multicast, routes, TCP/UDP sockets
- **Server info:** thread dump, server logs, configurations

### Heap Dumps

```bash
# Full heap dump
server heap-dump

# Live objects only
server heap-dump --live
```

### Connection and Connector Management

```bash
# List active connections
server connections ls
server connections ls --global

# List connectors
server connector ls

# Describe a connector
server connector describe endpoint-default

# Start/stop a connector
server connector start endpoint-default
server connector stop endpoint-default
```

### Datasource Management

```bash
# List datasources
server datasource ls

# Test a datasource
server datasource test my-datasource
```

### Security and Principals

```bash
# List authenticated principals
server principals ls

# Flush ACL caches
server aclcache flush
```

### IP Filtering

```bash
# List IP filter rules
server connector ipfilter ls endpoint-default

# Set IP filter rules
server connector ipfilter set endpoint-default --rules=ACCEPT/192.168.0.0/16,REJECT/10.0.0.0/8

# Clear all IP filter rules
server connector ipfilter clear endpoint-default
```

### Shutdown

```bash
# Graceful cluster shutdown (saves cluster state)
shutdown cluster

# Stop an individual server
shutdown server <hostname>
```

## Cross-Site Operations

```bash
# Check backup site status for a cache
site status --cache=mycache --site=NYC

# Check all backup locations
site status --cache=mycache

# Bring a backup site online
site bring-online --cache=mycache --site=NYC

# Take a backup site offline
site take-offline --cache=mycache --site=NYC

# Push state to a backup site
site push-site-state --cache=mycache --site=NYC

# Use --all-caches flag for all caches
site status --all-caches --site=NYC

# Get state transfer mode
site state-transfer-mode get --cache=mycache --site=NYC

# Set automatic state transfer
site state-transfer-mode set --cache=mycache --site=NYC --mode=AUTO

# Cancel ongoing state push/receive
site cancel-push-state --cache=mycache --site=NYC
site cancel-receive-state --cache=mycache --site=NYC

# Check push status
site push-site-status --cache=mycache

# Site info commands
site name            # local site name
site view            # list all sites
site is-relay-node   # check if current node is a relay node
site relay-nodes     # list relay nodes
```

## Logging Control

```bash
# List appenders (returns JSON)
logging list-appenders

# List logger configurations (returns JSON)
logging list-loggers

# Set log level for a package
logging set --level=DEBUG org.infinispan

# Set level and direct to specific appender
logging set --level=DEBUG --appenders=FILE org.infinispan

# Remove a logger config (falls back to root)
logging remove org.infinispan
```

**Log levels:** OFF, TRACE, DEBUG, INFO, WARN, ERROR, ALL

**Note:** Logging changes are runtime-only and are not persisted to `log4j2.xml`.

### Access Logs

Enable access logging for specific protocols:

```bash
logging set --level=TRACE org.infinispan.HOTROD_ACCESS_LOG
logging set --level=TRACE org.infinispan.REST_ACCESS_LOG
logging set --level=TRACE org.infinispan.MEMCACHED_ACCESS_LOG
logging set --level=TRACE org.infinispan.RESP_ACCESS_LOG
```

## Scripting and Tasks

```bash
# Upload a script
task upload --file=multiplication.js multiplication

# Execute a task with parameters
task exec multiplier.js -Pmultiplicand=10 -Pmultiplier=20

# Execute a built-in task
task exec @@cache@names

# List available tasks
ls tasks
```

Scripts use a header comment for metadata: `// mode=local,language=javascript`

## Troubleshooting Commands

### Access Log Analysis

```bash
# Analyze access log statistics grouped by operations
troubleshoot log access.log access.log.1

# Group stats by client
troubleshoot log --by-client access.log

# Show top 10 longest operations
troubleshoot log -t 10 access.log

# Filter by operation, time range
troubleshoot log -o PUT --start 'dd/MMM/yyyy:HH:mm:ss' --end '...' access.log

# Exclude specific operations, group by client
troubleshoot log -x GET,ERROR --by-client access.log
```

**Flags:** `--operation`, `--excludeOperations`, `--highest`, `--duration`, `--by-client`, `--start`, `--end`

### Persistent State Inspection

```bash
# List persistent states
troubleshoot persistent-state server/data

# Show specific state
troubleshoot persistent-state server/data --show org.infinispan.CONFIG

# Delete specific state
troubleshoot persistent-state server/data --delete org.infinispan.CONFIG
```

## Command Aliases

```bash
# Create an alias
alias q=quit

# List all aliases
alias

# Remove an alias
unalias q
```

## RAFT Membership

```bash
# List RAFT members
raft list

# Add a RAFT member
raft add NODE

# Remove a RAFT member
raft remove NODE
```

RAFT operations require quorum on every state machine.

## Complete Command Reference

| Command | Description |
|---------|-------------|
| `add` | Add counter delta |
| `alias` / `unalias` | Manage command aliases |
| `alter` | Modify cache configuration |
| `availability` | Check/set cache availability |
| `backup` | Create, restore, list, delete backups |
| `benchmark` | Run performance benchmarks |
| `bind` | Bind a variable |
| `cache` | Select cache context |
| `cas` | Compare-and-swap |
| `cd` | Navigate resource tree |
| `clearcache` | Clear all cache entries |
| `config` | Get/set CLI configuration properties |
| `connect` / `disconnect` | Connect to / disconnect from server |
| `container` | Select container |
| `counter` | Select counter context |
| `create` | Create caches, counters |
| `credentials` | Manage credentials |
| `describe` | Describe a resource |
| `drop` | Drop caches, counters |
| `encoding` | Set default encoding |
| `get` | Retrieve cache entry |
| `help` | Display help for commands |
| `index` | Manage cache indexes |
| `install` | Install server artifacts |
| `logging` | Manage server logging |
| `ls` | List resources |
| `migrate` | Migrate data between clusters |
| `patch` | Apply server patches |
| `put` | Add or update cache entry |
| `query` | Run Ickle queries |
| `quit` | Exit the CLI |
| `rebalance` | Enable/disable cache rebalancing |
| `remove` | Remove cache entry |
| `reset` | Reset counter |
| `schema` | Upload/manage Protobuf schemas |
| `server` | Server operations (report, heap-dump, connectors, datasources, IP filters) |
| `shutdown` | Shut down server or cluster |
| `site` | Cross-site replication management |
| `stats` | View statistics |
| `task` | Upload and execute scripts/tasks |
| `troubleshoot` | Access log analysis, persistent state inspection |
| `user` | User and role management |
| `version` | Display CLI version |

Use `help <command>` in the CLI session to view the manual page for any command.

## Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| Connection refused | Verify server is running and port is correct (default 11222) |
| Authentication failure | Check username/password; use `user create` to add credentials |
| TLS handshake failure | Verify truststore path and password with `config get truststore` |
| "Cache not found" errors | Use `ls caches` to list available caches; names are case-sensitive |
| Backup restore fails | Container name in backup must match target container name |
| User changes not visible | Property realm is per-node; run `server aclcache flush` after changes |
| Batch file fails silently | Check system property expansion; verify connection in batch file |

## Official Docs

- CLI guide: https://infinispan.org/docs/stable/titles/cli/cli.html
- Server guide: https://infinispan.org/docs/stable/titles/server/server.html
