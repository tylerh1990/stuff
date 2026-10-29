# Dex homelab proof of concept

Installs the upstream Dex OCI chart `0.26.0` (Dex `2.46.0`) from
`oci://ghcr.io/dexidp/helm-charts/dex` using Argo CD's OCI artifact source
with `path: .`. It runs two issuer replicas and a three-instance
CloudNativePG database. Each database instance has a 1 Gi PVC. Two separate
OAuth2 Proxy endpoints test the `dex-test` and `dex-test2` static OIDC clients
and forward authenticated requests to the existing `whoami` service. Grafana
and Argo CD continue using ZITADEL.

## Complete before reconciling

The AD host, bind DN, and domain search bases are configured as literals in
`kustomization.yaml`. Kustomize creates the `ldap-config` ConfigMap, and the
Dex chart loads it into the pod through `envFrom`. Dex expands those variables
in `config-values.yaml`. Confirm that the account can search users and groups, and
that users have `cn`, `sAMAccountName`, and `userPrincipalName` attributes.
Dex uses `userPrincipalName` for the OIDC email claim because the POC user
accounts do not all have `mail`. Adjust the search and
attribute mapping if your directory differs. Users sign in with their short
AD username (`sAMAccountName`); group lookup uses direct AD membership. Nested
groups need additional LDAP configuration.

Create these entries in the Infisical `kubernetes` project, `prod` environment:

| Path | Value |
| --- | --- |
| `/platform/dex/ldap-bind-password` | Password of the AD search account. |
| `/platform/dex/ldap-ca-pem` | PEM certificate of the CA that signs the AD LDAPS endpoint. |
| `/platform/dex/test-client-secret` | Random confidential OIDC client secret. |
| `/platform/dex/test-cookie-secret` | Base64-encoded 32-byte random OAuth2 Proxy cookie secret. |
| `/platform/dex/test2-client-secret` | Distinct random confidential OIDC client secret for `dex-test2`. |
| `/platform/dex/test2-cookie-secret` | Distinct base64-encoded 32-byte random OAuth2 Proxy cookie secret. |

CNPG generates `dex-db-app` with the `dex` database owner's username and
password. Dex reads both keys directly from that Secret. For the AD bind
password, use a random value made from letters, digits, `_`, and `-`: Dex
expands environment variables before parsing its YAML configuration, and
quotes or newlines could invalidate it. ESO creates the remaining Kubernetes
Secrets; do not commit secret values to Git.

Dex settings live in `config-values.yaml`. Argo CD passes this file to the
OCI chart, which generates the mounted configuration Secret and restarts Dex
when its contents change. `oauth2.skipApprovalScreen` is enabled so test users
proceed directly to the client after signing in. The test proxy sets
`approval_prompt=auto` because its default of `force` overrides Dex's setting.
The generated LDAP ConfigMap uses a stable name because the Helm source cannot
reference Kustomize's generated hash suffix. After changing a literal in
`kustomization.yaml`, restart the Dex Deployment so its environment variables
are refreshed.

Dex authentication sessions are enabled with `DEX_SESSIONS_ENABLED=true` and
stored in PostgreSQL. The default SSO sharing policy is `all`, so clients
without an explicit `ssoSharedWith` policy can reuse each other's Dex login.
A browser session is
valid for up to 24 hours and expires after one hour of inactivity. The two
OAuth2 Proxy endpoints use separate client secrets and separate cookies.

Dex uses `ssl.mode: verify-full` for its PostgreSQL connection. Its Pod mounts
only the public `ca.crt` key from CNPG's generated `dex-db-ca` Secret to verify
the database server. CNPG manages the server certificate and also uses TLS for
its own replication and operator traffic.

Ensure `dex.k8s.hasapes.net`, `dex-test.k8s.hasapes.net`, and
`dex-test2.k8s.hasapes.net` resolve to the existing Gateway and are covered by
its wildcard TLS certificate. Dex must be
able to reach the AD host on TCP 636, and OAuth2 Proxy must be able to reach
the public Dex issuer URL for OIDC discovery.

## Placement and recovery

Dex uses soft hostname and zone topology spread. CNPG uses preferred pod
anti-affinity across hostnames and soft zone topology spread. At implementation
time the live nodes report zone values `pve-01` and `proxmox`, so the zone
preference can affect placement. Confirm those values represent real failure
domains before treating placement as availability-zone resilience. Node labels
are owned by `talos-infrastructure`.
CNPG provides automatic primary failover; this POC does not configure database
backups or restore.

The chart creates a ServiceMonitor in `dex` for its telemetry port on 5558.
Prometheus discovers it through the `release: kube-prometheus-stack` label and
scrapes `/metrics` every 30 seconds.

## Verify

```bash
kubectl -n dex get externalsecret
kubectl -n dex get cluster dex-db
kubectl -n dex get pods -o wide
kubectl -n dex get httproute
kubectl -n dex get servicemonitor dex
curl -fsS https://dex.k8s.hasapes.net/dex/.well-known/openid-configuration
```

Open `https://dex-test.k8s.hasapes.net/` and sign in with a homelab AD user.
The protected whoami response should include `X-Forwarded-User`,
`X-Forwarded-Email`, and `X-Forwarded-Groups` with the expected AD values.
To test SSO, start a new private browser session after this configuration has
synced. Sign in at `https://dex-test.k8s.hasapes.net/`, then visit
`https://dex-test2.k8s.hasapes.net/` in the same browser session. The second
client should reach whoami without asking for AD credentials again. The
second proxy has its own cookie, so this verifies Dex's shared session.
Remove one Dex pod to check issuer availability, then perform a
controlled CNPG primary switchover and confirm login recovers. Keep the
generated Secrets and database volume claims when removing the POC until
their data is no longer needed.

After rotating a credential in Infisical, wait for the ExternalSecret refresh
and restart the affected Dex or OAuth2 Proxy Deployment so its environment
variables are refreshed. CNPG owns the `dex-db-app` database credentials.
