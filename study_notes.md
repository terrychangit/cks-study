# CKS EXAM REVISION NOTES - Enhanced Memory Edition

## 🧠 COMMAND PATTERNS TO MEMORIZE (Master These First)

### The "SSH-EDIT-RESTART-VERIFY-EXIT" Pattern (90% of questions)
```bash
# PATTERN: All questions follow this flow
ssh cks00X00X          # 1. Jump to node
sudo -i                # 2. Become root (if needed)
vim /path/to/file      # 3. Edit config
systemctl daemon-reload && systemctl restart kubelet  # 4. Apply changes
kubectl get pod -A     # 5. Verify it works
exit; exit             # 6. Return to base (exit twice)
```
**Memory Trick:** "**S**ally **E**ats **R**ed **V**egetables **E**veryday" = SSH-Edit-Restart-Verify-Exit[1]

### The "3 Files Pattern" (API Server Questions: Q1, Q6, Q14, Q16)
```bash
# Whenever you edit kube-apiserver.yaml, you touch 3 sections:
1. Add flags in "command:" section (--something=value)
2. Add volumeMounts (where to mount inside container)
3. Add volumes (what to mount from host)

# Memory: "CFV" = Command, volumeMounts (F=Files in container), Volumes
```

### Common kubectl Shortcuts[2]
```bash
# Use short names to save time
po  = pods
deploy = deployment  
svc = service
ns = namespace
sa = serviceaccount
netpol = networkpolicy
cm = configmap

# Example: kubectl get po -n prod (instead of kubectl get pods -n prod)
```

***

## 📋 16 QUESTIONS - ORGANIZED BY LEARNING DIFFICULTY

### 🟢 SUPER EASY - Learn These First (Days 1-3)

#### Q2: Create TLS Secret (5 min) ⭐ CONFIDENCE BUILDER
**What you're doing:** Creating a password-protected certificate (like HTTPS for websites)
**Why:** Services need certificates to prove their identity

```bash
# PATTERN: "secret tls NAME --cert=CRT --key=KEY"
ssh cks002002
kubectl -n clever-cactus create secret tls clever-cactus \
  --cert=/home/candidate/ca-cert/web.k8s.local.crt \
  --key=/home/candidate/ca-cert/web.k8s.local.key

# VERIFY (always verify!)
kubectl -n clever-cactus get secrets
```
**Memory Aid:** "TLS = **T**wo **L**inks **S**ecure" (cert + key = 2 files)[1]

***

#### Q8: Ingress with TLS (7 min)
**What you're doing:** Setting up the front door to your services with HTTPS
**Why:** Allows external traffic to reach your apps securely

```bash
ssh cks008008

# SEARCH: Type "ingress" in K8s docs → Click "TLS" example
vim ingress.yaml

# PATTERN: TLS has 3 parts: tls block, hosts, secretName
apiVersion: networking.k8s.io/v1
kind: Ingress
meta
  name: web
  namespace: prod02
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  tls:                           # 1. TLS block
  - hosts:                       # 2. Which domain
      - web.k8sng.local
    secretName: web-cert         # 3. Which certificate (from Q2!)
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
kubectl get ingress -n prod02
```
**Memory:** "TLS = **T**LS block, **L**ocation (hosts), **S**ecret name"

***

#### Q16: Disable Anonymous Access (6 min)
**What you're doing:** Locking the cluster door - no guests allowed
**Why:** Anonymous users can spy on your cluster

```bash
ssh cks016016
sudo -i

# SEARCH: "anonymous" in K8s docs → "kube-apiserver"
vim /etc/kubernetes/manifests/kube-apiserver.yaml

# ADD this line (just one flag!)
- --anonymous-auth=false

systemctl daemon-reload && systemctl restart kubelet

# VERIFY (use admin.conf because you disabled anonymous!)
kubectl --kubeconfig=/etc/kubernetes/admin.conf get pod -A
kubectl --kubeconfig=/etc/kubernetes/admin.conf delete clusterrolebinding system:anonymous
```
**Memory:** "Anonymous = **A**lways **F**alse" (anon auth = false)

***

### 🟡 EASY - Security Context Group (Days 4-7)

#### Q3: Dockerfile Security (6 min)
**What you're doing:** Making container run as nobody (not root) + restricting permissions
**Why:** If hacker breaks in, they can't do much without root

