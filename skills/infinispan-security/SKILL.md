---
name: infinispan-security
description: Use when user needs help with Infinispan security - authentication, authorization, TLS/SSL, security realms, SASL mechanisms, Kerberos, client certificates, credential stores, audit logging.
---

# Infinispan Security Guide

Help users secure Infinispan deployments by understanding their requirements, explaining security concepts and trade-offs, and providing ready-to-use configuration snippets.

## Workflow

1. Ask what aspect of security the user needs help with
2. Determine if this is server-side (remote caches) or embedded (library mode)
3. Determine the config format they prefer (XML, YAML, or programmatic Java)
4. Explain relevant concepts, mechanisms, and trade-offs
5. Provide concrete configuration snippets
6. Warn about common pitfalls and anti-patterns

## Security Overview

Infinispan provides security across three layers:

| Layer | Description |
|-------|-------------|
| **Core library (embedded)** | Role-based access control (RBAC) for CacheManagers, caches, and data |
| **Remote protocols** | Authentication of client requests and encryption of network traffic |
| **Cluster transport** | Authentication of new cluster members and encryption of JGroups traffic |

Infinispan uses standard Java security libraries: JAAS, JSSE, JCA, JCE, and SASL.

## Security Realms

Security realms integrate Infinispan Server with infrastructure that controls access and verifies user identities. When you add security realms to configuration, Infinispan Server automatically enables matching authentication mechanisms for Hot Rod and REST endpoints.

### Realm Types

| Realm Type | Description | Use When |
|------------|-------------|----------|
| **Property realm** | Uses `users.properties` and `groups.properties` files | Simple deployments, development, small user sets |
| **LDAP realm** | Connects to LDAP/Active Directory servers | Enterprise environments with centralized identity management |
| **Token realm** | Validates OAuth2 tokens via RFC-7662 introspection | OAuth/OIDC environments, Keycloak integration |
| **Trust store realm** | Authenticates via client certificates (mTLS) | High-security environments requiring mutual TLS |
| **Kerberos** | Uses Kerberos tickets from a KDC | Enterprise environments with Kerberos/Active Directory |
| **Distributed realm** | Combines multiple realm types together | Mixed authentication requirements |
| **Aggregate realm** | Combines multiple realm types for identity and group loading | When identity and groups come from different sources |
| **Local realm** | Trusts local OS-level connections | CLI access on the same host |

### Property Realm (XML)

```xml
<server xmlns="urn:infinispan:server:16.0">
  <security>
    <security-realms>
      <security-realm name="default">
        <properties-realm groups-attribute="Roles">
          <user-properties path="users.properties"
                           relative-to="infinispan.server.config.path"
                           plain-text="true"/>
          <group-properties path="groups.properties"
                            relative-to="infinispan.server.config.path"/>
        </properties-realm>
      </security-realm>
    </security-realms>
  </security>
</server>
```

### Property Realm (YAML)

```yaml
server:
  security:
    securityRealms:
      - name: "default"
        propertiesRealm:
          groupsAttribute: "Roles"
          userProperties:
            path: "users.properties"
            relative-to: "infinispan.server.config.path"
            plainText: "true"
          groupProperties:
            path: "groups.properties"
            relative-to: "infinispan.server.config.path"
```

**Important:** Adding credentials to a property realm with the CLI creates the user only on the server instance you are connected to. You must manually synchronize property realm credentials to each node in the cluster.

Create users with the CLI:

```bash
bin/cli.sh user create myuser -p changeme -g application
```

### LDAP Realm (XML)

```xml
<server xmlns="urn:infinispan:server:16.0">
  <security>
    <security-realms>
      <security-realm name="ldap-realm">
        <ldap-realm url="ldap://my-ldap-server:10389"
                    principal="uid=admin,ou=People,dc=infinispan,dc=org"
                    credential="strongPassword"
                    connection-timeout="3s"
                    read-timeout="30s"
                    connection-pooling="true"
                    referral-mode="ignore"
                    page-size="30"
                    direct-verification="true">
          <identity-mapping rdn-identifier="uid"
                            search-dn="ou=People,dc=infinispan,dc=org"
                            search-recursive="false">
            <attribute-mapping>
              <attribute from="cn" to="Roles"
                         filter="(&amp;(objectClass=groupOfNames)(member={1}))"
                         filter-dn="ou=Roles,dc=infinispan,dc=org"/>
            </attribute-mapping>
          </identity-mapping>
        </ldap-realm>
      </security-realm>
    </security-realms>
  </security>
</server>
```

