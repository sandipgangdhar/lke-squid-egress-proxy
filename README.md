# LKE Squid Egress Proxy with Kyverno Injection

## 1. Purpose

This repository provides a reusable pattern for workloads running on Linode Kubernetes Engine (LKE) that require controlled outbound **HTTP and HTTPS** access when a managed NAT Gateway is not available or is not the right fit.

The solution deploys a highly available Squid forward-proxy tier on a dedicated LKE node pool. Kyverno automatically injects standard proxy environment variables into newly created pods in explicitly labelled application namespaces.

This is an **explicit web proxy solution**, not a general-purpose NAT Gateway replacement.

### Supported traffic

- HTTP
- HTTPS through the `CONNECT` method
- Domain-based allow/deny controls, if added to Squid
- Centralized proxy access logging
- A small, identifiable set of worker-node public egress IPs

### Not covered

- Arbitrary TCP/UDP protocols
- DNS, ICMP, databases, message brokers, or applications that ignore proxy settings
- A guaranteed single outbound public IP
- Transparent interception of all pod traffic
- TLS inspection by default

## 2. Architecture

```text
Application Pod
  HTTP_PROXY / HTTPS_PROXY
          |
          v
squid-proxy.squid-proxy.svc.cluster.local:3128
          |
          v
Kubernetes ClusterIP Service
        /   \
       v     v
 Squid Pod  Squid Pod
 Node A     Node B
       \     /
        Internet
```

Each Squid pod runs on a dedicated node. Internet destinations normally see the public egress IP of the worker node that hosts the selected Squid pod. With two proxy nodes, customers should normally allowlist both public IPs.

## 3. Internal service-to-service traffic

When Deployment A calls Deployment B using a Kubernetes service name such as:

```text
http://service-b.customer-app.svc.cluster.local:8080
```

that request should bypass Squid because `.svc`, `.svc.cluster.local`, and `.cluster.local` are included in `NO_PROXY` and `no_proxy`.

Expected path:

```text
Deployment A -> Kubernetes Service B -> Deployment B
```

Not:

```text
Deployment A -> Squid -> Deployment B
```

Important limitations:

- The application or HTTP library must honor proxy environment variables.
- CIDR handling in `NO_PROXY` is not consistent across every runtime.
- Kubernetes DNS suffixes are generally more reliable than CIDR-only exclusions.
- If an internal service is called through a public or corporate FQDN, add that domain to both `NO_PROXY` and `no_proxy`.

## 4. Repository contents

```text
lke-squid-egress/
├── README.md
├── kyverno-values-ha.yaml
├── manifests/
│   ├── squid-lke.yaml
│   └── inject-squid-proxy.yaml
└── tests/
    ├── proxy-test-deployment.yaml
    └── internal-web.yaml
```

The `*-original.yaml` files preserve the initially tested versions and can be removed before publishing the repository.

## 5. Prerequisites

- Working LKE cluster and kubeconfig
- `kubectl`
- Helm 3
- Permission to create namespaces, deployments, services, PDBs, CRDs, webhooks, and ClusterPolicies
- At least two dedicated LKE worker nodes for Squid
- At least three schedulable worker nodes for fully effective Kyverno admission-controller HA
- Outbound internet connectivity from the Squid worker nodes

Validate cluster access:

```bash
kubectl cluster-info
kubectl get nodes -o wide
```

## 6. Create the dedicated LKE node pool

Suggested starting configuration:

```text
Node pool name: squid-egress
Nodes:          2
Plan:           Dedicated 4 GB or 8 GB
Label:          workload=squid-egress
Taint:          dedicated=squid-egress:NoSchedule
```

Why use a dedicated pool:

- Isolates proxy CPU, memory, and connection load
- Provides predictable egress IPs
- Prevents general workloads from occupying proxy nodes
- Allows independent scaling
- Simplifies capacity monitoring and troubleshooting

Verify labels and taints:

```bash
kubectl get nodes -L workload
kubectl describe node <SQUID-NODE-NAME> | grep -A3 Taints
```

Expected label:

```text
workload=squid-egress
```

Expected taint:

```text
dedicated=squid-egress:NoSchedule
```

Do not continue until both nodes have the expected label and taint.

## 7. Deploy Squid

Review the image in `manifests/squid-lke.yaml`. Before production use, mirror it into an approved registry, scan it, and pin it by immutable digest.

Apply:

```bash
kubectl apply -f manifests/squid-lke.yaml
```

