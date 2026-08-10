# RabbitMQ Platform Design

## Goal

Deploy a compact, production-like RabbitMQ cluster into the existing Yandex Managed Service for Kubernetes learning environment. The cluster must tolerate the loss of one RabbitMQ pod or one Kubernetes worker, persist messages on Yandex Cloud disks, keep credentials out of Git, and expose no public endpoint.

This phase prepares the messaging platform only. It does not deploy or modify the three microservices.

## Existing environment

The design targets the current `lab-k8s` cluster:

- Kubernetes `1.35`;
- three private worker nodes distributed across three availability zones;
- each worker has `2 vCPU`, `4 GiB` RAM, and no public IP;
- the `platform` namespace allows trusted platform workloads;
- the `platform` ResourceQuota allows `1500m` CPU requests, `3` CPU limits, `3Gi` memory requests, `5Gi` memory limits, and ten PVCs;
- Yandex Cloud NAT provides outbound access for image pulls;
- application namespaces have default-deny ingress and egress policies;
- the project uses plain Kubernetes manifests and manual application through `kubectl`, with later adoption by Argo CD.

The application code declares a durable, non-exclusive RabbitMQ queue without an explicit `x-queue-type`. RabbitMQ therefore has to set quorum as the server-side default queue type for the queue to be replicated without changing application code.

## Selected approach

Use the official RabbitMQ Cluster Kubernetes Operator instead of a Helm-packaged RabbitMQ installation or a handwritten StatefulSet.

The operator supplies the RabbitMQ-specific reconciliation needed for clustering, rolling updates, probes, peer discovery, and persistent-volume attachment. A handwritten StatefulSet would duplicate that operational logic. A packaged RabbitMQ Helm chart would make the learning workflow less transparent and introduce local Helm usage that is not needed for this phase.

The selected component versions are:

- cert-manager `v1.21.1`;
- RabbitMQ Cluster Operator `v2.22.3`;
- RabbitMQ Server `4.3.1-management`;
- External Secrets Operator with Yandex Lockbox support, Yandex Cloud Marketplace chart `2.5.0-2`.

The operator and cert-manager versions are pinned in the operator instructions. The repository will not copy their large upstream manifests. External Secrets Operator is installed from the official Yandex Cloud Marketplace OCI Helm chart `2.5.0-2`. The Marketplace UI was attempted first, but its product-access backend returned repeated HTTP 503 and internal gRPC timeouts; using the published chart keeps the same Yandex integration while making the installation reproducible.

cert-manager is included because RabbitMQ Cluster Operator versions `2.20` and later require it for admission-webhook certificates. It is a reusable platform prerequisite for later TLS work.

## Namespace placement

The components use three namespaces:

- `cert-manager` for cert-manager;
- `rabbitmq-system` for RabbitMQ Cluster Operator;
- `platform` for the `lab-rabbitmq` data-plane pods and project-owned RabbitMQ resources.

External Secrets Operator uses the non-default namespace selected during its Marketplace installation, named `external-secrets` in the operating guide.

Only RabbitMQ data-plane pods and their PVCs consume the existing `platform` quota. Operator workloads remain isolated in their own namespaces.

## Secret flow

RabbitMQ administrator credentials originate in Yandex Lockbox and never appear in Git.

The flow is:

1. The operator creates a Yandex Cloud service account named `lab-external-secrets`.
2. The operator creates a Lockbox secret named `lab-rabbitmq-admin` with payload entries `username` and `password`.
3. The service account receives `lockbox.payloadViewer` on that one secret, not on the folder.
4. External Secrets Operator is installed through Yandex Cloud Marketplace using one authorized key for `lab-external-secrets`.
5. A `ClusterSecretStore` references the Marketplace-managed Kubernetes Secret containing that authorized key.
6. An `ExternalSecret` reads `username` and `password` from `lab-rabbitmq-admin` and renders the Kubernetes Secret `lab-rabbitmq-default-user` in `platform`.
7. `RabbitmqCluster.spec.secretBackend.externalSecret` references `lab-rabbitmq-default-user`.

The rendered Kubernetes Secret follows the RabbitMQ service-binding format and contains:

- `default_user.conf`;
- `host` set to the internal RabbitMQ Service DNS name;
- `username`;
- `password`;
- `port` set to `5672`;
- `provider` and `type` set to `rabbitmq`.

The current Yandex Lockbox provider for External Secrets Operator supports authorized-key authentication. The key is therefore an explicit bootstrap credential. It is never committed, is scoped through least-privilege IAM, and must be revoked and deleted during teardown or rotation.

This administrator identity will not be used by application pods. A later application-delivery phase will create a dedicated virtual host and least-privilege application user.

Changing the Lockbox password is not treated as sufficient runtime rotation by itself. Password rotation will be a separate controlled operation that verifies External Secrets synchronization and the RabbitMQ Operator credential-update behavior before consumers switch to the new value.

## RabbitMQ cluster

Create one `RabbitmqCluster` named `lab-rabbitmq` in `platform` with:

- three replicas;
- RabbitMQ image release `rabbitmq:4.3.1-management`, resolved to an immutable digest during manifest implementation;
- a `ClusterIP` Service only;
- external administrator credentials from `lab-rabbitmq-default-user`;
- management and Prometheus plugins supplied by the operator-managed image;
- no public LoadBalancer or NodePort;
- no AMQP TLS in this phase.

The internal endpoints are:

- AMQP: `lab-rabbitmq.platform.svc.cluster.local:5672`;
- management API and UI: `lab-rabbitmq.platform.svc.cluster.local:15672`;
- Prometheus metrics: `lab-rabbitmq.platform.svc.cluster.local:15692`.

The management UI is accessed only with a temporary `kubectl port-forward` session.

## Queue availability

RabbitMQ configuration sets:

```text
default_queue_type = quorum
```

The existing service declares its queue as durable and non-exclusive, which is compatible with a quorum queue. On a three-node RabbitMQ cluster, a new quorum queue receives one member on each node and remains available while two members form a majority.

The cluster also sets:

```text
vm_memory_high_watermark.absolute = 512MiB
disk_free_limit.absolute = 1GB
collect_statistics_interval = 10000
```

No classic mirrored-queue or legacy partition-handling configuration is added.

## Scheduling and resources

Each RabbitMQ pod requests and limits:

```yaml
requests:
  cpu: 200m
  memory: 512Mi
limits:
  cpu: 500m
  memory: 768Mi
```

Across three replicas this consumes:

- requests: `600m` CPU and `1536Mi` memory;
- limits: `1500m` CPU and `2304Mi` memory.

This fits within the existing `platform` ResourceQuota and leaves capacity for later observability components.

Required pod anti-affinity uses `kubernetes.io/hostname`, ensuring that no worker runs more than one RabbitMQ pod. With the current topology of one worker per availability zone, this also distributes RabbitMQ across all three zones.

## Persistence

Each RabbitMQ replica receives one `5Gi` `ReadWriteOnce` PVC using `yc-network-ssd`.

The Yandex Cloud storage class uses dynamic CSI provisioning and `WaitForFirstConsumer`, so every disk is provisioned in the zone selected for its RabbitMQ pod. The three replicas therefore use three independent persistent network SSDs.

The standard `yc-network-ssd` reclaim policy is `Delete`. Final teardown must still verify both PVC removal and the absence of leftover Compute Cloud disks. Scaling RabbitMQ to zero preserves its PVCs and data but does not by itself stop Kubernetes worker billing.

## Disruption safety

Add a project-owned `PodDisruptionBudget` for `lab-rabbitmq` with:

```yaml
minAvailable: 2
```

This permits one voluntary disruption while preventing a normal node drain from voluntarily removing a second RabbitMQ pod. Operator-managed readiness and startup probes control Service membership. Persistent volumes allow a recreated pod to rejoin with its stored state.

Kubernetes cannot preserve RabbitMQ quorum after two simultaneous node losses. That is an accepted limit of the three-node learning topology.

## Network exposure

RabbitMQ receives no public IP, LoadBalancer, NodePort, or Ingress.