**LDAP authentication methods:**
- **Hashed password comparison**: Compares the hashed password stored in the user's `userPassword` attribute
- **Direct verification** (`direct-verification="true"`): Authenticates against the LDAP server using supplied credentials. Required for Active Directory since access to the `password` attribute is forbidden.

**Active Directory note:** With `direct-verification`, you must use `BASIC`/`PLAIN` authentication mechanisms, or use Kerberos for `SPNEGO`/`GSSAPI`/`GS2-KRB5`.

**Group membership mapping:**
- Standard LDAP (groupOfNames): Use `attribute` filter element
- Active Directory (memberOf): Use `<attribute-reference reference="memberOf" from="cn" to="Roles"/>`

**Performance tip:** Enable `connection-pooling="true"` to significantly improve LDAP authentication performance.

### Token Realm - OAuth2/Keycloak (XML)

```xml
<server xmlns="urn:infinispan:server:16.0">
  <security>
    <security-realms>
      <security-realm name="token-realm">
        <token-realm name="token"
                     auth-server-url="https://oauth-server/auth/">
          <oauth2-introspection introspection-url="https://oauth-server/auth/realms/infinispan/protocol/openid-connect/token/introspect"
                                client-id="infinispan-server"
                                client-secret="1fdca4ec-c416-47e0-867a-3d471af7050f"/>
        </token-realm>
      </security-realm>
    </security-realms>
  </security>
</server>
```

If the introspection URL uses HTTPS, configure a server identity with an appropriate truststore and reference it in the `oauth2-introspection.client-ssl-context` attribute.

### Kerberos Realm (XML)

```xml
<server xmlns="urn:infinispan:server:16.0">
  <security>
    <security-realms>
      <security-realm name="kerberos-realm">
        <server-identities>
          <kerberos keytab-path="hotrod.keytab"
                    principal="hotrod/datagrid@INFINISPAN.ORG"
                    required="true"/>
          <kerberos keytab-path="http.keytab"
                    principal="HTTP/localhost@INFINISPAN.ORG"
                    required="true"/>
        </server-identities>
      </security-realm>
    </security-realms>
  </security>
  <endpoints>
    <endpoint socket-binding="default"
              security-realm="kerberos-realm">
      <hotrod-connector>
        <authentication>
          <sasl server-name="datagrid"
                server-principal="hotrod/datagrid@INFINISPAN.ORG"/>
        </authentication>
      </hotrod-connector>
      <rest-connector>
        <authentication server-principal="HTTP/localhost@INFINISPAN.ORG"/>
      </rest-connector>
    </endpoint>
  </endpoints>
</server>
```

### Kerberos Realm (YAML)

```yaml
server:
  security:
    securityRealms:
      - name: "kerberos-realm"
        serverIdentities:
          - kerberos:
              principal: "hotrod/datagrid@INFINISPAN.ORG"
              keytabPath: "hotrod.keytab"
              required: "true"
          - kerberos:
              principal: "HTTP/localhost@INFINISPAN.ORG"
              keytabPath: "http.keytab"
              required: "true"
  endpoints:
    endpoint:
      socketBinding: "default"
      securityRealm: "kerberos-realm"
      hotrodConnector:
        authentication:
          sasl:
            serverName: "datagrid"
            serverPrincipal: "hotrod/datagrid@INFINISPAN.ORG"
      restConnector:
        authentication:
          serverPrincipal: "HTTP/localhost@INFINISPAN.ORG"
```

## Authentication

### Per-Protocol Authentication Mechanisms

| SASL Mechanism (Hot Rod/Memcached) | HTTP Mechanism (REST) | Security Realm | Performance Impact |
|---|---|---|---|
| `PLAIN` | `BASIC` | Property, LDAP | Fastest but least secure. Only use with TLS/SSL. |
| `DIGEST-SHA-256/384/512` | `DIGEST` | Property, LDAP | SHA-based hashing, no plain-text credentials on the wire. Recommended with TLS. |
| `SCRAM-SHA-256/384/512` | `DIGEST` | Property, LDAP | Salt + hashing + nonce. More secure than DIGEST, slightly slower. |
| `GSSAPI` / `GS2-KRB5` | `SPNEGO` | Kerberos | Authentication offloaded to KDC. Performance depends on KDC quality. |
| `OAUTHBEARER` | `BEARER_TOKEN` | Token | Federated identity. Lower server penalty; depends on identity provider. |
| `EXTERNAL` | `CLIENT_CERT` | Trust store | Client certificate authentication. Performance varies by trust store size. |

