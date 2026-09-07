# EduEasy Chatbot API Quick Reference

> Essential commands and shortcuts for epy-chatbot-api

---

## Common Commands

### Build & Run

```bash
# Build
mvn clean compile

# Run tests (unit only)
mvn test

# Run all tests (unit + integration)
mvn verify

# Run with coverage
mvn test jacoco:report

# Start locally
mvn spring-boot:run -Dspring-boot.run.profiles=local

# Package
mvn clean package -DskipTests
```

### Environment URLs

| Environment | API URL | Swagger UI |
|-------------|---------|------------|
| Local | http://localhost:8090 | http://localhost:8090/swagger-ui.html |
| Dev | https://devapi.edueasy.co.za/epy-chatbot-api | /swagger-ui.html |
| Test | https://testapi.edueasy.co.za/epy-chatbot-api | /swagger-ui.html |
| Train | https://trainapi.edueasy.co.za/epy-chatbot-api | N/A |
| Prod | https://api.edueasy.co.za/epy-chatbot-api | N/A |

---

## API Endpoints

### Public (No Auth)

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/v1/chat/sessions` | Create session |
| GET | `/api/v1/chat/sessions/{token}` | Get session |
| PUT | `/api/v1/chat/sessions/{token}/close` | Close session |
| GET | `/api/v1/chat/sessions/{token}/messages` | Get messages |
| POST | `/api/v1/chat/sessions/{token}/messages` | Send message |
| GET | `/api/v1/chat/faqs` | List FAQs |
| GET | `/api/v1/chat/faqs/{id}` | Get FAQ |
| GET | `/api/v1/chat/faqs/categories` | List categories |

### Admin (JWT Required)

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/v1/chat/faqs` | Create FAQ |
| PUT | `/api/v1/chat/faqs/{id}` | Update FAQ |
| DELETE | `/api/v1/chat/faqs/{id}` | Soft-delete FAQ |
| GET | `/api/v1/chat/faqs/drafts` | List drafts |
| PUT | `/api/v1/chat/faqs/{id}/approve` | Approve draft |
| GET | `/api/v1/chat/escalations` | List escalations |
| PUT | `/api/v1/chat/escalations/{token}/resolve` | Resolve |

### Student (JWT Required)

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/v1/chat/students/me/status` | Application status |

---

## Quick curl Examples

```bash
# Set base URL
export BASE="http://localhost:8090"

# Create session
curl -X POST "$BASE/api/v1/chat/sessions" -H "Content-Type: application/json"

# Send message
curl -X POST "$BASE/api/v1/chat/sessions/TOKEN/messages" \
  -H "Content-Type: application/json" \
  -d '{"message": "Hello"}'

# List FAQs
curl "$BASE/api/v1/chat/faqs"

# List FAQ categories
curl "$BASE/api/v1/chat/faqs/categories"

# Admin: Create FAQ
curl -X POST "$BASE/api/v1/chat/faqs" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"question":"Q","answer":"A","faqCategoryCode":"GENERAL","keywords":"test"}'

# Admin: List escalations
curl "$BASE/api/v1/chat/escalations" -H "Authorization: Bearer $TOKEN"
```

---

## Health Checks

```bash
# Health
curl http://localhost:8090/actuator/health

# Info
curl http://localhost:8090/actuator/info
```

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| 401 Unauthorized | Check JWT token is valid and not expired |
| 403 Forbidden | Verify your token has `edueasy-edueasy-chatbot.admin` role |
| 404 Not Found | Check endpoint URL, session token, or FAQ ID |
| 409 Conflict | Invalid state transition (e.g., closing already closed session) |
| 422 Validation Error | Check request body matches DTO requirements |
| Connection refused | Ensure app is running on port 8090 |
| JASYPT decrypt error | Set `JASYPT_ENCRYPTOR_PASSWORD` env var |

---

## Useful Profiles

```bash
# Local development (H2 database, debug logging)
-Dspring-boot.run.profiles=local

# Development environment
-Dspring-boot.run.profiles=dev

# Test environment
-Dspring-boot.run.profiles=test

# Production (minimal logging)
-Dspring-boot.run.profiles=prod
```

---

## Key Configuration

| Property | Default | Description |
|----------|---------|-------------|
| server.port | 8090 | HTTP port |
| spring.profiles.active | local | Active profile |
| chatbot.confidence-threshold | 0.7 | FAQ match threshold |
| chatbot.max-suggestions | 3 | Suggestions when low confidence |
| chatbot.recurring-faq-threshold | 3 | Trigger for recurring FAQ alert |

---

## Contacts

- **API Support:** api-support@edueasy.co.za
- **Documentation:** See `/docs` folder
