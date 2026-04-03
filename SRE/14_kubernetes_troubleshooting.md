# Kubernetes Troubleshooting Guide

A practical, scenario-based guide for diagnosing and fixing the most common Kubernetes issues. Organized as decision trees so you can go from symptom to fix as fast as possible.

---

## Troubleshooting Flowchart

```
Pod not running?
├── Pending         → Check scheduling (Step 1)
├── CrashLoopBackOff → Check container crashes (Step 2)
├── ImagePullBackOff → Check image/registry (Step 3)
├── OOMKilled        → Check memory (Step 4)
├── Evicted          → Check node resources (Step 5)
└── Running but not working → Check networking/config (Step 6)

Service not reachable?
├── No endpoints    → Check selectors (Step 7)
├── DNS not resolving → Check CoreDNS (Step 8)
└── Connection refused → Check ports/probes (Step 9)

Node issues?
├── NotReady        → Check kubelet/network (Step 10)
├── Disk pressure   → Check disk usage (Step 11)
└── Memory pressure → Check node memory (Step 12)
```

---

## Step 1: Pod Stuck in Pending

**Symptom**: Pod stays in `Pending` state and never gets scheduled.

**Diagnosis**:
```bash
# Check why the pod is pending
kubectl describe pod <pod-name> -n <namespace>
# Look at the "Events" section at the bottom
```

**Common Causes & Fixes**:

| Cause | Event Message | Fix |
|-------|--------------|-----|
| Insufficient CPU/memory | `Insufficient cpu` / `Insufficient memory` | Reduce resource requests, add nodes, or use Cluster Autoscaler |
| No matching nodes | `0/3 nodes are available: 3 node(s) didn't match Pod's node affinity` | Fix nodeSelector/affinity rules or add matching nodes |
| PVC not bound | `persistentvolumeclaim "data" not found` | Create the PVC or fix the StorageClass |
| Taint/toleration mismatch | `1 node(s) had taints that the pod didn't tolerate` | Add tolerations to pod spec or remove taints |
| Too many pods on node | `Too many pods` | Increase maxPods on nodes or add more nodes |
| ResourceQuota exceeded | `exceeded quota` | Request quota increase or reduce resource requests |

```bash
# Check node capacity vs. allocations
kubectl describe nodes | grep -A 5 "Allocated resources"

# Check resource quotas
kubectl get resourcequota -n <namespace>

# Check PVC status
kubectl get pvc -n <namespace>
```

---

## Step 2: CrashLoopBackOff

**Symptom**: Pod starts, crashes, restarts, crashes again. Backoff delay increases each time.

**Diagnosis**:
```bash
# Check container logs (current crash)
kubectl logs <pod-name> -n <namespace>

# Check previous crash logs
kubectl logs <pod-name> -n <namespace> --previous

# Check exit code
kubectl describe pod <pod-name> -n <namespace> | grep -A 3 "Last State"
```

**Exit Code Reference**:

| Exit Code | Meaning | Common Cause |
|-----------|---------|-------------|
| 0 | Success (but K8s restarts because restartPolicy) | Command completes; use a Job instead of Deployment |
| 1 | Application error | Bug in code, missing config, unhandled exception |
| 126 | Permission denied | Binary not executable |
| 127 | Command not found | Wrong entrypoint/command in Dockerfile or pod spec |
| 137 | SIGKILL (OOMKilled or manual kill) | Memory limit exceeded → see Step 4 |
| 139 | SIGSEGV (segfault) | Memory corruption, native code crash |
| 143 | SIGTERM (graceful shutdown failed) | App doesn't handle SIGTERM within terminationGracePeriodSeconds |

**Common Fixes**:

```bash
# Debug by getting a shell into a running container
kubectl exec -it <pod-name> -n <namespace> -- /bin/sh

# If container crashes too fast, override the command to keep it alive
kubectl run debug --image=<same-image> --command -- sleep 3600
kubectl exec -it debug -- /bin/sh

# Check if config/secrets are mounted correctly
kubectl exec -it <pod-name> -- env | grep DB_
kubectl exec -it <pod-name> -- cat /etc/config/app.conf
```

---

## Step 3: ImagePullBackOff / ErrImagePull

**Symptom**: Pod can't pull the container image.

**Diagnosis**:
```bash
kubectl describe pod <pod-name> -n <namespace> | grep -A 5 "Events"
```

**Common Causes & Fixes**:

| Cause | Fix |
|-------|-----|
| Image doesn't exist | Verify: `docker pull <image>` locally |
| Tag doesn't exist | Check registry for available tags |
| Private registry, no credentials | Create imagePullSecret (see below) |
| Registry rate limit (Docker Hub) | Use a pull-through cache or authenticate |
| Typo in image name | Double-check spelling and registry URL |
| Network issue | Check node can reach registry: `curl https://registry.example.com/v2/` |

**Creating an imagePullSecret**:
```bash
# Create secret
kubectl create secret docker-registry regcred \
  --docker-server=registry.example.com \
  --docker-username=user \
  --docker-password=pass \
  --docker-email=user@example.com \
  -n <namespace>

# Reference in pod spec
spec:
  imagePullSecrets:
    - name: regcred
  containers:
    - name: app
      image: registry.example.com/app:v1
```

---

## Step 4: OOMKilled (Exit Code 137)

**Symptom**: Container is killed because it exceeded its memory limit.

**Diagnosis**:
```bash
# Confirm OOMKilled
kubectl describe pod <pod-name> -n <namespace> | grep -A 5 "Last State"
# Look for: Reason: OOMKilled

# Check current memory usage vs. limits
kubectl top pod <pod-name> -n <namespace>

# Check memory limits set
kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.spec.containers[*].resources}'
```

**Fixes (in order of preference)**:

1. **Fix the memory leak** — Profile the application, check for unbounded caches, connection leaks, or growing data structures
2. **Increase memory limit** — If the app genuinely needs more memory
   ```yaml
   resources:
     requests:
       memory: 256Mi
     limits:
       memory: 512Mi   # Increase this
   ```
3. **Use VPA** — Let Vertical Pod Autoscaler right-size automatically
4. **Add JVM/runtime flags** — For Java: `-Xmx`, for Node.js: `--max-old-space-size`

**Prevention**:
```yaml
# Set up alerts for pods approaching memory limits
# PromQL: container approaching 90% of memory limit
(container_memory_working_set_bytes / container_spec_memory_limit_bytes) > 0.9
```

---

## Step 5: Evicted Pods

**Symptom**: Pod status shows `Evicted`. Often many pods evicted at once.

**Diagnosis**:
```bash
# List evicted pods
kubectl get pods -n <namespace> --field-selector=status.phase=Failed | grep Evicted

# Check why
kubectl describe pod <evicted-pod> -n <namespace> | grep -i evict

# Check node conditions
kubectl describe node <node-name> | grep -A 5 "Conditions"
```

**Common Causes**:

| Cause | Node Condition | Fix |
|-------|---------------|-----|
| Disk pressure | `DiskPressure=True` | Clean up images: `docker system prune`, increase disk, add log rotation |
| Memory pressure | `MemoryPressure=True` | Reduce workload memory, add nodes |
| PID pressure | `PIDPressure=True` | Find process leak, increase PID limit |
| Ephemeral storage exceeded | `EphemeralStoragePressure` | Set ephemeral-storage limits, clean temp files |

**Cleanup evicted pods**:
```bash
# Delete all evicted pods across all namespaces
kubectl get pods --all-namespaces --field-selector=status.phase=Failed \
  -o json | kubectl delete -f -
```

---

## Step 6: Pod Running But Not Working

**Symptom**: Pod shows `Running` and `1/1 Ready` but the application isn't behaving correctly.

**Diagnosis Checklist**:

```bash
# 1. Check application logs
kubectl logs <pod-name> -n <namespace> -f

# 2. Check if the right config is loaded
kubectl exec <pod-name> -- env
kubectl exec <pod-name> -- cat /etc/config/app.yaml

# 3. Check if the app is listening on the expected port
kubectl exec <pod-name> -- netstat -tlnp
# or
kubectl exec <pod-name> -- ss -tlnp

# 4. Test from inside the pod
kubectl exec <pod-name> -- curl -v localhost:8080/healthz

# 5. Check if probes are misconfigured
kubectl describe pod <pod-name> | grep -A 10 "Liveness\|Readiness"

# 6. Check resource throttling (CPU throttled)
kubectl top pod <pod-name>
```

**Common Issues**:

| Problem | Symptom | Fix |
|---------|---------|-----|
| Wrong port | App listens on 3000, probe checks 8080 | Align containerPort, probe port, and app config |
| Wrong config | App uses dev database in production | Check ConfigMap/Secret values and environment variables |
| CPU throttling | App is slow but not crashing | Increase CPU limits or remove CPU limits (use requests only) |
| DNS resolution fails inside pod | Can't reach other services | Check CoreDNS (Step 8), check NetworkPolicy |
| Readiness probe too strict | Pod marked not-ready intermittently | Tune probe thresholds (increase failureThreshold) |

