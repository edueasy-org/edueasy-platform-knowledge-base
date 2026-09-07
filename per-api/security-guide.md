# EduEasy Chatbot API Security Guide

> Authentication, authorization, and security best practices

---

## Authentication

### OAuth2 with Azure AD

This API uses OAuth2 with JWT tokens issued by Azure AD. Chat session and public FAQ endpoints are **public** (no auth required). Admin and student endpoints require valid JWTs.

```
┌──────────────┐    ┌──────────────┐    ┌──────────────────┐
│   Client     │───▶│  Azure AD    │───▶│  Chatbot API     │
│              │◀───│  (Token)     │    │  (JWT Validate)  │
└──────────────┘    └──────────────┘    └──────────────────┘
```

### Getting a Token

```bash
curl -X POST "https://login.microsoftonline.com/95d6a08d-8d38-495b-9c70-ab0cacd42d64/oauth2/v2.0/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "client_id={client-id}" \
  -d "client_secret={client-secret}" \
  -d "scope=api://{api-client-id}/.default" \
  -d "grant_type=client_credentials"
```

### Using the Token

```bash
curl -X GET "http://localhost:8090/api/v1/chat/faqs/drafts" \
  -H "Authorization: Bearer {token}"
```

---

## Authorization

### Roles

| Role | Description | Access Level |
|------|-------------|--------------|
| `edueasy-api.admin` | Global admin | Full access to all APIs |
| `edueasy-edueasy-chatbot.admin` | Chatbot admin | Full access to FAQ CRUD, escalations |
| `edueasy-edueasy-chatbot.user` | Student user | Access to student status endpoint |

### Scopes

| Scope | Permission |
|-------|------------|
| `student:read` | View student application status |

### Endpoint Permissions

| Endpoint | Method | Auth Required | Required Role/Scope |
|----------|--------|---------------|---------------------|
| `/api/v1/chat/sessions/**` | ALL | No | Public |
| `/api/v1/chat/faqs` | GET | No | Public |
| `/api/v1/chat/faqs/{id}` | GET | No | Public |
| `/api/v1/chat/faqs/categories` | GET | No | Public |
| `/api/v1/chat/faqs` | POST | Yes | `edueasy-api.admin` or `edueasy-edueasy-chatbot.admin` |
| `/api/v1/chat/faqs/{id}` | PUT | Yes | `edueasy-api.admin` or `edueasy-edueasy-chatbot.admin` |
| `/api/v1/chat/faqs/{id}` | DELETE | Yes | `edueasy-api.admin` or `edueasy-edueasy-chatbot.admin` |
| `/api/v1/chat/faqs/drafts` | GET | Yes | `edueasy-api.admin` or `edueasy-edueasy-chatbot.admin` |
| `/api/v1/chat/faqs/{id}/approve` | PUT | Yes | `edueasy-api.admin` or `edueasy-edueasy-chatbot.admin` |
| `/api/v1/chat/escalations` | GET | Yes | `edueasy-api.admin` or `edueasy-edueasy-chatbot.admin` |
| `/api/v1/chat/escalations/{token}/resolve` | PUT | Yes | `edueasy-api.admin` or `edueasy-edueasy-chatbot.admin` |
| `/api/v1/chat/students/me/status` | GET | Yes | `SCOPE_student:read` or `edueasy-edueasy-chatbot.user` |
| `/actuator/**` | GET | Yes | `edueasy-api.admin` |

---

## Security Headers

All responses include:

```
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
X-XSS-Protection: 1; mode=block
Strict-Transport-Security: max-age=31536000; includeSubDomains
```

---

## Input Validation & Sanitization

### Protected Against

| Attack | Protection |
|--------|------------|
| SQL Injection | Parameterized queries (JPA) |
| XSS | HTML/script tag sanitization on user messages (BR8) |
| CSRF | Stateless API (no cookies) |
| Mass Assignment | Explicit DTO field mapping |
| Message Flooding | 60 messages per session per hour (BR8) |
| Oversized Input | Max 4000 chars per message (BR8) |

### Validation Rules

- All inputs validated against DTO constraints (`@Size`, `@NotBlank`, validation groups)
- Maximum field lengths enforced (question: 1000, answer: 4000, keywords: 500, message: 4000)
- Enum values validated (`FaqCategoryCode`, `ChatStatusCode`, etc.)
- HTML/script tags stripped from user chat messages
- Application reference format validated: `^EPY-\d{4}-\d{6}$`

---

## Data Privacy (BR9)

- Never echo back ID numbers, email addresses, or phone numbers in chat responses
- Guest chat sessions are anonymised (no PII stored)
- Authenticated sessions linked to students are deletable on request (POPIA right to erasure)
- Student data accessed read-only from cross-schema tables

---

## Secrets Management

### Vault

All secrets stored under `edueasy/edueasy-chatbot-service/`:
- `db-password`
- `oauth2-client-secret`
- `rabbitmq-host`
- `rabbitmq-username`
- `rabbitmq-password`

### Jasypt Encryption

Sensitive values in configuration are encrypted:

```yaml
spring:
  datasource:
    password: ENC(encrypted-value-here)
```

Decrypt at runtime with:
```bash
export JASYPT_ENCRYPTOR_PASSWORD=your-secret-key
```

### Never Commit

- `.env` files
- Jasypt passwords
- API keys or client secrets
- Vault tokens

---

## Security Checklist

- [ ] JWT tokens validated on every admin/student request
- [ ] Public endpoints accessible without auth
- [ ] Roles enforced at endpoint level via `@PreAuthorize`
- [ ] Sensitive data encrypted (Jasypt/Vault)
- [ ] HTTPS enforced in production
- [ ] Security headers configured
- [ ] Input validation and sanitization active
- [ ] No PII echoed in chat responses
- [ ] Rate limiting enforced (60 msg/session/hour)
- [ ] Audit logging enabled

---

## Reporting Security Issues

Report security vulnerabilities to: security@edueasy.co.za