Wait for rollout:

```bash
kubectl rollout status deployment/squid -n squid-proxy --timeout=180s
```

Check placement:

```bash
kubectl get pods -n squid-proxy -o wide
```

Expected:

- Two Running pods
- One pod on each dedicated proxy node
- `READY 1/1`

Check the service and endpoints:

```bash
kubectl get service -n squid-proxy
kubectl get endpointslice -n squid-proxy \
  -l kubernetes.io/service-name=squid-proxy
```

Check startup logs:

```bash
kubectl logs -n squid-proxy deployment/squid \
  --all-containers=true --prefix
```

Check Squid cache logs:

```bash
for pod in $(kubectl get pods -n squid-proxy \
  -l app.kubernetes.io/name=squid -o name); do
  echo "===== $pod ====="
  kubectl exec -n squid-proxy "$pod" -- \
    tail -n 30 /var/log/squid/cache.log
done
```

## 8. Manual proxy validation

Create a temporary pod:

```bash
kubectl run squid-test \
  --image=curlimages/curl:8.12.1 \
  --restart=Never \
  --command -- sleep 3600
```

Test HTTP:

```bash
kubectl exec squid-test -- \
  curl -v \
  --proxy http://squid-proxy.squid-proxy.svc.cluster.local:3128 \
  http://example.com
```

Test HTTPS:

```bash
kubectl exec squid-test -- \
  curl -v \
  --proxy http://squid-proxy.squid-proxy.svc.cluster.local:3128 \
  https://example.com
```

For HTTPS, expect:

```text
HTTP/1.1 200 Connection established
CONNECT tunnel established
```

Check the observed public egress IP:

```bash
kubectl exec squid-test -- \
  curl -s \
  --proxy http://squid-proxy.squid-proxy.svc.cluster.local:3128 \
  https://api.ipify.org
```

Run multiple new requests and document every observed IP. External systems may need to allowlist both Squid worker-node public IPs.

Delete the temporary pod:

```bash
kubectl delete pod squid-test
```

## 9. Install Kyverno in high-availability mode

Add and refresh the Helm repository:

```bash
helm repo add kyverno https://kyverno.github.io/kyverno/
helm repo update
```

Install using the included HA values:

```bash
helm upgrade --install kyverno kyverno/kyverno \
  --namespace kyverno \
  --create-namespace \
  --values kyverno-values-ha.yaml
```

The values file sets:

```yaml
admissionController:
  replicas: 3
backgroundController:
  replicas: 3
cleanupController:
  replicas: 3
reportsController:
  replicas: 3
```

The admission controller is the critical component for pod mutation. Three replicas are used so admission requests can continue during a single replica or node failure.

Verify:

```bash
kubectl get deployments -n kyverno
kubectl get pods -n kyverno -o wide
```

Expected: each controller deployment reports the requested replica count as available.

For an existing single-replica installation, use the same `helm upgrade --install` command. Do not install a second release with another name.

## 10. Apply the proxy-injection policy

Apply:

```bash
kubectl apply -f manifests/inject-squid-proxy.yaml
```

Verify:

```bash
kubectl get clusterpolicy inject-squid-proxy
kubectl describe clusterpolicy inject-squid-proxy
```

The policy:

- Mutates only newly created Pods
- Applies only in namespaces labelled `squid-proxy-injection=enabled`
- Excludes `squid-proxy`, `kube-system`, and `kyverno`
- Injects variables into regular containers
- Injects variables into init containers when present
- Adds uppercase and lowercase variants for runtime compatibility

## 11. Enable proxy injection for an application namespace

Enable:

```bash
kubectl label namespace <APPLICATION-NAMESPACE> \
  squid-proxy-injection=enabled --overwrite
```

Example:

```bash
kubectl label namespace customer-app \
  squid-proxy-injection=enabled --overwrite
```

Verify:

```bash
kubectl get namespace customer-app --show-labels
```

Existing pods are not changed. Restart application workloads after labelling:

```bash
kubectl rollout restart deployment -n customer-app
```

Review stateful workloads, Jobs, CronJobs, and custom controllers individually before restarting them.

## 12. Validate automatic injection

Apply the test workload:

```bash
kubectl apply -f tests/proxy-test-deployment.yaml
kubectl rollout status deployment/proxy-test -n customer-app
```

Check variables:

```bash
kubectl exec -n customer-app deployment/proxy-test -- \
  env | grep -i proxy
```

Expected variables:

