# EKS + ArgoCD + Terraform Practice Project

A self-contained GitOps practice project: Terraform provisions an EKS cluster and
installs ArgoCD, then ArgoCD deploys a simple Flask app from this same Git repo.

```
terraform/   -> VPC, EKS cluster, ArgoCD (installed via Helm)
app/         -> Sample Flask app + Dockerfile
k8s/         -> Plain Kubernetes manifests ArgoCD will sync
argocd/      -> ArgoCD Application CRD (the GitOps entrypoint)
```

## Prerequisites

- AWS account (fresh account is fine) + IAM user/role with admin access for practice
- AWS CLI configured: `aws configure`
- Terraform >= 1.5, `kubectl`, `helm`, `docker`, a Docker Hub (or ECR) account
- A GitHub repo to push this project to

## Step 1 — Push this project to GitHub

```bash
cd eks-argocd-project
git init
git add .
git commit -m "initial commit: eks + argocd practice project"
git branch -M main
git remote add origin https://github.com/YOUR_USERNAME/eks-argocd-practice.git
git push -u origin main
```

## Step 2 — Provision the cluster + ArgoCD with Terraform

```bash
cd terraform
terraform init
terraform plan
terraform apply
```

This creates: a VPC (2 AZs, public+private subnets, 1 NAT gateway), an EKS cluster,
one managed node group (2x t3.medium), and installs ArgoCD into the `argocd`
namespace via the official Helm chart.

Takes ~15-20 minutes (EKS control plane is the slow part).

## Step 3 — Connect kubectl

```bash
aws eks update-kubeconfig --region us-east-1 --name eks-argocd-practice
kubectl get nodes
```

## Step 4 — Log into ArgoCD

```bash
# Get the admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d

# Get the ArgoCD UI URL (LoadBalancer hostname)
kubectl -n argocd get svc argocd-server -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
```

Open that hostname in a browser, login as `admin` with the password above.

## Step 5 — Build and push the app image

```bash
cd app
docker build -t YOUR_DOCKERHUB_USERNAME/eks-argocd-demo-app:v1 .
docker push YOUR_DOCKERHUB_USERNAME/eks-argocd-demo-app:v1
```

Update the `image:` field in `k8s/deployment.yaml` to match, then commit + push
that change to GitHub.

## Step 6 — Point ArgoCD at your repo

Edit `argocd/application.yaml`: set `repoURL` to your GitHub repo URL. Then apply it:

```bash
kubectl apply -f argocd/application.yaml
```

ArgoCD will now pull the `k8s/` folder from your repo and deploy it automatically.
Check sync status:

```bash
kubectl get applications -n argocd
kubectl get all -n demo-app
```

## Step 7 — Try the GitOps loop (the actual point of this project)

1. Change `APP_VERSION` env value in `k8s/deployment.yaml` (or the image tag)
2. Commit + push to GitHub
3. Watch ArgoCD auto-sync the change within ~3 minutes (or hit "Sync" in the UI)
4. `selfHeal: true` means if you manually `kubectl edit` something in the cluster,
   ArgoCD reverts it back to match Git — try it and watch it happen

## Step 8 — Tear down (avoid AWS charges)

```bash
kubectl delete -f argocd/application.yaml
kubectl delete namespace demo-app
cd terraform
terraform destroy
```

## Talking points for interviews

- Why separate `k8s/` manifests from `terraform/`: infra (cluster) vs. app
  deployment (workloads) have different lifecycles and often different owners
- `selfHeal` + `prune` = drift correction, a core GitOps concept
- Node group sizing, single NAT gateway = deliberate cost control for non-prod
- Could extend with: Kustomize overlays per environment, ArgoCD ApplicationSet
  for multi-cluster, sealed-secrets for secret management, an ALB Ingress
  Controller instead of a LoadBalancer Service per app
