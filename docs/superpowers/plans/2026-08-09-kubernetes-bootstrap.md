# Kubernetes Bootstrap Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a reproducible namespace, resource-policy, and network-policy baseline that can be applied through the Yandex Cloud Kubernetes UI and later managed unchanged by Argo CD.

**Architecture:** Store plain multi-document Kubernetes YAML under `deploy/k8s/bootstrap`, ordered by numeric filename prefixes. Keep application namespaces restricted and deny their network traffic by default, while leaving the trusted `platform` namespace open for component-specific policies added later.

**Tech Stack:** Kubernetes 1.35 API objects, Pod Security Admission labels, ResourceQuota, LimitRange, NetworkPolicy, Ruby YAML validation, kubectl 1.36 client-side dry-run.

## Global Constraints

- Do not modify application code, Dockerfiles, Docker Compose, migrations, or service configuration.
- Create only the four files approved in the design under `deploy/k8s/bootstrap`.
- Do not add application Deployments, Services, Secrets, databases, RabbitMQ, Argo CD, monitoring, logging, ingress, or load balancers.
- Keep the manifests usable as plain YAML through both the Yandex Cloud UI and a future Argo CD directory application.
- Use `platform`, `staging`, and `production` as the only namespaces created by this bootstrap.
- Enforce the restricted Pod Security profile in `staging` and `production`.
- Allow privileged enforcement only in `platform`, with restricted audit and warning labels.
- Do not store credentials or secret values in Git.
- Configuration validation is performed with test-first ephemeral Ruby assertions; no persistent test harness is added for these declarative files.

---

### Task 1: Namespace and Pod Security baseline

**Files:**
- Create: `deploy/k8s/bootstrap/00-namespaces.yaml`
- Test: ephemeral Ruby inventory assertion and kubectl client dry-run

**Interfaces:**
- Consumes: Kubernetes Pod Security Admission namespace labels.
- Produces: namespaces consumed by every later bootstrap manifest.

- [ ] **Step 1: Run the namespace inventory assertion before creating the file**

Run:

```bash
ruby -ryaml -e '
docs = YAML.load_stream(File.read(ARGV.fetch(0)))
expected = {
  "platform" => "privileged",
  "staging" => "restricted",
  "production" => "restricted"
}
actual = docs.to_h do |doc|
  [doc.dig("metadata", "name"), doc.dig("metadata", "labels", "pod-security.kubernetes.io/enforce")]
end
abort "namespace security mismatch: #{actual.inspect}" unless actual == expected
' deploy/k8s/bootstrap/00-namespaces.yaml
```

Expected: FAIL with `Errno::ENOENT` because the manifest does not exist yet.

- [ ] **Step 2: Create the namespace manifest**

Create `deploy/k8s/bootstrap/00-namespaces.yaml` with:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: platform
  labels:
    app.kubernetes.io/part-of: restaurant-booking
    environment: platform
    pod-security.kubernetes.io/enforce: privileged
    pod-security.kubernetes.io/enforce-version: latest
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/audit-version: latest
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/warn-version: latest
---
apiVersion: v1
kind: Namespace
metadata:
  name: staging
  labels:
    app.kubernetes.io/part-of: restaurant-booking
    environment: staging
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: latest
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/audit-version: latest
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/warn-version: latest
---
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    app.kubernetes.io/part-of: restaurant-booking
    environment: production
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: latest
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/audit-version: latest
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/warn-version: latest
```

- [ ] **Step 3: Verify the namespace manifest**

Run the Step 1 assertion again.

Expected: exit code `0` with no output.

Run:

```bash
kubectl apply --dry-run=client --validate=false -f deploy/k8s/bootstrap/00-namespaces.yaml
```

Expected: three namespace objects reported as configured or created in client dry-run mode.

- [ ] **Step 4: Commit the namespace baseline**

```bash
git add deploy/k8s/bootstrap/00-namespaces.yaml
git commit -m "feat: add kubernetes bootstrap namespaces"
```

---

### Task 2: Namespace quotas and container defaults

**Files:**
- Create: `deploy/k8s/bootstrap/10-resource-policies.yaml`
- Test: ephemeral Ruby quota assertion and kubectl client dry-run

**Interfaces:**
- Consumes: the three namespaces created by `00-namespaces.yaml`.
- Produces: one ResourceQuota and one LimitRange per namespace.

- [ ] **Step 1: Run the resource-policy assertion before creating the file**

Run:

```bash
ruby -ryaml -e '
docs = YAML.load_stream(File.read(ARGV.fetch(0)))
quotas = docs.select { |doc| doc["kind"] == "ResourceQuota" }
limits = docs.select { |doc| doc["kind"] == "LimitRange" }
expected = {
  "platform" => ["1500m", "3", "3Gi", "5Gi", "40"],
  "staging" => ["1000m", "2", "2Gi", "4Gi", "25"],
  "production" => ["3000m", "6", "5Gi", "8Gi", "40"]
}
actual = quotas.to_h do |doc|
  hard = doc.dig("spec", "hard")
  [doc.dig("metadata", "namespace"), [hard["requests.cpu"], hard["limits.cpu"], hard["requests.memory"], hard["limits.memory"], hard["pods"]]]
