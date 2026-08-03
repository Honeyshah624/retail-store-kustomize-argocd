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

## Notes

- Do not install AWS Load Balancer Controller separately for this Auto Mode setup.
- Do not expose catalog, carts, orders, or checkout publicly.
- The in-memory configuration is suitable for learning/testing, not production.
- Pod restarts can erase cart, checkout, and order state.
