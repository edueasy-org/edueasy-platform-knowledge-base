# EduEasy Chatbot API Production Readiness Checklist

> Pre-deployment verification checklist

---

## Build & Tests

- [ ] `mvn clean compile` passes with zero warnings
- [ ] All 140 unit tests pass (`mvn test`)
- [ ] All 35 integration tests pass (`mvn verify`)
- [ ] Code coverage ≥ 80%
- [ ] No TODO comments in production code
- [ ] Checkstyle passes (0 violations)
- [ ] E2E tests pass against target environment

---

## Configuration

- [ ] All secrets encrypted with Jasypt
- [ ] `JASYPT_ENCRYPTOR_PASSWORD` configured in deployment
- [ ] Correct Oracle database credentials for prod
- [ ] Correct RabbitMQ credentials for prod
- [ ] OAuth2 audience set to production client ID (`58b42ace-949f-42ad-9c7c-d24e563c3c68`)
- [ ] Logging level set to WARN/ERROR for prod profile
- [ ] Vault secrets configured under `edueasy/edueasy-chatbot-service/`

---

## Security

- [ ] OAuth2 JWT validation enabled
- [ ] Role-based access control configured (`@PreAuthorize`)
- [ ] Public endpoints accessible without auth (chat sessions, FAQ reads)
- [ ] Admin endpoints reject 401 without token
- [ ] Student endpoint requires `student:read` scope
- [ ] No sensitive data in logs (BR9)
- [ ] HTTPS enforced
- [ ] Security headers configured
- [ ] Input sanitization active (HTML/script tags)
- [ ] Rate limiting enforced (60 msg/session/hour)

---

## API

- [ ] All endpoints documented in OpenAPI (Swagger)
- [ ] Request validation active (DTO constraints)
- [ ] Error responses follow standard format (`ErrorResponse`)
- [ ] FAQ list supports category filtering and pagination
- [ ] CORS configured appropriately
- [ ] Actuator health/info accessible

---

## Monitoring

- [ ] Health endpoint accessible: `/actuator/health`
- [ ] Info endpoint configured: `/actuator/info`
- [ ] Sentry configured for error tracking (if DSN provided)
- [ ] Structured logging enabled
- [ ] Monitoring thresholds configured:
  - Escalation rate > 30%
  - Average confidence < 0.6
  - Unresolved rate > 20%
  - Response time > 2s
  - Active sessions > 500

---

## Database

- [ ] Oracle connection pool sized appropriately
- [ ] Indexes on frequently queried columns (session token, FAQ keywords)
- [ ] Cross-schema read-only entities verified (`@Immutable`)
- [ ] DDL deployed (EPY_FAQ, EPY_CHAT_SESSION, EPY_CHAT_MESSAGE, EPY_FAQ_CATEGORY, EPY_CHAT_STATUS)
- [ ] FAQ seed data loaded (16+ entries across 6 categories)
- [ ] Audit trigger fields configured (`@Transient`)

---

## Events (RabbitMQ)

- [ ] RabbitMQ exchange exists
- [ ] Queues bound with `SEND_MAIL` routing key
- [ ] Dead letter queue configured
- [ ] `ChatEscalationEmailTemplate` tested
- [ ] `RecurringFaqAlertEmailTemplate` tested
- [ ] `edueasy-comms-api` consuming events

---

## Performance

- [ ] Response times acceptable under load (< 2s)
- [ ] No N+1 query issues
- [ ] Appropriate timeouts configured
- [ ] Connection pools sized correctly (DB + RabbitMQ)

---

## Documentation

- [ ] README.md complete
- [ ] API documentation generated (`/docs`)
- [ ] Architecture documented
- [ ] Security guide reviewed
- [ ] This checklist completed

---

## Deployment

- [ ] Docker image builds successfully
- [ ] Environment variables documented
- [ ] Rollback procedure tested
- [ ] Kong Gateway routes registered
- [ ] Blue-green deployment configured

---

## Sign-off

| Role | Name | Date | Approved |
|------|------|------|----------|
| Developer | | | ☐ |
| Tech Lead | | | ☐ |
| QA | | | ☐ |
| DevOps | | | ☐ |
