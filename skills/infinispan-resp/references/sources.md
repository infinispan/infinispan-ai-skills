# Infinispan RESP Endpoint - References

## Official Documentation

- RESP endpoint guide: https://infinispan.org/docs/stable/titles/resp/resp-endpoint.html
- Server guide: https://infinispan.org/docs/stable/titles/server/server.html
- Security guide: https://infinispan.org/docs/stable/titles/security/security.html

## Supported Command Categories

Infinispan's RESP endpoint supports commands across these categories:
strings, hashes, lists, sets, sorted sets, bitmaps, HyperLogLog, geo, JSON, pub/sub, transactions, scripting, cluster, bloom filters, cuckoo filters, count-min sketch, top-k, search, connection, generic, iteration.

For the full list of supported commands, see the [official RESP endpoint documentation](https://infinispan.org/docs/stable/titles/resp/resp-endpoint.html).

## Client Libraries

- Jedis (Java): https://github.com/redis/jedis
- Lettuce (Java): https://github.com/lettuce-io/lettuce-core
- Spring Data Redis: https://spring.io/projects/spring-data-redis

## Key Configuration Requirements

- Cache key encoding must be `application/octet-stream`
- Cache must use `RESPHashFunctionPartitioner` for distributed caches
- Infinispan only supports RESP version 3 (RESP3)

## Provenance

This skill targets **Infinispan 16.x**. Content was derived from the official Infinispan documentation and verified against the source repository.
