# vp-cloudnative-pg-cluster

![Version: 0.1.0](https://img.shields.io/badge/Version-0.1.0-informational?style=flat-square) ![Type: application](https://img.shields.io/badge/Type-application-informational?style=flat-square) ![AppVersion: 16](https://img.shields.io/badge/AppVersion-16-informational?style=flat-square)

Validated Patterns wrapper chart for the CloudNativePG cluster Helm chart

This chart wraps the upstream
[CloudNativePG `cluster` Helm chart](https://github.com/cloudnative-pg/charts/blob/main/charts/cluster/README.md)
for Validated Patterns.

## Scope

| Concern | Where it lives |
| --- | --- |
| CloudNativePG operator Subscription / OperatorGroup | ClusterGroup (pattern `applications`) |
| Target namespace | ClusterGroup |
| PostgreSQL `Cluster`, `Database` CRs, and related upstream resources | Packaged `cluster` subchart (`cluster.*` values) |
| Bootstrap and role credentials | This chart: ExternalSecret resources; pattern `values-secret` supplies or generates material |

This chart does **not** install the CloudNativePG operator, create its
namespace, or create an Operator Subscription.

Red Hat does not support CloudNativePG. Prefer an externally supported
PostgreSQL instance when a supported database is required.

## Packaging

```yaml
dependencies:
  - name: cluster
    version: "0.9.0"
    repository: https://cloudnative-pg.github.io/charts
```

The dependency version is pinned in `Chart.lock`. The chart tarball under
`charts/` is not committed; run `make helm-deps` (or `helm dependency update .`)
before lint, test, or install.

Upstream values are passed under the top-level `cluster` key (the subchart
name). Nested `cluster.cluster.*` settings map to the upstream Cluster CR
configuration. Additional databases use upstream `cluster.databases`. Managed
roles use `cluster.cluster.roles`. See the
[upstream README](https://github.com/cloudnative-pg/charts/blob/main/charts/cluster/README.md)
for the full subchart value set.

Wrapper-specific keys:

- `secretStore` — ClusterSecretStore / SecretStore reference used by ExternalSecret
- `secrets` — whether and how credentials are projected

By default the Cluster (and upstream Database resources) use sync-wave `1`, and
ExternalSecrets use sync-wave `0`. Negative sync-waves are avoided because they
can cause Argo CD problems in some situations.

When backups are enabled later, `cluster.backups.objectStore.helmHook` defaults
to `false` so ObjectStore creation works under Argo CD.

### Secrets

Validated Patterns normally keep secret material out of Git. Database passwords
default to generation through the pattern secrets loader:

```yaml
# values-secret.yaml.template (pattern repo, not this chart)
version: "2.0"
secrets:
  - name: app-secret
    vaultPrefixes:
      - hub
    fields:
      - name: username
        value: app
      - name: password
        onMissingValue: generate
        vaultPolicy: validatedPatternDefaultPolicy
      - name: database
        value: app
```

Users who prefer fixed credentials set `value:` (or `path:` / `ini_file:`)
instead of `onMissingValue: generate`.

This chart projects backend entries into namespace Secrets with ExternalSecret
objects (default store `vault-backend` / `ClusterSecretStore`). Secret names are
taken from:

1. `cluster.cluster.initdb.secret.name`
2. `cluster.cluster.roles[].passwordSecret.name`
3. `secrets.additional`

Each maps to remote key `<secrets.keyPrefix>/<name>` (for example
`secret/data/hub/app-secret`). Target Secrets use type
`kubernetes.io/basic-auth` and extract all properties from that backend key.

### Additional databases

The upstream chart bootstraps one application database through
`cluster.cluster.initdb`. For more databases, set `cluster.databases` and, when
owners are not the bootstrap owner, `cluster.cluster.roles` with
`passwordSecret` references. Add any app-facing Secret names that are not
already covered to `secrets.additional`.

## Example: Automation Orchestrator

Automation Orchestrator (Red Hat AAP) is one consumer of this chart. A pattern
would typically:

1. Install the CloudNativePG operator and create the target namespace from the
   ClusterGroup
2. Deploy this chart with a bootstrap database plus Temporal databases
3. Deploy the automation-orchestrator chart against the resulting service and
   Secrets

Complete example files live under [`examples/`](examples/):

- [`examples/automation-orchestrator.yaml`](examples/automation-orchestrator.yaml) — chart values
- [`examples/values-secret-automation-orchestrator.yaml`](examples/values-secret-automation-orchestrator.yaml) — pattern `values-secret` entries

Illustrative values (not defaults of this chart):

```yaml
secrets:
  enabled: true
  keyPrefix: secret/data/hub
  additional:
    - temporal-visibility-postgres-secret

cluster:
  # Align Cluster / Services / CA secret names with automation-orchestrator defaults.
  fullnameOverride: orchestrator-postgres
  type: postgresql
  mode: standalone
  version:
    postgresql: "15"
  databases:
    - name: temporal
      owner: temporal
    - name: temporal_visibility
      owner: temporal
  cluster:
    instances: 1
    storage:
      size: 5Gi
    postgresql:
      parameters:
        max_connections: "200"
    # Databases share the Cluster endpoint; one RW LoadBalancer covers
    # orchestrator, temporal, and temporal_visibility.
    # See https://cloudnative-pg.io/documentation/current/service_management/
    services:
      additional:
        - selectorType: rw
          serviceTemplate:
            metadata:
              name: orchestrator-postgres-lb
              # Opt in to Validated Patterns External DNS (external-dns-operator chart).
              # Default FQDN: <service-name>.<global.clusterDomain>
              labels:
                external-dns.validatedpatterns.io/publish: "yes"
              # Optional: override the generated name (hostnameAnnotation: Allow).
              # annotations:
              #   external-dns.alpha.kubernetes.io/hostname: postgres.apps.example.com
            spec:
              type: LoadBalancer
              ports:
                - name: postgres
                  port: 5432
                  targetPort: 5432
                  protocol: TCP
    initdb:
      database: orchestrator
      owner: orchestrator
      secret:
        name: orchestrator-postgres-secret
      postInitApplicationSQL:
        - CREATE EXTENSION IF NOT EXISTS pgcrypto;
    roles:
      - name: temporal
        ensure: present
        login: true
        passwordSecret:
          name: temporal-postgres-secret
    annotations:
      argocd.argoproj.io/sync-wave: "1"
  backups:
    enabled: false
```

Matching pattern `values-secret` entries (also in
`examples/values-secret-automation-orchestrator.yaml`):

```yaml
version: "2.0"

secrets:
  - name: orchestrator-postgres-secret
    vaultPrefixes:
      - hub
    fields:
      - name: username
        value: orchestrator
      - name: password
        onMissingValue: generate
        vaultPolicy: validatedPatternDefaultPolicy
      - name: database
        value: orchestrator
  - name: temporal-postgres-secret
    vaultPrefixes:
      - hub
    fields:
      - name: username
        value: temporal
      - name: password
        onMissingValue: generate
        vaultPolicy: validatedPatternDefaultPolicy
      - name: database
        value: temporal
  - name: temporal-visibility-postgres-secret
    vaultPrefixes:
      - hub
    fields:
      - name: username
        value: temporal
      - name: password
        onMissingValue: generate
        vaultPolicy: validatedPatternDefaultPolicy
      - name: database
        value: temporal_visibility
```

`temporal` and `temporal_visibility` share the `temporal` role; set the same
password on both secrets if you do not want independent generated passwords.
Each Kubernetes Secret can still carry its own `database` key.

The automation-orchestrator chart would then point at the CloudNativePG
read-write service (`orchestrator-postgres-rw.<namespace>.svc` with the
`fullnameOverride` above), optional LoadBalancer `orchestrator-postgres-lb`,
SSL mode `verify-ca`, the cluster CA Secret (`orchestrator-postgres-ca`), and
the backend / Temporal credential Secrets above.

## External DNS for LoadBalancer Services

The Validated Patterns
[external-dns-operator chart](https://github.com/validatedpatterns/external-dns-operator-chart)
can publish DNS names for Kubernetes Services of type `LoadBalancer`.

When that chart is installed in the pattern (ClusterGroup concern: operator
Subscription plus the `external-dns-operator` application), it creates an
`ExternalDNS` resource that:

- Watches **LoadBalancer** Services only
- Requires the opt-in label `external-dns.validatedpatterns.io/publish: "yes"`
- Builds a default FQDN as `<service-name>.<global.clusterDomain>` using the
  chart default `fqdnTemplate` (`{{.Name}}.` plus the pattern
  `global.clusterDomain`)
- Allows an explicit hostname via
  `external-dns.alpha.kubernetes.io/hostname` (`hostnameAnnotation: Allow`)
- Targets the DNS zone that is the **parent** of `global.clusterDomain`, using
  the cluster cloud credentials (`global.clusterPlatform`)

For a CloudNativePG additional service created through
`cluster.cluster.services.additional`, put the label (and optional annotation)
on the `serviceTemplate.metadata`:

```yaml
cluster:
  cluster:
    services:
      additional:
        - selectorType: rw
          serviceTemplate:
            metadata:
              name: orchestrator-postgres-lb
              labels:
                external-dns.validatedpatterns.io/publish: "yes"
              # annotations:
              #   external-dns.alpha.kubernetes.io/hostname: postgres.apps.example.com
            spec:
              type: LoadBalancer
```

With the default `fqdnTemplate` and `fullnameOverride: orchestrator-postgres`, the
published name is typically:

`orchestrator-postgres-lb.<global.clusterDomain>`

for example `orchestrator-postgres-lb.apps.cluster.example.com` when
`global.clusterDomain` is `apps.cluster.example.com`. Override with the hostname
annotation when you need a different name. Ensure the parent zone is managed in
the same cloud account the External DNS Operator uses.

See also `examples/automation-orchestrator.yaml`.

## Notable changes

### 0.1.0

- Initial Validated Patterns wrapper around upstream `cluster` 0.9.0
- External Secrets projection with pattern `onMissingValue: generate`
- Additional databases via upstream `cluster.databases` and managed roles
- Operator Subscription and namespace remain ClusterGroup concerns

**Homepage:** <https://github.com/validatedpatterns/vp-cloudnative-pg-cluster-chart>

## Maintainers

| Name | Email | Url |
| ---- | ------ | --- |
| Validated Patterns Team | <validatedpatterns@googlegroups.com> |  |

## Source Code

* <https://github.com/validatedpatterns/vp-cloudnative-pg-cluster-chart>
* <https://github.com/cloudnative-pg/charts>

## Requirements

| Repository | Name | Version |
|------------|------|---------|
| https://cloudnative-pg.github.io/charts | cluster | 0.9.0 |

## Values

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| cluster.backups.enabled | bool | `false` | Disable backups by default. Enable and configure when an object store is available in the pattern. |
| cluster.backups.objectStore.helmHook | bool | `false` | Disable Helm hooks so ObjectStore works with Argo CD when backups are enabled later. |
| cluster.cluster.annotations | object | `{"argocd.argoproj.io/sync-wave":"1"}` | Annotations applied to the Cluster and Database resources. |
| cluster.cluster.initdb | object | `{}` | Bootstrap initdb settings. Set secret.name and project that Secret with External Secrets (or provide an existing Secret). |
| cluster.cluster.instances | int | `1` | Number of PostgreSQL instances. |
| cluster.cluster.roles | list | `[]` | Declarative managed roles. Use passwordSecret.name for credentials projected by this chart. |
| cluster.cluster.storage.size | string | `"8Gi"` | Persistent volume size for PostgreSQL data. |
| cluster.cluster.storage.storageClass | string | `""` | Storage class for PostgreSQL volumes. Empty uses the cluster default. |
| cluster.databases | list | `[]` | Additional CloudNativePG Database CRs managed by the upstream subchart. See upstream docs for owner, extensions, schemas, and related fields. |
| cluster.mode | string | `"standalone"` | Cluster mode. One of standalone, replica, recovery. |
| cluster.type | string | `"postgresql"` | Database type. One of postgresql, postgis, timescaledb. |
| cluster.version.postgresql | string | `"16"` | PostgreSQL major version used by the upstream chart image selection. |
| global | object | `{}` | Global pattern values. Patterns commonly inject hub/cluster metadata here. |
| secretStore.kind | string | `"ClusterSecretStore"` | External Secrets store kind. |
| secretStore.name | string | `"vault-backend"` | External Secrets store used to project database credentials. |
| secrets.additional | list | `[]` | Extra Secret names to project beyond those inferred from initdb and roles. |
| secrets.enabled | bool | `true` | Create ExternalSecret resources for database credentials. |
| secrets.keyPrefix | string | `"secret/data/hub"` | Vault (or other backend) key prefix used by ExternalSecret remote refs. Patterns typically use secret/data/hub/<name> with the vault backend. |
| secrets.refreshInterval | string | `"15s"` | Refresh interval for ExternalSecret resources. |
| secrets.syncWave | string | `"0"` | Argo CD sync-wave for ExternalSecret resources so credentials exist before the CloudNativePG Cluster is applied. Avoid negative waves. |

----------------------------------------------
Autogenerated from chart metadata using [helm-docs v1.14.2](https://github.com/norwoodj/helm-docs/releases/v1.14.2)
