# API Specification — EduEasy Chatbot API

> Detailed endpoint documentation for the edueasy-chatbot-api

---

## Quick Start

### Base URL

| Environment | URL |
|-------------|-----|
| Local | `http://localhost:8090` |
| Development | `https://devapi.edueasy.co.za/epy-chatbot-api` |
| Test | `https://testapi.edueasy.co.za/epy-chatbot-api` |
| Training | `https://trainapi.edueasy.co.za/epy-chatbot-api` |
| Production | `https://api.edueasy.co.za/epy-chatbot-api` |

### Authentication

Admin and student endpoints require a valid JWT bearer token:

```bash
curl -X GET "https://api.edueasy.co.za/epy-chatbot-api/api/v1/chat/faqs/drafts" \
  -H "Authorization: Bearer $JWT_TOKEN" \
  -H "Content-Type: application/json"
```

Chat session and public FAQ endpoints do **not** require authentication.

### Get a Token

```bash
# Azure AD OAuth2 Client Credentials Flow
curl -X POST "https://login.microsoftonline.com/95d6a08d-8d38-495b-9c70-ab0cacd42d64/oauth2/v2.0/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "client_id={client-id}" \
  -d "client_secret={client-secret}" \
  -d "scope=api://{api-client-id}/.default" \
  -d "grant_type=client_credentials"
```

---

## Endpoints Summary

### Public Endpoints (No Auth Required)

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/v1/chat/sessions` | Create a new chat session |
| GET | `/api/v1/chat/sessions/{sessionToken}` | Get session details |
| PUT | `/api/v1/chat/sessions/{sessionToken}/close` | Close a chat session |
| GET | `/api/v1/chat/sessions/{sessionToken}/messages` | Get message history |
| POST | `/api/v1/chat/sessions/{sessionToken}/messages` | Send message, receive bot response |
| GET | `/api/v1/chat/faqs` | List active FAQs |
| GET | `/api/v1/chat/faqs/{faqId}` | Get FAQ by ID |
| GET | `/api/v1/chat/faqs/categories` | List FAQ categories |

### Admin Endpoints (Role Required)

| Method | Endpoint | Required Role | Description |
|--------|----------|---------------|-------------|
| POST | `/api/v1/chat/faqs` | `edueasy-api.admin` or `edueasy-edueasy-chatbot.admin` | Create FAQ |
| PUT | `/api/v1/chat/faqs/{faqId}` | `edueasy-api.admin` or `edueasy-edueasy-chatbot.admin` | Update FAQ |
| DELETE | `/api/v1/chat/faqs/{faqId}` | `edueasy-api.admin` or `edueasy-edueasy-chatbot.admin` | Soft-delete FAQ |
| GET | `/api/v1/chat/faqs/drafts` | `edueasy-api.admin` or `edueasy-edueasy-chatbot.admin` | List draft FAQs |
| PUT | `/api/v1/chat/faqs/{faqId}/approve` | `edueasy-api.admin` or `edueasy-edueasy-chatbot.admin` | Approve draft FAQ |
| GET | `/api/v1/chat/escalations` | `edueasy-api.admin` or `edueasy-edueasy-chatbot.admin` | List escalated sessions |
| PUT | `/api/v1/chat/escalations/{sessionToken}/resolve` | `edueasy-api.admin` or `edueasy-edueasy-chatbot.admin` | Resolve escalation |

### Student Endpoint

| Method | Endpoint | Required Scope/Role | Description |
|--------|----------|---------------------|-------------|
| GET | `/api/v1/chat/students/me/status` | `SCOPE_student:read` or `edueasy-edueasy-chatbot.user` | Get application status |

---

## Chat Session Endpoints

### Create Chat Session

**POST** `/api/v1/chat/sessions`

Creates a new chat session. If a JWT is present, links the session to the authenticated student.

```bash
curl -X POST "http://localhost:8090/api/v1/chat/sessions" \
  -H "Content-Type: application/json"