```text
HTTP_PROXY
HTTPS_PROXY
NO_PROXY
http_proxy
https_proxy
no_proxy
```

Test internet access without `--proxy`:

```bash
kubectl exec -n customer-app deployment/proxy-test -- \
  curl -v https://api.ipify.org
```

Expected output includes:

```text
Uses proxy env variable https_proxy
CONNECT api.ipify.org:443
HTTP/1.1 200 Connection established
```

## 13. Confirm requests reached Squid

```bash
for pod in $(kubectl get pods -n squid-proxy \
  -l app.kubernetes.io/name=squid -o name); do
  echo "===== $pod ====="
  kubectl exec -n squid-proxy "$pod" -- \
    tail -n 20 /var/log/squid/access.log
done
```

A successful HTTPS request looks similar to:

```text
TCP_TUNNEL/200 ... CONNECT api.ipify.org:443
```

`NONE_NONE/000 error:transaction-end-before-headers` entries are commonly produced by TCP liveness/readiness probes which connect and close without sending HTTP headers. They are normally harmless.

## 14. Validate internal traffic bypass

Deploy an internal service:

```bash
kubectl apply -f tests/internal-web.yaml
kubectl rollout status deployment/internal-web -n customer-app
```

Call it from the proxy-test pod:

```bash
kubectl exec -n customer-app deployment/proxy-test -- \
  curl -v http://internal-web.customer-app.svc.cluster.local
```

Expected:

- Curl reports matching `no_proxy`
- Curl connects directly to the internal Kubernetes Service IP on port 80
- Curl does not connect to the Squid service on port 3128
- No corresponding internal-web request appears in Squid access logs

This validates that service-to-service traffic stays inside the cluster.

## 15. Add customer-specific NO_PROXY domains

Edit `manifests/inject-squid-proxy.yaml` and append domains to **both** `NO_PROXY` and `no_proxy`.

Example:

```text
.customer.internal,.corp.example.com,metadata.internal.example
```

Use a leading dot when all subdomains should bypass the proxy:

```text
.corp.example.com
```

Add exact hostnames when only one endpoint should bypass:

```text
auth.corp.example.com
```

After changing the policy:

```bash
kubectl apply -f manifests/inject-squid-proxy.yaml
kubectl rollout restart deployment -n <APPLICATION-NAMESPACE>
```

Why restart: environment variables are fixed when a pod is created. Updating a Kyverno policy does not modify existing pods.

Recommended items to include in `NO_PROXY`:

- Kubernetes service suffixes
- Pod/service/private network ranges, after runtime testing
- Internal corporate domains
- Private database endpoints
- Private object-storage or API endpoints
- Cloud metadata endpoint only when workloads legitimately require it
- Any service that must not traverse the web proxy

Avoid adding broad public suffixes because that may unintentionally bypass egress control.

## 16. High-availability behavior

### Squid

- Two replicas are scheduled on separate nodes.
- The ClusterIP Service distributes new TCP connections across healthy endpoints.
- Existing long-lived connections remain attached to the selected Squid pod.
- The PDB keeps at least one replica available during API-driven voluntary eviction.
- The PDB does not prevent sudden node, VM, kernel, or network failure.

Failover test:

```bash
kubectl delete pod -n squid-proxy \
  $(kubectl get pod -n squid-proxy \
    -l app.kubernetes.io/name=squid \
    -o jsonpath='{.items[0].metadata.name}')
```

Then immediately test from the application pod:

```bash
kubectl exec -n customer-app deployment/proxy-test -- \
  curl -s https://api.ipify.org
```

### Kyverno

- Three admission-controller replicas allow webhook requests to continue when one replica is unavailable.
- Additional controller replicas improve component availability, but some controller functions use leader election and are processed by one active leader at a time.
- Spread Kyverno pods across nodes by ensuring sufficient schedulable capacity and reviewing the Helm chart's default affinity/topology settings.

## 17. Security considerations

- This solution does not automatically block direct pod-to-internet access. Proxy variables instruct compatible applications to use Squid, but an application can bypass them.
- To enforce the path, add carefully tested Kubernetes/Calico/Cilium egress NetworkPolicies which permit DNS, internal services, and Squid while denying direct external egress.
- Do not apply a deny policy until cluster CIDRs, DNS endpoints, API dependencies, and application-specific destinations are confirmed.
- Use domain allowlists in Squid where the customer requires restricted web access.
- Do not enable TLS interception unless the customer explicitly requires it and accepts certificate, privacy, and operational implications.
- Export access logs to the customer's logging platform.
- Restrict access to Squid port 3128 to approved application namespaces.
- Pin and scan all container images.

