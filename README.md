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
| Operator            | `operator/central-marketplace.yaml`               | `dsc/data-space-connector`  | 10.3.2  | Central marketplace (BAE + IdP)        |
| Operator            | `operator/onboarding-portal.yaml`                 | onboarding-portal chart     | —       | Participant onboarding UI              |
| Consumer            | `consumer/consumer.yaml`                          | `dsc/data-space-connector`  | 10.3.2  |                                        |
| Provider            | `provider/provider.yaml`                          | `dsc/data-space-connector`  | 10.3.2  |                                        |
| Provider            | `provider/provider-central-marketplace.yaml`      | `dsc/data-space-connector`  | 10.3.2  | Provider as central-marketplace peer   |
| Provider (overlay)  | `provider/fdsc-dashboard.yaml`                    | —                           | —       | Dashboard ingress overlay (nginx)      |
| Consumer + Provider | `consumer-and-provider/consumer-and-provider.yaml`| `dsc/data-space-connector`  | 10.3.2  |                                        |

## Prerequisites

- Kubernetes cluster (the examples assume AWS EKS — `gp2` StorageClass; adapt for other clouds)
- Helm 3
- `nginx` ingress controller
- `cert-manager` with a `prod` ClusterIssuer reachable from your namespaces
- An `issuance-secret` Secret pre-provisioned (sealed-secrets / Vault / External Secrets)
  containing at least `keycloak-admin` and `store-pass` keys

## Install

```sh
helm repo add dsc https://fiware.github.io/data-space-connector/
helm repo update
```

Operator (trust anchor):
```sh
helm install trust-anchor dsc/trust-anchor -n trust-anchor --create-namespace \
  -f operator/trust-anchor.yaml --version 1.0.0
```

Consumer:
```sh
helm install consumer dsc/data-space-connector -n consumer --create-namespace \
  -f consumer/consumer.yaml --version 10.3.2
```

Provider:
```sh
helm install provider dsc/data-space-connector -n provider --create-namespace \
  -f provider/provider.yaml --version 10.3.2
```

Consumer + Provider (note the release name `coprov` is referenced by service URLs inside the values file):
```sh
helm install coprov dsc/data-space-connector -n coprov --create-namespace \
  -f consumer-and-provider/consumer-and-provider.yaml --version 10.3.2
```

Operator (central marketplace) and a provider that peers with it (release names are
referenced by service URLs inside the values files):
```sh
helm install central-marketplace dsc/data-space-connector -n central-marketplace --create-namespace \
  -f operator/central-marketplace.yaml --version 10.3.2

helm install provider-central-mp dsc/data-space-connector -n provider-central-mp --create-namespace \
  -f provider/provider-central-marketplace.yaml --version 10.3.2
```

## What each role brings up

- **Operator** — Trusted Issuers Registry (TIR) + Postgres.
- **Consumer** — Keycloak (VC issuer for own users), DID helper, Postgres.
- **Provider** — Verifier + CCS + TIL, APISIX + OPA + ODRL-PAP, Keycloak, DID,
  Scorpio data plane, TMForum APIs, Marketplace (BAE), contract-management,
  dashboard.
- **Consumer + Provider** — Same as Provider plus extra Keycloak realm config
  (`OperatorCredential` issuance + per-peer client) so this participant can
  also consume from other providers.

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

## References

- [DSC roles documentation](https://github.com/FIWARE/data-space-connector/tree/main/doc/deployment-integration/roles)
- [DSC chart source](https://github.com/FIWARE/data-space-connector/tree/main/charts/data-space-connector)
- [Local k3s examples (testing/dev only)](https://github.com/FIWARE/data-space-connector/tree/main/k3s)
