# CKS EXAM QUICK REFERENCE CARD - One Page Summary

## 16 QUESTIONS QUICK LOOKUP

### Q1: CIS Benchmark (7 min)
```bash
ssh cks001001 → vim /var/lib/kubelet/config.yaml
# anonymous: enabled: false (was true)
# webhook enabled: true (was false)
# authorization mode: Webhook (was AlwaysAllow)

vim /etc/kubernetes/manifests/etcd.yaml
# --client-cert-auth=true
systemctl daemon-reload
systemctl restart kubelet
kubectl get pod -A
```

### Q2: TLS Secret (5 min) ⭐ EASIEST
```bash
ssh cks002002
kubectl -n clever-cactus create secret tls clever-cactus --cert=/home/candidate/ca-cert/web.k8s.local.crt --key=/home/candidate/ca-cert/web.k8s.local.key
kubectl -n clever-cactus get secrets
```

### Q3: Dockerfile Security (6 min)
```bash
ssh cks003003
# FILE 1: /cks/docker/Dockerfile - Change USER root → USER nobody
vim /cks/docker/Dockerfile

USER nobody

# FILE 2: /cks/docker/deployment.yaml - Add securityContext:
vim /cks/docker/deployment.yaml

securityContext:
  runAsUser: 65535
  readOnlyRootFilesystem: true
  allowPrivilegeEscalation: false
```

### Q4: Falco /dev/mem Detection (10 min) ⭐ HARD
```bash
ssh cks004004
# Create rule: /etc/falco/falco_rules.local.yaml
# Run: sudo falco -M 30 -r /etc/falco/falco_rules.local.yaml >> devmem.log
# Find: cat devmem.log | grep container_id
# Scale: kubectl scale deployment ollama --replicas=0
kubectl get deployment
vim /etc/falco/falco_rules.local.yaml
#Search 'https://falco.org/docs/concepts/rules/basic-elements/&#39; copy and modify (shell_binaries)
- list: mem_file
  items:[/dev/mem]
- rule: devmem
  desc: devmem
  condition: >
    fd.name in (mem_file)    
  output: >
    Shell (command=%proc.cmdline file=%fd.name container_id=%container.id)
  priority: NOTICE
  tag: [file]

sudo falco -M 30 -r /etc/falco/falco_rules.local.yaml >> devmem.log
cat devmem.log
sudo crictl ps | grep {container_id}
kubectl get pod, deployment
kubectl scale deployment ollama --replicas=0
kubectl get deployments
```

### Q5: Pod Security Context (6 min)
```bash
ssh cks005005
vim /home/candidate/sec-ns_deployment.yaml
# Add to containers both sec-ctx-demo-1 and 2:
securityContext:
  runAsUser: 30000
  readOnlyRootFilesystem: true
  allowPrivilegeEscalation: false
kubectl apply -f /home/candidate/sec-ns_deployment.yaml
kubectl -n sec-ns get pod, deployment
```

### Q6: Audit Logging (12 min) ⭐ COMPLEX
```bash
ssh cks006006
#Open kubernetes document search 'log audit', audit/audit-policy.yaml
sudo -i
# 1. vim /etc/kubernetes/logpolicy/sample-policy.yaml (add audit rules)
# Log pod changes at RequestResponse level
  - level: RequestResponse
    resources:
    - group: ""
      resources: ["persistentvolumes"]
  # Log the request body of configmap changes in kube-system.
  - level: Request
    resources:
    - group: ""
      resources: ["configmaps"]
    namespaces: ["front-apps"]
  # Log configmap and secret changes in all other namespaces at the Metadata level.
  - level: Metadata
    resources:
    - group: ""
      resources: ["secrets", "configmaps"]
  # A catch-all rule to log all other requests at the Metadata level.
  - level: Metadata
    omitStages:
      - "RequestReceived"
# 2. vim /etc/kubernetes/manifests/kube-apiserver.yaml (add flags) - Click into "audit-backends" link:
--audit-policy-file=/etc/kubernetes/logpolicy/sample-policy.yaml
--audit-log-path=/var/log/kubernetes/audit-logs.txt
--audit-log-maxage=10
--audit-log-maxbackup=2
# 3. Update volumeMounts and hostPath
volumeMounts:
  - mountPath: /etc/kubernetes/logpolicy/sample-policy.yaml
    name: audit
    readOnly: true
  - mountPath: /var/log/kubernetes/
    name: audit-log
    readOnly: false

volumes:
- name: audit
  hostPath:
    path: /etc/kubernetes/logpolicy/sample-policy.yaml
    type: File

- name: audit-log
  hostPath:
    path: /var/log/kubernetes/
    type: DirectoryOrCreate

systemctl daemon-reload
systemctl restart kubelet
kubectl get pod -A
tail /var/log/kubernetes/audit-logs.txt
```

