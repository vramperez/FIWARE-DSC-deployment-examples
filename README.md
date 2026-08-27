# FIWARE DSC deployment examples

Training/educational `values.yaml` examples for the
[FIWARE Data Space Connector](https://github.com/FIWARE/data-space-connector) Helm charts,
one per dataspace role.

> ⚠️ **DISCLAIMER** — These files are for training purposes only.
> Do **NOT** deploy as-is in production. Review credentials, hostnames, cert
> issuers, resource sizing, persistence, and disabled subcharts first.

## Layout

| Role                | File                                              | Chart                       | Version | Notes                                  |
| ------------------- | ------------------------------------------------- | --------------------------- | ------- | -------------------------------------- |
| Operator            | `operator/trust-anchor.yaml`                      | `dsc/trust-anchor`          | 1.0.0   |                                        |
| Operator            | `operator/central-marketplace.yaml`               | `dsc/data-space-connector`  | 10.4.12 | Central marketplace (BAE + IdP)        |
| Operator            | `operator/onboarding-portal.yaml`                 | `fiware/onboarding-portal`  | 1.4.0   | Participant onboarding UI (app 0.2.0)  |
| Consumer            | `consumer/consumer.yaml`                          | `dsc/data-space-connector`  | 10.4.12 |                                        |
| Provider            | `provider/provider.yaml`                          | `dsc/data-space-connector`  | 10.4.12 |                                        |
| Provider            | `provider/provider-central-marketplace.yaml`      | `dsc/data-space-connector`  | 10.4.12 | Provider as central-marketplace peer   |
| Provider (overlay)  | `provider/fdsc-dashboard.yaml`                    | `fdsc-dashboard` (subchart) | 0.6.8   | Operator UI overlay — see below        |
| Consumer + Provider | `consumer-and-provider/consumer-and-provider.yaml`| `dsc/data-space-connector`  | 10.4.12 |                                        |

## Prerequisites

- Kubernetes cluster (the examples assume AWS EKS — `gp2` StorageClass; adapt for other clouds).
  The Marketplace (BAE) PVCs are the exception: the examples leave them on the cluster's
  default StorageClass. Since BAE 1.1.0 (shipped in DSC 10.4.x) you can pin them explicitly
  with `marketplace.bizEcosystemChargingBackend.persistence.storageClass` and the equivalent
  key on the IDM PVC.
- Helm 3
- `nginx` ingress controller
- `cert-manager` with a `prod` ClusterIssuer reachable from your namespaces
- An `issuance-secret` Secret pre-provisioned (sealed-secrets / Vault / External Secrets)
  containing at least `keycloak-admin` and `store-pass` keys

## Install

```sh
helm repo add dsc https://fiware.github.io/data-space-connector/
# the onboarding portal lives in a different chart repo
helm repo add fiware https://fiware.github.io/helm-charts
helm repo update
```

Operator (trust anchor):
```sh
helm install trust-anchor dsc/trust-anchor -n trust-anchor --create-namespace \
  -f operator/trust-anchor.yaml --version 1.0.0
```

Operator (onboarding portal) — a standalone chart on its own release cycle, deployed
into the `central-marketplace` namespace alongside the trust anchor:
```sh
helm install onboarding-portal fiware/onboarding-portal -n central-marketplace \
  -f operator/onboarding-portal.yaml --version 1.4.0
```

Consumer:
```sh
helm install consumer dsc/data-space-connector -n consumer --create-namespace \
  -f consumer/consumer.yaml --version 10.4.12
```

Provider:
```sh
helm install provider dsc/data-space-connector -n provider --create-namespace \
  -f provider/provider.yaml --version 10.4.12
```

Consumer + Provider (note the release name `coprov` is referenced by service URLs inside the values file):
```sh
helm install coprov dsc/data-space-connector -n coprov --create-namespace \
  -f consumer-and-provider/consumer-and-provider.yaml --version 10.4.12
```

Operator (central marketplace) and a provider that peers with it (release names are
referenced by service URLs inside the values files):
```sh
helm install central-marketplace dsc/data-space-connector -n central-marketplace --create-namespace \
  -f operator/central-marketplace.yaml --version 10.4.12

helm install provider-central-mp dsc/data-space-connector -n provider-central-mp --create-namespace \
  -f provider/provider-central-marketplace.yaml --version 10.4.12
```

## Upgrading from 10.3.x

DSC **10.4.0** drops the SEAMWARE-patched Keycloak image
(`quay.io/seamware/keycloak:26.6.3`) and runs the upstream default
`docker.io/keycloak/keycloak:26.7.0` instead (CloudPirates chart `0.21.28`). The
fork only existed to carry the OID4VCI QR-endpoint fix
([keycloak/keycloak#44623](https://github.com/keycloak/keycloak/issues/44623)) and the
Liquibase changeset `26.7.0-verifiable-credential`; both are upstream now. Two
consequences:

**1. The PKCS12 keystore must live under `data/<realm-name>`.** Since Keycloak
**26.6.4** the `java-keystore` realm key provider only reads keystores from a
directory named after the realm, inside `${kc.home.dir}/data` (i.e.
`/opt/keycloak/data` on the official image). A keystore outside that boundary still
works for a read-only startup, but any create/update on the key provider fails. All
the example values files already apply the move — the directory name matches
`keycloak.realm.name`:

```yaml
keycloak:
  extraVolumeMounts:
    - { name: did-material, mountPath: /opt/keycloak/data/provider }  # was: /did-material
    - { name: realms,       mountPath: /opt/keycloak/data/import }
  signingKey:
    storePath: /opt/keycloak/data/provider/cert.pfx                   # was: /did-material/cert.pfx
```

The `get-did` initContainer keeps writing to its own `/did-material` mount of the
same `emptyDir` — only the Keycloak container's view is constrained. If you use a
`did:elsi` issuer (disabled in every example here), apply the same move to
`elsi.storePath`.

**2. An existing Keycloak database from `26.6.3` needs a manual migration.** The
`26.6.3` fork carried early, differently-named versions of the
`user_ver_credential` / `issued_ver_credential` schema changes that upstream
`26.7.0` introduces under its own changeset ids, so Liquibase checksums no longer
match on startup. Run
[`doc/scripts/migration_26.6.3_to_26.7.0.sql`](https://github.com/FIWARE/data-space-connector/blob/main/doc/scripts/migration_26.6.3_to_26.7.0.sql)
once against that database **before** starting Keycloak on `26.7.0`, and back the
database up first — it alters constraints and column types in place. This does not
apply to fresh installs or to deployments already on `26.7.0`.

Everything else in these values files renders unchanged against 10.4.12; the
`decentralized-iam` bump (2.1.17 → 2.1.19) and the Marketplace bump (BAE 1.0.4 →
1.1.0) are additive.

**3. The onboarding portal needs its own bump.** It is a separate chart, so the DSC
version says nothing about it: `fiware/onboarding-portal` **1.4.0** (app **0.2.0**) is
the first release that speaks the post-26.4 OID4VCI model, and anything older silently
fails to issue against Keycloak 26.7.0. Its values move with it — credential scopes
migrate from `config.app.keycloak.additionalClientScopes` to
`defaultRealmConfig.clientScopes` with `protocol: oid4vc`, mappers swap
`subjectProperty` for `claim.name` and drop `supportedCredentialTypes`, the format
identifier goes `vc+sd-jwt` → `dc+sd-jwt`, `vct` → `verifiable_credential_type`, and
the VC client needs `attributes: {oid4vci.enabled: "true"}`. The flat
`vc.<name>.*` realm attributes are gone. See `operator/onboarding-portal.yaml`.

## What each role brings up

- **Operator** — Trusted Issuers Registry (TIR) + Postgres.
- **Consumer** — Keycloak (VC issuer for own users), DID helper, Postgres.
- **Provider** — Verifier + CCS + TIL, APISIX + OPA + ODRL-PAP, Keycloak, DID,
  Scorpio data plane, TMForum APIs, Marketplace (BAE), contract-management,
  dashboard.
- **Consumer + Provider** — Same as Provider plus extra Keycloak realm config
  (`OperatorCredential` issuance + per-peer client) so this participant can
  also consume from other providers.

## Operator dashboard (overlay)

`provider/fdsc-dashboard.yaml` enables the `fdsc-dashboard` subchart (0.6.8, shipped
by DSC 10.4.12) and is applied as a second `-f` on top of `provider/provider.yaml`:

```sh
helm install provider dsc/data-space-connector -n provider --create-namespace \
  -f provider/provider.yaml -f provider/fdsc-dashboard.yaml --version 10.4.12
```

Besides the TIL / TIR / CCS / ODRL-PAP / APISIX views it demonstrates three optional
integrations, each of which needs something switched on elsewhere:

| Overlay key | What it adds | Depends on |
| ----------- | ------------ | ---------- |
| `grafana.*` | Embedded Grafana panels (`panelsJson` is a JSON string) | `grafana.enabled: true` — see [Observability](#observability-optional-disabled-by-default) |
| `tracing.*` | Tempo traces view | `grafana.enabled` + `tempo.enabled` |
| `keycloak.url` | Credential revocation status read from Keycloak's status-list endpoint | [Credential revocation](#credential-revocation-optional-disabled-by-default) |

Two gotchas the overlay comments repeat:

- `tracing.tempoDatasourceId` is **not** deterministic. The chart provisions the Tempo
  datasource without a fixed `uid`, so Grafana mints one per install — read it off the
  running instance (`/api/datasources/name/Tempo`) before filling it in.
- Neither `provider.yaml` nor the overlay creates the `dashboard` Keycloak client the
  UI authenticates against. The overlay lists exactly what to add under
  `keycloak.realm`; working copies live in `provider/provider-central-marketplace.yaml`
  and `consumer-and-provider/consumer-and-provider.yaml`.

## Observability (optional, disabled by default)

DSC 10.x ships an opt-in OpenTelemetry tracing stack (Collector + Grafana Tempo +
Grafana), all disabled by default. The example values keep it **off**. To enable
end-to-end tracing for any scenario, add to the values file (or a `-f` overlay):

```yaml
tracing:
  enabled: true
opentelemetry-collector:
  enabled: true
tempo:
  enabled: true
grafana:
  enabled: true
```

This wires workloads (Keycloak, Scorpio, tm-forum-api, contract-management,
marketplace, IdentityHub, fdsc-edc) → OTEL Collector → Tempo → Grafana, with the
Tempo datasource auto-provisioned. Per-component service names can be overridden
via each component's `tracing.serviceName` (e.g. `scorpio.tracing.serviceName`).
See the [DSC observability guide](https://github.com/FIWARE/data-space-connector/tree/main/doc/deployment-integration/observability).

## Credential revocation (optional, disabled by default)

DSC 10.4.x adds opt-in credential revocation via the
[Token Status List](https://datatracker.ietf.org/doc/draft-ietf-oauth-status-list/)
draft. It has two halves, both off in the example values — enable them together:

```yaml
# The status-list server itself (Rust service + Postgres + Redis).
statusListServer:
  enabled: true
  postgresql:
    host: postgres
    database: statuslistdb
    auth:
      username: statuslist
      existingSecret: statuslist.postgres.credentials.postgresql.acid.zalan.do
  env:
    serverDomain: status-list.example.com

# The Keycloak plugin that registers each issued VC with that server.
keycloak:
  tokenStatusList:
    enabled: true
    serverUrl: http://status-list-server:8081
    # credential types that become revocable
    credentialTypes: ["LegalPersonCredential", "OperatorCredential"]
```

The plugin JAR (`io.github.wistefan:keycloak-token-status-plugin`) is pulled from
Maven Central by an init container at pod startup; the server image is
`quay.io/wi_stefan/status-list-server`. Signing keys can be handed to the server as a
cert-manager Secret via `statusListServer.signing.*` instead of its built-in ACME
manager. See the [upstream server docs](https://github.com/adorsys/status-list-server)
and the [plugin docs](https://github.com/ADORSYS-GIS/token-status-link).

## Keycloak behind an outbound HTTP proxy (optional)

Also new in 10.4.x: `keycloak.proxy.*` injects the standard `HTTPS_PROXY` /
`HTTP_PROXY` / `NO_PROXY` env vars plus the Keycloak SPI proxy mapping, so Keycloak
(and the token-status-list plugin) can reach outside the cluster through a forward
proxy.

```yaml
keycloak:
  proxy:
    enabled: true
    httpsProxy: http://squid-proxy.infra.svc.cluster.local:8888
    noProxy: "localhost,*.cluster.local,*.svc"
```

## References

- [DSC roles documentation](https://github.com/FIWARE/data-space-connector/tree/main/doc/deployment-integration/roles)
- [DSC chart source](https://github.com/FIWARE/data-space-connector/tree/main/charts/data-space-connector)
- [Local k3s examples (testing/dev only)](https://github.com/FIWARE/data-space-connector/tree/main/k3s)
