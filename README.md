# Deploy Strimzi-Kafka with Conduktor and Postgres


# Create Namespace
kubectl create namespace kafka

# DB Setup
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# Deploy PostgreSQL

# Reinstall PostgreSQL with explicit configuration
helm install postgres bitnami/postgresql -n kafka \
  --set auth.postgresPassword="conduktor123" \
  --set auth.username="conduktor" \
  --set auth.password="conduktor123" \
  --set auth.database="conduktor"

#                # Remove the existing PostgreSQL installation   (IF NOT NEW DEPLOYMENT)
                helm uninstall postgres -n kafka

#                # Wait for the pod to be deleted     (IF NOT NEW DEPLOYMENT)
                kubectl wait --for=delete pod/postgres-postgresql-0 -n kafka --timeout=60s

#                # Also delete the PVC if it exists (this will remove old data and passwords)   (IF NOT NEW DEPLOYMENT)
                kubectl delete pvc data-postgres-postgresql-0 -n kafka --ignore-not-found

#                # Reinstall PostgreSQL with explicit configuration
                helm install postgres bitnami/postgresql -n kafka \
                --set auth.postgresPassword="conduktor123" \
                --set auth.username="conduktor" \
                --set auth.password="conduktor123" \
                --set auth.database="conduktor"

#               wait for ready 
                kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=postgresql -n kafka --timeout=300s

#               test connection with default user
                kubectl run postgres-client --rm -it --restart=Never --namespace kafka --image postgres:13 --env="PGPASSWORD=conduktor123" -- psql -h postgres-postgresql -U postgres -d postgres -p 5432

#               test connection with created user
                kubectl run postgres-client --rm -it --restart=Never --namespace kafka --image postgres:13 --env="PGPASSWORD=conduktor123" -- psql -h postgres-postgresql -U conduktor -d conduktor -p 5432

# Setup Helm repository
helm repo add conduktor https://helm.conduktor.io
helm repo update

export ADMIN_EMAIL="admin@netsol.pk"
export ADMIN_PASSWORD="Netsolpk1@"  # Added @ symbol, Password must contain at least 8 characters.
export ORG_NAME="Netsol"
export NAMESPACE="kafka"

# Deploy Helm chart
helm install console conduktor/console \
  -n ${NAMESPACE} \
  --set config.organization.name="${ORG_NAME}" \
  --set config.admin.email="${ADMIN_EMAIL}" \
  --set config.admin.password="${ADMIN_PASSWORD}" \
  --set config.database.password="conduktor123" \
  --set config.database.username="conduktor" \
  --set config.database.host="postgres-postgresql" \
  --set config.database.port="5432" \
  --set config.database.name="conduktor" \
  --set resources.limits.cpu="4" \
  --set resources.requests.cpu="10000m"
    
# Port forward to access Conduktor
kubectl port-forward deployment/console -n ${NAMESPACE} 8080:8080
open http://localhost:8080


# Deploy Kafka


kubectl apply -f 'https://strimzi.io/install/latest?namespace=kafka' -n kafka

# Wait for the operator to be ready
kubectl wait --for=condition=ready pod -l name=strimzi-cluster-operator -n kafka --timeout=300s

# show operator pod(s)
kubectl get pods -n kafka

# verify Strimzi CRDs are installed
kubectl get crd | grep strimzi || true

# or list Strimzi custom resources (short names work)
kubectl get kafkas -n kafka

# create a single-node Kafka cluster (official example)
kubectl apply -f https://strimzi.io/examples/latest/kafka/kafka-single-node.yaml -n kafka

# wait until Strimzi reports the Kafka resource is Ready
kubectl wait kafka/my-cluster --for=condition=Ready --timeout=600s -n kafka

# check the pods and services
kubectl get pods -n kafka -l strimzi.io/cluster=my-cluster
kubectl get svc -n kafka | grep my-cluster

