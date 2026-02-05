## 02/02/2026

### Q1 CIS Benchmark

```bash
sudo -i
vim /var/lib/kubelet/config.yaml
anonymous: false
webhook enabled: true
auth mode: Webhook

vim /etc/kubernetes/manifest/etcd.yaml
--client-cert-ahth: true

systemctl deamon-reload
systemcll restart kubelet
---

sudo -i
vim /var/lib/kubelet/config.yaml
anonymous
    enabled: false
webhook
    enabled: true
authorization
    mode: Webhook

vim /etc/kubernetes/manifests/etcd.yaml
--client-cert-auth=true

systemctl daemon-reload
systemctl restart kubelet

```

### Q2 TLS Secret

```bash
kubectl -n clever-cactus create secret tls clever-cactus --cert=/home/candidate/ca-cert/web.k8s.local.crt --key=/home/candidate/ca-cert/web.k8s.local.key

```

### Q3 Docker Security

```bash
vim /cks/docker/Dockerfile

USER nobody

vim /cks/docker/deployment.yaml

securityContext:
  runAsUser: 65535
  readOnlyRootFilesystem: true
  allowPrevilegeEscalation: false

```

### Q4 Falco **********
```bash
```

### Q5 Pod Security Context

```bash
vim /home/candidate/sec-ns-deployment.yaml

securityContenxt:
  runAsUser: 30000
  readOnlyRootFilesystem: true
  allowPrevilegeEscalation: false

kubectl apply -f /home/candidate/sec-ns-deployment.yaml
kubectl -n sec-ns get pod, deployment
```

### Q6 Audit Logging ************
```bash
```


### Q7 Network Policies

```bash
vim deny-policy.yaml

---
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

vim allow-from-prod.yaml

---
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
          project: prod

kubectl apply -f allow-from-prod.yaml
kubectl get networkpolicy -n data

```

### Q8 Ingress TLS

```bash
vim ingress.yaml

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web
  namespace: prod02
  annotations:
    ingressclass.kubernetes.io/ssl-redirect: "true"
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
```

### Q9 Disable Token Auto-Mount

```bash
kubectl -n monitoring edit sa stats-monitor-sa

automountServiceAccountToken: false

vim /home/candidate/stats-monitor/deployment.yaml


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
kubectl get deployment -n monitoring
```

### Q10 SBOM Generation

```bash
kubectl get pod -n apline
cat /home/candidate/apline-deployment.yaml

kubectl -n apline exec -it alpine-7685556c79-nmxjn -c apline-a -- apk list| grep libcrypto3
kubectl -n apline exec -it alpine-7685556c79-nmxjn -c apline-b -- apk list| grep libcrypto3
kubectl -n apline exec -it alpine-7685556c79-nmxjn -c apline-c -- apk list| grep libcrypto3

bom generate --image registry.cn-qingdao.aliyuncs.com/containerhub/alpine:3.19.1 --output /home/candidate/apline.spdx

vim /home/candidate/apline-deployment.yaml

delete apline-b

kubectl apply -f /home/candidate/apline-deployment.yaml
kubectl get pod -n apline
```

### Q11 Pod security

```bash
kubectl -n confidential get deployment 
kubectl delete -f /home/candidate/nginx-unprivileged.yaml
kubectl apply -f /home/candidate/nginx-unprivileged.yaml

#Check the logs after create deployment
# Search "security context"
vim /home/candidate/nginx-unprivileged.yaml

securityContext:
  allowPrivilegeEscalation: false
  capabilities:
    drop: ["ALL"]
  runAsNonRoot: true
  seccompProfile:
    type: RuntimeDefault 

kubectl apply -f /home/candidate/nginx-unprivileged.yaml
kubectl -n confidential get deployment 
```

### Q12 Docker Security

```bash
sudo -i

# Remove docker socket
id developer
gpasswd -d developer docker
id developer

vim /usr/lib/systemd/system/docker.socket
SocketGroup: root

vim /usr/lib/systemd/system/docker.service
# Remove -H tcp://0.0.0.0:2375
systemctl daemon-reload
systemctl restart docker.socket
systemctl restart docker.service
```

### Q13 Istio mTLS

```bash
kubectl label namespace mtls istio-injection=enabled
kubectl rollout restart deployment -n mtls

kubectl get ns mtls --show-labels
kubectl -n mtls get pod
kubectl -n mtls get deployment

kubectl get peerauthentication -n mlts
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

### Q14 ImagePolicyWebhook

```bash

```

### Q15 Node Upgrade

```bash

ssh node02
sudo -i

kubectl get node
apt update
apt-cache madison kubeadm

apt install kubeadm='1.32.1-1'

systemctl daemon-reload
systemctl restart kubelet


```

### Q16 API Server

```bash
vim /etc/kubernetes/manifests/kube-apiserver.yaml

- --anonymous-auth=false
- --enable-admission-plugins=NodeRestriction

kubectl daemon-reload
kubectl restart kubelet

kubectl --kubeconfig=/etc/kubernetes/admin.conf get pod -A
kubectl --kubeconfig=/etc/kubernetes/admin.conf delete clusterrolebinding system:anonymous
```

