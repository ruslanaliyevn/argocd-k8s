# Example Application - Nginx Deployment

This is a complete example showing how to deploy an application using ArgoCD and GitOps workflow.

## 📋 What's Included

- **Namespace:** `nginx-test`
- **Deployment:** 2 replicas of nginx
- **Service:** ClusterIP service
- **ConfigMap:** Custom HTML page
- **Istio Gateway:** External access configuration

## 🚀 Step-by-Step Deployment

### Step 1: Create Git Repository

1. Create a new repository on GitLab or GitHub
   - Example name: `my-k8s-apps`
   - Visibility: Public or Private

2. Clone the repository:
   ```bash
   git clone https://gitlab.com/username/my-k8s-apps.git
   cd my-k8s-apps
   ```

3. Copy `nginx-app.yaml` to the repository:
   ```bash
   # Copy the nginx-app.yaml file to repository root
   git add nginx-app.yaml
   git commit -m "Add nginx application"
   git push origin main
   ```

### Step 2: Connect Repository to ArgoCD

#### Option A: Via ArgoCD UI (Recommended)

1. Login to ArgoCD UI (`https://argocd.yourdomain.com`)
2. Go to **Settings** (gear icon) → **Repositories**
3. Click **Connect Repo** button
4. Select connection method: **VIA HTTPS**
5. Fill in the form:
   ```
   Repository URL: https://gitlab.com/username/my-k8s-apps.git
   Username: git
   Password: glpat-your-token-here
   ```
6. Check **Skip server verification** (if using self-signed certificate)
7. Click **Connect**
8. Wait for "Successful" status

#### Option B: Via kubectl

Create a secret with your repository credentials:

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: my-k8s-apps-repo
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
stringData:
  type: git
  url: https://gitlab.com/username/my-k8s-apps.git
  username: git
  password: glpat-your-token-here
EOF
```

**How to get GitLab Personal Access Token:**
1. Go to GitLab → User Settings → Access Tokens
2. Token name: `argocd-access`
3. Select scopes: `read_repository`
4. Click "Create personal access token"
5. Copy the token (starts with `glpat-`)

### Step 3: Create Application in ArgoCD

#### Option A: Via ArgoCD UI (Recommended)

1. Go to **Applications** → Click **+ New App**

2. Fill in **GENERAL** section:
   ```
   Application Name: nginx-test
   Project: default
   Sync Policy: Automatic
   ```

3. Check these options:
   - ✅ **Prune Resources** - Delete resources not in Git
   - ✅ **Self Heal** - Revert manual changes

4. Fill in **SOURCE** section:
   ```
   Repository URL: https://gitlab.com/username/my-k8s-apps.git
   Revision: main (or master)
   Path: . (dot means root directory)
   ```

5. Fill in **DESTINATION** section:
   ```
   Cluster URL: https://kubernetes.default.svc
   Namespace: nginx-test
   ```

6. In **SYNC OPTIONS** section:
   - ✅ **Auto-Create Namespace**

7. Click **Create** at the top

8. Your application is created! Now click **Sync** → **Synchronize**

#### Option B: Via kubectl

```bash
kubectl apply -f - <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: nginx-test
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://gitlab.com/username/my-k8s-apps.git
    targetRevision: main
    path: .
  destination:
    server: https://kubernetes.default.svc
    namespace: nginx-test
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
EOF
```

### Step 4: Verify Deployment

Check application status in ArgoCD UI or via kubectl:

```bash
# Check application
kubectl get application nginx-test -n argocd

# Check pods
kubectl get pods -n nginx-test

# Expected output:
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-xxxxxxxxxx-xxxxx   1/1     Running   0          2m
nginx-deployment-xxxxxxxxxx-xxxxx   1/1     Running   0          2m

# Check service
kubectl get svc -n nginx-test

# Expected output:
NAME            TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
nginx-service   ClusterIP   10.233.x.x      <none>        80/TCP    2m
```

### Step 5: Access the Application

#### Option A: Port Forward (Quick Test)

```bash
kubectl port-forward -n nginx-test svc/nginx-service 8080:80
```

Open browser: `http://localhost:8080`

You should see the custom HTML page!