end
abort "quota mismatch: #{actual.inspect}" unless actual == expected
abort "expected three limit ranges" unless limits.size == 3
limits.each do |doc|
  rule = doc.dig("spec", "limits", 0)
  abort "limit defaults mismatch" unless rule.dig("defaultRequest", "cpu") == "100m" && rule.dig("default", "memory") == "512Mi"
end
' deploy/k8s/bootstrap/10-resource-policies.yaml
```

Expected: FAIL with `Errno::ENOENT` because the manifest does not exist yet.

- [ ] **Step 2: Create the resource-policy manifest**

Create `deploy/k8s/bootstrap/10-resource-policies.yaml` with:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: namespace-resources
  namespace: platform
  labels:
    app.kubernetes.io/part-of: restaurant-booking
spec:
  hard:
    requests.cpu: "1500m"
    limits.cpu: "3"
    requests.memory: 3Gi
    limits.memory: 5Gi
    pods: "40"
    services: "20"
    configmaps: "50"
    secrets: "50"
    persistentvolumeclaims: "10"
---
apiVersion: v1
kind: LimitRange
metadata:
  name: container-resources
  namespace: platform
  labels:
    app.kubernetes.io/part-of: restaurant-booking
spec:
  limits:
    - type: Container
      defaultRequest:
        cpu: 100m
        memory: 128Mi
      default:
        cpu: 500m
        memory: 512Mi
      min:
        cpu: 25m
        memory: 32Mi
      max:
        cpu: "2"
        memory: 2Gi
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: namespace-resources
  namespace: staging
  labels:
    app.kubernetes.io/part-of: restaurant-booking
spec:
  hard:
    requests.cpu: "1000m"
    limits.cpu: "2"
    requests.memory: 2Gi
    limits.memory: 4Gi
    pods: "25"
    services: "15"
    configmaps: "30"
    secrets: "30"
    persistentvolumeclaims: "5"
---
apiVersion: v1
kind: LimitRange
metadata:
  name: container-resources
  namespace: staging
  labels:
    app.kubernetes.io/part-of: restaurant-booking
spec:
  limits:
    - type: Container
      defaultRequest:
        cpu: 100m
        memory: 128Mi
      default:
        cpu: 500m
        memory: 512Mi
      min:
        cpu: 25m
        memory: 32Mi
      max:
        cpu: "2"
        memory: 2Gi
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: namespace-resources
  namespace: production
  labels:
    app.kubernetes.io/part-of: restaurant-booking
spec:
  hard:
    requests.cpu: "3000m"
    limits.cpu: "6"
    requests.memory: 5Gi
    limits.memory: 8Gi
    pods: "40"
    services: "20"
    configmaps: "50"
    secrets: "50"
    persistentvolumeclaims: "10"
---
apiVersion: v1
kind: LimitRange
metadata:
  name: container-resources
  namespace: production
  labels:
    app.kubernetes.io/part-of: restaurant-booking
spec:
  limits:
    - type: Container
      defaultRequest:
        cpu: 100m
        memory: 128Mi
      default:
        cpu: 500m
        memory: 512Mi
      min:
        cpu: 25m
        memory: 32Mi
      max:
        cpu: "2"
        memory: 2Gi
```

- [ ] **Step 3: Verify the resource policies**

Run the Step 1 assertion again.

Expected: exit code `0` with no output.

Run:

```bash
kubectl apply --dry-run=client --validate=false -f deploy/k8s/bootstrap/10-resource-policies.yaml
```

Expected: three ResourceQuota and three LimitRange objects reported in client dry-run mode.

- [ ] **Step 4: Commit the resource policies**

```bash
git add deploy/k8s/bootstrap/10-resource-policies.yaml
git commit -m "feat: add kubernetes namespace resource policies"
```

---

### Task 3: Default-deny application networking

**Files:**
- Create: `deploy/k8s/bootstrap/20-network-policies.yaml`
- Test: ephemeral Ruby policy assertion and kubectl client dry-run

**Interfaces:**
- Consumes: `staging`, `production`, and the standard `kube-system` namespace label.
- Produces: default-deny ingress, default-deny egress, and CoreDNS egress in each application namespace.

- [ ] **Step 1: Run the network-policy assertion before creating the file**

Run:

