Below implimentation steps is not the final refined one but the original one I used as the starting point. You may get minor blockers and errors which is requried as part of the learning WF. I'll later post the updated, refined implimentation steps as a reference. Please note, the refined implimentation steps offer fewer learning opportunities and should only be used as a reference.

This project demonstrates **GitOps maturity** from basic CD to progressive delivery, multi-cluster management, and production hardening - exactly what senior DevOps roles look for.

## ArgoCD Implementation Roadmap - DevOps Interview Perspective

### Prerequisites Validation (5 min)
```bash
# Verify your environment
kubectl get nodes
kubectl get ns
kubectl get svc -n metallb-system
helm version
```

---

## Step 1: Install ArgoCD (10 min)

```bash
# Create namespace
kubectl create namespace argocd

# Install ArgoCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for all pods to be ready
kubectl get pods -n argocd -w
```

**Interview Talking Point**: "I'm using the non-HA installation for demonstration. In production, we'd use the HA manifest with Redis HA, multiple replicas, and proper resource limits."

---

## Step 2: Expose ArgoCD Server (10 min)

### Option A: NodePort (for Vagrant VM)
```bash
# Patch ArgoCD server service
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "NodePort", "ports": [{"port": 443, "nodePort": 30443}]}}'

# Get access URL
NODE_IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[0].address}')
echo "https://$NODE_IP:30443"
```

### Option B: Ingress (if you have NGINX Ingress)
```bash
# Install NGINX Ingress (if not present)
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.1/deploy/static/provider/cloud/deploy.yaml

# Create Ingress for ArgoCD
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-server
  namespace: argocd
  annotations:
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
    nginx.ingress.kubernetes.io/ssl-passthrough: "true"
spec:
  rules:
  - host: argocd.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: argocd-server
            port:
              number: 443
EOF

# Add to /etc/hosts (on host machine, not VM)
# echo "192.168.56.10 argocd.local" | sudo tee -a /etc/hosts
```

---

## Step 3: Initial Authentication & CLI Setup (5 min)

```bash
# Get initial admin password
ARGO_PWD=$(kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d)
echo "Admin Password: $ARGO_PWD"

# Install ArgoCD CLI
curl -sSL -o argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
chmod +x argocd
sudo mv argocd /usr/local/bin/

# Login via CLI
argocd login $NODE_IP:30443 --username admin --password $ARGO_PWD --insecure

# Change default password
argocd account update-password --current-password $ARGO_PWD --new-password devops123
```

---

## Step 4: Create Demo Application Manifests (15 min)

```bash
# Create a Git repo structure (simulating a real repo)
mkdir -p ~/demo-app/{base,overlays/dev,overlays/prod}
cd ~/demo-app

# Base deployment
cat <<EOF > base/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: guestbook
spec:
  replicas: 2
  selector:
    matchLabels:
      app: guestbook
  template:
    metadata:
      labels:
        app: guestbook
    spec:
      containers:
      - name: guestbook
        image: gcr.io/heptio-images/ks-guestbook-demo:0.2
        ports:
        - containerPort: 80
EOF

# Base service
cat <<EOF > base/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: guestbook
spec:
  selector:
    app: guestbook
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
EOF

# Base kustomization
cat <<EOF > base/kustomization.yaml
apiVersion: kustomize.toolkit.istio/v1beta1
kind: Kustomization
resources:
- deployment.yaml
- service.yaml
EOF

# Dev overlay
cat <<EOF > overlays/dev/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- ../../base
namespace: dev
patches:
- target:
    kind: Deployment
    name: guestbook
  patch: |-
    - op: replace
      path: /spec/replicas
      value: 1
EOF

# Prod overlay
cat <<EOF > overlays/prod/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- ../../base
namespace: prod
patches:
- target:
    kind: Deployment
    name: guestbook
  patch: |-
    - op: replace
      path: /spec/replicas
      value: 3
EOF

# Initialize git
git init
git add .
git commit -m "Initial guestbook application"
```

**Interview Talking Point**: "I'm using Kustomize for environment-specific configurations. This is crucial for maintaining DRY principles while supporting multiple environments."

---

## Step 5: Configure ArgoCD Application (10 min)

### Method 1: CLI-Based
```bash
# Create dev namespace
kubectl create ns dev

# Create ArgoCD app via CLI
argocd app create guestbook-dev \
  --repo file:///home/vagrant/demo-app \
  --path overlays/dev \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace dev \
  --sync-policy automated \
  --auto-prune \
  --self-heal

# Sync app
argocd app sync guestbook-dev
```