---

## Step 7: Service Has No Endpoints

**Symptom**: Service exists but `kubectl get endpoints <service>` shows no endpoints.

**Diagnosis**:
```bash
# Check service selectors
kubectl describe svc <service-name> -n <namespace>

# Check if pods match the selector
kubectl get pods -n <namespace> -l <selector-from-service>

# Check if pods are Ready
kubectl get pods -n <namespace> -l <selector-from-service> -o wide
```

**Common Causes**:

| Cause | Fix |
|-------|-----|
| Selector doesn't match pod labels | Align `svc.spec.selector` with `pod.metadata.labels` |
| Pods exist but aren't Ready | Fix readiness probe (pods must pass readiness to get endpoints) |
| Pods are in wrong namespace | Service and pods must be in the same namespace (for non-headless) |
| No pods running at all | Check Deployment replicas, check if pods are crashing |

**Quick check**:
```bash
# Compare service selector with pod labels
SVC_SELECTOR=$(kubectl get svc <svc> -n <ns> -o jsonpath='{.spec.selector}')
echo "Service selects: $SVC_SELECTOR"
kubectl get pods -n <ns> --show-labels
```

---

## Step 8: DNS Not Resolving

**Symptom**: Pods can't resolve service names (`curl http://my-service` fails with DNS errors).

**Diagnosis**:
```bash
# Test DNS from inside a pod
kubectl exec -it <pod-name> -- nslookup my-service
kubectl exec -it <pod-name> -- nslookup kubernetes.default

# Check CoreDNS is running
kubectl get pods -n kube-system -l k8s-app=kube-dns

# Check CoreDNS logs
kubectl logs -n kube-system -l k8s-app=kube-dns --tail=50

# Check resolv.conf inside pod
kubectl exec <pod-name> -- cat /etc/resolv.conf
```

**Common Fixes**:

| Problem | Fix |
|---------|-----|
| CoreDNS pods crashed | `kubectl rollout restart deployment/coredns -n kube-system` |
| resolv.conf wrong | Check pod's dnsPolicy (should be `ClusterFirst` for most cases) |
| NetworkPolicy blocking DNS | Allow egress to `kube-system` namespace on port 53 (TCP+UDP) |
| Cross-namespace resolution | Use FQDN: `my-service.other-namespace.svc.cluster.local` |

**NetworkPolicy allowing DNS**:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
```

---

## Step 9: Connection Refused / Timeout

**Symptom**: Requests to a service return connection refused or time out.

**Diagnosis**:
```bash
# Check service and endpoints
kubectl get svc,endpoints <service-name> -n <namespace>

# Test connectivity from another pod
kubectl run test --rm -it --image=busybox -- wget -qO- http://<service>:<port>/healthz

# Check if target port matches container port
kubectl describe svc <service-name>
kubectl describe pod <pod-name> | grep "Port:"

# Check NetworkPolicies
kubectl get networkpolicy -n <namespace>
```

**Common Causes**:

| Symptom | Cause | Fix |
|---------|-------|-----|
| Connection refused | App not listening on targetPort | Align service targetPort with container's listening port |
| Timeout | NetworkPolicy blocking traffic | Add ingress rule allowing traffic from source |
| Intermittent failures | Pod failing readiness probe | Fix probe or app health check |
| Works inside cluster, not outside | Service type is ClusterIP | Use NodePort, LoadBalancer, or Ingress |

---

## Step 10: Node NotReady

**Symptom**: `kubectl get nodes` shows a node as `NotReady`.

**Diagnosis**:
```bash
# Check node conditions
kubectl describe node <node-name> | grep -A 20 "Conditions"

# Check kubelet status (SSH to node)
systemctl status kubelet
journalctl -u kubelet --since "10 minutes ago" | tail -50

# Check if node can reach API server
# (from the node)
curl -k https://<api-server>:6443/healthz
```

**Common Causes**:

| Cause | Fix |
|-------|-----|
| Kubelet crashed | `systemctl restart kubelet` |
| Network partition | Check VPC/security groups, CNI plugin health |
| Certificate expired | Renew kubelet certificates |
| Disk full | Clean up: `docker system prune`, remove old logs |
| Out of memory | Identify and kill memory-heavy processes, add capacity |
| CNI plugin failure | Restart CNI pods: `kubectl rollout restart ds/<cni-name> -n kube-system` |

---

## Step 11: Disk Pressure

**Symptom**: Node condition shows `DiskPressure=True`. Pods start getting evicted.

**Diagnosis** (SSH to node):
```bash
# Check disk usage
df -h

