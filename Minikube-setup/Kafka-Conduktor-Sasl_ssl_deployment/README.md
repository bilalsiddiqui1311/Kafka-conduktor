# step-by-step set of commands that addresses the issues and includes all necessary files for your Minikube setup.


#!/bin/bash

# Create a temporary directory for cert generation
mkdir certs_temp
cd certs_temp

# 1. Generate CA certificate and private key
openssl req -new -x509 -keyout ca-key.pem -out ca-cert.pem -days 365 \
  -subj "/CN=kafka-ca/OU=Test/O=Test/L=Test/ST=Test/C=US" -nodes

# 2. Generate Kafka server keystore and CSR
keytool -keystore kafka.server.keystore.jks -alias kafka-server \
  -validity 365 -genkeypair -keyalg RSA \
  -dname "CN=kafka1/OU=Test/O=Test/L=Test/ST=Test/C=US" \
  -ext "SAN=DNS:kafka1,DNS:localhost" \
  -storepass changeit -keypass changeit -noprompt

keytool -keystore kafka.server.keystore.jks -alias kafka-server \
  -certreq -file cert-file.csr -storepass changeit -noprompt

# 3. Sign the server certificate
openssl x509 -req -CA ca-cert.pem -CAkey ca-key.pem -in cert-file.csr \
  -out cert-signed.pem -days 365 -CAcreateserial -passin pass:

# 4. Import CA and signed certificate into server keystore
keytool -keystore kafka.server.keystore.jks -alias CARoot \
  -importcert -file ca-cert.pem -storepass changeit -noprompt

keytool -keystore kafka.server.keystore.jks -alias kafka-server \
  -importcert -file cert-signed.pem -storepass changeit -noprompt

# 5. Create a truststore for both Kafka and Conduktor
keytool -keystore kafka.server.truststore.jks -alias CARoot \
  -importcert -file ca-cert.pem -storepass changeit -noprompt

# 6. Create password files
echo "changeit" > keystore_creds
echo "changeit" > key_creds
echo "changeit" > truststore_creds

# Move files to a new `ssl` directory
mkdir ../ssl
mv kafka.server.keystore.jks ../ssl/
mv kafka.server.truststore.jks ../ssl/
mv keystore_creds ../ssl/
mv key_creds ../ssl/
mv truststore_creds ../ssl/

# Clean up temporary files
cd ..
rm -rf certs_temp

# 7. Create the JAAS configuration file
echo 'KafkaServer {' > kafka_server_jaas.conf
echo '  org.apache.kafka.common.security.plain.PlainLoginModule required' >> kafka_server_jaas.conf
echo '  user_admin="admin-secret"' >> kafka_server_jaas.conf
echo '  user_producer="producer-secret";' >> kafka_server_jaas.conf
echo '};' >> kafka_server_jaas.conf

echo "Files generated successfully. You can now use the 'ssl' directory and 'kafka_server_jaas.conf' file."


# Create namespace
kubectl create ns kafka-conduktor

# Create Secrets
kubectl create secret generic kafka-ssl-certs -n kafka-conduktor --from-file=ssl/
kubectl create configmap kafka-jaas-config -n kafka-conduktor --from-file=kafka_server_jaas.conf

# Apply postgress
kubectl apply -f postgresql.yml -n kafka-conduktor

# Apply Kafka => Wait for Postgres pod till running
kubectl apply -f kafka.yml -n kafka-conduktor

# Apply Conduktor =>  Wait for Kafka pod till running
kubectl apply -f conduktor.yml -n kafka-conduktor

# Ensure all pods are in a Running state.
kubectl get all -n kafka-conduktor

# Access Conduktor
minikube service conduktor-console -n kafka-conduktor --url