**Key difference:** Infinispan validates credentials once per user session over Hot Rod, but potentially for every request over HTTP/REST.

### Memcached Authentication

- **Text protocol**: Authentication via initial `set` command with username and password concatenated with a space. Requires a realm that supports plain-text.
- **Binary protocol**: Authentication via dedicated SASL challenge/response operations. Works with all Infinispan security realm types.

### Configuring Endpoint Authentication (XML)

```xml
<endpoints>
  <endpoint socket-binding="default"
            security-realm="default">
    <hotrod-connector>
      <authentication>
        <sasl mechanisms="SCRAM-SHA-512 DIGEST-SHA-512 PLAIN"
              server-name="infinispan"
              qop="auth"/>
      </authentication>
    </hotrod-connector>
    <rest-connector>
      <authentication mechanisms="DIGEST BASIC"/>
    </rest-connector>
  </endpoint>
</endpoints>
```

### SASL Quality of Protection (QoP)

| QoP Value | Description |
|-----------|-------------|
| `auth` | Authentication only (default) |
| `auth-int` | Authentication with integrity protection |
| `auth-conf` | Authentication with integrity and confidentiality |

### Disabling Authentication

For local development or isolated networks only:

```xml
<server xmlns="urn:infinispan:server:16.0">
  <security>
    <security-realms>
      <security-realm name="default">
        <properties-realm groups-attribute="Roles">
          <user-properties path="users.properties"
                           relative-to="infinispan.server.config.path"/>
          <group-properties path="groups.properties"
                            relative-to="infinispan.server.config.path"/>
        </properties-realm>
      </security-realm>
    </security-realms>
  </security>
  <!-- No security-realm attribute on the endpoint -->
  <endpoints socket-binding="default"/>
</server>
```

**Warning:** When you disable authentication, also remove all `authorization` elements from the cache-container and cache configurations.

## Authorization (RBAC)

### Built-in Roles

| Role | Permissions | Description |
|------|-------------|-------------|
| `admin` | ALL | Superuser with all permissions including Cache Manager lifecycle |
| `deployer` | ALL_READ, ALL_WRITE, LISTEN, EXEC, MONITOR, CREATE | Can create and delete Infinispan resources |
| `application` | ALL_READ, ALL_WRITE, LISTEN, EXEC, MONITOR | Read/write access, event listening, task execution |
| `observer` | ALL_READ, MONITOR | Read-only access plus monitoring |
| `monitor` | MONITOR | View statistics via JMX and metrics endpoint only |

### Cache Manager Permissions

| Permission | Function | Description |
|------------|----------|-------------|
| CONFIGURATION | `defineConfiguration` | Define new cache configurations |
| LISTEN | `addListener` | Register listeners on Cache Manager |
| LIFECYCLE | `stop` | Stop the Cache Manager |
| CREATE | `createCache`, `removeCache` | Create and remove caches, counters, schemas, scripts |
| MONITOR | `getStats` | Access JMX statistics and metrics endpoint |
| ALL | - | All Cache Manager permissions |

### Cache Permissions

| Permission | Function | Description |
|------------|----------|-------------|
| READ | `get`, `contains` | Retrieve entries |
| WRITE | `put`, `putIfAbsent`, `replace`, `remove`, `evict` | Write, replace, remove, evict entries |
| EXEC | `distexec`, `streams` | Execute code against a cache |
| LISTEN | `addListener` | Register listeners on a cache |
| BULK_READ | `keySet`, `values`, `entrySet`, `query` | Bulk retrieve operations |
| BULK_WRITE | `clear`, `putAll` | Bulk write operations |
| LIFECYCLE | `start`, `stop` | Start and stop a cache |
| ADMIN | various | Access underlying components and internal structures |
| MONITOR | `getStats` | Access JMX statistics and metrics |
| ALL | - | All cache permissions |
| ALL_READ | - | Combines READ and BULK_READ |
| ALL_WRITE | - | Combines WRITE and BULK_WRITE |

