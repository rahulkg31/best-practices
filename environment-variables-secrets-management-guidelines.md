# Environment Variables & Secrets Management

## 1. Core Principles

1. **Never commit secrets.** No raw access keys, passwords, or tokens in `application.yml`, `application-*.yml`, or any file tracked by git.
2. **Externalize via placeholders.** Use Spring's `${VAR_NAME}` syntax in YAML. Spring's `Environment` resolves these from OS environment variables and JVM system properties automatically - this works identically whether the app runs as a bare JAR, in Docker, or in a Kubernetes pod.
3. **No default fallbacks for secrets.** Don't write `${AWS_S3_ACCESS_KEY:changeme}` — a silent default masks misconfiguration. Defaults are fine for non-sensitive values (e.g. endpoints, ports).
4. **Prefer short-lived, identity-based credentials over static keys.** Wherever the hosting platform supports it (IAM roles, Managed Identity, Workload Identity), use that instead of long-lived secret keys — even for authenticating *to* the secrets manager itself.
5. **Fail fast.** If a required secret is missing, the app should fail to start, not run with a null/blank credential.
6. **Assume committed history is compromised.** If a secret was ever pushed to git, rotate it — scrubbing history alone is not sufficient.

---

## 2. Standard YAML Pattern

```yaml
# application-dev.yml
aws:
  s3:
    access-key: "${AWS_S3_ACCESS_KEY}"
    secret-key: "${AWS_S3_SECRET_KEY}"
    endpoint: https://fsn1.your-objectstorage.com
    bucket:
      IoT-Sense: iotsense
```

- Sensitive values → `${ENV_VAR}` with **no default**.
- Non-sensitive values (endpoints, bucket names, feature flags) can stay literal or use `${VAR:default}`.
- Keep the same env var **name** across local, Docker, and Kubernetes so the YAML never needs per-environment edits — only the mechanism that populates the env var changes.

---

## 3. Scenario-by-Scenario Setup

### 3.1 Plain JVM (bare metal / VM)

Set variables in the shell or process manager before launching the JAR.

```bash
export AWS_S3_ACCESS_KEY=xxxx
export AWS_S3_SECRET_KEY=yyyy
java -jar iotsense-object-storage-service.jar
```

**systemd unit:**
```ini
[Service]
Environment="AWS_S3_ACCESS_KEY=xxxx"
Environment="AWS_S3_SECRET_KEY=yyyy"
ExecStart=/usr/bin/java -jar /opt/app/iotsense-object-storage-service.jar
```

**Local development:** use a `.env` file loaded via `direnv`, an IDE run configuration, or `dotenv-java`. Add `.env` to `.gitignore`.

### 3.2 Docker

Never bake secrets into the image (`ENV` in a Dockerfile persists in image layers). Inject at runtime instead:

```bash
docker run -e AWS_S3_ACCESS_KEY=xxxx -e AWS_S3_SECRET_KEY=yyyy iotsense-object-storage-service
```

```yaml
# docker-compose.yml
services:
  object-storage-service:
    image: iotsense-object-storage-service:latest
    env_file:
      - .env.production   # not committed to git
```

### 3.3 Kubernetes

Route secrets through a K8s `Secret` object and inject with `envFrom`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: iotsense-aws-secrets
type: Opaque
stringData:
  AWS_S3_ACCESS_KEY: xxxx
  AWS_S3_SECRET_KEY: yyyy
```

```yaml
spec:
  containers:
    - name: object-storage-service
      image: iotsense-object-storage-service:latest
      envFrom:
        - secretRef:
            name: iotsense-aws-secrets
```

Don't hand-write the `Secret` manifest with real values in a repo. Generate/sync it at deploy time using **Sealed Secrets**, **External Secrets Operator**, or a Vault/cloud-native integration (see §4).

---

## 4. Pulling Secrets from a Secret Manager

Two integration patterns, usable with any backend:

- **Pattern A — Entrypoint/wrapper script**: fetch the secret via CLI/SDK before launching the JVM, export as env vars. Works the same for a bare JAR (wrapper script) or Docker (as `ENTRYPOINT`).
- **Pattern B — Spring-native connector**: the app itself pulls secrets on boot via a Spring Cloud integration (`spring.config.import`). No wrapper script needed, but couples the app to one vendor's library.

### 4.1 AWS Secrets Manager

**Pattern A (entrypoint.sh):**
```bash
#!/bin/bash
SECRET_JSON=$(aws secretsmanager get-secret-value \
  --secret-id iotsense/object-storage/aws-s3 \
  --query 'SecretString' --output text)