### Q7: Network Policies (7 min)
```bash
ssh cks007007
# FILE 1: deny-policy.yaml
# Search "Network Policies" in Kubernetes Document, search deny
# Copy and modify "network-policy-default-deny-ingress.yaml":
vim deny-policy.yaml

apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-policy
  namespace: prod
spec:
  podSelector: {}
  policyTypes:
  - Ingress

kubectl apply -f deny-policy.yaml
kubectl get networkpolicy -n prod

# FILE 2: allow-from-prod.yaml (in data namespace, allow from prod)
# Create allow-from-prod policy (Search NetworkPolicy resource)
vim allow-from-prod.yaml

apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-prod
  namespace: data
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          env: prod

kubectl apply -f allow-from-prod.yaml
kubectl get networkpolicy -n data
```
### Q8: Ingress TLS (7 min)
```bash
ssh cks008008

# Search "ingress" in Kubernetes Document (Ingress -> Make your HTTP (or HTTPS)), Click "TLS"
# Copy and modify "tls-example-ingress.yaml"
vim ingress.yaml

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web
  namespace: prod02
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  tls:
  - hosts:
      - web.k8sng.local
    secretName: web-cert
  rules:
  - host: web.k8sng.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web
            port:
              number: 80

kubectl apply -f ingress.yaml
kubektl get ingress -n prod02
```

### Q9: Disable Token Auto-Mount (8 min)
```bash
ssh cks009009

# Search "ServiceAccount" in Kubernetes Document
# Click "Configure Service Accounts for Pods" > "Opt out of API credential automounting"
# 1. Inline edit ServiceAccount
kubectl -n monitoring edit sa stats-monitor-sa
# Add under "secrets>name", align to outer left
automountServiceAccountToken: false
# 2. Edit Deployment mountPath (Click the "Launch a Pod using service account token projection)
# Copy and modify "pods/pod-projected-svc-token.yaml"
vim /home/candidate/stats-monitor/deployment.yaml
# Add volumeMounts + projected volume for token
volumeMounts:
- mountPath: /var/run/secrets/kubernetes.io/serviceaccount/token
  name: token

volumes:
- name: token
    projected:
      sources:
      - serviceAccountToken:
          path: token

kubectl apply -f /home/candidate/stats-monitor/deployment.yaml
kubectl -n monitoring get deployments
```

### Q10: SBOM Generation (8 min)
```bash
ssh cks010010
# Find which container has vulnerable package:
kubectl get pod -n alpine
cat /home/candidate/alpine-deployment.yaml
kubectl -n alpine exec -it {pod_name} -c alpine-a -- apk list | grep libcrypto3
kubectl -n alpine exec -it {pod_name} -c alpine-b -- apk list | grep libcrypto3
kubectl -n alpine exec -it {pod_name} -c alpine-c -- apk list | grep libcrypto3
# libcrypto3-3.1.4-r5 (vulnerable in alpine:3.19.1)
# Generate SBOM:
bom generate --image registry.cn-qingdao.aliyuncs.com/containerhub/alpine:3.19.1 --output /home/candidate/alpine.spdx
# Delete container from deployment (alpine-b)
vim /home/candidate/alpine-deployment.yaml
# Find and Remove below lines
- name: alpine-b
  image: registry.cn-qingdao.aliyuncs.com/containerhub/alpine: 3.29.2
  imagePullPolicy: IfNotPresent
  args:
  - /bin/sh
  - -c
  - while true; do sleep; done

kubectl apply -f /home/candidate/alpine-deployment.yaml
kubectl -n alpine get pod
```

### Q11: Pod Security Standards (6 min)
```bash
# Configure restrictive PSP or Pod Security Standards
# Enforce "restricted" standard in namespace
# Block privileged pods, require non-root
kubectl -n confidential get deployments
kubectl delete -f /home/candidate/nginx-unprivileged.yaml
kubectl apply -f /home/candidate/nginx-unprivileged.yaml
# Search "security standards"
vim /home/candidate/nginx-unprivileged.yaml

securityContext:
  allowPrivilegeEscalation: false
  capabilities:
    drop: ["ALL"]
  runAsNonRoot: true
  seccompProfile
    type: RuntimeDefault

kubectl apply -f /home/candidate/nginx-unprivileged.yaml
kubectl -n confidential get deployment
```

