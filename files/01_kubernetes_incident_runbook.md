# Kubernetes Cluster Incident Response Runbook

**Space:** Engineering > SRE > Runbooks  
**Owner:** Platform Engineering Team  
**Last Updated:** February 2025  
**Review Cycle:** Quarterly  
**Severity Coverage:** P1 / P2 / P3

---

## Overview

This runbook covers incident response procedures for the production Kubernetes clusters running on AWS EKS. It applies to all services deployed under the `prod-*` namespace family across regions `us-east-1` (primary) and `eu-west-1` (DR).

Our EKS clusters run **Kubernetes 1.29** across three node groups:
- `system-ng` — 3× m5.xlarge (core infra: CoreDNS, metrics-server, cluster-autoscaler)
- `app-ng` — 8–40× m5.2xlarge (application workloads, autoscaled)
- `gpu-ng` — 0–6× g4dn.2xlarge (ML inference workloads, scaled to zero when idle)

---

## Severity Definitions

| Severity | Description | Response SLA | Escalation |
|----------|-------------|--------------|------------|
| P1 | Full cluster or multi-service outage. Revenue impact. | 15 min | On-call lead + Eng VP |
| P2 | Single critical service degraded. Partial data loss risk. | 30 min | On-call engineer |
| P3 | Non-critical service degraded. No user impact. | 4 hours | Next business day |

---

## 1. Initial Triage (All Severities)

### 1.1 Check Cluster Health

```bash
# Verify all nodes are Ready
kubectl get nodes -o wide

# Check for any nodes in NotReady state
kubectl get nodes | grep -v Ready

# Check system pod health
kubectl get pods -n kube-system

# View recent events (last 30 minutes)
kubectl get events --sort-by='.lastTimestamp' -A | tail -50
```

### 1.2 Identify Affected Namespace

```bash
# List all namespaces and pod counts
kubectl get ns
kubectl get pods -A | grep -v Running | grep -v Completed

# Check pod restarts across all namespaces
kubectl get pods -A --sort-by='.status.containerStatuses[0].restartCount' | tail -20
```

### 1.3 Pull Logs for Failing Pods

```bash
# Get logs for a crashing pod
kubectl logs <pod-name> -n <namespace> --previous

# Stream logs in real time
kubectl logs -f <pod-name> -n <namespace>

# Get logs from a specific container in a multi-container pod
kubectl logs <pod-name> -n <namespace> -c <container-name>
```

---

## 2. Common Failure Scenarios

### 2.1 OOMKilled Pods

**Symptoms:** Pods cycling with `OOMKilled` exit code (137). High restart counts.

**Diagnosis:**
```bash
# Find OOMKilled pods
kubectl get pods -A | grep -i oom
kubectl describe pod <pod-name> -n <namespace> | grep -A5 "Last State"

# Check current resource usage
kubectl top pods -n <namespace>
kubectl top nodes
```

**Resolution:**
1. Identify which container is being OOM-killed via `kubectl describe pod`.
2. Temporarily scale down the deployment to reduce load:
   ```bash
   kubectl scale deployment <name> -n <namespace> --replicas=1
   ```
3. Patch memory limits (temporary fix — raise a PR for permanent change):
   ```bash
   kubectl set resources deployment <name> -n <namespace> \
     --limits=memory=2Gi --requests=memory=1Gi
   ```
4. Root cause: Check if recent code deployment introduced a memory leak. Review metrics in Datadog under `kubernetes.memory.usage` filtered by pod.

**Escalation trigger:** If more than 3 deployments are OOMKilling simultaneously → P1, wake on-call lead.

---

### 2.2 Node Not Ready

**Symptoms:** `kubectl get nodes` shows one or more nodes in `NotReady` state.

**Diagnosis:**
```bash
# Describe the problem node
kubectl describe node <node-name>

# Check kubelet logs on the node (requires SSM access)
aws ssm start-session --target <ec2-instance-id>
sudo journalctl -u kubelet -n 100 --no-pager
```