# Find what's using space
du -sh /var/lib/docker/*
du -sh /var/log/*
du -sh /var/lib/kubelet/*

# Check container image usage
docker system df
# or
crictl images | sort -k 3 -h
```

**Fixes**:

```bash
# Clean unused Docker images and containers
docker system prune -af

# Clean old container logs
find /var/lib/docker/containers -name "*.log" -size +100M -exec truncate -s 0 {} \;

# Configure log rotation (add to /etc/docker/daemon.json)
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}

# Clean up completed/evicted pods
kubectl get pods --all-namespaces --field-selector=status.phase=Failed -o name | xargs kubectl delete
```

**Prevention**:
```yaml
# Set ephemeral-storage limits on pods
resources:
  requests:
    ephemeral-storage: 1Gi
  limits:
    ephemeral-storage: 2Gi
```

---

## Step 12: Memory Pressure

**Symptom**: Node condition shows `MemoryPressure=True`.

**Diagnosis** (SSH to node):
```bash
# Check memory usage
free -h

# Find memory-heavy processes
ps aux --sort=-%mem | head -20

# Check kubelet memory thresholds
cat /var/lib/kubelet/config.yaml | grep -A 5 eviction
```

**Default eviction thresholds**:
```
memory.available < 100Mi  → eviction starts
nodefs.available < 10%    → eviction starts
imagefs.available < 15%   → eviction starts
```

**Fixes**:
- Reduce pod memory requests/limits to allow better bin packing
- Add more nodes to the cluster
- Use Cluster Autoscaler or Karpenter for automatic node scaling
- Investigate and fix memory leaks in applications

---

## Quick Command Reference

### Pod Debugging
```bash
kubectl get pods -n <ns> -o wide                    # Pod status + node
kubectl describe pod <pod> -n <ns>                   # Full details + events
kubectl logs <pod> -n <ns> --tail=100               # Recent logs
kubectl logs <pod> -n <ns> --previous               # Previous crash logs
kubectl logs <pod> -n <ns> -c <container>           # Specific container
kubectl exec -it <pod> -n <ns> -- /bin/sh           # Shell into container
kubectl top pod -n <ns>                              # CPU/memory usage
kubectl get events -n <ns> --sort-by=.lastTimestamp  # Recent events
```

### Node Debugging
```bash
kubectl get nodes -o wide                            # Node status
kubectl describe node <node>                         # Full details
kubectl top nodes                                    # CPU/memory per node
kubectl drain <node> --ignore-daemonsets             # Safely remove workloads
kubectl cordon <node>                                # Prevent scheduling
kubectl uncordon <node>                              # Allow scheduling again
```

### Networking Debugging
```bash
kubectl get svc,endpoints -n <ns>                    # Service + endpoints
kubectl get networkpolicy -n <ns>                    # Network policies
kubectl run test --rm -it --image=nicolaka/netshoot -- /bin/bash  # Network debug pod
```

### Cluster Health
```bash
kubectl cluster-info                                 # API server info
kubectl get componentstatuses                        # Control plane health
kubectl get pods -n kube-system                      # System pods
kubectl api-resources                                # Available API resources
```

---

## Troubleshooting Decision Tree — Quick Reference

```
App not working
│
├── Pod not starting?
│   ├── Status: Pending     → kubectl describe pod → check resources, PVC, affinity
│   ├── Status: Error       → kubectl logs --previous → check entrypoint, config
│   ├── CrashLoopBackOff    → kubectl logs --previous → check exit code
│   ├── ImagePullBackOff    → kubectl describe pod → check image name, registry creds
│   └── Init:Error          → kubectl logs <pod> -c <init-container> → check init container
│
├── Pod running but broken?
│   ├── Returns errors      → kubectl logs → check application logs
│   ├── Slow responses      → kubectl top → check CPU throttling, memory
│   ├── Can't reach deps    → kubectl exec → test DNS, network connectivity
│   └── Wrong behavior      → kubectl exec → check env vars, config files
│
├── Service not reachable?
│   ├── No endpoints        → check label selector alignment
│   ├── DNS fails           → check CoreDNS pods, NetworkPolicy
│   ├── Connection refused  → check targetPort vs containerPort
│   └── Timeout             → check NetworkPolicy, security groups
│
└── Node problems?
    ├── NotReady            → check kubelet, CNI, certificates
    ├── DiskPressure        → clean images, logs, set log rotation
    └── MemoryPressure      → scale down workloads, add nodes
```