### Q12: Docker Socket Security (8 min)
```bash
ssh cks012012
sudo -i
# Check: ls -l /var/run/docker.sock (should be 660, root:root)
# Remove docker socket
id developer
gpasswd -d developer docker
id developer

vim /usr/lib/systemd/system/docker.socket
# Change SocketGroup to root
SocketGroup=root
vim /usr/lib/systemd/system/docker.service
# Remove -H tcp://0.0.0.0:2375 from the "ExecStart"
systemctl daemon-reload
systemclt restart docker.socket
systemctl restart docker.service
ls -l /var/run/docker.dock
ss -tunlp | grep 2375
```

### Q13: Istio mTLS (10 min)
```bash
ssh cks013013
# 1. kubectl label namespace mtls istio-injection=enabled
kubectl get ns mtls --show-labels
kubectl -n mtls get deployment
kubectl -n mtls get pod
# Label the namespace
kubectl label namespace mtls istio-injection=enabled
kubectl -n mtls rollout restart deployment
# Check after completed
kubectl get ns mtls --show-labels
kubectl -n mtls get deployment
kubectl -n mtls get pod
# 2. Create PeerAuthentication with mode: STRICT
kubectl get peerauthentication -n mtls
kubectl exec -n mtls deploy/frontend -c istio-proxy -- curl -s http://backend.mtls.svc.cluster.local
vim mtls.yaml

apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
  name: default
  namespace: mtls
spec:
  mtls:
    mode: STRICT

kubectl apply -f mtls.yaml
kubectl get peerauthentication -n mtls
kubectl exec -n mtls deploy/frontend -c istio-proxy -- curl -s http://backend.mtls.svc.cluster.local
```

### Q14: ImagePolicyWebhook (15 min)
```bash
ssh cks014014
sudo -i
# Search "ImagePolicyWebhook" in Kubernetes Document
cd /etc/kubernetes/epconfig
ls -lh
vim image-policy-config.yaml

imagePolicy:
  kubeConfigFile: /etc/kubernetes/epconfig/kube-config.yaml
  # time in s to cache approval
  allowTTL: 50
  # time in s to cache denial
  denyTTL: 50
  # time in ms to wait between retries
  retryBackoff: 500
  # determines behavior if the webhook backend fails
  defaultAllow: false

vim kube-config.yaml

- cluster:
    certificate-authority: /etc/kubernetes/epconfig/webhook.pem
    server: https://image-bouncer-webhook.default.svc:1323/image_policy
  name: bouncer_webhook

vim /etc/kubernetes/manifests/kube-apiserver.yaml
# Add Webhook
--enable-admission-plugins=NodeRestriction,ImagePolicyWebhook
--admission-control-config-file=/etc/kubernetes/epconfig/admission-control-config.json

# Add Mount Path
volumeMounts:
  - name: image-policy
    mountPath: /etc/kubernetes/epconfig
volumes:
  - name: image-policy
    hostPath:
      path: /etc/kubernetes/epconfig
      type: DirectoryOrCreate
systemctl daemon-reload
systemctl restart kubelet
kubectl -n kube-system get pod
kubectl apply -f /home/candidate/web1.yaml

```

### Q15: Node Upgrade (8 min)
```bash
ssh cks015015
ssh node02
sudo -i
kubectl get node
apt-get update
sudo apt-cache madison kubeadm
apt install kubelet=1.32.1-1.1
systemctl daemon-reload
systemctl restart kubelet
```

### Q16: API Server Authentication (6 min)
```bash
ssh cks016016
sudo -i
# Search "anonymous" > Click "kube-apiserver" > "anonymous-auth"
vim /etc/kubernetes/manifests/kube-apiserver.yaml
- --enable-admission-plugins=NodeRestriction
- --anonymous-auth=false

systemctl daemon-reload
systemctl restart kubelet
kubectl --kubeconfig=/etc/kubernetes/admin.conf get pod -A
kubectl --kubeconfig=/etc/kubernetes/admin.conf delete clusterrolebinding system:anonymous
```

---

## DOMAINS BREAKDOWN

| Domain | Questions | Time | Focus |
|---|---|---|---|
| **Cluster Setup** | 1,2,7,8,16 | 32 min | Network, Secrets, Ingress, Auth |
| **Cluster Hardening** | 3,5,9,13 | 32 min | Security Context, RBAC, mTLS |
| **System Hardening** | 12,15 | 16 min | OS security, Node upgrade |
| **Monitoring/Logging** | 4,6 | 22 min | Falco, Audit logging |
| **Supply Chain** | 10,14 | 23 min | SBOM, Image validation |