```bash
ssh cks003003

# FILE 1: Dockerfile - Change USER
vim /cks/docker/Dockerfile
USER nobody              # Memory: "**No**body is **No**t root"

# FILE 2: Deployment - Add 3 security rules
vim /cks/docker/deployment.yaml

# PATTERN: "3 Rules of Security Context" = RRA
securityContext:
  runAsUser: 65535                      # R = Run as user
  readOnlyRootFilesystem: true          # R = Read-only
  allowPrivilegeEscalation: false       # A = Allow privilege = false
```
**Memory:** "**RRA** = **R**un **R**eadonly **A**llow-false"[1]

***

#### Q5: Pod Security Context (6 min)
**What you're doing:** Same as Q3 but for 2 containers in one deployment
**Why:** Both containers need to be secure

```bash
ssh cks005005
vim /home/candidate/sec-ns_deployment.yaml

# Add to BOTH containers (sec-ctx-demo-1 AND sec-ctx-demo-2)
securityContext:
  runAsUser: 30000                      # Different user ID than Q3!
  readOnlyRootFilesystem: true
  allowPrivilegeEscalation: false

kubectl apply -f /home/candidate/sec-ns_deployment.yaml
kubectl -n sec-ns get pod
```
**Pattern Recognition:** Q3 = 65535, Q5 = 30000 (write these on scratch paper!)

***

#### Q11: Pod Security Standards (6 min)
**What you're doing:** Maximum security lockdown for sensitive apps
**Why:** "Restricted" is the strictest security policy

```bash
kubectl -n confidential get deployments

# SEARCH: "security standards" in K8s docs
vim /home/candidate/nginx-unprivileged.yaml

# PATTERN: "4 Restricted Rules" = ACRS
securityContext:
  allowPrivilegeEscalation: false       # A = Allow = false
  capabilities:                          # C = Capabilities drop ALL
    drop: ["ALL"]
  runAsNonRoot: true                    # R = Run as non-root
  seccompProfile:                       # S = Seccomp profile
    type: RuntimeDefault

kubectl apply -f /home/candidate/nginx-unprivileged.yaml
kubectl -n confidential get deployment
```
**Memory:** "**ACRS** = **A**ll **C**apabilities **R**emoved for **S**ecurity"[1]

***

### 🟡 MEDIUM - Network & Access Control (Days 8-12)

#### Q7: Network Policies (7 min)
**What you're doing:** Building firewalls between namespaces
**Why:** Block all traffic, then allow only what's needed (zero-trust)

```bash
ssh cks007007

# PART 1: Deny everything in "prod" namespace
# SEARCH: "Network Policies" → "deny"
vim deny-policy.yaml

apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
meta
  name: deny-policy
  namespace: prod              # 🔒 Lock this namespace
spec:
  podSelector: {}              # Empty = ALL pods
  policyTypes:
  - Ingress                    # Block incoming traffic

kubectl apply -f deny-policy.yaml

# PART 2: Allow "data" namespace to receive from "prod"
vim allow-from-prod.yaml

apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
meta
  name: allow-from-prod
  namespace: data              # 🔓 Open this namespace
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:       # Only from namespaces with label
        matchLabels:
          env: prod            # Label must match!

kubectl apply -f allow-from-prod.yaml
```
**Memory:** "**DIA** pattern = **D**eny **I**ngress to **A**ll, then allow specific"[3]

***

#### Q1: CIS Benchmark (7 min)
**What you're doing:** Fixing security audit findings (3 fixes in 2 files)
**Why:** CIS = industry standard security checklist

```bash
ssh cks001001
sudo -i

# FILE 1: Kubelet config (3 fixes)
vim /var/lib/kubelet/config.yaml

# PATTERN: "Change 3 things from BAD→GOOD"
anonymous: enabled: false      # ❌ true → ✅ false
webhook: enabled: true         # ❌ false → ✅ true
authorization mode: Webhook    # ❌ AlwaysAllow → ✅ Webhook

# FILE 2: etcd manifest (1 fix)
vim /etc/kubernetes/manifests/etcd.yaml
--client-cert-auth=true        # Require certificates

# RESTART & VERIFY
systemctl daemon-reload && systemctl restart kubelet
kubectl get pod -A
```
**Memory:** "**AWA** = **A**nonymous false, **W**ebhook true, **A**uth=Webhook"[1]

***