### Method 2: GitOps Declarative
```bash
# Create App of Apps pattern
cat <<EOF > argocd-apps.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook-prod
  namespace: argocd
spec:
  project: default
  source:
    repoURL: file:///home/vagrant/demo-app
    targetRevision: HEAD
    path: overlays/prod
  destination:
    server: https://kubernetes.default.svc
    namespace: prod
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
---
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: demo-project
  namespace: argocd
spec:
  description: Demo project for DevOps interview
  sourceRepos:
  - '*'
  destinations:
  - namespace: '*'
    server: https://kubernetes.default.svc
  clusterResourceWhitelist:
  - group: '*'
    kind: '*'
EOF

kubectl apply -f argocd-apps.yaml
```

---

## Step 6: Demonstrate GitOps Workflow (15 min)

### Demo Scenario 1: Auto-healing
```bash
# Manual change outside GitOps
kubectl scale deployment guestbook -n dev --replicas=5

# Watch ArgoCD revert it (30 sec - 3 min)
argocd app get guestbook-dev --refresh
# or
watch kubectl get pods -n dev
```

### Demo Scenario 2: Application Update
```bash
# Update image version in Git
cd ~/demo-app
sed -i 's|image: gcr.io/heptio-images/ks-guestbook-demo:0.2|image: gcr.io/heptio-images/ks-guestbook-demo:0.3|' base/deployment.yaml

git add .
git commit -m "Update guestbook to v0.3"

# ArgoCD will auto-sync (or manual sync)
argocd app sync guestbook-dev
```

### Demo Scenario 3: Rollback
```bash
# View history
argocd app history guestbook-dev

# Rollback to previous version (e.g., ID 1)
argocd app rollback guestbook-dev 1
```

---

## Step 7: Advanced Features - Progressive Delivery (20 min)

### Blue-Green Deployment with ArgoCD Rollouts
```bash
# Install Argo Rollouts
kubectl create namespace argo-rollouts
kubectl apply -n argo-rollouts -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml

# Install Rollouts CLI
curl -LO https://github.com/argoproj/argo-rollouts/releases/latest/download/kubectl-argo-rollouts-linux-amd64
chmod +x kubectl-argo-rollouts-linux-amd64
sudo mv kubectl-argo-rollouts-linux-amd64 /usr/local/bin/kubectl-argo-rollouts

# Create Rollout definition
cat <<EOF > ~/demo-app/base/rollout.yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: guestbook-rollout
spec:
  replicas: 3
  strategy:
    blueGreen:
      activeService: guestbook-active
      previewService: guestbook-preview
      autoPromotionEnabled: false
  selector:
    matchLabels:
      app: guestbook
  template:
    metadata:
      labels:
        app: guestbook
    spec:
      containers:
      - name: guestbook
        image: gcr.io/heptio-images/ks-guestbook-demo:0.2
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: guestbook-active
spec:
  selector:
    app: guestbook
  ports:
  - port: 80
---
apiVersion: v1
kind: Service
metadata:
  name: guestbook-preview
spec:
  selector:
    app: guestbook
  ports:
  - port: 80
EOF

# Update kustomization
cd ~/demo-app
git add .
git commit -m "Add Blue-Green rollout strategy"

# Promote rollout
kubectl argo rollouts promote guestbook-rollout -n dev
```

---

## Step 8: Monitoring & Troubleshooting (10 min)

### ArgoCD Web UI Commands
```bash
# Check sync status
argocd app get guestbook-dev --show-params

# View diff between live and desired state
argocd app diff guestbook-dev

# Resource tree view
argocd app manifests guestbook-dev

# Logs
argocd app logs guestbook-dev --tail=50
```

### Web UI Operations
```bash
# Access UI
echo "Access: https://$NODE_IP:30443"
echo "Username: admin, Password: devops123"

# In UI, demonstrate:
# 1. App topology view
# 2. Sync/Rollback operations
# 3. Resource visualization
# 4. Parameter overrides
```

---

## Step 9: Security & RBAC (15 min)

### Configure SSO and RBAC
```bash
# Create custom role
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-rbac-cm
  namespace: argocd
data:
  policy.default: role:readonly
  policy.csv: |
    p, role:dev-lead, applications, sync, dev/*, allow
    p, role:dev-lead, applications, update, dev/*, allow
    g, alice, role:dev-lead
    p, role:viewer, applications, get, */*, allow
    g, bob, role:viewer
EOF

# Restart ArgoCD server
kubectl rollout restart deployment argocd-server -n argocd
```