export AWS_S3_ACCESS_KEY=$(echo "$SECRET_JSON" | jq -r '.access_key')
export AWS_S3_SECRET_KEY=$(echo "$SECRET_JSON" | jq -r '.secret_key')

exec java -jar /app/app.jar
```

**Pattern B (Spring Cloud AWS):**
```xml
<dependency>
    <groupId>io.awspring.cloud</groupId>
    <artifactId>spring-cloud-aws-starter-secrets-manager</artifactId>
</dependency>
```
```yaml
spring:
  config:
    import: aws-secretsmanager:iotsense/object-storage/aws-s3
```

**Auth to AWS itself** (never static keys where avoidable):
| Runtime | Mechanism |
|---|---|
| EC2 (JAR or Docker on EC2) | IAM instance profile |
| ECS | IAM Task Role |
| EKS | IRSA (IAM Roles for Service Accounts) |
| Local dev | `aws-vault` or SSO profile — short-lived creds, never committed |

### 4.2 HashiCorp Vault

**Pattern B (Spring Cloud Vault):**
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-vault-config</artifactId>
</dependency>
```
```yaml
spring:
  cloud:
    vault:
      uri: https://vault.internal:8200
      authentication: KUBERNETES   # or TOKEN, APPROLE, AWS_IAM
      kv:
        backend: secret
        default-context: iotsense/object-storage
  config:
    import: vault://
```

**Vault Agent Injector (recommended for K8s):** a sidecar fetches secrets and renders them as files or env vars before the app starts. The app stays completely vault-agnostic — preferred when you want to avoid vendor coupling in app code.

### 4.3 Azure Key Vault

```xml
<dependency>
    <groupId>com.azure.spring</groupId>
    <artifactId>spring-cloud-azure-starter-keyvault-secrets</artifactId>
</dependency>
```
```yaml
spring:
  cloud:
    azure:
      keyvault:
        secret:
          property-sources:
            - endpoint: https://iotsense-vault.vault.azure.net/
```
**Auth:** Managed Identity (Azure VM, AKS, App Service) — no static keys.

### 4.4 GCP Secret Manager

```xml
<dependency>
    <groupId>com.google.cloud</groupId>
    <artifactId>spring-cloud-gcp-starter-secretmanager</artifactId>
</dependency>
```
```yaml
aws:
  s3:
    access-key: "${sm://iotsense-aws-s3-access-key}"
```
**Auth:** Workload Identity (GKE) or the VM's attached service account (GCE).

### 4.5 Backend-agnostic entrypoint (multi-cloud / migration scenarios)

One Docker image, pluggable backend selected at deploy time:

```bash
#!/bin/bash
# entrypoint.sh
case "$SECRET_BACKEND" in
  aws)
    SECRET_JSON=$(aws secretsmanager get-secret-value --secret-id "$SECRET_ID" --query SecretString --output text)
    ;;
  vault)
    SECRET_JSON=$(vault kv get -format=json "$SECRET_PATH" | jq -r '.data.data')
    ;;
  azure)
    SECRET_JSON=$(az keyvault secret show --name "$SECRET_NAME" --vault-name "$VAULT_NAME" --query value -o tsv)
    ;;
  gcp)
    SECRET_JSON=$(gcloud secrets versions access latest --secret="$SECRET_NAME")
    ;;
esac

export AWS_S3_ACCESS_KEY=$(echo "$SECRET_JSON" | jq -r '.access_key')
export AWS_S3_SECRET_KEY=$(echo "$SECRET_JSON" | jq -r '.secret_key')

exec java -jar /app/app.jar
```

---

## 5. Choosing a Pattern

- **Vault Agent Injector** (or equivalent sidecar/init-container pattern): best when you want app code fully decoupled from the secrets backend. Most portable across clouds.
- **Entrypoint script abstraction (§4.5)**: best when supporting multiple backends with one Docker image, or during a cloud migration.
- **Spring-native connector (Pattern B)**: simplest to wire up, but couples the app to one vendor's SDK — fine if you're committed long-term to a single cloud.
