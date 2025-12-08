prod: 8080
staging: 8180
argocd: 8980


./set-up.sh

istioctl install -f install-istio.yaml -y
curl -sI http://localhost:15021/healthz/ready // to check if working

helm install argocd ../g11-helm-charts/argo-cd/ -n argocd --create-namespace

kubectl get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" -n argocd | base64 -d

// Install Postgres Operator (ONCE per cluster, manages all namespaces)
helm install -n postgres-operator --create-namespace \
    pgo oci://registry.developers.crunchydata.com/crunchydata/pgo

// Setup Production namespace
kubectl create namespace g11-prod
kubectl label namespace g11-prod istio-injection=enabled

helm install hippo ../g11-helm-charts/postgres/ -n g11-prod

kubectl get pods -n g11-prod -l postgres-operator.crunchydata.com/pgbackrest-backup // to check if it is working

// Setup Staging namespace (reuses the same PGO operator)
kubectl create namespace g11-stg
kubectl label namespace g11-stg istio-injection=enabled

helm install hippo-stg ../g11-helm-charts/postgres/ -n g11-stg

kubectl get pods -n g11-stg -l postgres-operator.crunchydata.com/pgbackrest-backup // to check if it is working

// Then go into argoCD and create application make sure to enter secretes in the yaml file

https://github.com/G11-Engineering/g11-helm-charts

// enter the JWT secret and m2msecret

// change the port for staging