```

**Response (201 Created):**
```json
{
  "chatSessionId": 1,
  "sessionToken": "550e8400-e29b-41d4-a716-446655440000",
  "chatStatusCode": "ACTIVE",
  "conversationState": "IDLE",
  "isAuthenticated": "N",
  "startedAt": "2026-04-25T10:30:00Z",
  "message": "Welcome to EduEasy! How can I help you today?"
}
```

### Get Chat Session

**GET** `/api/v1/chat/sessions/{sessionToken}`

```bash
curl "http://localhost:8090/api/v1/chat/sessions/550e8400-e29b-41d4-a716-446655440000"
```

**Response (200 OK):**
```json
{
  "chatSessionId": 1,
  "sessionToken": "550e8400-e29b-41d4-a716-446655440000",
  "chatStatusCode": "ACTIVE",
  "conversationState": "IDLE",
  "isAuthenticated": "N",
  "startedAt": "2026-04-25T10:30:00Z"
}
```

### Close Chat Session

**PUT** `/api/v1/chat/sessions/{sessionToken}/close`

Only ACTIVE or ESCALATED sessions can be closed.

```bash
curl -X PUT "http://localhost:8090/api/v1/chat/sessions/550e8400-e29b-41d4-a716-446655440000/close"
```

**Response (200 OK):**
```json
{
  "chatSessionId": 1,
  "sessionToken": "550e8400-e29b-41d4-a716-446655440000",
  "chatStatusCode": "CLOSED",
  "endedAt": "2026-04-25T11:00:00Z"
}
```

### Get Chat Messages

**GET** `/api/v1/chat/sessions/{sessionToken}/messages`

Returns message history ordered by sentAt ascending.

```bash
curl "http://localhost:8090/api/v1/chat/sessions/550e8400-e29b-41d4-a716-446655440000/messages"
```

**Response (200 OK):**
```json
[
  {
    "chatMessageId": 1,
    "messageType": "USER",
    "content": "What is my application status?",
    "intentDetected": "CHECK_STATUS",
    "confidenceScore": 0.85,
    "sentAt": "2026-04-25T10:31:00Z"
  },
  {
    "chatMessageId": 2,
    "messageType": "BOT",
    "content": "I can help you check your application status. Please provide your application reference (e.g., EPY-2026-001234).",
    "sentAt": "2026-04-25T10:31:01Z"
  }
]
```

### Send Chat Message

**POST** `/api/v1/chat/sessions/{sessionToken}/messages`

Sends a user message and returns the bot response. Processes intent detection, FAQ matching, conversational prompting, and escalation.

```bash
curl -X POST "http://localhost:8090/api/v1/chat/sessions/550e8400-e29b-41d4-a716-446655440000/messages" \
  -H "Content-Type: application/json" \
  -d '{"message": "How do I apply?"}'
```

**Response (200 OK):**
```json
{
  "userMessage": {
    "chatMessageId": 3,
    "messageType": "USER",
    "content": "How do I apply?",
    "intentDetected": "FAQ_APPLICATION",
    "confidenceScore": 0.92,
    "faqId": 5,
    "sentAt": "2026-04-25T10:32:00Z"
  },
  "botMessage": {
    "chatMessageId": 4,
    "messageType": "BOT",
    "content": "To apply to EduEasy, visit our application portal and follow the steps...",
    "sentAt": "2026-04-25T10:32:01Z"
  }
}
```

---

## FAQ Endpoints

### List Active FAQs

**GET** `/api/v1/chat/faqs`

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| categoryCode | string | - | Filter by category (GENERAL, APPLICATION, PAYMENT, SPONSORSHIP, ACCOUNT, PROGRAMME) |
| limit | integer | 20 | Maximum results (max 100) |
| offset | integer | 0 | Starting index |

```bash
curl "http://localhost:8090/api/v1/chat/faqs?categoryCode=PAYMENT&limit=10"
```

### Get FAQ by ID

**GET** `/api/v1/chat/faqs/{faqId}`

```bash
curl "http://localhost:8090/api/v1/chat/faqs/5"
```

**Response (200 OK):**
```json
{
  "faqId": 5,
  "faqCategoryCode": "APPLICATION",
  "question": "How do I apply to EduEasy?",
  "answer": "Visit our application portal and follow the steps...",
  "keywords": "apply,application,register,how to",
  "isActive": "Y",
  "viewCount": 142
}
```

### List FAQ Categories

**GET** `/api/v1/chat/faqs/categories`

```bash
curl "http://localhost:8090/api/v1/chat/faqs/categories"
```

**Response (200 OK):**
```json
[
  {"code": "GENERAL", "meaning": "General questions about EduEasy"},
  {"code": "APPLICATION", "meaning": "Application process questions"},
  {"code": "PAYMENT", "meaning": "Payment and fee questions"},
  {"code": "SPONSORSHIP", "meaning": "Sponsorship and bursary questions"},
  {"code": "ACCOUNT", "meaning": "Account and login questions"},
  {"code": "PROGRAMME", "meaning": "Programme and course questions"}
]
```

### Create FAQ (Admin)

**POST** `/api/v1/chat/faqs`

```bash
curl -X POST "http://localhost:8090/api/v1/chat/faqs" \
  -H "Authorization: Bearer $JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "question": "How do I reset my password?",
    "answer": "Click the Forgot Password link on the login page...",
    "faqCategoryCode": "ACCOUNT",
    "keywords": "password,reset,forgot,login"
  }'
```

**Response (201 Created):**
```json
{
  "id": 17
}
```

### Update FAQ (Admin)

**PUT** `/api/v1/chat/faqs/{faqId}`

```bash
curl -X PUT "http://localhost:8090/api/v1/chat/faqs/17" \
  -H "Authorization: Bearer $JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "question": "How do I reset my password?",
    "answer": "Updated answer with more detail...",
    "faqCategoryCode": "ACCOUNT",
    "keywords": "password,reset,forgot,login,change"
  }'
