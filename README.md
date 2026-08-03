# Retail Store Deployment (Quick Commands)



# Architecture

```
                    GitHub Repository
                           |
                           |
                     ArgoCD Watches
                           |
                           |
                  +----------------+
                  |    ArgoCD      |
                  +----------------+
                           |
                    Kubernetes API
                           |
                  Amazon EKS Auto Mode
                           |
      -----------------------------------------
      |        |        |       |             |
   Catalog   Orders   Carts  Checkout       UI
      |        |        |       |             |
      -----------------------------------------
                           |
                     AWS Network Load Balancer
                           |
                       Internet Users
```

## 1. Configure kubectl

```bash
aws eks update-kubeconfig \
--region ap-south-1 \
--name demo-app

kubectl get nodes
```

---

# 2. Install ArgoCD

```bash
kubectl create namespace argocd

kubectl apply \
--server-side \
--force-conflicts \
-n argocd \
-f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

---

# 3. Wait for ArgoCD

```bash
kubectl wait \
--for=condition=Available \
deployment/argocd-server \
-n argocd \
--timeout=600s
```

---

# 4. If ImagePullBackOff (Private Cluster)

## Login to ECR

```bash
AWS_ACCOUNT_ID=<ACCOUNT_ID>

AWS_REGION=ap-south-1

ECR_REGISTRY="${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"

aws ecr get-login-password \
--region "$AWS_REGION" \
| docker login \
--username AWS \
--password-stdin "$ECR_REGISTRY"
```

---

## Pull Images

```bash
docker pull quay.io/argoproj/argocd:v3.4.6

docker pull ghcr.io/dexidp/dex:v2.45.0

docker pull public.ecr.aws/docker/library/redis:8.2.3-alpine
```

---

## Tag Images

```bash
docker tag quay.io/argoproj/argocd:v3.4.6 \
$ECR_REGISTRY/platform/argocd:v3.4.6

docker tag ghcr.io/dexidp/dex:v2.45.0 \
$ECR_REGISTRY/platform/dex:v2.45.0

docker tag public.ecr.aws/docker/library/redis:8.2.3-alpine \
$ECR_REGISTRY/platform/redis:8.2.3-alpine
```

---

## Push Images

```bash
docker push $ECR_REGISTRY/platform/argocd:v3.4.6

docker push $ECR_REGISTRY/platform/dex:v2.45.0

docker push $ECR_REGISTRY/platform/redis:8.2.3-alpine
```

---

# 5. Create Required VPC Endpoints

Create

- ECR API
- ECR DKR
- S3 Gateway

Verify

```bash
aws ec2 describe-vpc-endpoints \
--region ap-south-1
```

---

# 6. Verify ArgoCD

```bash
kubectl get pods -n argocd
```

Expected

```
Running
```

---

# 7. Clone Repository

```bash
git clone https://github.com/<username>/retail-store-kustomize-argocd.git

cd retail-store-kustomize-argocd
```

---

# 8. Apply ArgoCD Project

```bash
kubectl apply -k argocd
```

---

# 9. Verify Applications

```bash
kubectl get applications -n argocd
```

---

# 10. Force Refresh (Optional)

```bash
for app in retail-catalog retail-carts retail-orders retail-checkout retail-ui
do
kubectl annotate application $app \
-n argocd \
argocd.argoproj.io/refresh=hard \
--overwrite
done
```

---

# 11. Verify Sync

```bash
kubectl get applications -n argocd
```

Expected

```
Synced