### Configuring Custom Roles (XML)

```xml
<infinispan>
  <cache-container name="secured">
    <security>
      <authorization>
        <role name="admin" permissions="ALL"/>
        <role name="reader" permissions="READ BULK_READ"/>
        <role name="writer" permissions="READ WRITE BULK_READ"/>
        <role name="supervisor" permissions="READ WRITE BULK_READ BULK_WRITE LISTEN"/>
      </authorization>
    </security>
  </cache-container>
</infinispan>
```

### Cache-Level Authorization (XML)

Implicit (all roles from Cache Manager apply):

```xml
<distributed-cache name="secured">
  <security>
    <authorization/>
  </security>
</distributed-cache>
```

Explicit (only specified roles can access this cache):

```xml
<distributed-cache name="restricted">
  <security>
    <authorization roles="admin supervisor"/>
  </security>
</distributed-cache>
```

### Authorization (YAML)

```yaml
distributedCache:
  name: "secured"
  security:
    authorization:
      enabled: true
      roles: "admin supervisor"
```

### Managing Roles via REST API

```bash
# List all roles
GET /rest/v2/security/roles

# Get roles for a principal
GET /rest/v2/security/roles/{principal}

# Grant roles to a principal
PUT /rest/v2/security/roles/{principal}?role=deployer&role=application

# Deny (remove) roles from a principal
DELETE /rest/v2/security/roles/{principal}?role=deployer

# Create a custom role
POST /rest/v2/security/roles/{roleName}?permission=READ&permission=WRITE

# List principals
GET /rest/v2/security/principals

# View current user ACL
GET /rest/v2/security/user/acl

# Flush security caches across the cluster
GET /rest/v2/security/cache?action=flush
```

## TLS/SSL Encryption

### Configuring a Server Keystore (XML)

```xml
<server xmlns="urn:infinispan:server:16.0">
  <security>
    <security-realms>
      <security-realm name="default">
        <server-identities>
          <ssl>
            <keystore path="server.p12"
                      relative-to="infinispan.server.config.path"
                      password="secret"
                      alias="my-server"/>
          </ssl>
        </server-identities>
        <properties-realm groups-attribute="Roles">
          <user-properties path="users.properties"
                           relative-to="infinispan.server.config.path"/>
          <group-properties path="groups.properties"
                            relative-to="infinispan.server.config.path"/>
        </properties-realm>
      </security-realm>
    </security-realms>
  </security>
</server>
```

### Configuring a Server Keystore (YAML)

```yaml
server:
  security:
    securityRealms:
      - name: "default"
        serverIdentities:
          ssl:
            keystore:
              path: "server.p12"
              relative-to: "infinispan.server.config.path"
              password: "secret"
              alias: "my-server"
        propertiesRealm:
          groupsAttribute: "Roles"
          userProperties:
            path: "users.properties"
            relative-to: "infinispan.server.config.path"
          groupProperties:
            path: "groups.properties"
            relative-to: "infinispan.server.config.path"
```

### Auto-Generated Keystore (Development Only)

```xml
<server-identities>
  <ssl>
    <keystore path="server.p12"
              relative-to="infinispan.server.config.path"
              password="secret"
              alias="server"
              generate-self-signed-certificate-host="localhost"/>
  </ssl>
</server-identities>
```

**Warning:** Auto-generated self-signed certificates are for development only. Use proper CA-signed certificates in production.

### TLS Engine Configuration

Control TLS protocol versions, cipher suites, and named groups:

```xml
<ssl>
  <keystore path="server.p12" password="secret" alias="server"/>
  <engine enabled-protocols="TLSv1.3 TLSv1.2"
          enabled-ciphersuites="TLS_DHE_RSA_WITH_AES_128_CBC_SHA256"
          enabled-ciphersuites-tls13="TLS_AES_256_GCM_SHA384:TLS_AES_128_GCM_SHA256"/>
</ssl>
```

**WARNING:** If you modify the `engine` element, you MUST explicitly set `enabled-protocols`. Omitting it allows any TLS version, including insecure ones. Never enable TLS versions below 1.2.

### Performance Notes on Encryption