#### Q9: Disable Token Auto-Mount (8 min)
**What you're doing:** Stop auto-injecting API credentials into pods
**Why:** If pod is compromised, attacker can't use stolen credentials

```bash
ssh cks009009

# SEARCH: "ServiceAccount" → "Opt out of automounting"

# STEP 1: Edit ServiceAccount (inline)
kubectl -n monitoring edit sa stats-monitor-sa

# ADD this line (align left!)
automountServiceAccountToken: false

# STEP 2: Edit Deployment - add token projection
vim /home/candidate/stats-monitor/deployment.yaml

# PATTERN: volumeMounts + volumes (2 sections)
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
```
**Memory:** "**SAT** = **S**erviceAccount **A**uto-mount, **T**oken projected"

***

### 🟠 MEDIUM-HARD - System & Infrastructure (Days 13-16)

#### Q12: Docker Socket Security (8 min)
**What you're doing:** Locking down Docker daemon (remove user from group + close network port)
**Why:** Docker socket = root access to host

```bash
ssh cks012012
sudo -i

# STEP 1: Remove user from docker group
gpasswd -d developer docker
id developer                   # Verify: should NOT show "docker" group

# STEP 2: Change socket ownership to root-only
vim /usr/lib/systemd/system/docker.socket
SocketGroup=root              # Was: docker → Now: root

# STEP 3: Remove network exposure
vim /usr/lib/systemd/system/docker.service
# DELETE: -H tcp://0.0.0.0:2375 from ExecStart line

# RESTART & VERIFY
systemctl daemon-reload
systemctl restart docker.socket
systemctl restart docker.service

ls -l /var/run/docker.sock    # Should be: root:root 660
ss -tunlp | grep 2375          # Should be: empty (port closed)
```
**Memory:** "**3 Docker Locks** = **G**roup, **S**ocket, **P**ort (GSP)"[4]

***

#### Q15: Node Upgrade (8 min)
**What you're doing:** Upgrading kubelet on worker node to specific version
**Why:** Keep cluster components updated

```bash
ssh cks015015
ssh node02
sudo -i

# PATTERN: "Update-Install-Restart"
apt-get update
apt-cache madison kubeadm           # Check available versions
apt install kubelet=1.32.1-1.1      # Specific version!

systemctl daemon-reload
systemctl restart kubelet
kubectl get node                     # Verify version
```
**Memory:** "**UIR** = **U**pdate, **I**nstall, **R**estart"[3]

***

#### Q10: SBOM Generation (8 min)
**What you're doing:** Find vulnerable container, generate software bill of materials, remove it
**Why:** Track which images have security holes

```bash
ssh cks010010

# STEP 1: Find vulnerable container (check all 3)
kubectl get pod -n alpine
POD=$(kubectl get pod -n alpine -o name | head -1)

kubectl -n alpine exec -it $POD -c alpine-a -- apk list | grep libcrypto3
kubectl -n alpine exec -it $POD -c alpine-b -- apk list | grep libcrypto3  # ← Found!
kubectl -n alpine exec -it $POD -c alpine-c -- apk list | grep libcrypto3

# STEP 2: Generate SBOM report
bom generate \
  --image registry.cn-qingdao.aliyuncs.com/containerhub/alpine:3.19.1 \
  --output /home/candidate/alpine.spdx

# STEP 3: Remove vulnerable container from deployment
vim /home/candidate/alpine-deployment.yaml
# DELETE the entire "alpine-b" container block (name + image + args)

kubectl apply -f /home/candidate/alpine-deployment.yaml
```
**Memory:** "**FGR** = **F**ind, **G**enerate SBOM, **R**emove container"

***

### 🔴 HARD - Advanced Topics (Days 17-20)

#### Q13: Istio mTLS (10 min)
**What you're doing:** Enable mutual TLS in service mesh (both sides verify each other)
**Why:** Encrypt all pod-to-pod traffic automatically

```bash
ssh cks013013

# STEP 1: Enable Istio injection (label namespace)
kubectl label namespace mtls istio-injection=enabled
kubectl -n mtls rollout restart deployment   # Restart to inject sidecar

# Verify: pods should now have 2 containers (app + istio-proxy)
kubectl -n mtls get pod

# STEP 2: Create PeerAuthentication (enforce mTLS)
vim mtls.yaml

apiVersion: security.istio.io/v1
kind: PeerAuthentication
meta
  name: default
  namespace: mtls
spec:
  mtls:
    mode: STRICT            # Memory: STRICT = both sides verify

kubectl apply -f mtls.yaml

# TEST: Should still work (now encrypted)
kubectl exec -n mtls deploy/frontend -c istio-proxy -- \
  curl -s http://backend.mtls.svc.cluster.local
```
**Memory:** "**LRS** = **L**abel ns, **R**estart, **S**TRICT mode"

