# Kubernetes Bootstrap Design

## Goal

Create a small, reproducible Kubernetes baseline for the `restaurant-booking` learning cluster before deploying applications or platform components. The baseline will be stored in Git, applied initially through the Yandex Cloud Kubernetes UI, and later adopted by Argo CD without changing the manifests.

## Scope

This bootstrap covers only cluster policy and namespace preparation:

- namespaces `platform`, `staging`, and `production`;
- Pod Security Admission labels;
- namespace-level resource quotas;
- default container requests and limits;
- default-deny network policies for application namespaces;
- DNS egress from application namespaces;
- an operator-facing application and verification guide.

It does not deploy microservices, databases, RabbitMQ, secrets, Argo CD, Grafana, monitoring agents, logging agents, ingress, or load balancers.

## Repository layout

The manifests will be added under the existing deployment directory:

```text
deploy/k8s/bootstrap/
├── 00-namespaces.yaml
├── 10-resource-policies.yaml
├── 20-network-policies.yaml
└── README.md
```

The numeric prefixes define the manual application order. Every YAML file may contain multiple Kubernetes documents separated with `---`.

No Helm chart or local infrastructure tool is required. Argo CD can later manage this directory as a plain manifest directory.

## Namespace security model

### `platform`

The namespace hosts trusted cluster components such as Argo CD, Grafana, Prometheus Agent, and Fluent Bit.

- `pod-security.kubernetes.io/enforce: privileged`
- `pod-security.kubernetes.io/audit: restricted`
- `pod-security.kubernetes.io/warn: restricted`
- Pod Security version labels use `latest`.

The privileged enforcement level is intentional because node-level observability agents may need host mounts or elevated permissions. Only platform administrators should be allowed to deploy into this namespace. Fine-grained platform RBAC will be added with the platform components.

### `staging` and `production`

Application workloads must satisfy the restricted Pod Security profile:

- `pod-security.kubernetes.io/enforce: restricted`
- `pod-security.kubernetes.io/audit: restricted`
- `pod-security.kubernetes.io/warn: restricted`
- Pod Security version labels use `latest`.

Future Deployment manifests therefore must run as non-root, use the runtime-default seccomp profile, disable privilege escalation, and drop Linux capabilities.

## Resource policy

Quotas are ceilings, not reserved capacity. They prevent a namespace from consuming the entire three-node cluster while still allowing CPU overcommit through limits.

| Namespace | CPU requests | CPU limits | Memory requests | Memory limits | Pods | Services | ConfigMaps | Secrets | PVCs |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| `platform` | `1500m` | `3` | `3Gi` | `5Gi` | `40` | `20` | `50` | `50` | `10` |
| `staging` | `1000m` | `2` | `2Gi` | `4Gi` | `25` | `15` | `30` | `30` | `5` |
| `production` | `3000m` | `6` | `5Gi` | `8Gi` | `40` | `20` | `50` | `50` | `10` |

Each namespace receives the same `LimitRange` defaults for a container:

- default request: `100m` CPU and `128Mi` memory;
- default limit: `500m` CPU and `512Mi` memory;
- minimum: `25m` CPU and `32Mi` memory;
- maximum: `2` CPU and `2Gi` memory.

Application and platform manifests should still declare explicit resources. The defaults are a safety net for workloads that omit them.

## Network policy

The `staging` and `production` namespaces receive:

1. a policy denying all ingress by default;
2. a policy denying all egress by default;
3. a policy allowing UDP and TCP port `53` to CoreDNS pods in `kube-system`.

No application-to-application, internet, PostgreSQL, RabbitMQ, Lockbox, or monitoring traffic is allowed at this stage. Those paths will be opened later with narrowly scoped policies alongside the workloads that require them.

The `platform` namespace does not receive a default-deny policy in this bootstrap. Its controllers and node agents have varied connectivity requirements; component-specific policies will be added when each component is installed.

## Application flow

The first application is manual and uses the Kubernetes object console in Yandex Cloud:

1. Apply `00-namespaces.yaml`.
2. Apply `10-resource-policies.yaml`.
3. Apply `20-network-policies.yaml`.
4. Verify all objects and namespace labels in the console.

The network policies are applied while the application namespaces are empty, so they cannot interrupt existing workloads. Application-specific allow policies must be applied together with, or before, future Deployments.

## Validation

Before delivery, the generated manifests must pass:

- YAML syntax parsing;
- Kubernetes structural validation using an available local validator or Kubernetes client dry-run;
- checks that every namespaced object targets one of the three declared namespaces;
- checks that only `platform` uses privileged Pod Security enforcement;
- checks that `staging` and `production` have both default-deny policies and DNS egress.

After manual application, success means:

- all three namespaces are Active;
- every namespace has one ResourceQuota and one LimitRange;
- `staging` and `production` each have three NetworkPolicy objects;
- there are no application pods yet;
- the cluster remains Healthy and all three worker nodes remain Ready.

## Rollback

Before application workloads exist, rollback is performed in reverse order:

1. delete the NetworkPolicy objects;
2. delete the LimitRange and ResourceQuota objects;
3. delete the three namespaces.

Once workloads are deployed, namespace deletion is destructive and is not part of routine rollback. Argo CD will eventually own these resources and handle reconciliation from Git.