- TLS/SSL handshakes carry a slight performance penalty and increased latency during connection establishment
- Once connections are established, encryption latency is not a concern
- Use TLSv1.3 for best performance
- Java 17+ TLS performance is on par with native implementations

## Client Certificate Authentication (mTLS)

### Two Trust Store Approaches

1. **CA-only trust store**: Contains only the signing CA certificate. Any client with a certificate signed by the CA can connect. Simpler to manage but grants broader trust.
2. **Full trust store**: Contains the CA certificate plus all individual client certificates. Only clients with certificates present in the trust store can connect. More secure but more overhead.

### Client Certificate Authentication (XML)

```xml
<server xmlns="urn:infinispan:server:16.0">
  <security>
    <security-realms>
      <security-realm name="trust-store-realm">
        <server-identities>
          <ssl>
            <keystore path="server.p12"
                      relative-to="infinispan.server.config.path"
                      keystore-password="secret"
                      alias="server"/>
            <truststore path="trust.p12"
                        relative-to="infinispan.server.config.path"
                        password="secret"/>
          </ssl>
        </server-identities>
        <!-- Authenticates each client certificate against the trust store -->
        <truststore-realm/>
      </security-realm>
    </security-realms>
  </security>
  <endpoints>
    <endpoint socket-binding="default"
              security-realm="trust-store-realm"
              require-ssl-client-auth="true">
      <hotrod-connector>
        <authentication>
          <sasl mechanisms="EXTERNAL"
                server-name="infinispan"
                qop="auth"/>
        </authentication>
      </hotrod-connector>
      <rest-connector>
        <authentication mechanisms="CLIENT_CERT"/>
      </rest-connector>
    </endpoint>
  </endpoints>
</server>
```

### Client Certificate Authentication (YAML)

```yaml
server:
  security:
    securityRealms:
      - name: "trust-store-realm"
        serverIdentities:
          ssl:
            keystore:
              path: "server.p12"
              relative-to: "infinispan.server.config.path"
              keystore-password: "secret"
              alias: "server"
            truststore:
              path: "trust.p12"
              relative-to: "infinispan.server.config.path"
              password: "secret"
        truststoreRealm: ~
  endpoints:
    socketBinding: "default"
    securityRealm: "trust-store-realm"
    requireSslClientAuth: "true"
    connectors:
      - hotrod:
          hotrodConnector:
            authentication:
              sasl:
                mechanisms: "EXTERNAL"
                serverName: "infinispan"
                qop: "auth"
      - rest:
          restConnector:
            authentication:
              mechanisms: "CLIENT_CERT"
```

**Note:** PEM files can be used as trust stores if they contain one or more certificates. Configure them with an empty password: `password=""`.

### Client Certificate Authorization

Map the Common Name (CN) from client certificates to Infinispan roles:

```xml
<infinispan>
  <cache-container name="certificate-authentication" statistics="true">
    <security>
      <authorization group-only-mapping="false">
        <common-name-role-mapper/>
        <!-- If a client certificate contains CN=Client1,
             clients with matching certificates get ALL permissions -->
        <role name="Client1" permissions="ALL"/>
      </authorization>
    </security>
  </cache-container>
</infinispan>
```

**Important:** Infinispan extracts the certificate principal from the CN field. Subject Alternative Names (SANs) are currently ignored. The `group-only-mapping` attribute must be set to `false`.

## Credential Stores

Credential stores encrypt sensitive passwords (database credentials, LDAP passwords, keystore passwords) so they do not appear in plain text in configuration files.

### Setting Up a Credential Keystore

```bash
# Create a credential keystore with an alias
bin/cli.sh credentials add dbpassword -c changeme -p "secret1234!"

# List aliases in the keystore
bin/cli.sh credentials ls -p "secret1234!"

# Mask the keystore password for additional security
bin/cli.sh credentials mask -i 100 -s pepper99 "secret1234!"
```

Masked passwords use Password Based Encryption (PBE) and must follow the format: `<MASKED_VALUE;SALT;ITERATION>`.

### Credential Store Configuration (XML)

```xml
<server xmlns="urn:infinispan:server:16.0">
  <security>
    <credential-stores>
      <credential-store name="credentials" path="credentials.pfx">
        <clear-text-credential clear-text="secret1234!"/>
      </credential-store>
    </credential-stores>
  </security>
</server>
```

