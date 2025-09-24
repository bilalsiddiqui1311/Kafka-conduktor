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
# Create Kafka deployment with StatefulSet instead of Deployment
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: kafka
  namespace: dev-kafka
spec:
  serviceName: kafka-headless
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
        # Fixed message size configurations
        - name: KAFKA_MESSAGE_MAX_BYTES
          value: "52428800"  # 50MB (increased from 10MB)
        - name: KAFKA_REPLICA_FETCH_MAX_BYTES
          value: "52428800"  # 50MB
        - name: KAFKA_FETCH_MESSAGE_MAX_BYTES
          value: "52428800"  # 50MB for consumers
        - name: KAFKA_SOCKET_REQUEST_MAX_BYTES
          value: "52428800"  # 50MB for socket requests
        # Additional buffer configurations
        - name: KAFKA_PRODUCER_PURGATORY_PURGE_INTERVAL_REQUESTS
          value: "1000"
        - name: KAFKA_FETCH_PURGATORY_PURGE_INTERVAL_REQUESTS
          value: "1000"
        - name: KAFKA_AUTO_CREATE_TOPICS_ENABLE
          value: "true"
        # JVM heap settings for better performance
        - name: KAFKA_HEAP_OPTS
          value: "-Xmx1G -Xms1G"
        # Graceful shutdown configuration
        - name: KAFKA_CONTROLLED_SHUTDOWN_ENABLE
          value: "true"
        - name: KAFKA_CONTROLLED_SHUTDOWN_MAX_RETRIES
          value: "3"
        - name: KAFKA_CONTROLLED_SHUTDOWN_RETRY_BACKOFF_MS
          value: "5000"
        volumeMounts:
        - name: kafka-data
          mountPath: /kafka/data
        readinessProbe:
          tcpSocket:
            port: 9092
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
        livenessProbe:
          tcpSocket:
            port: 9092
          initialDelaySeconds: 60
          periodSeconds: 30
          timeoutSeconds: 10
        # Graceful termination
        lifecycle:
          preStop:
            exec:
              command:
              - /bin/bash
              - -c
              - "kafka-server-stop.sh"
        # Resource limits to prevent OOM
        resources:
          requests:
            memory: "1Gi"
            cpu: "500m"
          limits:
            memory: "2Gi"
            cpu: "1000m"
      # Proper termination grace period
      terminationGracePeriodSeconds: 60
      volumes:
      - name: kafka-data
        persistentVolumeClaim:
          claimName: kafka-data-pvc
---
# Headless service for StatefulSet
apiVersion: v1
kind: Service
metadata:
  name: kafka-headless
  namespace: dev-kafka
spec:
  clusterIP: None
  selector:
    app: kafka
  ports:
  - name: kafka
    port: 9092
    targetPort: 9092
  - name: controller
    port: 9093
    targetPort: 9093
---
# Regular service for external access
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
        # Increase UI limits for large messages
        - name: SPRING_KAFKA_CONSUMER_MAX_POLL_RECORDS
          value: "10"
        - name: SPRING_KAFKA_CONSUMER_FETCH_MAX_BYTES
          value: "52428800"
        - name: SPRING_KAFKA_CONSUMER_FETCH_MIN_BYTES
          value: "1"
        - name: SERVER_MAX_HTTP_HEADER_SIZE
          value: "50485760"
        - name: SERVER_MAX_HTTP_POST_SIZE
          value: "52428800"
        - name: SPRING_SERVLET_MULTIPART_MAX_FILE_SIZE
          value: "50MB"
        - name: SPRING_SERVLET_MULTIPART_MAX_REQUEST_SIZE
          value: "50MB"
        # Kafka UI specific buffer settings
        - name: KAFKA_CLUSTERS_0_PROPERTIES_MAX_POLL_RECORDS
          value: "10"
        - name: KAFKA_CLUSTERS_0_PROPERTIES_FETCH_MAX_BYTES
          value: "52428800"
        - name: KAFKA_CLUSTERS_0_PROPERTIES_MAX_PARTITION_FETCH_BYTES
          value: "52428800"
        # JVM settings for Kafka UI
        - name: JAVA_OPTS
          value: "-Xmx2G -Xms512M"
        # WebFlux buffer limit - this is the missing piece!
        - name: SPRING_CODEC_MAX_IN_MEMORY_SIZE
          value: "52428800"
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
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "1Gi"
            cpu: "500m"
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

# Wait for StatefulSet to be ready
echo "Waiting for Kafka StatefulSet..."
kubectl wait --for=condition=ready pod/kafka-0 -n dev-kafka --timeout=300s

echo "Waiting for Kafka UI deployment..."
kubectl wait --for=condition=available --timeout=300s deployment/kafka-ui -n dev-kafka

# Check pod status
echo "Checking pod status..."
kubectl get pods -n dev-kafka

# Get service information
echo "Getting service information..."
kubectl get services -n dev-kafka