## 18. Capacity planning

Measure before production:

- Concurrent connections
- New connections per second
- Requests per second
- Average and peak throughput
- CPU and memory usage
- File descriptor consumption
- DNS latency and failures
- Upstream connection errors
- Failover load when only one Squid replica remains

Do not assume a 4 GB or 8 GB node is sufficient for every customer. Load test using expected request concurrency and response sizes.

## 19. Troubleshooting

### Squid CrashLoop: setgid/initgroups not permitted

Cause: all capabilities were dropped and the image could not switch to the `proxy` user/group.

Required capabilities in this tested image:

```yaml
add:
  - SETUID
  - SETGID
  - CHOWN
```

### Squid cannot write `/dev/stdout`

Use writable files under `/var/log/squid` and `fsGroup: 13`, as included in the manifest.

### `Squid is already running` or fresh PID file

Do not run `squid -z` before the foreground process when caching is disabled. Remove stale `/run/squid.pid`, parse the config, then start Squid once.

### Third Squid pod remains Pending during rollout

With two eligible nodes and required anti-affinity, use:

```yaml
maxSurge: 0
maxUnavailable: 1
```

### Variables are not injected

Check:

```bash
kubectl get ns <NAMESPACE> --show-labels
kubectl get clusterpolicy inject-squid-proxy
kubectl get pods -n kyverno
kubectl describe pod <POD> -n <NAMESPACE>
```

Delete/restart the workload pod after enabling injection.

### Application still connects directly

The application may not honor standard proxy variables. Review its runtime-specific proxy configuration. Java applications, some Node.js libraries, and vendor software may require dedicated settings.

### Internal traffic goes through Squid

- Confirm destination DNS name matches an entry in `NO_PROXY`.
- Add the internal FQDN/domain to both variable variants.
- Restart the pod.
- Test the specific runtime because CIDR and suffix matching behavior varies.

## 20. Cleanup and rollback

Disable injection for future pods:

```bash
kubectl label namespace <APPLICATION-NAMESPACE> \
  squid-proxy-injection-
```

Restart workloads to remove injected environment variables:

```bash
kubectl rollout restart deployment -n <APPLICATION-NAMESPACE>
```

Remove policy:

```bash
kubectl delete -f manifests/inject-squid-proxy.yaml
```

Remove Squid:

```bash
kubectl delete -f manifests/squid-lke.yaml
```

Remove Kyverno only when no other cluster policies depend on it:

```bash
helm uninstall kyverno -n kyverno
```

Test cleanup:

```bash
kubectl delete -f tests/internal-web.yaml --ignore-not-found
kubectl delete -f tests/proxy-test-deployment.yaml --ignore-not-found
```

## 21. Production acceptance checklist

- [ ] Two dedicated proxy nodes are labelled and tainted correctly.
- [ ] Two Squid replicas are Running on separate nodes.
- [ ] Squid service has two ready endpoints.
- [ ] HTTP and HTTPS tests succeed.
- [ ] Every possible outbound public IP is documented and allowlisted.
- [ ] Kyverno admission controller has three available replicas.
- [ ] Proxy injection is enabled only for approved namespaces.
- [ ] Existing workloads were restarted in a controlled manner.
- [ ] Internal service-to-service traffic bypasses Squid.
- [ ] Customer internal domains are in both NO_PROXY variants.
- [ ] Application runtimes were verified to honor proxy variables.
- [ ] Squid failover and Kyverno replica failure were tested.
- [ ] Logs and metrics are exported and monitored.
- [ ] Container images are approved, scanned, and digest-pinned.
- [ ] Direct egress bypass risk is documented or controlled by NetworkPolicy.
- [ ] Capacity was load-tested under single-replica failover conditions.

## 22. GitHub repository or document?

Use a **GitHub repository as the primary source of truth**, with this README serving as the operational document.

Recommended repository controls:

- Protected main branch
- Mandatory pull-request review
- CODEOWNERS for the cloud/network team
- Semantic release tags such as `v1.0.0`
- Customer-specific values in separate overlays or branches, not hardcoded into the shared baseline
- CI checks for YAML parsing, `kubectl --dry-run=client`, Kyverno policy tests, and image scanning
- No credentials, tokens, customer secrets, or private endpoint details committed