```

### Delete FAQ (Admin)

**DELETE** `/api/v1/chat/faqs/{faqId}`

Soft-deletes by setting `isActive='N'`.

```bash
curl -X DELETE "http://localhost:8090/api/v1/chat/faqs/17" \
  -H "Authorization: Bearer $JWT_TOKEN"
```

### List Draft FAQs (Admin)

**GET** `/api/v1/chat/faqs/drafts`

```bash
curl "http://localhost:8090/api/v1/chat/faqs/drafts" \
  -H "Authorization: Bearer $JWT_TOKEN"
```

### Approve Draft FAQ (Admin)

**PUT** `/api/v1/chat/faqs/{faqId}/approve`

Sets `isActive='Y'` on a draft FAQ.

```bash
curl -X PUT "http://localhost:8090/api/v1/chat/faqs/17/approve" \
  -H "Authorization: Bearer $JWT_TOKEN"
```

---

## Escalation Endpoints

### List Escalated Sessions (Admin)

**GET** `/api/v1/chat/escalations`

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| limit | integer | 20 | Maximum results (max 100) |
| offset | integer | 0 | Starting index |

```bash
curl "http://localhost:8090/api/v1/chat/escalations" \
  -H "Authorization: Bearer $JWT_TOKEN"
```

### Resolve Escalated Session (Admin)

**PUT** `/api/v1/chat/escalations/{sessionToken}/resolve`

Only ESCALATED sessions can be resolved. Sets status to CLOSED.

```bash
curl -X PUT "http://localhost:8090/api/v1/chat/escalations/550e8400-e29b-41d4-a716-446655440000/resolve" \
  -H "Authorization: Bearer $JWT_TOKEN"
```

---

## Student Status Endpoint

### Get Student Application Status

**GET** `/api/v1/chat/students/me/status`

Requires `SCOPE_student:read` or `edueasy-edueasy-chatbot.user` role. Student ID extracted from JWT.

```bash
curl "http://localhost:8090/api/v1/chat/students/me/status" \
  -H "Authorization: Bearer $STUDENT_JWT_TOKEN"
```

**Response (200 OK):**
```json
{
  "applicationReference": "EPY-2026-001234",
  "applicationStatus": "Submitted",
  "programmeChoices": [
    {
      "choiceOrder": 1,
      "programmeName": "Bachelor of Science",
      "institutionName": "University of Cape Town",
      "choiceStatus": "Pending"
    }
  ],
  "paymentStatus": "Paid",
  "paymentAmount": 250.00
}
```

---

## Error Responses

### HTTP Status Codes

| Code | Meaning | When |
|------|---------|------|
| 200 | OK | Successful GET, PUT |
| 201 | Created | Successful POST |
| 400 | Bad Request | Malformed JSON, validation failure |
| 401 | Unauthorized | Missing/invalid JWT (admin/student endpoints) |
| 403 | Forbidden | Insufficient permissions |
| 404 | Not Found | Resource doesn't exist |
| 409 | Conflict | Invalid state transition |
| 422 | Unprocessable Entity | Business rule violation |
| 500 | Internal Server Error | Server-side error |

### Error Response Format

```json
{
  "message": "Error summary",
  "details": "Detailed error information",
  "errors": [
    {
      "field": "fieldName",
      "message": "Field-specific error"
    }
  ]
}
```

---

## Data Models

### ChatSessionDTO

| Field | Type | Description |
|-------|------|-------------|
| chatSessionId | integer | Primary key (read-only) |
| sessionToken | string | UUID session identifier (read-only) |
| chatStatusCode | ChatStatusCode | ACTIVE, CLOSED, ESCALATED, TIMEOUT |
| conversationState | ConversationStateCode | Current prompting state |
| isAuthenticated | string | 'Y' or 'N' |
| startedAt | datetime | Session start |
| endedAt | datetime | Session end (if closed) |

### ChatMessageDTO

| Field | Type | Description |
|-------|------|-------------|
| chatMessageId | integer | Primary key (read-only) |
| messageType | MessageTypeCode | USER or BOT |
| content | string | Message text (max 4000 chars) |
| intentDetected | string | Detected intent code |
| confidenceScore | number | 0.0 to 1.0 |
| faqId | integer | Source FAQ if matched |
| sentAt | datetime | Message timestamp |

### FaqDTO

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| faqId | integer | Read-only | Primary key |
| faqCategoryCode | FaqCategoryCode | Yes | GENERAL, APPLICATION, PAYMENT, SPONSORSHIP, ACCOUNT, PROGRAMME |
| question | string | Yes | FAQ question (max 1000 chars) |
| answer | string | Yes | FAQ answer (max 4000 chars) |
| keywords | string | Yes (create) | Comma-separated keywords (max 500 chars) |
| isActive | string | Read-only | 'Y' or 'N' |
| viewCount | integer | Read-only | Times served |

---

## Versioning

The API uses URL path versioning: `/api/v1/...`

Breaking changes will be introduced with a new version (`/api/v2/...`).

---

## Support

For API issues, contact: api-support@edueasy.co.za
