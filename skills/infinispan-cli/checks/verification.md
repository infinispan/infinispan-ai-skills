# Verification Checks for Infinispan CLI Skill

Use these checks to verify that CLI guidance is accurate and complete.

## Connection Verification

- [ ] Command uses correct default port (11222)
- [ ] TLS config uses `config set truststore` / `config set keystore` (not Java system properties)
- [ ] Auto-connect URL format is correct: `http[s]://<user>:<pass>@<host>:<port>`
- [ ] Container connection uses `--net=host` for host network access

## Cache Operations Verification

- [ ] `create cache` uses `--template=` or `--file=` flag correctly
- [ ] `put`/`get`/`remove` commands use `--cache=` flag when not in cache context
- [ ] `clearcache` clears entries; `drop cache` deletes the cache itself (distinct operations)
- [ ] Inline XML/JSON cache definitions use correct quoting

## Counter Verification

- [ ] Counter type is specified (`--type=strong` or `--type=weak`)
- [ ] `--storage` is `PERSISTENT` or `VOLATILE` (not other values)
- [ ] `--concurrency-level` is only used with weak counters
- [ ] `add --delta=` is used for both increment and decrement

## Backup/Restore Verification

- [ ] `backup restore -u` flag is used for local file upload (vs server path)
- [ ] Container name matching requirement is mentioned for restore
- [ ] Selective backup flags (`--caches=*`, `--templates=*`, `--proto-schemas=`) are correct

## User Management Verification

- [ ] `user create` includes `--groups=` and `-p` flags
- [ ] Property realm per-node limitation is mentioned
- [ ] `server aclcache flush` is recommended after user changes
- [ ] Role permissions use valid values (ALL_READ, ALL_WRITE, LISTEN, etc.)

## Batch Mode Verification

- [ ] Batch file uses `-f` flag
- [ ] System property expansion `${property}` syntax is mentioned
- [ ] Interactive stdin mode uses `-f -`

## Cross-Site Verification

- [ ] `site` commands use `--cache=` and `--site=` flags
- [ ] `--all-caches` flag is mentioned as alternative to `--cache=`
- [ ] State transfer mode values are correct (AUTO, MANUAL)

## General Checks

- [ ] All commands are for Infinispan 16.x
- [ ] `help <command>` is suggested for detailed usage
- [ ] Resource tree navigation (cd/ls/describe) is explained
- [ ] Common error scenarios have troubleshooting guidance