***

#### Q6: Audit Logging (12 min) ⚠️ MOST COMPLEX
**What you're doing:** Recording all API requests to log file (3-file config)
**Why:** Track who did what for security investigations

```bash
ssh cks006006
sudo -i

# SEARCH: "log audit" → audit-policy.yaml

# FILE 1: Create audit rules
vim /etc/kubernetes/logpolicy/sample-policy.yaml

# PATTERN: "4 Levels in order: RequestResponse > Request > Metadata > Metadata"
- level: RequestResponse       # 1. Full details for PVs
  resources:
  - group: ""
    resources: ["persistentvolumes"]

- level: Request               # 2. Body only for ConfigMaps in front-apps
  resources:
  - group: ""
    resources: ["configmaps"]
  namespaces: ["front-apps"]

- level: Metadata              # 3. Metadata for secrets/configmaps everywhere
  resources:
  - group: ""
    resources: ["secrets", "configmaps"]

- level: Metadata              # 4. Catch-all for everything else
  omitStages:
    - "RequestReceived"

# FILE 2: Configure kube-apiserver
vim /etc/kubernetes/manifests/kube-apiserver.yaml

# ADD 3 flags under "command:"
--audit-policy-file=/etc/kubernetes/logpolicy/sample-policy.yaml
--audit-log-path=/var/log/kubernetes/audit-logs.txt
--audit-log-maxage=10
--audit-log-maxbackup=2

# ADD volumeMounts (2 mounts)
volumeMounts:
  - mountPath: /etc/kubernetes/logpolicy/sample-policy.yaml
    name: audit
    readOnly: true
  - mountPath: /var/log/kubernetes/
    name: audit-log
    readOnly: false

# ADD volumes (2 volumes)
volumes:
- name: audit
  hostPath:
    path: /etc/kubernetes/logpolicy/sample-policy.yaml
    type: File
- name: audit-log
  hostPath:
    path: /var/log/kubernetes/
    type: DirectoryOrCreate

# RESTART & VERIFY
systemctl daemon-reload && systemctl restart kubelet
kubectl get pod -A
tail /var/log/kubernetes/audit-logs.txt   # Should see logs!
```
**Memory:** "**CFV** = **C**ommand flags, volumeMounts (**F**iles), **V**olumes" + "**RRM** levels = **R**esponse > **R**equest > **M**eta"[1]

***

#### Q4: Falco /dev/mem Detection (10 min) ⚠️ TRICKY
**What you're doing:** Create Falco rule to detect suspicious file access, find culprit, kill it
**Why:** /dev/mem = direct memory access = usually malicious

```bash
ssh cks004004
sudo -i

# SEARCH: "falco rules" → basic-elements

# Create custom rule
vim /etc/falco/falco_rules.local.yaml

# PATTERN: "List + Rule + Output"
- list: mem_file
  items: [/dev/mem]         # 1. Define suspicious file

- rule: devmem              # 2. Define rule
  desc: devmem
  condition: >
    fd.name in (mem_file)   # Trigger when file accessed
  output: >
    Shell (command=%proc.cmdline file=%fd.name container_id=%container.id)
  priority: NOTICE
  tags: [file]

# Run Falco for 30 seconds, save to log
sudo falco -M 30 -r /etc/falco/falco_rules.local.yaml >> devmem.log

# Find guilty container
cat devmem.log | grep container_id
# Copy the container_id value

sudo crictl ps | grep {container_id}   # Find which pod
kubectl get pod,deployment
kubectl scale deployment ollama --replicas=0   # Kill it!
```
**Memory:** "**LRO** = **L**ist items, **R**ule condition, **O**utput format"[1]

***

#### Q14: ImagePolicyWebhook (15 min) ⚠️ HARDEST - Skip if time low
**What you're doing:** Block unapproved container images from running
**Why:** Prevents running images from untrusted registries

