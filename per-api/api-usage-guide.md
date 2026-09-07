# EduEasy Chatbot API Usage Guide

> Consumer integration guide for the epy-chatbot-api

---

## Overview

This guide covers common integration patterns for the EduEasy Chatbot API. The API supports three main use cases:

1. **Chat Sessions** — Anonymous or authenticated conversational chatbot
2. **FAQ Management** — Admin CRUD with draft/approve workflow
3. **Escalation Handling** — Admin resolution of escalated sessions

---

## Quick Start

### 1. Start a Chat Session (No Auth Required)

```bash
# Create anonymous session
RESPONSE=$(curl -s -X POST "http://localhost:8090/api/v1/chat/sessions" \
  -H "Content-Type: application/json")

SESSION_TOKEN=$(echo $RESPONSE | jq -r '.sessionToken')
echo "Session: $SESSION_TOKEN"
```

### 2. Send Messages

```bash
# Send a message and get bot response
curl -s -X POST "http://localhost:8090/api/v1/chat/sessions/$SESSION_TOKEN/messages" \
  -H "Content-Type: application/json" \
  -d '{"message": "How do I apply?"}'
```

### 3. Close Session

```bash
curl -X PUT "http://localhost:8090/api/v1/chat/sessions/$SESSION_TOKEN/close"
```

---

## Common Workflows

### Chat Conversation Flow

```bash
# 1. Create session
SESSION=$(curl -s -X POST "http://localhost:8090/api/v1/chat/sessions" | jq -r '.sessionToken')

# 2. User asks a question → bot responds with FAQ match or prompt
curl -s -X POST "http://localhost:8090/api/v1/chat/sessions/$SESSION/messages" \
  -H "Content-Type: application/json" \
  -d '{"message": "What is my application status?"}'

# 3. Bot prompts for application reference → user provides it
curl -s -X POST "http://localhost:8090/api/v1/chat/sessions/$SESSION/messages" \
  -H "Content-Type: application/json" \
  -d '{"message": "EPY-2026-001234"}'

# 4. Bot responds with status lookup results

# 5. Close session
curl -X PUT "http://localhost:8090/api/v1/chat/sessions/$SESSION/close"
```

### Authenticated Session (Student)

```bash
# Get student JWT token first
TOKEN=$(curl -s -X POST "https://login.microsoftonline.com/{tenant}/oauth2/v2.0/token" \
  -d "client_id=$CLIENT_ID&client_secret=$SECRET&scope=api://$API_ID/.default&grant_type=client_credentials" \
  | jq -r '.access_token')

# Create authenticated session — auto-links to student
curl -s -X POST "http://localhost:8090/api/v1/chat/sessions" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json"
```

### FAQ Admin Workflow

```bash
export TOKEN="your-admin-jwt-token"

# Create FAQ
FAQ_ID=$(curl -s -X POST "http://localhost:8090/api/v1/chat/faqs" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"question":"How to apply?","answer":"Visit the portal...","faqCategoryCode":"APPLICATION","keywords":"apply,application,register"}' \
  | jq -r '.id')

# Review drafts
curl -s "http://localhost:8090/api/v1/chat/faqs/drafts" \
  -H "Authorization: Bearer $TOKEN"

# Approve a draft
curl -X PUT "http://localhost:8090/api/v1/chat/faqs/$FAQ_ID/approve" \
  -H "Authorization: Bearer $TOKEN"

# Update
curl -X PUT "http://localhost:8090/api/v1/chat/faqs/$FAQ_ID" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"question":"How to apply?","answer":"Updated answer...","faqCategoryCode":"APPLICATION","keywords":"apply,application"}'

# Soft-delete
curl -X DELETE "http://localhost:8090/api/v1/chat/faqs/$FAQ_ID" \
  -H "Authorization: Bearer $TOKEN"
```

### Escalation Resolution

```bash
# List escalated sessions
curl -s "http://localhost:8090/api/v1/chat/escalations" \
  -H "Authorization: Bearer $TOKEN"

# Resolve an escalation
curl -X PUT "http://localhost:8090/api/v1/chat/escalations/$SESSION_TOKEN/resolve" \
  -H "Authorization: Bearer $TOKEN"
```

---

## Code Examples

### JavaScript/TypeScript

```typescript
const BASE_URL = 'http://localhost:8090';

// Create session
const sessionRes = await fetch(`${BASE_URL}/api/v1/chat/sessions`, {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' }
});
const { sessionToken } = await sessionRes.json();

// Send message
const msgRes = await fetch(`${BASE_URL}/api/v1/chat/sessions/${sessionToken}/messages`, {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ message: 'How do I apply?' })
});
const { botMessage } = await msgRes.json();
console.log(botMessage.content);

// Close
await fetch(`${BASE_URL}/api/v1/chat/sessions/${sessionToken}/close`, { method: 'PUT' });
```

### Python

```python
import requests

BASE_URL = "http://localhost:8090"

# Create session
session = requests.post(f"{BASE_URL}/api/v1/chat/sessions").json()
token = session["sessionToken"]

# Send message
response = requests.post(
    f"{BASE_URL}/api/v1/chat/sessions/{token}/messages",
    json={"message": "How do I apply?"}
).json()
print(response["botMessage"]["content"])

# Close
requests.put(f"{BASE_URL}/api/v1/chat/sessions/{token}/close")
```

---

## Error Handling

### Common Error Responses

| Status | Meaning | Action |
|--------|---------|--------|
| 400 | Bad request / validation failure | Check request body |
| 401 | Token expired/invalid | Refresh your token |
| 403 | Insufficient permissions | Check your roles |
| 404 | Resource not found | Verify the ID/token exists |
| 409 | Invalid state transition | Check session status |
| 422 | Business rule violation | Review business rules |

---

## Best Practices

1. **Reuse sessions** — Don't create a new session for every message
2. **Handle conversational state** — The bot may prompt for details; send user input as next message
3. **Cache tokens** — Don't request a new OAuth2 token for every API call
4. **Close sessions** — Always close sessions when the user is done
5. **Check bot suggestions** — When confidence is low, bot returns suggestions; present them to the user

---

## Support

For integration support: api-support@edueasy.co.za
