# Create namespace
kubectl create namespace dev-kafka

# Create PersistentVolumeClaim for Kafka data
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: kafka-data-pvc
  namespace: dev-kafka
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
---
# Create Kafka deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kafka
  namespace: dev-kafka
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kafka
  template:
    metadata:
      labels:
        app: kafka
    spec:
      containers:
      - name: kafka
        image: confluentinc/cp-kafka:7.5.0
        ports:
        - containerPort: 9092
        - containerPort: 9093
        env:
        - name: KAFKA_PROCESS_ROLES
          value: "broker,controller"
        - name: KAFKA_NODE_ID
          value: "1"
        - name: KAFKA_CONTROLLER_QUORUM_VOTERS
          value: "1@localhost:9093"
        - name: KAFKA_LISTENERS
          value: "PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093"
        - name: KAFKA_ADVERTISED_LISTENERS
          value: "PLAINTEXT://kafka-service:9092"
        - name: KAFKA_LISTENER_SECURITY_PROTOCOL_MAP
          value: "PLAINTEXT:PLAINTEXT,CONTROLLER:PLAINTEXT"
        - name: KAFKA_CONTROLLER_LISTENER_NAMES
          value: "CONTROLLER"
        - name: KAFKA_INTER_BROKER_LISTENER_NAME
          value: "PLAINTEXT"
        - name: KAFKA_LOG_DIRS
          value: "/kafka/data"
        - name: CLUSTER_ID
          value: "4L6g3nShT-eMCtK--X86sw"
        - name: KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR
          value: "1"
        - name: KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS
          value: "0"
        - name: KAFKA_TRANSACTION_STATE_LOG_MIN_ISR
          value: "1"
        - name: KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR
          value: "1"
        - name: KAFKA_MESSAGE_MAX_BYTES
          value: "10485760"  # 10MB
        - name: KAFKA_REPLICA_FETCH_MAX_BYTES
          value: "10485760" # 10MB
        - name: KAFKA_FETCH_MESSAGE_MAX_BYTES
          value: "10485760"   # for consumers          
        volumeMounts:
        - name: kafka-data
          mountPath: /kafka/data
        readinessProbe:
          tcpSocket:
            port: 9092
          initialDelaySeconds: 30
          periodSeconds: 10
        livenessProbe:
          tcpSocket:
            port: 9092
          initialDelaySeconds: 60
          periodSeconds: 30
      volumes:
      - name: kafka-data
        persistentVolumeClaim:
          claimName: kafka-data-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: kafka-service
  namespace: dev-kafka
spec:
  selector:
    app: kafka
  ports:
  - name: kafka
    port: 9092
    targetPort: 9092
  - name: controller
    port: 9093
    targetPort: 9093
  type: ClusterIP
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kafka-ui
  namespace: dev-kafka
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
          value: "dev-kafka"
        - name: KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS
          value: "kafka-service:9092"
        - name: KAFKA_CLUSTERS_0_PROPERTIES_SECURITY_PROTOCOL
          value: "PLAINTEXT"
        readinessProbe:
          httpGet:
            path: /
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /
            port: 8080
          initialDelaySeconds: 60
          periodSeconds: 30
---
apiVersion: v1
kind: Service
metadata:
  name: kafka-ui-service
  namespace: dev-kafka
spec:
  selector:
    app: kafka-ui
  ports:
  - name: http
    port: 8080
    targetPort: 8080
  type: NodePort
EOF

# Wait for deployments to be ready
echo "Waiting for Kafka deployment..."
kubectl wait --for=condition=available --timeout=300s deployment/kafka -n dev-kafka

echo "Waiting for Kafka UI deployment..."
kubectl wait --for=condition=available --timeout=300s deployment/kafka-ui -n dev-kafka

# Check pod status
echo "Checking pod status..."
kubectl get pods -n dev-kafka

# Get service information
echo "Getting service information..."
kubectl get services -n dev-kafka