```bash
ssh cks014014
sudo -i

# SEARCH: "ImagePolicyWebhook"

cd /etc/kubernetes/epconfig

# FILE 1: Policy config
vim image-policy-config.yaml

imagePolicy:
  kubeConfigFile: /etc/kubernetes/epconfig/kube-config.yaml
  allowTTL: 50
  denyTTL: 50
  retryBackoff: 500
  defaultAllow: false       # Memory: default = false (deny unless approved)

# FILE 2: Webhook server config
vim kube-config.yaml

- cluster:
    certificate-authority: /etc/kubernetes/epconfig/webhook.pem
    server: https://image-bouncer-webhook.default.svc:1323/image_policy
  name: bouncer_webhook

# FILE 3: Enable in kube-apiserver
vim /etc/kubernetes/manifests/kube-apiserver.yaml

# ADD to admission plugins
--enable-admission-plugins=NodeRestriction,ImagePolicyWebhook
--admission-control-config-file=/etc/kubernetes/epconfig/admission-control-config.json

# ADD volumeMounts + volumes (same pattern as Q6!)
volumeMounts:
  - name: image-policy
    mountPath: /etc/kubernetes/epconfig
volumes:
  - name: image-policy
    hostPath:
      path: /etc/kubernetes/epconfig
      type: DirectoryOrCreate

systemctl daemon-reload && systemctl restart kubelet
kubectl apply -f /home/candidate/web1.yaml   # Test it!
```
**Memory:** "**3 Files** = **P**olicy, **W**ebhook server, **A**PI server (PWA)"

***

## 🎯 EXAM STRATEGY - EXECUTION ORDER

### Phase 1: Build Confidence (0-40 min) - Do These First
```
✅ Q2  (5 min) - TLS Secret        [Easiest - start here!]
✅ Q8  (7 min) - Ingress
✅ Q16 (6 min) - Anonymous Auth
✅ Q3  (6 min) - Dockerfile
✅ Q5  (6 min) - Pod Security
✅ Q11 (6 min) - Pod Standards
✅ Q7  (7 min) - Network Policy

Total: 43 min | Score: 7 questions = 44% ✓
```

### Phase 2: Build Score (40-85 min) - Medium Questions
```
✅ Q1  (7 min) - CIS Benchmark
✅ Q9  (8 min) - Token Mount
✅ Q12 (8 min) - Docker Socket
✅ Q15 (8 min) - Node Upgrade
✅ Q10 (8 min) - SBOM
✅ Q13 (10 min) - Istio mTLS

Total: 49 min | Score: 13 questions = 81% ✓ PASS SECURED
```

### Phase 3: Bonus Points (85-120 min) - Hard Questions
```
⚠️ Q4  (10 min) - Falco         [Skip if <25 min left]
⚠️ Q6  (12 min) - Audit         [Skip if <20 min left]
⚠️ Q14 (15 min) - ImagePolicy   [Skip if <18 min left]

If time left: Do Q4 only (easier than Q6/Q14)
If 25+ min: Attempt all three
If 15- min: VERIFY Phase 1 & 2 answers instead!
```

**Critical Rule:** If you have < 15 minutes left, STOP doing new questions. Go back and verify your easy questions![5]

***

## 🔧 ESSENTIAL COMMANDS - ORGANIZED BY ACTION

### Setup Commands (Do Once at Start)
```bash
alias k='kubectl'              # Save 6 keystrokes per command [web:16]
alias kgp='kubectl get pod'
alias kga='kubectl get all'
export do='--dry-run=client -o yaml'  # Fast YAML generation
```

### Navigation Pattern (Every Question)
```bash
ssh cks00X00X              # Jump to node
sudo -i                    # Become root (if editing /etc or /var)
cd /path/to/files          # Go to work directory
exit                       # Exit from root
exit                       # Exit from node (back to base)
```

### Verification Commands (After Every Edit)
```bash
# After editing YAML files
kubectl get pod -A              # All pods should be Running
kubectl get deployment -n NS    # Check deployment updated
kubectl describe pod NAME       # Debug if failing

# After editing systemd/kubelet
systemctl daemon-reload         # Reload configs
systemctl restart kubelet       # Apply changes
systemctl status kubelet        # Check it started OK

# After audit/Falco changes
tail -f /var/log/kubernetes/audit-logs.txt   # Watch audit logs
cat devmem.log | grep container_id            # Find Falco results
```

