# Security Configuration Verification Checklist

Use this checklist to verify that Infinispan security configurations are correct and complete before deployment.

## Authentication Verification

- [ ] A security realm is defined in the server configuration
- [ ] The `security-realm` attribute is set on the `endpoints` element
- [ ] Authentication mechanisms are appropriate for the security realm type:
  - Property/LDAP realm: `SCRAM-SHA-512`, `DIGEST-SHA-512`, or `PLAIN` (with TLS only)
  - Trust store realm: `EXTERNAL` (Hot Rod) / `CLIENT_CERT` (REST)
  - Token realm: `OAUTHBEARER` (Hot Rod) / `BEARER_TOKEN` (REST)
  - Kerberos realm: `GSSAPI` or `GS2-KRB5` (Hot Rod) / `SPNEGO` (REST)
- [ ] Users are created with the CLI: `bin/cli.sh user create <username> -p <password> -g <role>`
- [ ] Property files (`users.properties`, `groups.properties`) are synchronized across all cluster nodes
- [ ] For LDAP with Active Directory: `direct-verification="true"` is set and `PLAIN`/`BASIC` mechanisms are used (or Kerberos)
- [ ] For LDAP: the principal has sufficient privileges for LDAP queries

## Authorization Verification

- [ ] Authorization is enabled in the `cache-container` security section
- [ ] Roles are defined with appropriate permissions (principle of least privilege)
- [ ] Cache-level authorization is configured where needed (implicit or explicit roles)
- [ ] Users are assigned to correct groups in `groups.properties` or LDAP
- [ ] For client certificate authorization: `group-only-mapping="false"` is set
- [ ] For client certificate authorization: `common-name-role-mapper` is configured
- [ ] Custom roles use only the minimum permissions required

## TLS/SSL Verification

- [ ] Keystore exists at the configured path under `server/conf`
- [ ] Keystore password is correct
- [ ] Keystore alias matches the configured alias
- [ ] Certificate is not expired
- [ ] Certificate hostname matches the server hostname (or SNI is configured)
- [ ] `generate-self-signed-certificate-host` is NOT used in production
- [ ] For mTLS: `require-ssl-client-auth="true"` is set on the endpoint
- [ ] For mTLS: trust store is configured with client certificates or CA certificate
- [ ] PEM trust stores use empty password: `password=""`
- [ ] TLSv1.3 is preferred for best performance

## Credential Store Verification

- [ ] Credential keystore file exists and is accessible
- [ ] Credential keystore is readable only by the Infinispan server process user
- [ ] All sensitive passwords use credential references instead of plain text
- [ ] `credential-reference` elements specify correct `store` and `alias` values
- [ ] Keystore password is masked or provided via external command (not clear text in production)

## Cluster Security Verification

- [ ] Cluster transport uses a dedicated security realm (separate from endpoints)
- [ ] Cluster transport keystore contains valid certificates
- [ ] If using ASYM_ENCRYPT: keystores are provided for certificate authentication
- [ ] If using SYM_ENCRYPT: shared keystore is distributed to all nodes
- [ ] On startup, verify log message: `ISPN080060: SSL Transport using realm <name>`

## Audit Logging Verification

- [ ] `org.infinispan.AUDIT` logging category is set to `INFO` in `log4j2.xml`
- [ ] `audit.log` file is being written to `server/log` directory
- [ ] Audit log file rotation is configured to prevent disk space exhaustion

## Common Verification Commands

```bash
# Create a user
bin/cli.sh user create testuser -p testpass -g application

# List users
bin/cli.sh user ls

# Test Hot Rod connection with authentication
bin/cli.sh connect --username=testuser --password=testpass

# Flush security caches after property realm changes
bin/cli.sh -c "server aclcache flush"

# Check credentials in a keystore
bin/cli.sh credentials ls -p "keystorepassword"

# View user ACL via REST
curl -u admin:password https://localhost:11222/rest/v2/security/user/acl

# List roles via REST
curl -u admin:password https://localhost:11222/rest/v2/security/roles

# Flush security cache via REST
curl -u admin:password https://localhost:11222/rest/v2/security/cache?action=flush
```

## Security Smoke Test Sequence

1. Start the server and verify no authentication/SSL errors in logs
2. Attempt connection with valid credentials -- should succeed
3. Attempt connection with invalid credentials -- should fail with authentication error
4. Attempt an operation that exceeds the user's role permissions -- should fail with authorization error
5. Verify audit log entries are being recorded (if audit logging is enabled)
6. For TLS: verify certificate chain with `openssl s_client -connect host:11222`
7. For cluster security: verify the `ISPN080060` log message on each node
