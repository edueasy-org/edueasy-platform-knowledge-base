# EduEasy Chatbot API Setup and Configuration Guide

> Development environment setup and configuration

---

## Prerequisites

| Requirement | Version | Check Command |
|-------------|---------|---------------|
| Java | 21 | `java -version` |
| Maven | 3.9+ | `mvn -version` |
| Git | Latest | `git --version` |
| Node.js | 18+ | `node --version` (for E2E tests) |

---

## Quick Start

### 1. Clone Repository

```bash
git clone <your-repo-url>
cd edueasy-chatbot-api
```

### 2. Set Environment Variables

```powershell
# Required for encrypted properties
$env:JASYPT_ENCRYPTOR_PASSWORD = "your-secret-key"

# Optional: Override defaults
$env:SERVER_PORT = "8090"
```

### 3. Build and Run

```bash
# Build
mvn clean compile

# Run locally
mvn spring-boot:run -Dspring-boot.run.profiles=local
```

### 4. Verify

```bash
# Health check
curl http://localhost:8090/actuator/health

# Swagger UI
start http://localhost:8090/swagger-ui.html
```

---

## Configuration Properties

### Core Properties

| Property | Description | Default |
|----------|-------------|---------|
| `server.port` | HTTP port | 8090 |
| `spring.profiles.active` | Active profile | local |
| `logging.level.root` | Log level | INFO |

### Chatbot Properties

| Property | Description | Default |
|----------|-------------|---------|
| `chatbot.confidence-threshold` | FAQ match confidence threshold | 0.7 |
| `chatbot.max-suggestions` | Number of suggestions for low confidence | 3 |
| `chatbot.recurring-faq-threshold` | Occurrences before recurring FAQ alert | 3 |
| `chatbot.escalation.admin-email` | Escalation notification recipient | edueasy-support@edueasy.co.za |
| `chatbot.escalation.faq-admin-email` | FAQ alert recipient | edueasy-faq-admin@edueasy.co.za |

### Security Properties

| Property | Description | Required |
|----------|-------------|----------|
| `spring.security.oauth2.resourceserver.jwt.issuer-uri` | Azure AD issuer | Yes (auto from parent) |
| `spring.security.oauth2.resourceserver.jwt.audiences` | API client ID | Yes (auto from parent) |

### Database Properties

| Property | Description |
|----------|-------------|
| `spring.datasource.url` | Oracle JDBC URL |
| `spring.datasource.username` | DB username |
| `spring.datasource.password` | DB password (Jasypt encrypted) |

### RabbitMQ Properties

| Property | Description |
|----------|-------------|
| `spring.rabbitmq.host` | RabbitMQ host |
| `spring.rabbitmq.port` | RabbitMQ port (default 5672) |
| `spring.rabbitmq.username` | Username |
| `spring.rabbitmq.password` | Password (Jasypt encrypted) |

---

## Environment Profiles

### local
- H2 in-memory database (Oracle mode)
- DEBUG logging
- Swagger UI enabled
- Mock JWT for testing

### dev
- Development Oracle database
- INFO logging
- Swagger UI enabled

### test
- Test Oracle database
- INFO logging
- Swagger UI enabled

### prod
- Production Oracle database
- WARN logging
- Swagger UI disabled

---

## Jasypt Encryption

### Encrypt a Value

```bash
# Using Maven plugin
mvn jasypt:encrypt-value \
  -Djasypt.encryptor.password=your-secret-key \
  -Djasypt.plugin.value="secret-to-encrypt"
```

### Use in Configuration

```yaml
spring:
  datasource:
    password: ENC(encrypted-value-here)
```

### Runtime Decryption

```powershell
$env:JASYPT_ENCRYPTOR_PASSWORD = "your-secret-key"
mvn spring-boot:run
```

---

## IDE Setup

### IntelliJ IDEA

1. Import as Maven project
2. Set JDK to 21
3. Enable annotation processing
4. Set run configuration:
   - Main class: `za.co.edueasy.api.chatbot.Application`
   - VM options: `-Dspring.profiles.active=local`
   - Environment: `JASYPT_ENCRYPTOR_PASSWORD=your-key`

### VS Code

1. Install Java Extension Pack
2. Open folder
3. `.vscode/launch.json`:

```json
{
  "configurations": [{
    "type": "java",
    "name": "Launch Application",
    "request": "launch",
    "mainClass": "za.co.edueasy.api.chatbot.Application",
    "args": "--spring.profiles.active=local",
    "env": {
      "JASYPT_ENCRYPTOR_PASSWORD": "your-key"
    }
  }]
}
```

---

## Docker (Optional)

### Build Image

```bash
mvn clean package -DskipTests
docker build -t epy-chatbot-api .
```

### Run Container

```bash
docker run -p 8090:8090 \
  -e SPRING_PROFILES_ACTIVE=local \
  -e JASYPT_ENCRYPTOR_PASSWORD=your-key \
  epy-chatbot-api
```

---

## Vault Secrets

All secrets are stored in Vault under `edueasy/edueasy-chatbot-service/`:

| Secret | Description |
|--------|-------------|
| `db-password` | Oracle database password |
| `oauth2-client-secret` | Azure AD client secret |
| `rabbitmq-host` | RabbitMQ hostname |
| `rabbitmq-username` | RabbitMQ username |
| `rabbitmq-password` | RabbitMQ password |

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Port already in use | Change `server.port` or kill existing process on 8090 |
| JASYPT decrypt error | Set `JASYPT_ENCRYPTOR_PASSWORD` env var |
| Maven build fails | Run `mvn clean` and retry |
| Java version mismatch | Ensure `JAVA_HOME` points to JDK 21 |
| H2 errors in local | Check application-local.yml uses `MODE=Oracle` |
| RabbitMQ connection refused | Check RabbitMQ is running and credentials are correct |

---

## Next Steps

- [API Specification](api-specification.md)
- [Testing Guide](testing-guide.md)
- [Architecture](architecture.md)
