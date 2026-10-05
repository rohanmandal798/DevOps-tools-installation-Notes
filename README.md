# DevOps Tools Installation Notes

Two setups in one file:

- **[Part 1: Dev / Learning Setup](#part-1-dev--learning-setup)**: quick installs for a laptop, VM or lab. Fast, not hardened.
- **[Part 2: Production Setup](#part-2-production-setup)**: repeatable, secured, persistent, HA where it matters.

**Tools covered:** Docker, Jenkins, kubectl, Kubernetes, ArgoCD, Nginx, Prometheus, Grafana

---

# Part 1: Dev / Learning Setup

> For local machines and test servers only. Not for production.

## 1.1 Docker

```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
rm get-docker.sh
sudo usermod -aG docker $USER
newgrp docker
docker run hello-world
```

---

## 1.2 Jenkins

### Install Java

```bash
sudo apt update
sudo apt install fontconfig openjdk-21-jre
java -version
```

### Add Jenkins repository and install

```bash
sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2026.key
echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/" | \
  sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt update
sudo apt install jenkins
```

### Enable and start

```bash
sudo systemctl enable jenkins
sudo systemctl start jenkins
sudo systemctl status jenkins
```

Open `http://<server-ip>:8080`. Initial password:

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

---

## 1.3 kubectl

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
kubectl version --client
```

---

## 1.4 Kubernetes (Minikube)

```bash
curl -LO https://github.com/kubernetes/minikube/releases/latest/download/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube && rm minikube-linux-amd64

minikube start
kubectl get nodes
```

---

## 1.5 ArgoCD

### CLI

```bash
VERSION=$(curl -L -s https://raw.githubusercontent.com/argoproj/argo-cd/stable/VERSION)
curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/download/v$VERSION/argocd-linux-amd64
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
rm argocd-linux-amd64
```

### Server on Minikube (optional)

```bash
kubectl create namespace argocd
kubectl apply -n argocd --server-side --force-conflicts \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

kubectl port-forward svc/argocd-server -n argocd 8080:443

# Admin password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d; echo
```

Open `https://localhost:8080` (user: `admin`).

---

## 1.6 Nginx

```bash
sudo apt install nginx
sudo systemctl enable --now nginx
```

Test: open `http://<server-ip>`.

---

## 1.7 Prometheus + Grafana (optional, on Minikube)

```bash
# Helm
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace

# Access
kubectl port-forward svc/monitoring-grafana -n monitoring 3000:80
kubectl port-forward svc/monitoring-kube-prometheus-prometheus -n monitoring 9090:9090

# Grafana admin password (default user: admin)
kubectl get secret monitoring-grafana -n monitoring \
  -o jsonpath="{.data.admin-password}" | base64 -d; echo
```

---

# Part 2: Production Setup

> Pin versions, use TLS, persistent storage and HA, and keep secrets out of config files.

## 2.0 Prerequisites

```bash
# Helm
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version

# Cluster access check
kubectl get nodes
kubectl get storageclass     # a default StorageClass is needed for persistent volumes
```

You also need an Ingress controller (e.g. ingress-nginx) and cert-manager for TLS on the cluster.

---

## 2.1 Docker

Use the official apt repository (updates via `apt`, no piped scripts).

```bash
sudo apt update
sudo apt install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

sudo systemctl enable --now docker
sudo docker run hello-world
```

### Daemon hardening: `/etc/docker/daemon.json`

```json
{
  "log-driver": "json-file",
  "log-opts": { "max-size": "10m", "max-file": "3" },
  "live-restore": true
}
```

```bash
sudo systemctl restart docker
```

Notes:
- Membership in the `docker` group is **root-equivalent**. On production servers avoid `usermod -aG docker` for regular users; use `sudo docker` or rootless mode.
- Pin versions: `apt-cache madison docker-ce`, then `apt install docker-ce=<version>`.
- Run containers with `--restart unless-stopped`, resource limits and non-root users.

---

## 2.2 Jenkins

Install Java and the Jenkins repo as in Part 1, then bind Jenkins to localhost and put Nginx (TLS) in front of it.

```bash
sudo systemctl edit jenkins
```

Add:

```ini
[Service]
Environment="JENKINS_LISTEN_ADDRESS=127.0.0.1"
Environment="JAVA_OPTS=-Djava.awt.headless=true -Xms1g -Xmx2g"
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now jenkins
sudo systemctl status jenkins

# Initial admin password
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

### Checklist
- Put `/var/lib/jenkins` (JENKINS_HOME) on a dedicated disk and back it up (ThinBackup plugin or volume snapshots).
- Run builds on **agents**, not the built-in controller (set controller executors to 0).
- Enable security: matrix/role-based authorization, SSO/LDAP, disable anonymous access.
- Store credentials in the Jenkins Credentials store, never in Jobfiles.
- Keep Jenkins and plugins updated; install only needed plugins.
- Avoid adding the `jenkins` user to the `docker` group on the controller; use Docker-capable agents.
- Set the Jenkins URL to `https://jenkins.example.com/` under Manage Jenkins > System.

---

## 2.3 kubectl

Install from the official apt repo and match the cluster version (kubectl is supported within one minor version of the API server).

```bash
K8S_MINOR=v1.34    # set to your cluster's minor version

sudo apt update
sudo apt install -y apt-transport-https ca-certificates curl gpg
sudo mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/${K8S_MINOR}/deb/Release.key | \
  sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/${K8S_MINOR}/deb/ /" | \
  sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt update
sudo apt install -y kubectl
sudo apt-mark hold kubectl

kubectl version --client
```

Tips:
- Keep kubeconfig private: `chmod 600 ~/.kube/config`.
- Use separate contexts per cluster (`kubectl config get-contexts`) and RBAC-scoped users, not cluster-admin, for daily work.

---

## 2.4 Kubernetes (kubeadm)

Minikube is single-node and for dev only. For production use a managed service (EKS / GKE / AKS) or kubeadm with multiple nodes.

### On every node (control plane and workers)

```bash
# 1. Disable swap
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab

# 2. Kernel modules and sysctl
cat <<EOT | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOT
sudo modprobe overlay
sudo modprobe br_netfilter

cat <<EOT | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOT
sudo sysctl --system

# 3. Container runtime (containerd, from the Docker repo set up in 2.1)
sudo apt install -y containerd.io
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl enable containerd

# 4. kubeadm, kubelet, kubectl (apt repo from 2.3)
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

### Initialize the control plane (first node)

For HA, put a load balancer in front of 3 control-plane nodes and use its address as the endpoint.

```bash
sudo kubeadm init \
  --control-plane-endpoint "k8s-api.example.com:6443" \
  --upload-certs \
  --pod-network-cidr=192.168.0.0/16

mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### Install a CNI (network plugin), e.g. Calico or Cilium

Follow the CNI's current install guide, then verify:

```bash
kubectl get pods -A
kubectl get nodes
```

### Join nodes

`kubeadm init` prints the join commands. To regenerate a worker join command:

```bash
kubeadm token create --print-join-command
```

### Checklist
- 3 control-plane nodes (odd number) behind a load balancer; 3+ workers.
- Regular **etcd backups** (`etcdctl snapshot save`).
- Enable RBAC (default), NetworkPolicies and Pod Security Standards.
- Add an Ingress controller (ingress-nginx), cert-manager, metrics-server and a CSI storage driver.
- Plan upgrades: `kubeadm upgrade plan` / `kubeadm upgrade apply`, one minor version at a time.

---

## 2.5 Nginx

Use as a TLS-terminating reverse proxy (here in front of Jenkins).

```bash
sudo apt update
sudo apt install -y nginx
sudo systemctl enable --now nginx

# Firewall
sudo ufw allow 'Nginx Full'
sudo ufw allow OpenSSH
sudo ufw enable
```

### Reverse proxy for Jenkins: `/etc/nginx/sites-available/jenkins`

```nginx
server {
    listen 80;
    server_name jenkins.example.com;

    location / {
        proxy_pass         http://127.0.0.1:8080;
        proxy_set_header   Host              $host;
        proxy_set_header   X-Real-IP         $remote_addr;
        proxy_set_header   X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Proto $scheme;
        proxy_http_version 1.1;
        proxy_request_buffering off;
        proxy_read_timeout 90s;
    }
}
```

```bash
sudo ln -s /etc/nginx/sites-available/jenkins /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

### TLS with Let's Encrypt

```bash
sudo apt install -y certbot python3-certbot-nginx
sudo certbot --nginx -d jenkins.example.com
sudo certbot renew --dry-run
```

### Hardening: in the `http {}` block of `/etc/nginx/nginx.conf`

```nginx
server_tokens off;
client_max_body_size 50m;
```

---

## 2.6 ArgoCD (HA, Helm)

### Add repo and create namespace

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
kubectl create namespace argocd
```

### `argocd-values.yaml`

```yaml
global:
  domain: argocd.example.com

redis-ha:
  enabled: true

controller:
  replicas: 1          # scale via sharding if managing many clusters

server:
  replicas: 2
  ingress:
    enabled: true
    ingressClassName: nginx
    annotations:
      cert-manager.io/cluster-issuer: letsencrypt-prod
      nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
    tls: true

repoServer:
  replicas: 2

applicationSet:
  replicas: 2
```

### Install

```bash
# pin a chart version in production: --version <x.y.z>
helm install argocd argo/argo-cd \
  --namespace argocd \
  -f argocd-values.yaml

kubectl get pods -n argocd
```

### Initial admin password and login

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d; echo

# Without ingress (testing)
kubectl port-forward svc/argocd-server -n argocd 8080:443

argocd login localhost:8080 --username admin --insecure
argocd account update-password
```

After changing the password, delete the initial secret:

```bash
kubectl -n argocd delete secret argocd-initial-admin-secret
```

---

## 2.7 Prometheus + Grafana (kube-prometheus-stack)

One chart installs Prometheus, Alertmanager, Grafana, exporters and dashboards.

### Add repo and create namespace

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
kubectl create namespace monitoring
```

### Create Grafana admin secret (don't hardcode passwords)

```bash
kubectl create secret generic grafana-admin \
  --namespace monitoring \
  --from-literal=admin-user=admin \
  --from-literal=admin-password='CHANGE_ME_STRONG_PASSWORD'
```

### `monitoring-values.yaml`

```yaml
prometheus:
  prometheusSpec:
    retention: 15d
    replicas: 2
    storageSpec:
      volumeClaimTemplate:
        spec:
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 50Gi
    resources:
      requests:
        cpu: 500m
        memory: 2Gi
      limits:
        memory: 4Gi

alertmanager:
  alertmanagerSpec:
    replicas: 3
    storage:
      volumeClaimTemplate:
        spec:
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 5Gi

grafana:
  admin:
    existingSecret: grafana-admin
    userKey: admin-user
    passwordKey: admin-password
  persistence:
    enabled: true
    size: 10Gi
  ingress:
    enabled: true
    ingressClassName: nginx
    hosts:
      - grafana.example.com
    annotations:
      cert-manager.io/cluster-issuer: letsencrypt-prod
    tls:
      - secretName: grafana-tls
        hosts:
          - grafana.example.com
```

### Install

```bash
# pin a chart version in production: --version <x.y.z>
helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  -f monitoring-values.yaml

kubectl get pods -n monitoring
```

### Access (testing without ingress)

```bash
# Grafana
kubectl port-forward svc/monitoring-grafana -n monitoring 3000:80

# Prometheus
kubectl port-forward svc/monitoring-kube-prometheus-prometheus -n monitoring 9090:9090

# Alertmanager
kubectl port-forward svc/monitoring-kube-prometheus-alertmanager -n monitoring 9093:9093
```

---

## 2.8 Monitor ArgoCD with Prometheus

Add to `argocd-values.yaml`, then upgrade:

```yaml
controller:
  metrics:
    enabled: true
    serviceMonitor:
      enabled: true
      additionalLabels:
        release: monitoring     # must match the kube-prometheus-stack release name
server:
  metrics:
    enabled: true
    serviceMonitor:
      enabled: true
      additionalLabels:
        release: monitoring
repoServer:
  metrics:
    enabled: true
    serviceMonitor:
      enabled: true
      additionalLabels:
        release: monitoring
```

```bash
helm upgrade argocd argo/argo-cd -n argocd -f argocd-values.yaml
```

Import the official ArgoCD dashboard in Grafana (Dashboards > Import) using ID `14584`.

---

## 2.9 Upgrade / Uninstall

```bash
helm upgrade monitoring prometheus-community/kube-prometheus-stack -n monitoring -f monitoring-values.yaml
helm upgrade argocd argo/argo-cd -n argocd -f argocd-values.yaml

helm uninstall monitoring -n monitoring
helm uninstall argocd -n argocd
```

---

## 2.10 Production Checklist

- [ ] Pin versions (apt packages and Helm `--version`)
- [ ] TLS via cert-manager + Ingress, or certbot on Nginx
- [ ] Persistent volumes for Prometheus, Alertmanager, Grafana
- [ ] Strong admin passwords from Secrets; SSO/OIDC for ArgoCD, Grafana and Jenkins
- [ ] Resource requests/limits set
- [ ] Alertmanager receivers configured (Slack / email / PagerDuty)
- [ ] Backups: etcd, JENKINS_HOME, ArgoCD config (`argocd admin export`), Grafana dashboards as code
- [ ] Firewall enabled, only needed ports open
- [ ] Manage these installs through ArgoCD (GitOps)

---

# Dev vs Production Summary

| Tool | Dev (Part 1) | Production (Part 2) |
|------|--------------|---------------------|
| Docker | `get.docker.com` script, user in docker group | apt repo, `daemon.json` log limits, no docker group |
| Jenkins | Direct on port 8080 | Localhost only + Nginx TLS, agents, backups |
| kubectl | Latest binary download | apt repo pinned to cluster version |
| Kubernetes | Minikube (single node) | kubeadm HA cluster or managed (EKS/GKE/AKS) |
| ArgoCD | Plain manifest + port-forward | Helm HA install, Ingress + TLS |
| Prometheus / Grafana | Default Helm install, port-forward | `kube-prometheus-stack` with persistence, secrets, ingress |
| Nginx | `apt install nginx` | TLS via certbot, hardened, reverse proxy |