---

## Step 10: Production-Ready Enhancements (Bonus - 10 min)

### Configure Notifications
```bash
# Install notifications controller
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj-labs/argocd-notifications/release-1.1/manifests/install.yaml

# Configure Slack notifications
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-notifications-cm
  namespace: argocd
data:
  service.slack: |
    token: \$slack-token
  template.app-sync-succeeded: |
    message: |
      Application {{.app.metadata.name}} sync succeeded!
  trigger.on-sync-succeeded: |
    - description: App sync succeeded
      oncePer: app.sync-status.succeeded
      send: [app-sync-succeeded]
EOF
```

### Image Updater Setup
```bash
# Install ArgoCD Image Updater
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj-labs/argocd-image-updater/stable/manifests/install.yaml

# Annotate application for auto-updates
argocd app set guestbook-dev \
  --annotations 'argocd-image-updater.argoproj.io/image-list=guestbook=gcr.io/heptio-images/ks-guestbook-demo' \
  --annotations 'argocd-image-updater.argoproj.io/guestbook.update-strategy=latest'
```

---

## Interview Q&A Preparation

### Key Questions You Should Answer:

**Q1: How does ArgoCD differ from other CD tools like Jenkins?**
- *Answer*: "ArgoCD is pull-based, GitOps-centric. Jenkins is push-based. ArgoCD constantly reconciles cluster state with Git, providing drift correction and declarative management."

**Q2: How do you handle secrets?**
- *Answer*: "Use SealedSecrets, ExternalSecretsOperator, or HashiCorp Vault. Never store plain secrets in Git. For demo, we use Kubernetes secrets encrypted with SOPS."

**Q3: Explain App of Apps pattern**
- *Answer*: "A parent Application that defines and deploys child Applications. Enables managing multiple microservices from a single Git repository with hierarchical configuration."

**Q4: Disaster recovery strategy?**
- *Answer*: "Backup ArgoCD's cluster-secret (encryption key), application CRDs, and project configs. Regular `argocd export` for full state backup."

**Q5: Multi-cluster management?**
- *Answer*: "Add clusters via `argocd cluster add`. Use cluster-scoped resources carefully. Implement cluster sharding for large deployments."

---

## Sample Interview Demo Script (30 min timed)

```bash
# 0:00 - Start
show_version() {
  echo "=== ArgoCD Demo - DevOps Interview ==="
  argocd version --short
}

# 1:00 - Show current state
show_state() {
  echo "=== Current Applications ==="
  argocd app list
  echo "=== Pods in dev namespace ==="
  kubectl get pods -n dev
}

# 5:00 - Introduce config drift
introduce_drift() {
  kubectl delete pod -n dev -l app=guestbook
  echo "Manually deleted pod - watch auto-healing"
  watch -n 2 'kubectl get pods -n dev'
}

# 10:00 - Rollout new version
rollout_new() {
  cd ~/demo-app
  sed -i 's/v0.2/v0.4/' base/deployment.yaml
  git add . && git commit -m "Version bump v0.4"
  argocd app sync guestbook-dev
}

# 15:00 - Rollback
rollback_demo() {
  argocd app rollback guestbook-dev 1
  echo "Rolled back to previous version"
}

# 20:00 - Scale demonstration
scale_demo() {
  argocd app set guestbook-dev -p "replicas=5"
  echo "Scaled via parameter override"
}

# 25:00 - Cleanup
cleanup() {
  argocd app delete guestbook-dev --yes
  kubectl delete ns dev prod argo-rollouts
}
```

---

## Success Criteria Checklist

- ✅ ArgoCD accessible via UI/CLI
- ✅ Application auto-syncing from Git
- ✅ Drift detection & auto-healing demonstrated
- ✅ Rollback functionality shown
- ✅ Multi-environment (dev/prod) management
- ✅ CLI operations fluent
- ✅ Can explain GitOps principles
- ✅ Troubleshooting commands known
- ✅ Security/RBAC configured
- ✅ Blue-Green or Canary strategy demoed

---

## Common Pitfalls to Avoid

1. **Long sync times** - Use `--reconcile-timeout` flags, check network
2. **OutOfSync errors** - Use `--prune` flag, check finalizers
3. **Resource conflicts** - Set correct resource ordering via sync waves
4. **Secret management** - Never use plain secrets in Git
5. **Performance issues** - Implement app-of-apps sharding

This implementation demonstrates **GitOps maturity** from basic CD to progressive delivery, multi-cluster management, and production hardening - exactly what senior DevOps roles look for.