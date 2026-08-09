# Kubernetes bootstrap

This directory contains the namespace and policy baseline for the `lab-k8s` learning cluster.

## Apply through Yandex Cloud

1. Start `lab-k8s` and wait until the cluster is `Running` and `Healthy` and all three worker nodes are `Ready`.
2. Open the cluster's Kubernetes object console in Yandex Cloud.
3. Create objects from `00-namespaces.yaml`.
4. After all three namespaces become `Active`, create objects from `10-resource-policies.yaml`.
5. Create objects from `20-network-policies.yaml`.

Apply the files in this order. Each file contains multiple YAML documents separated by `---`.

## Verify

- Namespaces: `platform`, `staging`, and `production` are `Active`.
- Resource quotas: each namespace contains `namespace-resources`.
- Limit ranges: each namespace contains `container-resources`.
- Network policies: `staging` and `production` each contain `default-deny-ingress`, `default-deny-egress`, and `allow-dns-egress`.
- There are no application pods at this stage.
- The cluster remains `Healthy` and all three worker nodes remain `Ready`.

The default-deny policies intentionally block application traffic. Apply future workload-specific allow policies together with the corresponding Deployments.

## Pause the lab

When you finish the session, stop the entire Kubernetes cluster from the Yandex Cloud console. Starting the cluster again will preserve these Kubernetes objects.

## Rollback while namespaces are empty

1. Delete the six NetworkPolicy objects.
2. Delete the three `container-resources` LimitRange objects.
3. Delete the three `namespace-resources` ResourceQuota objects.
4. Delete `platform`, `staging`, and `production` only if they still contain no required workloads.

Deleting a namespace also deletes the namespaced objects inside it. Do not use namespace deletion as routine rollback after applications are deployed.