#### Option B: Via Istio Gateway (Production)

Update the domain in `nginx-app.yaml`:
```yaml
hosts:
- "nginx.yourdomain.com"
```

Commit and push the change. ArgoCD will automatically sync!

Then access via: `http://nginx.yourdomain.com`

---

## 🔄 Testing Auto-Sync

Let's test GitOps workflow by making a change:

1. Edit `nginx-app.yaml` in your Git repository
2. Change the replica count:
   ```yaml
   spec:
     replicas: 3  # Changed from 2 to 3
   ```

3. Commit and push:
   ```bash
   git add nginx-app.yaml
   git commit -m "Scale nginx to 3 replicas"
   git push origin main
   ```

4. Watch ArgoCD automatically sync (within ~3 minutes):
   ```bash
   kubectl get pods -n nginx-test -w
   ```

You should see a third pod being created automatically!

5. Check in ArgoCD UI:
   - Application status should show "Synced"
   - You'll see the sync history with your commit message

---

## 🎨 Customize the Application

### Change HTML Content

Edit the ConfigMap section in `nginx-app.yaml`:
```yaml
data:
  index.html: |
    <!DOCTYPE html>
    <html>
    <head>
        <title>My Custom Page</title>
    </head>
    <body>
        <h1>Hello from ArgoCD!</h1>
    </body>
    </html>
```

Commit, push, and watch it update automatically!

### Change Nginx Version

Update the image version:
```yaml
containers:
- name: nginx
  image: nginx:1.26-alpine  # Changed from 1.25
```

### Add Environment Variables

```yaml
containers:
- name: nginx
  env:
  - name: MY_VAR
    value: "my-value"
```

---

## 🧹 Cleanup

To delete the application and all its resources:

#### Via ArgoCD UI:
1. Go to Applications
2. Click on `nginx-test`
3. Click **Delete** button
4. Confirm deletion

#### Via kubectl:
```bash
kubectl delete application nginx-test -n argocd
```

This will automatically remove all deployed resources (pods, service, configmap) because prune is enabled.

---

## 🐛 Troubleshooting

### Application stuck in "Progressing"
```bash
# Check application details
kubectl describe application nginx-test -n argocd

# Check ArgoCD logs
kubectl logs -n argocd deployment/argocd-server
```

### Sync fails with "path does not exist"
- Check that `nginx-app.yaml` exists in repository root
- Verify the path in Application configuration (should be `.` for root)
- Check repository connection in Settings → Repositories

### Pods not starting
```bash
# Check pod events
kubectl describe pod <pod-name> -n nginx-test

# Check pod logs
kubectl logs <pod-name> -n nginx-test
```

### Cannot access via domain
- Verify Gateway and VirtualService are created: `kubectl get gateway,virtualservice -n nginx-test`
- Check Istio ingress gateway IP: `kubectl get svc istio-ingressgateway -n istio-system`
- Verify DNS points to correct IP

---

## 📝 Key Concepts Demonstrated

### GitOps Workflow
Every change goes through Git → ArgoCD detects change → Kubernetes is updated

### Declarative Configuration
Everything is defined in YAML files, stored in Git

### Auto-Sync
Changes in Git automatically deployed to Kubernetes (within 3 minutes by default)

### Self-Heal
If someone manually changes a resource, ArgoCD reverts it to match Git

### Prune
If you delete a resource from Git, ArgoCD deletes it from Kubernetes

---

## 🎯 Next Steps

Try these exercises to learn more:

1. **Add a second service** - Create another deployment in the same namespace
2. **Use Helm charts** - Deploy a Helm-based application
3. **Multi-environment** - Create dev, staging, prod configurations
4. **Secrets management** - Use Sealed Secrets or External Secrets Operator
5. **Progressive delivery** - Implement canary deployments with Argo Rollouts

---

## 🔗 Related Documentation

- [ArgoCD Application Specs](https://argo-cd.readthedocs.io/en/stable/user-guide/application-specification/)
- [Sync Policies](https://argo-cd.readthedocs.io/en/stable/user-guide/sync-options/)
- [Health Assessment](https://argo-cd.readthedocs.io/en/stable/operator-manual/health/)
