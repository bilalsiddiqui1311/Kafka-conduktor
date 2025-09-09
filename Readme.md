**Let's do a complete clean restart:**

```bash
# 1. Clean everything completely
kubectl delete namespace kafka-demo
kubectl delete clusterrole strimzi-cluster-operator-namespaced strimzi-cluster-operator-global strimzi-cluster-operator-watched strimzi-cluster-operator-leader-election strimzi-entity-operator strimzi-kafka-broker strimzi-kafka-client --ignore-not-found=true
kubectl delete clusterrolebinding strimzi-cluster-operator strimzi-cluster-operator-kafka-broker-delegation strimzi-cluster-operator-kafka-client-delegation --ignore-not-found=true
kubectl delete crd kafkas.kafka.strimzi.io kafkatopics.kafka.strimzi.io kafkausers.kafka.strimzi.io kafkaconnects.kafka.strimzi.io kafkaconnectors.kafka.strimzi.io kafkamirrormakers.kafka.strimzi.io kafkamirrormaker2s.kafka.strimzi.io kafkabridges.kafka.strimzi.io kafkarebalances.kafka.strimzi.io kafkanodepools.kafka.strimzi.io strimzipodsets.core.strimzi.io --ignore-not-found=true

# Wait for cleanup
sleep 10

# 2. Create fresh namespace
kubectl create namespace kafka-demo
```

**Then install step by step:**

```bash
# 3. Install only basic components first (Keycloak + secrets)
kubectl apply -f - << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: keycloak
  namespace: kafka-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: keycloak
  template:
    metadata:
      labels:
        app: keycloak
    spec:
      containers:
      - name: keycloak
        image: quay.io/keycloak/keycloak:22.0.1
        args: ["start-dev"]
        env:
        - name: KEYCLOAK_ADMIN
          value: admin
        - name: KEYCLOAK_ADMIN_PASSWORD
          value: admin123
        - name: KC_HOSTNAME_STRICT
          value: "false"
        - name: KC_HOSTNAME_STRICT_HTTPS
          value: "false"
        ports:
        - containerPort: 8080
        readinessProbe:
          httpGet:
            path: /realms/master
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
---
apiVersion: v1
kind: Service
metadata:
  name: keycloak-service
  namespace: kafka-demo
spec:
  selector:
    app: keycloak
  ports:
  - port: 8080
    targetPort: 8080
    nodePort: 30080
  type: NodePort
---
apiVersion: v1
kind: Secret
metadata:
  name: kafka-oauth-secret
  namespace: kafka-demo
type: Opaque
stringData:
  clientSecret: "kafka-secret-123"
EOF

# 4. Wait for Keycloak to be ready
kubectl wait --for=condition=available --timeout=300s deployment/keycloak -n kafka-demo
```

**Then install Strimzi:**

```bash
# 5. Install Strimzi operator (latest stable)
kubectl create -f https://strimzi.io/install/latest?namespace=kafka-demo -n kafka-demo

# 6. Wait for operator to be ready
kubectl wait --for=condition=available --timeout=600s deployment/strimzi-cluster-operator -n kafka-demo
```

**Check if the operator is working:**

```bash
kubectl get pods -n kafka-demo
kubectl logs deployment/strimzi-cluster-operator -n kafka-demo --tail=50
```



Before this, it works fine. everything Up!!!


We need to configure Keycloak before deploying the Kafka cluster.


Step 1: Access Keycloak and Set Up the Realm
bash# Get Minikube IP
MINIKUBE_IP=$(minikube ip)
echo "Keycloak Admin Console: http://$MINIKUBE_IP:30080"
Manual Setup (Recommended for understanding):

Access Keycloak Admin Console:

Go to http://MINIKUBE_IP:30080
Login with: admin / admin123


Create the Kafka Realm:

Click "Create Realm"
Name: kafka
Click "Create"


Create Kafka Client:

Go to "Clients" → "Create Client"
Client ID: kafka
Client Type: OpenID Connect
Click "Next"
Enable "Client authentication"
Enable "Service accounts roles"
Click "Save"
Go to "Credentials" tab
Copy the "Client secret" (or set it to: kafka-secret-123)


Create Kafka UI Client:

Create another client with ID: kafka-ui
Same settings as above
Set client secret to: kafka-ui-secret-456
In "Valid redirect URIs" add: http://MINIKUBE_IP:30081/*


Create Test Users:

Go to "Users" → "Create new user"
Username: kafka-user
Set password: kafka123 (disable temporary)
Create another user: admin-user / admin123



# Deploy Kafka cluster (now Keycloak is ready)
kubectl apply -f kafka-resources.yaml

# Wait for Kafka to be ready
kubectl wait --for=condition=Ready --timeout=900s kafka/kafka-cluster -n kafka-demo



Please run these commands step by step and let me know at which point you encounter issues. Also, share the operator logs if it's still crashing.




kubectl apply -f - << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kafka-ui
  namespace: kafka-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kafka-ui
  template:
    metadata:
      labels:
        app: kafka-ui
    spec:
      containers:
      - name: kafka-ui
        image: provectuslabs/kafka-ui:latest
        ports:
        - containerPort: 8080
        env:
        - name: KAFKA_CLUSTERS_0_NAME
          value: kafka-cluster
        - name: KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS
          value: kafka-cluster-kafka-bootstrap:9092
        - name: AUTH_TYPE
          value: OAUTH2
        - name: AUTH_OAUTH2_CLIENT_KEYCLOAK_PROVIDER
          value: keycloak
        - name: AUTH_OAUTH2_CLIENT_KEYCLOAK_REALM
          value: kafka
        - name: AUTH_OAUTH2_CLIENT_KEYCLOAK_AUTH_SERVER_URL
          value: http://keycloak-service:8080
        - name: AUTH_OAUTH2_CLIENT_KEYCLOAK_CLIENT_ID
          value: kafka-ui
        - name: AUTH_OAUTH2_CLIENT_KEYCLOAK_CLIENT_SECRET
          value: kafka-ui-secret-456
        - name: AUTH_OAUTH2_CLIENT_KEYCLOAK_SCOPE
          value: openid
---
apiVersion: v1
kind: Service
metadata:
  name: kafka-ui-service
  namespace: kafka-demo
spec:
  selector:
    app: kafka-ui
  ports:
  - port: 8080
    targetPort: 8080
    nodePort: 30081
  type: NodePort
EOF


✅ Deploy Keycloak
✅ Configure Keycloak (realm, clients, users) ← You're here
Deploy Kafka cluster
Deploy Kafka UI
Test everything