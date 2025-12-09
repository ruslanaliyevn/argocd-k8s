<div align="center">
  <img src="https://argo-cd.readthedocs.io/en/stable/assets/logo.png" alt="ArgoCD Logo" width="200"/>
  <h1>ArgoCD Kubernetes Handbook</h1>
  <p>
    <img src="https://img.shields.io/badge/Kubernetes-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white" alt="Kubernetes"/>
    <img src="https://img.shields.io/badge/ArgoCD-EF7B4D?style=for-the-badge&logo=argo&logoColor=white" alt="ArgoCD"/>
    <img src="https://img.shields.io/badge/Istio-466BB0?style=for-the-badge&logo=istio&logoColor=white" alt="Istio"/>
    <img src="https://img.shields.io/badge/GitOps-FC6D26?style=for-the-badge&logo=git&logoColor=white" alt="GitOps"/>
    <img src="https://img.shields.io/badge/License-MIT-green?style=for-the-badge" alt="License"/>
  </p>
  <p><strong>Production-ready ArgoCD configurations and GitOps best practices for Kubernetes</strong></p>
</div>

---

## 📖 Table of Contents

- [About](#-about)
- [Prerequisites](#-prerequisites)
- [Repository Structure](#-repository-structure)
- [Quick Start](#-quick-start)
- [Installation Guide](#-installation-guide)
- [Usage](#-usage)
- [Key Features](#-key-features)
- [Learning Path](#-learning-path)
- [Best Practices](#-best-practices)
- [Troubleshooting](#-troubleshooting)
- [Resources](#-resources)
- [Contributing](#-contributing)
- [License](#-license)

---

## 🎯 About

**ArgoCD** is a declarative, GitOps continuous delivery tool for Kubernetes. This handbook provides everything you need to:

- Install and configure ArgoCD on Kubernetes
- Expose ArgoCD securely via Istio Gateway
- Connect to Git repositories (GitLab/GitHub)
- Deploy applications using GitOps principles
- Implement automated synchronization and self-healing

### Why This Repository?

✅ **Production-tested** configurations  
✅ **Step-by-step** instructions with examples  
✅ **Istio integration** with TLS Passthrough  
✅ **Real-world** deployment scenarios  
✅ **Best practices** and troubleshooting tips  

---

## 📋 Prerequisites

Before you begin, ensure you have:

- **Kubernetes cluster** (v1.20 or higher)
- **kubectl** installed and configured
- **Istio** installed (for ingress gateway configuration)
- **Git** repository (GitLab or GitHub)
- Basic understanding of Kubernetes concepts

### Optional Tools

- `argocd` CLI tool
- `helm` (for Helm-based deployments)

---

## 📁 Repository Structure

```
argocd-k8s/
├── README.md                          # This file
├── .gitignore                         # Git ignore rules
├── installation/                      # ArgoCD installation manifests
│   ├── 01-namespace.yaml              # Namespace creation
│   ├── 02-gateway.yaml                # Istio Gateway configuration
│   └── 03-virtualservice.yaml         # VirtualService for routing
└── example-app/                       # Complete application example
    ├── README.md                      # Detailed deployment guide
    └── nginx-app.yaml                 # Sample nginx application
```

---

## ⚡ Quick Start

Get ArgoCD up and running in 5 minutes:

```bash
# 1. Create namespace
kubectl create namespace argocd

# 2. Install ArgoCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 3. Wait for pods to be ready
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=argocd-server -n argocd --timeout=300s

# 4. Get admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo

# 5. Port forward (quick access)
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Access at: https://localhost:8080
# Username: admin
# Password: (output from step 4)
```

---

## 🔧 Installation Guide

### Step 1: Create Namespace

```bash
kubectl create namespace argocd
```

Or apply the manifest:
```bash
kubectl apply -f installation/01-namespace.yaml
```

### Step 2: Install ArgoCD

```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

This installs:
- ArgoCD API Server
- Repository Server
- Application Controller
- Redis
- Dex (SSO)
- Notifications Controller

### Step 3: Verify Installation

```bash
kubectl get pods -n argocd
```

Expected output:
```
NAME                                                READY   STATUS    RESTARTS   AGE
argocd-application-controller-0                     1/1     Running   0          2m
argocd-applicationset-controller-xxxxxxxxxx-xxxxx   1/1     Running   0          2m
argocd-dex-server-xxxxxxxxxx-xxxxx                  1/1     Running   0          2m
argocd-notifications-controller-xxxxxxxxxx-xxxxx    1/1     Running   0          2m
argocd-redis-xxxxxxxxxx-xxxxx                       1/1     Running   0          2m
argocd-repo-server-xxxxxxxxxx-xxxxx                 1/1     Running   0          2m
argocd-server-xxxxxxxxxx-xxxxx                      1/1     Running   0          2m
```

### Step 4: Get Admin Credentials

```bash
# Get password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo
```

**Default credentials:**
- Username: `admin`
- Password: (from command above)

⚠️ **Security Note:** Change the default password immediately after first login and delete the initial secret:
```bash
kubectl -n argocd delete secret argocd-initial-admin-secret
```

### Step 5: Expose ArgoCD UI

#### Option A: Port Forward (Development)

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```
Access: `https://localhost:8080`

#### Option B: Istio Gateway (Production)

Apply Gateway configuration:
```bash
kubectl apply -f installation/02-gateway.yaml
```

Apply VirtualService:
```bash
kubectl apply -f installation/03-virtualservice.yaml
```

**Important:** Update the domain name in both files before applying:
```yaml
hosts:
- "argocd.yourdomain.com"  # Change this
```

Verify:
```bash
kubectl get gateway -n argocd
kubectl get virtualservice -n argocd
```

Configure DNS to point to your Istio Ingress Gateway IP and access ArgoCD at: `https://argocd.yourdomain.com`

---

## 💻 Usage

### Accessing ArgoCD UI

Once ArgoCD is installed and exposed, access the UI:

```bash
# Get the Istio Ingress Gateway IP
kubectl get svc istio-ingressgateway -n istio-system

# Access via browser
https://argocd.yourdomain.com
```

**Login credentials:**
- Username: `admin`
- Password: Get from secret (see installation step 4)

### Connecting to Git Repository

#### Method 1: Via ArgoCD UI (Recommended)

1. **Login** to ArgoCD UI
2. Click **Settings** (gear icon) in the left sidebar
3. Select **Repositories**
4. Click **Connect Repo** button
5. Choose connection method: **VIA HTTPS**
6. Fill in the details:
   ```
   Repository URL: https://gitlab.com/username/my-k8s-apps.git
   Username: git
   Password: glpat-xxxxxxxxxxxxx (Personal Access Token)
   ```
7. Check **Skip server verification** (if needed for self-signed certs)
8. Click **Connect**
9. Wait for "Successful" connection status

**Creating Personal Access Token:**

For **GitLab**:
- Go to Settings → Access Tokens
- Name: `argocd-access`
- Scopes: ✅ `read_repository`
- Click "Create personal access token"
- Copy token (starts with `glpat-`)

For **GitHub**:
- Go to Settings → Developer Settings → Personal Access Tokens
- Generate new token (classic)
- Scopes: ✅ `repo` (Full control of private repositories)
- Generate and copy token (starts with `ghp_`)

#### Method 2: Via kubectl

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: my-git-repo
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
stringData:
  type: git
  url: https://gitlab.com/username/my-k8s-apps.git
  username: git
  password: glpat-your-personal-access-token
EOF
```

Verify connection:
```bash
kubectl get secrets -n argocd -l argocd.argoproj.io/secret-type=repository
```

### Creating Your First Application

#### Method 1: Via ArgoCD UI (Recommended for Beginners)

1. **Click Applications** in the left sidebar
2. Click **+ New App** button
3. Fill in the form:

**GENERAL:**
```
Application Name: my-first-app
Project: default
Sync Policy: Automatic
```

**Sync Options** (check these):
- ✅ **Prune Resources** - Delete resources removed from Git
- ✅ **Self Heal** - Revert manual changes automatically

**SOURCE:**
```
Repository URL: https://gitlab.com/username/my-k8s-apps.git
Revision: main (or master)
Path: . (if manifests are in root)
```

**DESTINATION:**
```
Cluster URL: https://kubernetes.default.svc
Namespace: default (or your target namespace)
```

**SYNC OPTIONS:**
- ✅ **Auto-Create Namespace** - Create namespace if it doesn't exist

4. Click **Create** at the top
5. Click **Sync** button → **Synchronize**

#### Method 2: Via kubectl

```bash
kubectl apply -f - <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-first-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://gitlab.com/username/my-k8s-apps.git
    targetRevision: main
    path: .
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
EOF
```

### Managing Applications

#### View Application Status

**Via UI:**
- Click on application card to see detailed view
- Check sync status, health status, and resource tree

**Via kubectl:**
```bash
# List all applications
kubectl get applications -n argocd

# Get detailed status
kubectl get application my-first-app -n argocd -o yaml

# Watch application sync
kubectl get application my-first-app -n argocd -w
```

#### Manual Sync 

**Via UI:**
- Click on application
- Click **Sync** button
- Review changes
- Click **Synchronize**

**Via kubectl:**
```bash
kubectl patch application my-first-app -n argocd \
  --type merge \
  -p '{"operation":{"sync":{}}}'
```

#### Refresh Application

Force ArgoCD to check Git repository for changes:

**Via UI:**
- Click on application
- Click **Refresh** button

**Via kubectl:**
```bash
kubectl patch application my-first-app -n argocd \
  --type merge \
  -p '{"metadata":{"annotations":{"argocd.argoproj.io/refresh":"hard"}}}'
```

#### Delete Application

**Via UI:**
- Click on application
- Click **Delete** button
- Choose whether to cascade delete (remove Kubernetes resources)
- Confirm deletion

**Via kubectl:**
```bash
# Delete application and all resources
kubectl delete application my-first-app -n argocd

# Delete application but keep resources
kubectl patch application my-first-app -n argocd \
  --type merge \
  -p '{"metadata":{"finalizers":null}}'
kubectl delete application my-first-app -n argocd
```

### Complete Example

For a full working example with step-by-step instructions, see the [`example-app/`](example-app/) directory:

- 📄 **nginx-app.yaml** - Complete nginx deployment with all resources
- 📚 **README.md** - Detailed deployment guide with:
  - Git repository setup
  - ArgoCD application creation
  - Testing auto-sync
  - Customization examples
  - Troubleshooting tips

**Quick start with example:**
```bash
# 1. Fork or clone this repository
# 2. Upload example-app/nginx-app.yaml to your Git repo
# 3. Connect your repo to ArgoCD (see above)
# 4. Create application pointing to your repo
# 5. Watch it deploy automatically!
```

---

## ✨ Key Features

### 🔄 GitOps Principles
- Git as single source of truth
- Declarative infrastructure
- Version-controlled deployments
- Audit trail via Git history

### 🤖 Automated Synchronization
- Automatic sync from Git repository
- Self-healing capabilities
- Automatic pruning of deleted resources
- Configurable sync policies

### 🔐 Security
- RBAC integration
- SSO support (via Dex)
- TLS encryption
- Secrets management

### 🌐 Multi-Cluster Support
- Manage multiple Kubernetes clusters
- Cluster credential management
- Cross-cluster deployments

### 📊 Visibility
- Real-time application status
- Resource health monitoring
- Sync status tracking
- Deployment history

---

## 📚 Learning Path

### Beginner
1. ✅ Install ArgoCD (this guide)
2. ✅ Deploy first application ([example-app](example-app/))
3. ✅ Understand sync policies
4. ✅ Test auto-sync and self-heal

### Intermediate
- Configure RBAC and SSO
- Multi-environment deployments (dev/staging/prod)
- Helm chart deployments
- Kustomize integration

### Advanced
- Multi-cluster management
- Progressive delivery with Argo Rollouts
- ApplicationSets for templating
- Webhook integrations

---

## 🎓 Best Practices

### Repository Organization
```
my-k8s-apps/
├── base/                 # Common configurations
├── overlays/
│   ├── dev/             # Development environment
│   ├── staging/         # Staging environment
│   └── production/      # Production environment
└── apps/                # Application definitions
```

### Security
- ✅ Use Personal Access Tokens, not passwords
- ✅ Implement RBAC policies
- ✅ Enable audit logging
- ✅ Regularly update ArgoCD
- ✅ Use encrypted secrets (Sealed Secrets, External Secrets)

### Sync Policies
- Use **automatic sync** for non-production environments
- Use **manual sync** for production (with approvals)
- Enable **prune** to keep cluster clean
- Enable **self-heal** to prevent configuration drift

### Application Design
- Keep manifests simple and readable
- Use Kustomize or Helm for environment-specific configs
- Implement health checks
- Define resource limits
- Use meaningful labels and annotations

---

## 🐛 Troubleshooting

### Common Issues

#### Application Stuck in "OutOfSync"
```bash
# Check application details
kubectl describe application <app-name> -n argocd

# Force sync
kubectl patch application <app-name> -n argocd --type merge -p '{"operation":{"sync":{}}}'
```

#### Cannot Access ArgoCD UI
```bash
# Check pods status
kubectl get pods -n argocd

# Check Gateway and VirtualService
kubectl get gateway,virtualservice -n argocd

# Check Istio ingress gateway
kubectl get svc istio-ingressgateway -n istio-system
```

#### Repository Connection Failed
- Verify credentials (token not expired)
- Check repository URL format
- Ensure network connectivity
- Verify firewall rules

#### Sync Failed
```bash
# View detailed error
kubectl get application <app-name> -n argocd -o yaml

# Check argocd-server logs
kubectl logs -n argocd deployment/argocd-server

# Check application controller logs
kubectl logs -n argocd statefulset/argocd-application-controller
```

### Debug Commands

```bash
# Get all applications
kubectl get applications -n argocd

# Describe application
kubectl describe application <app-name> -n argocd

# Get application status
kubectl get application <app-name> -n argocd -o jsonpath='{.status.sync.status}'

# Force refresh
kubectl patch application <app-name> -n argocd --type merge -p '{"metadata":{"annotations":{"argocd.argoproj.io/refresh":"hard"}}}'

# View sync history
kubectl get application <app-name> -n argocd -o jsonpath='{.status.history}'
```

---

## 📖 Resources

### Official Documentation
- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [ArgoCD GitHub](https://github.com/argoproj/argo-cd)
- [ArgoCD Best Practices](https://argo-cd.readthedocs.io/en/stable/user-guide/best_practices/)

### GitOps
- [GitOps Principles](https://opengitops.dev/)
- [CNCF GitOps Working Group](https://github.com/cncf/tag-app-delivery)

### Related Tools
- [Istio Documentation](https://istio.io/latest/docs/)
- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [Helm Documentation](https://helm.sh/docs/)
- [Kustomize Documentation](https://kustomize.io/)

### Tutorials & Guides
- [ArgoCD Tutorial](https://argo-cd.readthedocs.io/en/stable/getting_started/)
- [GitOps with ArgoCD](https://codefresh.io/learn/argo-cd/)

---

## 🤝 Contributing

Contributions are welcome! If you have improvements, examples, or fixes:

1. Fork the repository
2. Create a feature branch
3. Commit your changes
4. Open a Pull Request

Please ensure your contributions follow best practices and include documentation.

---

## 📝 License

This project is licensed under the MIT License - free to use for learning and production environments.

---

## 🌟 Acknowledgments

Built with experience from real-world Kubernetes deployments. Special thanks to the ArgoCD and CNCF communities for their excellent tools and documentation.

---

## 📬 Support

If you find this handbook helpful:
- ⭐ Star this repository
- 🐛 Report issues
- 💡 Suggest improvements
- 📢 Share with others

---

**Happy GitOps-ing! 🚀**