The current `platform` namespace intentionally has no default-deny policy because trusted platform controllers have varied requirements. Application namespaces remain default-deny. Their future deployment bundle will add narrowly scoped egress to RabbitMQ TCP `5672`, and RabbitMQ ingress restrictions will be introduced with those application identities so that policies are not based on labels that do not yet exist.

TLS for internal AMQP traffic is deferred to the later Certificate Manager and private-DNS phase. This phase first establishes clustering, storage, secrets, and recovery behavior.

## Repository layout

The project-owned resources will be stored as:

```text
deploy/k8s/platform/rabbitmq/
├── 00-external-secret.yaml
├── 10-rabbitmq-cluster.yaml
├── 20-pod-disruption-budget.yaml
└── README.md
```

`00-external-secret.yaml` is generated only after the user supplies the nonsensitive Lockbox secret ID. It contains the `ClusterSecretStore` and `ExternalSecret`, but no secret value or authorized key.

`README.md` contains pinned upstream install URLs, UI prerequisites, application order, validation, temporary management access, scale-to-zero instructions, and teardown order.

## Application order

The manual deployment order is:

1. Create `lab-external-secrets` and `lab-rabbitmq-admin` in the Yandex Cloud UI.
2. Grant the service account payload-view access only to that secret.
3. Install External Secrets Operator chart `2.5.0-2` from `oci://cr.yandex/yc-marketplace/yandex-cloud/external-secrets/charts/external-secrets` into `external-secrets` and verify the Helm release, `sa-creds`, and CRDs.
4. Install pinned cert-manager `v1.21.1` and wait until its webhook is ready.
5. Install pinned RabbitMQ Cluster Operator `v2.22.3` and wait until its webhook is ready.
6. Apply `00-external-secret.yaml` and wait for `lab-rabbitmq-default-user` to exist.
7. Apply `10-rabbitmq-cluster.yaml`.
8. Apply `20-pod-disruption-budget.yaml`.
9. Wait for three ready RabbitMQ pods and three bound PVCs.

The RabbitMQ custom resource is not applied before its external credential Secret exists.

## Validation

Repository validation must check:

- YAML parses successfully;
- Kubernetes client dry-run accepts built-in resources;
- `RabbitmqCluster` declares exactly three replicas;
- no `Secret` value or authorized key is present in Git;
- the external Secret name matches the RabbitMQ secret backend reference;
- resources fit the `platform` quota;
- the PDB selects the operator-generated RabbitMQ pod labels and requires two available pods;
- persistence uses `yc-network-ssd` with `5Gi` per replica;
- required anti-affinity targets `kubernetes.io/hostname`;
- the Service type is not public;
- quorum is the configured default queue type.

Post-deployment validation must confirm:

- cert-manager, External Secrets Operator, and RabbitMQ Cluster Operator are ready;
- `ExternalSecret` reports `Ready=True`;
- `lab-rabbitmq-default-user` exists without displaying its payload;
- three RabbitMQ pods are ready on three different workers;
- three PVCs are `Bound`;
- the RabbitMQ cluster reports three members and healthy quorum status;
- deleting one RabbitMQ pod leaves the Service available and the pod returns with its PVC;
- management UI is reachable only through a temporary port-forward;
- no public Service or external address is created.

Application behavior and business endpoints are outside this validation scope.

## Teardown and stopping

For a short pause while Kubernetes remains running, RabbitMQ Operator versions supporting scale-to-zero may scale the cluster to zero while preserving PVCs. This only releases pod resources; it does not remove disk charges or stop Kubernetes worker charges.

For final teardown:

1. remove application dependencies on RabbitMQ;
2. delete the `RabbitmqCluster` and PDB;
3. delete RabbitMQ PVCs;
4. verify that the three Compute Cloud disks were removed;
5. delete the `ExternalSecret` and project-owned secret store;
6. uninstall RabbitMQ Cluster Operator, External Secrets Operator, and cert-manager when no other component uses them;
7. delete the authorized key for `lab-external-secrets`;
8. delete the Lockbox secret and service account after confirming they have no remaining consumers.

Deleting the `platform` namespace is not used as a routine RabbitMQ teardown mechanism.
