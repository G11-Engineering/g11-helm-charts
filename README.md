prod: 8080
staging: 8180
argocd: 8980


./set-up.sh

istioctl install -f install-istio.yaml
curl -sI http://localhost:15021/healthz/ready // to check if working

helm install argocd ../g11-helm-charts/argo-cd/ -n argocd --create-namespace

kubectl get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" -n argocd | base64 -d

kubectl create namespace g11-prod

helm install -n g11-prod --create-namespace \
    pgo oci://registry.developers.crunchydata.com/crunchydata/pgo

kubectl label namespace g11-prod istio-injection=enabled

helm install hippo ../g11-helm-charts/postgres/ -n g11-prod

kubectl get pods -n g11-prod -l postgres-operator.crunchydata.com/pgbackrest-backup // to check if it is working

// Then go into argoCD and create application