Healthy
```

---

# 12. Verify Pods

```bash
kubectl get pods -n retail-store
```

---

# 13. Wait for Deployments

```bash
for service in catalog carts orders checkout ui
do
kubectl rollout status deployment/$service \
-n retail-store \
--timeout=300s
done
```

---

# 14. Verify UI Service

```bash
kubectl get svc ui -n retail-store
```

---

# 15. Get LoadBalancer URL

```bash
kubectl get svc ui \
-n retail-store \
-o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
```

---

# 16. Verify Target Health

```bash
LB_DNS=$(kubectl get svc ui \
-n retail-store \
-o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

LB_ARN=$(aws elbv2 describe-load-balancers \
--region ap-south-1 \
--query "LoadBalancers[?DNSName=='$LB_DNS'].LoadBalancerArn" \
--output text)

TG_ARN=$(aws elbv2 describe-target-groups \
--region ap-south-1 \
--load-balancer-arn "$LB_ARN" \
--query 'TargetGroups[0].TargetGroupArn' \
--output text)

aws elbv2 describe-target-health \
--region ap-south-1 \
--target-group-arn "$TG_ARN"
```

Expected

```
healthy
```

---

# 17. Open Application

```text
http://<LoadBalancerDNS>
```

---

# 18. Access ArgoCD UI

```bash
kubectl port-forward \
svc/argocd-server \
-n argocd \
8080:443
```

Open

```
https://localhost:8080
```

---

# 19. Get ArgoCD Password

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
-o jsonpath="{.data.password}" \
| base64 -d
```

Username

```
admin
```

---

# 20. Useful Commands

## Applications

```bash
kubectl get applications -n argocd
```

## Pods

```bash
kubectl get pods -A
```

## Services

```bash
kubectl get svc -A
```

## Deployments

```bash
kubectl get deployments -A
```

## Describe Application

```bash
kubectl describe application retail-ui -n argocd
```

## Sync Application

```bash
kubectl annotate application retail-ui \
-n argocd \
argocd.argoproj.io/refresh=hard \
--overwrite
```

## Logs

```bash
kubectl logs deployment/argocd-server -n argocd

kubectl logs deployment/argocd-repo-server -n argocd
```

## Delete Applications

```bash
kubectl delete -k argocd
```

## Delete Namespace

```bash
kubectl delete namespace retail-store
```

# Retail Store GitOps — EKS Auto Mode

GitHub repository expected by the Argo CD manifests:

`https://github.com/Honeyshah624/retail-store-gitops.git`

This repository deploys these independent Argo CD Applications:

- retail-catalog
- retail-carts
- retail-orders
- retail-checkout
- retail-ui

The development configuration uses in-memory backends. Only the UI is exposed
publicly. Its Kustomize overlay changes the Service to an EKS Auto Mode NLB.

## 1. Push this folder to GitHub

Create this repository in GitHub:

`Honeyshah624/retail-store-gitops`

Then run:

```bash
cd retail-store-gitops
git init
git branch -M main
git add .
git commit -m "Add Retail Store Argo CD Kustomize deployment"
git remote add origin https://github.com/Honeyshah624/retail-store-gitops.git
git push -u origin main
```

## 2. Connect to EKS

```bash
aws eks update-kubeconfig --region ap-south-1 --name demo-app
kubectl get nodes
```

No manually created managed node group is required when EKS Auto Mode is used.

## 3. Install Argo CD

```bash
kubectl create namespace argocd

kubectl apply --server-side --force-conflicts -n argocd   -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

kubectl wait --for=condition=Available deployment/argocd-server   -n argocd --timeout=600s
```

For production, pin a tested Argo CD release rather than permanently using the
moving `stable` manifest.

## 4. Validate the Kustomize overlays

```bash
for service in catalog carts orders checkout ui; do
  kubectl kustomize "overlays/dev/${service}" > "/tmp/${service}.yaml"
done
```

Optional API-server dry run:

```bash
for service in catalog carts orders checkout ui; do
  kubectl apply --dry-run=server -k "overlays/dev/${service}"
done
```

## 5. Create the Argo CD Applications

```bash
kubectl apply -k argocd
kubectl get applications -n argocd
```

## 6. Watch EKS Auto Mode provision capacity

```bash
kubectl get nodes -w
```

In another terminal:

```bash
kubectl get pods -n retail-store -w
```

## 7. Verify all workloads

```bash
kubectl get deploy,pods,svc -n retail-store

for service in catalog carts orders checkout ui; do
  kubectl rollout status "deployment/${service}"     -n retail-store --timeout=600s
done
```

## 8. Open the UI

```bash
kubectl get service ui -n retail-store -w
```

After the NLB hostname appears:

```bash
UI_HOST=$(kubectl get service ui -n retail-store   -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

echo "http://${UI_HOST}"
```

Port-forward fallback:

```bash
kubectl port-forward -n retail-store service/ui 8080:80
```

Then open `http://localhost:8080`.


Retail Store GitOps Deployment Summary

**Objective:**

Deploy a Retail Store microservices application on an **Amazon EKS Auto Mode** cluster using **ArgoCD** and  **Kustomize** , following the GitOps approach.

### Work Completed

1. **Configured the EKS Cluster**

* Connected to the existing EKS Auto Mode cluster.
* Verified node and cluster health.

1. **Installed ArgoCD**

* Installed ArgoCD in the `argocd` namespace.
* Verified all ArgoCD components were running.

1. **Resolved Private Cluster Image Pull Issues**

* Since the cluster had restricted internet access, ArgoCD images could not be pulled from public registries.
* Mirrored the required images (ArgoCD, Dex, Redis) into Amazon ECR.
* Created and verified ECR API, ECR DKR, and S3 VPC Endpoints.
* Diagnosed networking issues and found the node route table had a broken (blackhole) NAT route.
* Created the required S3 Gateway Endpoint in the correct VPC and verified image pulling worked successfully.

1. **Prepared the GitOps Repository**

* Organized the repository into:
  *  `base/`
  *  `overlays/dev/`
  *  `argocd/`
* Configured Kustomize manifests for all services.

1. **Configured ArgoCD**

* Created an `AppProject`.
* Created separate ArgoCD Applications for:
  *  Catalog
  *  Carts
  *  Orders
  *  Checkout
  *  UI
* Updated repository references to the new GitHub repository.

1. **Deployed the Application**

* Applied the ArgoCD manifests.
* Verified all applications reached:
  *  **Synced**
  *  **Healthy**

1. **Configured the Load Balancer**

* Exposed the UI using an AWS Network Load Balancer (NLB).
* Configured health checks and internet-facing access.
* Fixed subnet configuration so the NLB covered all required Availability Zones.
* Verified the target group became healthy and the application was accessible through the Load Balancer DNS.

1. **Verified GitOps Workflow**

* Confirmed ArgoCD continuously monitors the GitHub repository.
* Any future Git commit will automatically synchronize changes to the EKS cluster.

---

## Final Result

* ✅ ArgoCD installed successfully.
* ✅ Retail Store microservices deployed on EKS.
* ✅ All ArgoCD applications are **Synced** and  **Healthy** .
* ✅ Application accessible through the AWS Network Load Balancer.
* ✅ GitOps pipeline established using  **GitHub + ArgoCD + Kustomize** .

## Notes

- Do not install AWS Load Balancer Controller separately for this Auto Mode setup.
- Do not expose catalog, carts, orders, or checkout publicly.
- The in-memory configuration is suitable for learning/testing, not production.
- Pod restarts can erase cart, checkout, and order state.