---

## TIME ALLOCATION STRATEGY

```
EXAM TIME: 120 minutes total

Phase 1 (0-45 min): Easy Questions
- Q2: TLS Secret (5 min)
- Q3: Dockerfile (6 min)
- Q7: Network Policy (7 min)
- Q8: Ingress (7 min)
- Q16: API Auth (6 min)
- Q5: Pod Security (6 min)
✓ Score: ~6 questions ✓ Time: 37 min

Phase 2 (45-100 min): Medium Questions
- Q1: CIS Benchmark (7 min)
- Q9: Token Auto-Mount (8 min)
- Q12: Docker Socket (8 min)
- Q15: Node Upgrade (8 min)
- Q10: SBOM (8 min)
- Q13: Istio (10 min)
✓ Score: ~6 questions ✓ Time: 49 min

Phase 3 (100-120 min): Hard Questions
- Q4: Falco (10 min)
- Q6: Audit (12 min)
- Q14: ImagePolicy (15 min) ← SKIP IF RUNNING OUT OF TIME
✓ Score: 2-3 questions ✓ Time: 20 min

TOTAL: 14-15 correct questions = 82-88% ✓ PASS
```

---

## MUST-MEMORIZE COMMANDS

```bash
# Setup/Exit
ssh ckXXXXXX                    # SSH to node
sudo -i                         # Become root
exit                            # Exit once (from root)
exit                            # Exit twice (from node)

# Kubernetes
kubectl get pod -A              # All pods all namespaces
kubectl get deployment          # Check deployments
kubectl get secret -n ns        # Check secrets
kubectl apply -f file.yaml      # Apply config
kubectl describe pod NAME       # Detailed pod info
kubectl scale deployment X --replicas=0  # Scale down

# System
systemctl daemon-reload         # Reload systemd configs
systemctl restart kubelet       # Restart kubelet
systemctl restart docker        # Restart docker
tail /var/log/kubernetes/audit-logs.txt  # View audit logs

# Editing
vim /path/to/file              # Edit file
# Paste: ESC, :set paste, i
# Save: ESC, :wq

# Find
grep -r "search_term"          # Find in files
ss -tunlp | grep 2375          # Check if port listening
ps aux | grep process          # Find process
```

---

## COMMON MISTAKES TO AVOID ❌

1. ❌ **Forgetting to SSH to correct node** = 0 points
2. ❌ **Not verifying with kubectl get/describe** = missing issues
3. ❌ **Forgetting systemctl daemon-reload** before restart
4. ❌ **Wrong file locations** (check exact paths)
5. ❌ **YAML indentation errors** (spaces matter!)
6. ❌ **Not exiting back to base node** = can't switch nodes
7. ❌ **Typos in names/labels** (copy-paste is safer)
8. ❌ **Forgetting to apply kubectl commands** (vim only ≠ applied)
9. ❌ **Wrong parameter values** (e.g., 65535 vs 30000)
10. ❌ **Spending >8 min on one question** (skip and return)

---

## FINAL CONFIDENCE CHECKLIST ✓

Before exam, verify you can:

- [ ] SSH without issues (practice path: base → node → base)
- [ ] Edit config files in vim without losing formatting
- [ ] Copy-paste between windows properly
- [ ] Apply YAML files with kubectl
- [ ] Check resources with kubectl get/describe
- [ ] Restart services correctly
- [ ] Find and fix CIS violations
- [ ] Create TLS secrets
- [ ] Write network policies from memory
- [ ] Create Ingress with TLS
- [ ] Understand RBAC basics
- [ ] Know security context fields
- [ ] Verify audit logs
- [ ] Falco rule basics (memorized!)
- [ ] Manage deployments/pods

---

## PASSING SCORES

- **Minimum**: 67% = 11 correct questions
- **Safe Pass**: 75% = 13 correct questions  
- **Excellent**: 85% = 14-15 correct questions

---

## EXAM DAY FINAL TIPS 💡

✅ **DO**:
- Start with easy questions to build confidence
- Always verify your work before moving on
- Flag hard questions and return if time permits
- Read question carefully (some reverse requirements)
- Copy commands from blue text in exam
- Use keyboard shortcuts to save time
- Get 8+ hours of sleep before exam

❌ **DON'T**:
- Spend >8 minutes on any single question
- Skip verification steps
- Panic on hard questions (just skip and continue)
- Edit files without saving (ESC :wq)
- Forget to exit back to base node
- Start with hardest questions
- Try to memorize everything (understand instead)