### Credential Store Configuration (YAML)

```yaml
server:
  security:
    credentialStores:
      - name: credentials
        path: credentials.pfx
        clearTextCredential:
          clearText: "secret1234!"
```

### Referencing Credentials in LDAP Realm (XML)

```xml
<server xmlns="urn:infinispan:server:16.0">
  <security>
    <credential-stores>
      <credential-store name="credentials" path="credentials.pfx">
        <clear-text-credential clear-text="secret1234!"/>
      </credential-store>
    </credential-stores>
    <security-realms>
      <security-realm name="default">
        <ldap-realm name="ldap"
                    url="ldap://my-ldap-server:10389"
                    principal="uid=admin,ou=People,dc=infinispan,dc=org">
          <credential-reference store="credentials" alias="ldappassword"/>
        </ldap-realm>
      </security-realm>
    </security-realms>
  </security>
</server>
```

### Referencing Credentials in Datasource (XML)

```xml
<data-sources>
  <data-source name="postgres" jndi-name="jdbc/postgres">
    <connection-factory driver="org.postgresql.Driver"
                        username="dbuser"
                        url="jdbc:postgresql://localhost:5432/mydb">
      <credential-reference store="credentials" alias="dbpassword"/>
    </connection-factory>
    <connection-pool max-size="10" min-size="1"
                     background-validation="1000"
                     idle-removal="1" initial-size="1"
                     leak-detection="10000"/>
  </data-source>
</data-sources>
```

**Best practice:** Credential keystores should be readable only by the user who runs the Infinispan Server process.

## Audit Logging

Audit logs track changes to your Infinispan Server deployment, recording security events and administrative operations.

### Enabling Audit Logging

Edit `server/conf/log4j2.xml` and set the `org.infinispan.AUDIT` logging category to `INFO`:

```xml
<!-- Set to INFO to enable audit logging -->
<Logger name="org.infinispan.AUDIT" additivity="false" level="INFO">
   <AppenderRef ref="AUDIT-FILE"/>
</Logger>
```

Audit messages are written to `audit.log` in the `server/log` directory.

### Custom Audit Destinations

Redirect audit events to Kafka, syslog, or JDBC via Log4j appenders:

```xml
<!-- Kafka appender for audit events -->
<Kafka name="AUDIT-KAFKA" topic="audit">
  <PatternLayout pattern="%date %message"/>
  <Property name="bootstrap.servers">localhost:9092</Property>
</Kafka>

<Logger name="org.infinispan.AUDIT" additivity="false" level="INFO">
   <AppenderRef ref="AUDIT-KAFKA"/>
</Logger>
```

### MCP Audit Log Access

With the MCP server enabled (`-Dorg.infinispan.feature.mcp=true`), read audit logs via:
```
infinispan+logs://audit
```

## Cluster Security

### Securing Cluster Transport with TLS

Create a dedicated security realm for cluster transport (separate from endpoint security):

```xml
<server xmlns="urn:infinispan:server:16.0">
  <security>
    <security-realms>
      <security-realm name="cluster-transport">
        <server-identities>
          <ssl>
            <keystore path="server.pfx"
                      relative-to="infinispan.server.config.path"
                      password="secret"
                      alias="server"/>
          </ssl>
        </server-identities>
      </security-realm>
    </security-realms>
  </security>
</server>
```

Reference the security realm in the transport configuration:

```xml
<infinispan>
  <cache-container>
    <transport server:security-realm="cluster-transport"/>
  </cache-container>
</infinispan>
```

**Verification:** On startup, look for the log message:
```
[org.infinispan.SERVER] ISPN080060: SSL Transport using realm <security_realm_name>
```

### JGroups Encryption Protocols

| Protocol | Description | Use When |
|----------|-------------|----------|
| **ASYM_ENCRYPT** | Coordinator generates and distributes secret keys to joining nodes | Need automatic key rotation when membership changes |
| **SYM_ENCRYPT** | All nodes use a shared secret key from a keystore | Simpler setup, faster than ASYM_ENCRYPT |

**ASYM_ENCRYPT flow:**
1. Coordinator node generates a secret key
2. Joining node performs certificate authentication with the coordinator
3. Joining node requests the secret key (sends its public key)
4. Coordinator encrypts the secret key with the joining node's public key
5. Joining node decrypts and installs the secret key
6. Node joins the cluster and encrypts/decrypts messages with the secret key

