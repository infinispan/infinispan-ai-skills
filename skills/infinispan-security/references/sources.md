# Infinispan Security - References

## Official Documentation

- Security guide: https://infinispan.org/docs/stable/titles/security/security.html
- Server guide: https://infinispan.org/docs/stable/titles/server/server.html
- REST API reference: https://infinispan.org/docs/stable/titles/rest/rest.html
- Hot Rod client guide: https://infinispan.org/docs/stable/titles/hotrod_java/hotrod_java.html

## Key Topics

| Topic | Documentation Section |
|-------|----------------------|
| Security realms | Security guide > Configuring security realms |
| SASL mechanisms | Security guide > SASL authentication mechanisms |
| SASL policies | Security guide > SASL QoP and policies |
| Authorization (RBAC) | Security guide > Configuring authorization |
| Built-in roles | Security guide > Default user roles and permissions |
| Custom roles | Security guide > Adding custom roles |
| Cache-level authorization | Security guide > Configuring cache authorization |
| TLS/SSL keystores | Server guide > Configuring TLS/SSL encryption |
| Auto-generated keystores | Server guide > Automatically generated keystores |
| Client certificate auth | Security guide > Configuring client certificate authentication |
| Client certificate authz | Security guide > Configuring client certificate authorization |
| Credential stores | Security guide > Setting up credential keystores |
| Audit logging | Security guide > Enabling audit logging |
| JGroups encryption | Security guide > Encrypting cluster traffic |
| REST security API | REST API guide > Security endpoints |

## Built-in Roles Reference

| Role | Permissions |
|------|------------|
| `admin` | ALL |
| `deployer` | ALL_READ, ALL_WRITE, LISTEN, EXEC, MONITOR, CREATE, BULK_READ, BULK_WRITE |
| `application` | ALL_READ, ALL_WRITE, LISTEN, EXEC, BULK_READ, BULK_WRITE |
| `observer` | ALL_READ, LISTEN, BULK_READ |
| `monitor` | MONITOR |

## Provenance

This skill targets **Infinispan 16.x**. Content was derived from the official Infinispan documentation and verified against the source repository.