### Emergency Commands (When Things Break)
```bash
# Pod not starting?
kubectl describe pod POD_NAME -n NAMESPACE
kubectl logs POD_NAME -n NAMESPACE

# API server not starting?
sudo crictl ps -a | grep kube-apiserver
sudo crictl logs CONTAINER_ID

# Kubelet issues?
journalctl -u kubelet -f
systemctl status kubelet
```

***

## 🧠 MEMORY PALACE TECHNIQUE[1]

**Create a mental journey through your house for the 16 questions:**

1. **Front Door** (Q2) = TLS Secret (door lock = certificate)
2. **Living Room** (Q8) = Ingress (guests enter here)
3. **Kitchen** (Q7) = Network Policy (firewall between rooms)
4. **Bedroom** (Q3,5,11) = Security Context (private room = restricted)
5. **Bathroom** (Q16) = Anonymous Auth (no guests allowed!)
6. **Garage** (Q12) = Docker Socket (car engine = Docker daemon)
7. **Basement** (Q1) = CIS Benchmark (foundation = security basics)
8. **Attic** (Q4,6) = Falco & Audit (watching from above)
9. **Garden** (Q13) = Istio (mesh = garden fence)
10. **Gate** (Q14) = ImagePolicy (guard checks who enters)

When you see a question, visualize the room in your house![1]

***

## ✅ FINAL CHECKLIST - 3 Days Before Exam

Practice these until you can do them without looking:

- [ ] Complete all 7 Phase 1 questions in 40 minutes
- [ ] SSH to node and back without mistakes
- [ ] Edit kube-apiserver.yaml and add volumeMounts/volumes
- [ ] Create Network Policy from memory
- [ ] Write securityContext with all 3-4 fields
- [ ] Use vim efficiently (dd, yy, p, :wq, :set paste)[5]
- [ ] Find documentation in < 30 seconds
- [ ] Verify changes with kubectl get/describe

**Passing Score:** 11/16 = 67% | **Your Goal:** 13/16 = 81% (skip 3 hardest) [6]

***

## 💡 FINAL TIPS

**DO:**
- Write "RRA ACRS CFV" on scratch paper at start (memory triggers)[4]
- Start with Q2 always (confidence builder)
- Flag questions > 8 minutes and return later[5]
- Use `kubectl explain` when stuck (e.g., `kubectl explain pod.spec.securityContext`)
- Copy-paste from blue text in exam

**DON'T:**
- Try to do questions in numerical order
- Spend >10 min on any question first pass
- Skip verification steps (causes lost points)
- Edit files without `:set paste` in vim (causes indentation errors)
- Panic - you only need 11 correct to pass!

Good luck! Practice Phase 1 questions until they're automatic, and you'll pass easily! 🚀

來源
[1] 3 Ways to Memorize Linux Commands https://www.youtube.com/watch?v=cOwdNFy979o
[2] How to Pass the Certified Kubernetes Security Specialist Exam https://www.freecodecamp.org/news/how-to-pass-the-certified-kubernetes-security-specialist-exam/
[3] 7 ways to remember Linux commands https://www.networkworld.com/article/968201/7-ways-to-remember-linux-commands.html
[4] Terminal: How do folks memorize command, and how do folks know what command to run? https://www.reddit.com/r/linuxquestions/comments/rs78fe/terminal_how_do_folks_memorize_command_and_how_do/
[5] CKS/CKA exam tips (TL;DR) - LinkedIn https://www.linkedin.com/pulse/cks-exam-tips-tldr-walter-lee
[6] Certified Kubernetes Security Specialist (CKS) Exam Guide https://kodekloud.com/blog/certified-kubernetes-security-specialist-cks-exam-verification-guide/
[7] Linux memorizing commands ? : r/linux https://www.reddit.com/r/linux/comments/197sr6s/linux_memorizing_commands/
[8] Memorizing Network Commands - General Memory Chat https://forum.artofmemory.com/t/memorizing-network-commands/55078
[9] 3 Tools to Help You Remember Linux Commands https://www.linux.com/topic/desktop/3-tools-help-you-remember-linux-commands/
[10] Kubectl Cheat Sheet with Examples- 50 Quick Commands | Pomerium https://www.pomerium.com/blog/kubectl-cheat-sheet-with-examples
[11] Kubernetes Cheat Sheet | Mirantis https://www.mirantis.com/blog/kubernetes-cheat-sheet/