**Important:** When using ASYM_ENCRYPT, always provide keystores for certificate authentication to protect against man-in-the-middle attacks.

**SYM_ENCRYPT** is faster because nodes do not need to exchange keys, but you are responsible for generating and distributing the shared keystore to all nodes. There is no automatic key rotation when cluster membership changes.

### Symmetric Encryption - SYM_ENCRYPT (XML)

```xml
<infinispan>
  <jgroups>
    <stack name="encrypt-tcp" extends="tcp">
      <SYM_ENCRYPT keystore_name="myKeystore.p12"
                   keystore_type="PKCS12"
                   store_password="changeit"
                   key_password="changeit"
                   alias="myKey"
                   stack.combine="INSERT_AFTER"
                   stack.position="VERIFY_SUSPECT2"/>
    </stack>
  </jgroups>
  <cache-container name="default" statistics="true">
    <transport cluster="${infinispan.cluster.name}"
               stack="encrypt-tcp"
               node-name="${infinispan.node.name:}"/>
  </cache-container>
</infinispan>
```

### Asymmetric Encryption - ASYM_ENCRYPT (XML)

```xml
<infinispan>
  <jgroups>
    <stack name="encrypt-tcp" extends="tcp">
      <SSL_KEY_EXCHANGE keystore_name="mykeystore.jks"
                        keystore_password="changeit"
                        stack.combine="INSERT_AFTER"
                        stack.position="VERIFY_SUSPECT2"/>
      <ASYM_ENCRYPT asym_keylength="2048"
                    asym_algorithm="RSA"
                    change_key_on_coord_leave="false"
                    change_key_on_leave="false"
                    use_external_key_exchange="true"
                    stack.combine="INSERT_BEFORE"
                    stack.position="pbcast.NAKACK2"/>
    </stack>
  </jgroups>
  <cache-container name="default" statistics="true">
    <transport cluster="${infinispan.cluster.name}"
               stack="encrypt-tcp"
               node-name="${infinispan.node.name:}"/>
  </cache-container>
</infinispan>
```

**Verification:** On startup, verify cluster is using the encrypted stack:
```
ISPN000078: Starting JGroups channel cluster with stack encrypt-tcp
```

## Common Mistakes and Security Anti-Patterns

| Mistake | Impact | Fix |
|---------|--------|-----|
| Using `PLAIN`/`BASIC` without TLS | Credentials transmitted in clear text | Always enable TLS/SSL when using PLAIN or BASIC mechanisms |
| Using `generate-self-signed-certificate-host` in production | Self-signed certs are not trusted and vulnerable to MITM | Use CA-signed certificates in production |
| Sharing security realm between endpoints and cluster transport | Compromised endpoint credentials could access cluster transport | Create dedicated keystores and security realms for cluster transport |
| Not synchronizing property realm files across nodes | Users created on one node are not available on others | Manually copy `users.properties` and `groups.properties` to all nodes, then flush the ACL cache |
| Disabling authentication without disabling authorization | Authorization checks fail for anonymous users | Remove both `security-realm` from endpoints and `authorization` from cache configs |
| Storing passwords in plain text in server config | Configuration files expose secrets | Use credential stores to encrypt sensitive passwords |
| Using `DIGEST` without TLS on untrusted networks | Vulnerable to monkey-in-the-middle attacks | Use TLS/SSL or switch to `SCRAM-SHA-512` |
| Not enabling `connection-pooling` for LDAP | Poor authentication performance | Set `connection-pooling="true"` on the `ldap-realm` element |
| Not flushing ACL cache after property realm changes | Changes not reflected until server restart | Run `server aclcache flush` via CLI or REST API after changes |
| Using CA-only trust store when individual client verification is needed | Any client with a CA-signed cert can connect | Include all client certificates in the trust store, not just the CA |
| Forgetting `group-only-mapping="false"` for certificate authorization | Certificate CN not mapped to roles correctly | Set `group-only-mapping="false"` when using `common-name-role-mapper` |

## Official Documentation

- Security guide: https://infinispan.org/docs/stable/titles/security/security.html
- Server guide: https://infinispan.org/docs/stable/titles/server/server.html
- REST API security: https://infinispan.org/docs/stable/titles/rest/rest.html