**Resolution:**
1. If the node has been `NotReady` < 5 minutes — wait. Auto-recovery often triggers.
2. If `NotReady` > 5 minutes:
   ```bash
   # Cordon the node (stop new scheduling)
   kubectl cordon <node-name>
   
   # Drain existing pods safely
   kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data --grace-period=60
   ```
3. Terminate the EC2 instance via AWS console. ASG will launch a replacement.
4. Verify new node joins and reaches `Ready` state within 4 minutes.

---

### 2.3 Pending Pods / Insufficient Resources

**Symptoms:** Pods stuck in `Pending` state for > 2 minutes.

```bash
# Check why pod is pending
kubectl describe pod <pod-name> -n <namespace> | grep -A10 Events

# Check cluster-autoscaler logs
kubectl logs -n kube-system -l app=cluster-autoscaler --tail=100
```

**Common causes:**
- **No nodes available:** Cluster autoscaler should trigger. Typical scale-up time: 3–4 minutes. If not scaling, check CA logs for errors.
- **Resource requests too high:** Pod requests exceed any single node capacity. Check the pod spec against node instance types.
- **Taints/tolerations mismatch:** GPU workloads require `tolerations` for `nvidia.com/gpu`. Verify pod spec.
- **PVC not binding:** Check `kubectl get pvc -n <namespace>` for `Pending` PVCs.

---

### 2.4 CrashLoopBackOff

```bash
# Get exit code and reason
kubectl describe pod <pod-name> -n <namespace>

# Pull previous container logs
kubectl logs <pod-name> -n <namespace> --previous

# Check if ConfigMap or Secret is missing
kubectl get configmaps -n <namespace>
kubectl get secrets -n <namespace>
```

**Resolution checklist:**
- [ ] Check recent deployments — was a bad image pushed?
- [ ] Verify environment variables and config mounts
- [ ] Check liveness/readiness probe configuration
- [ ] Verify downstream dependency health (DB, Redis, external APIs)
- [ ] Roll back if caused by recent deploy: `kubectl rollout undo deployment/<name> -n <namespace>`

---

### 2.5 EKS Control Plane Unavailable

**Symptoms:** `kubectl` commands timing out. `kubectl cluster-info` fails.

```bash
# Check EKS cluster status
aws eks describe-cluster --name prod-eks-us-east-1 --query 'cluster.status'

# Check control plane logs in CloudWatch
aws logs get-log-events \
  --log-group-name /aws/eks/prod-eks-us-east-1/cluster \
  --log-stream-name kube-apiserver-audit \
  --limit 50
```

**Resolution:** Control plane issues are AWS-managed. Actions:
1. Open AWS Support ticket immediately (P1 — production impact).
2. Check [AWS Service Health Dashboard](https://health.aws.amazon.com).
3. If DR is required, initiate failover to `eu-west-1` cluster. See **DR Failover Runbook**.

---

## 3. Post-Incident Actions

All P1 and P2 incidents require:
- [ ] Incident timeline documented in the `#incidents` Slack channel
- [ ] RCA completed within 5 business days (template in Confluence: `Engineering > SRE > RCA Template`)
- [ ] Action items filed in Jira under project `SRE` with label `incident-followup`
- [ ] Runbook updated if new failure mode discovered

---

## 4. Useful Aliases & Scripts

```bash
# Add to ~/.bashrc for faster response
alias kgp='kubectl get pods -A | grep -v Running'
alias kge='kubectl get events --sort-by=.lastTimestamp -A | tail -30'
alias ktopn='kubectl top nodes'
alias ktopp='kubectl top pods -A | sort -k3 -rh | head -20'
```

---

## 5. Key Contacts

| Role | Name | Slack | PagerDuty |
|------|------|-------|-----------|
| Platform Lead | Arjun Mehta | @arjun.mehta | pd-platform-lead |
| On-Call Rotation | (see PD schedule) | #on-call | pd-engineering |
| AWS TAM | Jennifer Lowe | jennifer.lowe@amazon.com | — |
| Datadog Support | — | #monitoring | — |

---

*For cluster networking issues (VPC CNI, load balancer), see: `Engineering > SRE > Runbooks > Networking Incidents`*  
*For database-related incidents, see: `Engineering > SRE > Runbooks > Database Incidents`*
