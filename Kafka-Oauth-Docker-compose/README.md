# Local Kafka, Kafka UI, and OAuth

This stack runs a local Kafka broker, Kafka UI, and Keycloak-based OAuth login.

## Start

```bash
docker compose up -d
```

Kafka UI: http://localhost:8082

Kafka bootstrap from your host machine: `localhost:9092`

Kafka bootstrap from containers on this Compose network: `kafka:9093`

Keycloak admin: http://keycloak.localhost:8081

## Credentials

Kafka UI login is handled by Keycloak:

```text
username: kafka-admin
password: admin
```

Keycloak admin console:

```text
username: admin
password: admin
```

The Kafka UI OAuth client is pre-created in the `devops` realm:

```text
client id: kafka-ui
client secret: kafka-ui-local-secret
callback: http://localhost:8082/login/oauth2/code/keycloak
issuer: http://keycloak.localhost:8081/realms/devops
```

## Smoke Test

Create and consume a message:

```bash
docker compose exec kafka /opt/kafka/bin/kafka-console-producer.sh \
  --bootstrap-server localhost:9092 \
  --topic local.dev.events
```

Then in another terminal:

```bash
docker compose exec kafka /opt/kafka/bin/kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 \
  --topic local.dev.events \
  --from-beginning
```

## Stop

```bash
docker compose down
```

To remove containers and any named volumes created later:

```bash
docker compose down -v
```

These credentials and secrets are for local development only.
