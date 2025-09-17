# Deploy Strimzi-Kafka with Conduktor and Postgres On k3s


# Create Namespace
sudo k3s kubectl create namespace kafka-conduktor

# DB Setup
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# Deploy PostgreSQL

# Reinstall PostgreSQL with explicit configuration
helm install postgres bitnami/postgresql -n kafka-conduktor \
  --set auth.postgresPassword="conduktor123" \
  --set auth.username="conduktor" \
  --set auth.password="conduktor123" \
  --set auth.database="conduktor"

#                # Remove the existing PostgreSQL installation   (IF NOT NEW DEPLOYMENT)
                helm uninstall postgres -n kafka-conduktor

#                # Wait for the pod to be deleted     (IF NOT NEW DEPLOYMENT)
                sudo k3s kubectl wait --for=delete pod/postgres-postgresql-0 -n kafka-conduktor --timeout=60s

#                # Also delete the PVC if it exists (this will remove old data and passwords)   (IF NOT NEW DEPLOYMENT)
                sudo k3s kubectl delete pvc data-postgres-postgresql-0 -n kafka-conduktor --ignore-not-found

#                # Reinstall PostgreSQL with explicit configuration
                helm install postgres bitnami/postgresql -n kafka-conduktor \
                --set auth.postgresPassword="conduktor123" \
                --set auth.username="conduktor" \
                --set auth.password="conduktor123" \
                --set auth.database="conduktor"

#               wait for ready 
                sudo k3s kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=postgresql -n kafka-conduktor --timeout=300s

#               test connection with default user
                sudo k3s kubectl run postgres-client --rm -it --restart=Never --namespace kafka-conduktor --image postgres:13 --env="PGPASSWORD=conduktor123" -- psql -h postgres-postgresql -U postgres -d postgres -p 5432

#               test connection with created user
                sudo k3s kubectl run postgres-client --rm -it --restart=Never --namespace kafka-conduktor --image postgres:13 --env="PGPASSWORD=conduktor123" -- psql -h postgres-postgresql -U conduktor -d conduktor -p 5432

# Setup Helm repository
helm repo add conduktor https://helm.conduktor.io
helm repo update

export ADMIN_EMAIL="admin@netsol.pk"
export ADMIN_PASSWORD="Netsolpk1@"  # Added @ symbol, Password must contain at least 8 characters.
export ORG_NAME="Netsol"
export NAMESPACE="kafka-conduktor"

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
    
# Port forward to access Conduktor : its only accessible on Localhost within the cluster.
sudo k3s kubectl port-forward deployment/console -n ${NAMESPACE} 8080:8080
open http://localhost:8080

# Port forward to access Conduktor outside the cluster.
sudo k3s kubectl port-forward deployment/console -n kafka-conduktor 8080:8080 --address='0.0.0.0'
open http://<cluserip>:8080

# Deploy Kafka


sudo k3s kubectl apply -f 'https://strimzi.io/install/latest?namespace=kafka-conduktor' -n kafka-conduktor

# Wait for the operator to be ready
sudo k3s kubectl wait --for=condition=ready pod -l name=strimzi-cluster-operator -n kafka-conduktor --timeout=300s

# show operator pod(s)
sudo k3s kubectl get pods -n kafka-conduktor

# verify Strimzi CRDs are installed
sudo k3s kubectl get crd | grep strimzi || true

# or list Strimzi custom resources (short names work)
sudo k3s kubectl get kafka -n kafka-conduktor

# create a single-node Kafka cluster (official example)
sudo k3s kubectl apply -f https://strimzi.io/examples/latest/kafka/kafka-single-node.yaml -n kafka-conduktor

# wait until Strimzi reports the Kafka resource is Ready
sudo k3s kubectl wait kafka/my-cluster --for=condition=Ready --timeout=600s -n kafka-conduktor   

# check the pods and services
sudo k3s kubectl get pods -n kafka-conduktor -l strimzi.io/cluster=my-cluster
sudo k3s kubectl get svc -n kafka-conduktor | grep my-cluster

