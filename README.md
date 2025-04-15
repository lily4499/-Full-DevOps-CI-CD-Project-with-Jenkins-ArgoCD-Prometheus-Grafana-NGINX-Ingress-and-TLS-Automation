# -Full-DevOps-CI-CD-Project-with-Jenkins-ArgoCD-Prometheus-Grafana-NGINX-Ingress-and-TLS-Automation
![image](https://github.com/user-attachments/assets/0be54098-2926-43e3-8ea0-518e9c4dfb27)

---

```markdown
# Full DevOps CI/CD Project with Jenkins, ArgoCD, Prometheus, Grafana, NGINX Ingress, and TLS Automation

This guide provides a comprehensive walkthrough of setting up a full CI/CD pipeline using Jenkins, ArgoCD, Kubernetes (EKS), monitoring with Prometheus and Grafana, secure traffic with Ingress + Cert-Manager + Let's Encrypt, and GitHub Webhooks.

---

## 1. Jenkins Workspace Setup (Initial Configuration)

```bash
# Create Jenkins workspace for students (can be a folder or multibranch pipeline setup)
# Create and store credentials in Jenkins:
```

- DockerHub Credentials (username/password or token)
- GitHub Credentials (personal access token or username/password)
- Add these to Jenkins > Manage Jenkins > Credentials

---

## 2. Application and Manifest Repositories

- Fork the following:
  - Application source code repository
  - Kubernetes manifests repository (used by ArgoCD)
- Walk through the folder structure and make necessary configuration updates:
  - Update image names
  - Set correct namespaces
  - Add string parameters for automation

---

## 3. CI Phase: Jenkins Pipeline Jobs

### First Jenkins Job – CI Build

- Create a pipeline that builds your application
- Pushes the image to DockerHub
- Triggers the manifest update job

### Second Jenkins Job – Manifest Update

- This job watches the manifests repo
- Accepts string parameters for image tag or version
- Updates Kubernetes YAML files (e.g., Deployment)

---

## 4. Setup Kubernetes Cluster (EKS or Other)

```bash
aws eks --region <AWS_REGION> update-kubeconfig --name <EKS_CLUSTER_NAME>
kubectl get nodes
```

---

## 5. ArgoCD Installation and Setup

### Step 1: Install ArgoCD

```bash
brew install argocd  # or use binaries
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
watch kubectl get pods -n argocd
```

### Step 2: Access ArgoCD UI

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

- Access UI at: `http://localhost:8080`
- Default username: `admin`
- Retrieve password:
  ```bash
  kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
  ```

### Step 3: ArgoCD CLI Access

```bash
argocd login localhost:8080
argocd account update-password
```

---

## 6. Prometheus Monitoring Setup

```bash
kubectl create ns prometheus
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install my-prometheus prometheus-community/prometheus --namespace prometheus
kubectl --namespace prometheus port-forward deploy/my-prometheus-server 9090
```

---

## 7. Grafana Dashboard Setup

```bash
kubectl create ns grafana
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
helm install my-grafana grafana/grafana --namespace grafana
kubectl --namespace grafana get secret my-grafana -o jsonpath="{.data.admin-password}" | base64 --decode; echo
kubectl --namespace grafana port-forward svc/my-grafana 3000:80
```

---

## 8. Install Ingress Controller

```bash
kubectl create ns ingress-nginx
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install my-nginx ingress-nginx/ingress-nginx -n ingress-nginx
kubectl get all -n ingress-nginx
```

---

## 9. DNS Configuration

- Go to your domain registrar (e.g., Namecheap)
- Point the domain’s nameservers to your cloud provider (e.g., AWS)
- Use `kubectl get svc -n ingress-nginx` to find external IP for DNS setup

---

## 10. Deploy Applications and Create Ingress Rules

```bash
kubectl create ns dev
kubectl apply -f deployment.yaml  # or use ArgoCD to sync
```

### Example Ingress Manifest

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-prod
  annotations:
    cert-manager.io/issuer: letsencrypt-nginx
spec:
  tls:
  - hosts:
    - cicd.lilianedevops.online
    secretName: letsencrypt-nginx
  rules:
    - host: cicd.lilianedevops.online
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: amazon-service
                port:
                  number: 3000
  ingressClassName: nginx
```

```bash
kubectl apply -f ingress.yaml
kubectl get ingress -n dev
```

---

## 11. TLS with Cert-Manager and Let’s Encrypt

### Step 1: Install Cert-Manager

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.11.1/cert-manager.crds.yaml
helm repo add jetstack https://charts.jetstack.io
helm repo update
kubectl create ns cert-manager
helm install my-tls-cert jetstack/cert-manager --namespace cert-manager --version v1.11.1
```

### Step 2: Create Issuer

```yaml
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: letsencrypt-nginx
spec:
  acme:
    email: you@example.com
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-nginx-private-key
    solvers:
      - http01:
          ingress:
            class: nginx
```

```bash
kubectl apply -f issuer.yaml
kubectl get issuer
kubectl get secrets
```

---

## 12. Update Ingress Resources with TLS

Update your ingress manifest to include the following:

```yaml
annotations:
  cert-manager.io/issuer: letsencrypt-nginx

tls:
  - hosts:
    - cicd.lilianedevops.online
    secretName: letsencrypt-nginx
```

Then re-apply your ingress:

```bash
kubectl apply -f ingress.yaml
```

---

## 13. GitHub Webhook Automation (Optional)

1. Go to GitHub repo > Settings > Webhooks
2. Add a webhook with your Jenkins URL (`http://your-jenkins-url/github-webhook/`)
3. Push a change (bug fix, new feature)
4. Watch the Jenkins job trigger automatically, build the image, update manifests, and deploy via ArgoCD

---

## ✅ Summary

You now have a fully functioning CI/CD pipeline:

- Jenkins builds and updates manifests
- ArgoCD syncs deployments
- Prometheus and Grafana monitor your system
- TLS is issued automatically via Let's Encrypt
- Ingress exposes your app to the internet securely

---
```
