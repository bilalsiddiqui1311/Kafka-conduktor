# SASL_SSL Implementation Guide

## Step 1: Create SSL Certificates

First, create the directory structure and generate certificates:

```bash
# Create SSL directory
mkdir -p ssl
cd ssl

# Generate CA certificate
openssl req -new -x509 -keyout ca-key -out ca-cert -days 365 \
  -subj "/CN=ca/OU=Test/O=Test/L=Test/ST=Test/C=US" -nodes

# Generate server keystore
keytool -keystore kafka.server.keystore.jks -alias localhost \
  -validity 365 -genkey -keyalg RSA \
  -dname "CN=localhost,OU=Test,O=Test,L=Test,ST=Test,C=US" \
  -storepass changeit -keypass changeit -noprompt

# Generate certificate signing request
keytool -keystore kafka.server.keystore.jks -alias localhost \
  -certreq -file cert-file -storepass changeit -noprompt

# Sign the certificate
openssl x509 -req -CA ca-cert -CAkey ca-key -in cert-file \
  -out cert-signed -days 365 -CAcreateserial -passin pass:

# Import CA certificate into keystore
keytool -keystore kafka.server.keystore.jks -alias CARoot \
  -import -file ca-cert -storepass changeit -noprompt

# Import signed certificate
keytool -keystore kafka.server.keystore.jks -alias localhost \
  -import -file cert-signed -storepass changeit -noprompt

# Create truststore
keytool -keystore kafka.server.truststore.jks -alias CARoot \
  -import -file ca-cert -storepass changeit -noprompt

# Create credential files
echo "changeit" > keystore_creds
echo "changeit" > key_creds
echo "changeit" > truststore_creds

cd ..
```

## Step 2: Create JAAS Configuration

Create `kafka_server_jaas.conf`:

```ini
KafkaServer {
    org.apache.kafka.common.security.plain.PlainLoginModule required
    username="admin"
    password="admin-secret"
    user_admin="admin-secret"
    user_producer="producer-secret"
    user_consumer="consumer-secret";
};
```

## Step 3: Updated Docker Compose

Your directory structure should look like:
```
your-project/
├── docker-compose.yml
├── kafka_server_jaas.conf
└── ssl/
    ├── kafka.server.keystore.jks
    ├── kafka.server.truststore.jks
    ├── keystore_creds
    ├── key_creds
    └── truststore_creds
```

## Step 4: RUN Docker Compose

Implementation Steps:
1. Run the SSL certificate generation commands from the guide above
2. Create the JAAS config file as shown

## Clean up existing containers
docker-compose down -v

# Start the services
docker-compose up

# Test
http://localhost:8080