```bash
ruby -ryaml -e '
docs = YAML.load_stream(File.read(ARGV.fetch(0)))
abort "expected six policies" unless docs.size == 6
expected_names = ["allow-dns-egress", "default-deny-egress", "default-deny-ingress"]
["staging", "production"].each do |namespace|
  policies = docs.select { |doc| doc.dig("metadata", "namespace") == namespace }
  names = policies.map { |doc| doc.dig("metadata", "name") }.sort
  abort "policy inventory mismatch for #{namespace}: #{names.inspect}" unless names == expected_names
  dns = policies.find { |doc| doc.dig("metadata", "name") == "allow-dns-egress" }
  ports = dns.dig("spec", "egress", 0, "ports").map { |port| [port["protocol"], port["port"]] }.sort
  abort "DNS ports mismatch" unless ports == [["TCP", 53], ["UDP", 53]]
end
' deploy/k8s/bootstrap/20-network-policies.yaml
```

Expected: FAIL with `Errno::ENOENT` because the manifest does not exist yet.

- [ ] **Step 2: Create the network-policy manifest**

Create `deploy/k8s/bootstrap/20-network-policies.yaml` with:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: staging
  labels:
    app.kubernetes.io/part-of: restaurant-booking
spec:
  podSelector: {}
  policyTypes:
    - Ingress
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-egress
  namespace: staging
  labels:
    app.kubernetes.io/part-of: restaurant-booking
spec:
  podSelector: {}
  policyTypes:
    - Egress
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns-egress
  namespace: staging
  labels:
    app.kubernetes.io/part-of: restaurant-booking
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: production
  labels:
    app.kubernetes.io/part-of: restaurant-booking
spec:
  podSelector: {}
  policyTypes:
    - Ingress
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-egress
  namespace: production
  labels:
    app.kubernetes.io/part-of: restaurant-booking
spec:
  podSelector: {}
  policyTypes:
    - Egress
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns-egress
  namespace: production
  labels:
    app.kubernetes.io/part-of: restaurant-booking
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
```

- [ ] **Step 3: Verify the network policies**

Run the Step 1 assertion again.

Expected: exit code `0` with no output.

Run:

```bash
kubectl apply --dry-run=client --validate=false -f deploy/k8s/bootstrap/20-network-policies.yaml
```

Expected: six NetworkPolicy objects reported in client dry-run mode.

- [ ] **Step 4: Commit the network policies**

```bash
git add deploy/k8s/bootstrap/20-network-policies.yaml
git commit -m "feat: add kubernetes default network policies"
```

---

### Task 4: Yandex Cloud UI application guide and final verification

**Files:**
- Create: `deploy/k8s/bootstrap/README.md`
- Test: repository content checks and full client dry-run

**Interfaces:**
- Consumes: all three YAML manifests from Tasks 1–3.
- Produces: the exact manual application, verification, pause, and rollback instructions for the cluster operator.

- [ ] **Step 1: Run the documentation-content assertion before creating the file**

Run:

```bash
test -f deploy/k8s/bootstrap/README.md && \
rg -q '00-namespaces.yaml' deploy/k8s/bootstrap/README.md && \
rg -q '10-resource-policies.yaml' deploy/k8s/bootstrap/README.md && \
rg -q '20-network-policies.yaml' deploy/k8s/bootstrap/README.md && \
rg -q 'Yandex Cloud' deploy/k8s/bootstrap/README.md
```

Expected: FAIL because the guide does not exist yet.

- [ ] **Step 2: Create the UI application guide**

Create `deploy/k8s/bootstrap/README.md` with:

```markdown
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
```

- [ ] **Step 3: Run complete local validation**

Run the Step 1 documentation assertion again.

Expected: exit code `0`.

Run YAML parsing and inventory validation:

```bash
ruby -ryaml -e '
files = Dir["deploy/k8s/bootstrap/*.yaml"].sort
abort "expected three YAML files" unless files.size == 3
docs = files.flat_map { |file| YAML.load_stream(File.read(file)) }
counts = docs.group_by { |doc| doc["kind"] }.transform_values(&:size)
expected = {"Namespace" => 3, "ResourceQuota" => 3, "LimitRange" => 3, "NetworkPolicy" => 6}
abort "object inventory mismatch: #{counts.inspect}" unless counts == expected
allowed = [nil, "platform", "staging", "production"]
docs.each do |doc|
  namespace = doc.dig("metadata", "namespace")
  abort "unexpected namespace #{namespace}" unless allowed.include?(namespace)
end
' 
```

Expected: exit code `0` with no output.

Run client-side Kubernetes decoding for each file:

```bash
for file in deploy/k8s/bootstrap/*.yaml; do
  kubectl apply --dry-run=client --validate=false -f "$file"
done
```

Expected: all 15 objects are decoded successfully; no errors are printed.

Run repository checks:

```bash
git diff --check
git status --short
```

Expected: no whitespace errors; only `deploy/k8s/bootstrap/README.md` is uncommitted at this point.

- [ ] **Step 4: Commit the application guide**

```bash
git add deploy/k8s/bootstrap/README.md
git commit -m "docs: add kubernetes bootstrap instructions"
```

- [ ] **Step 5: Confirm the final repository state**

```bash
git status --short
git log -4 --oneline
```

Expected: clean working tree and four bootstrap commits, ending with the documentation commit.
