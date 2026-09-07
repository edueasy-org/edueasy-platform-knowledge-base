# EduEasy Chatbot API Test Cases Specification

> Detailed test scenarios and expected behaviors

---

## Test Categories

| Category | Description | Priority |
|----------|-------------|----------|
| Happy Path | Normal operations succeed | P0 |
| Validation | Input validation errors | P0 |
| Security | Authentication/Authorization | P0 |
| Error Handling | Service errors, edge cases | P1 |
| Integration | Security integration tests | P1 |

---

## Unit Tests

### Controller Tests — Chat

| ID | Scenario | Expected Result |
|----|----------|-----------------|
| CHAT-001 | Create anonymous chat session | 201 Created, sessionToken returned |
| CHAT-002 | Create authenticated chat session | 201 Created, isAuthenticated='Y' |
| CHAT-003 | Get chat session by token | 200 OK, session details |
| CHAT-004 | Close active session | 200 OK, status=CLOSED |
| CHAT-005 | Send message to active session | 200 OK, userMessage + botMessage |
| CHAT-006 | Get message history | 200 OK, ordered message list |

### Controller Tests — FAQ

| ID | Scenario | Expected Result |
|----|----------|-----------------|
| CHAT-010 | List active FAQs | 200 OK, list of active FAQs |
| CHAT-011 | List FAQs by category | 200 OK, filtered list |
| CHAT-012 | Get FAQ by ID | 200 OK, FAQ details |
| CHAT-013 | List FAQ categories | 200 OK, category list |
| CHAT-014 | Create FAQ (admin) | 201 Created, ID returned |
| CHAT-015 | Update FAQ (admin) | 200 OK, updated FAQ |
| CHAT-016 | Delete FAQ (admin) | 200 OK, soft-deleted |
| CHAT-017 | List draft FAQs (admin) | 200 OK, draft list |
| CHAT-018 | Approve draft FAQ (admin) | 200 OK, isActive='Y' |

### Controller Tests — Escalation

| ID | Scenario | Expected Result |
|----|----------|-----------------|
| CHAT-020 | List escalated sessions | 200 OK, escalated list |
| CHAT-021 | Resolve escalated session | 200 OK, status=CLOSED |

### Controller Tests — Student Status

| ID | Scenario | Expected Result |
|----|----------|-----------------|
| CHAT-025 | Get student application status | 200 OK, status summary |

### Controller Negative Tests

| ID | Scenario | Expected Result |
|----|----------|-----------------|
| CHATN-001 | Close already closed session | 400/409/422 |
| CHATN-002 | Send empty message | 400 Bad Request |
| CHATN-003 | Send oversized message (> 4000 chars) | 400 Bad Request |
| CHATN-004 | Get non-existent session | 404 Not Found |
| CHATN-005 | Get non-existent FAQ | 404 Not Found |
| CHATN-006 | Delete non-existent FAQ | 404 Not Found |
| CHATN-007 | Approve already active FAQ | 400/422 |
| CHATN-008 | Resolve non-escalated session | 400/422 |
| CHATN-009 | Send message to closed session | 400/409 |
| CHATN-010 | Message with XSS script tags | 400 or sanitized |

### Service Tests — ChatbotService

| ID | Scenario | Expected Result |
|----|----------|-----------------|
| CHAT-101 | Process greeting message | Welcome response returned |
| CHAT-102 | Process FAQ match (high confidence) | FAQ answer returned, viewCount incremented |
| CHAT-103 | Process low confidence match | Suggestions returned |
| CHAT-104 | Process status check (guest) | Prompt for application reference |
| CHAT-105 | Process status check (authenticated) | Auto-lookup by student ID |
| CHAT-106 | Process escalation request | Session escalated, email event published |

### Service Tests — IntentDetectionService

| ID | Scenario | Expected Result |
|----|----------|-----------------|
| CHAT-110 | Detect CHECK_STATUS intent | Intent matched with confidence |
| CHAT-111 | Detect CHECK_PAYMENT intent | Intent matched |
| CHAT-112 | Detect ESCALATE intent | Intent matched |
| CHAT-113 | Detect GREETING intent | Intent matched |
| CHAT-114 | Detect GOODBYE intent | Intent matched |
| CHAT-115 | Match FAQ keywords | Best FAQ returned with score |
| CHAT-116 | No match found | Empty result |

### Service Tests — ChatSessionService

| ID | Scenario | Expected Result |
|----|----------|-----------------|
| CHAT-120 | Create session generates UUID token | Token generated, status=ACTIVE |
| CHAT-121 | Close session updates status | Status=CLOSED, endedAt set |
| CHAT-122 | Invalid state transition rejected | 422 thrown |

### Service Tests — FaqService

| ID | Scenario | Expected Result |
|----|----------|-----------------|
| CHAT-130 | Create FAQ sets isActive='Y' | FAQ persisted |
| CHAT-131 | Update FAQ modifies fields | Updated FAQ returned |
| CHAT-132 | Delete FAQ sets isActive='N' | Soft-deleted |
| CHAT-133 | Approve draft sets isActive='Y' | FAQ activated |
| CHAT-134 | List active excludes drafts | Only isActive='Y' returned |

### Service Tests — ChatEscalationService

| ID | Scenario | Expected Result |
|----|----------|-----------------|
| CHAT-140 | Escalate session | Status=ESCALATED, email published |
| CHAT-141 | Auto-escalate after 3 low-confidence messages | Auto-escalation triggered |
| CHAT-142 | Sensitive topic escalation | Priority=High escalation |

### Transformer Tests

| ID | Scenario | Expected Result |
|----|----------|-----------------|
| CHAT-201 | FaqTransformer toDTO | All fields mapped |
| CHAT-202 | FaqTransformer toDTO with null | Returns null |
| CHAT-203 | FaqTransformer toEntity | All fields mapped |
| CHAT-204 | ChatSessionTransformer toDTO | All fields mapped |
| CHAT-205 | ChatMessageTransformer toDTO | All fields mapped |
| CHAT-206 | StudentStatusTransformer toDTO | All fields mapped, PII excluded |

### DTO Validation Tests

| ID | Scenario | Expected Result |
|----|----------|-----------------|
| CHAT-401 | Valid FaqDTO for create | No violations |
| CHAT-402 | Valid FaqDTO for update | No violations |
| CHAT-403 | Missing question (create) | Violation on question |
| CHAT-404 | Missing answer (create) | Violation on answer |
| CHAT-405 | Question exceeds 1000 chars | Violation |
| CHAT-406 | Answer exceeds 4000 chars | Violation |
| CHAT-407 | Keywords exceeds 500 chars | Violation |
| CHAT-408 | Invalid faqCategoryCode | Violation |

### Communication Tests

| ID | Scenario | Expected Result |
|----|----------|-----------------|
| CHAT-501 | ChatEscalationEmailTemplate builds correctly | Valid EventDTO |
| CHAT-502 | RecurringFaqAlertEmailTemplate builds correctly | Valid EventDTO |

---

## Integration Tests

### Security Integration Tests

| ID | Scenario | Expected Result |
|----|----------|-----------------|
| CHATI-001 | Chat session create without auth | 201 (public) |
| CHATI-002 | FAQ list without auth | 200 (public) |
| CHATI-003 | FAQ categories without auth | 200 (public) |
| CHATI-004 | FAQ create without auth | 401 |
| CHATI-005 | FAQ create with admin role | 201 |
| CHATI-006 | Escalation list without auth | 401 |
| CHATI-007 | Escalation list with admin role | 200 |
| CHATI-008 | Student status without auth | 401 |
| CHATI-009 | Student status with student scope | 200 |

### Input Security Tests

| ID | Scenario | Expected Result |
|----|----------|-----------------|
| CHATI-020 | XSS in chat message | 400 or sanitized |
| CHATI-021 | XSS in FAQ question | 400 or sanitized |
| CHATI-022 | Invalid JSON body | 400 Bad Request |
| CHATI-023 | Control characters in message | 400 or sanitized |
| CHATI-024 | Oversized request body | 400 Bad Request |

---

## E2E Tests

### Smoke Tests

| ID | Scenario | Expected Result |
|----|----------|-----------------|
| CHAT-SMOKE-001 | Actuator health endpoint | 200, status UP |
| CHAT-SMOKE-002 | Public FAQ endpoint accessible | 200 |
| CHAT-SMOKE-003 | Public categories endpoint accessible | 200 |
| CHAT-SMOKE-004 | Chat session creation | 201 |
| CHAT-SMOKE-005 | Admin endpoint requires auth | 401 |

### Chat Session E2E

| ID | Scenario | Expected Result |
|----|----------|-----------------|
| CHAT-E2E-001 | Create session → send messages → close | Full lifecycle |
| CHAT-E2E-002 | Greeting message returns bot response | Bot responds |
| CHAT-E2E-003 | Status check triggers intent detection | Intent detected |
| CHAT-E2E-004 | Message history contains all messages | History complete |
| CHAT-E2E-005 | Empty message returns 400 | Validation error |
| CHAT-E2E-006 | XSS message handled safely | Sanitized or rejected |

### FAQ E2E

| ID | Scenario | Expected Result |
|----|----------|-----------------|
| CHAT-E2E-010 | Admin creates FAQ | 201, FAQ persisted |
| CHAT-E2E-011 | Admin updates FAQ | 200, changes applied |
| CHAT-E2E-012 | Admin deletes FAQ | 200/204, soft-deleted |
| CHAT-E2E-013 | Admin approves draft | 200, isActive='Y' |
| CHAT-E2E-014 | Public reads FAQ list | 200, active FAQs only |

### Escalation E2E

| ID | Scenario | Expected Result |
|----|----------|-----------------|
| CHAT-E2E-020 | Admin lists escalations | 200, escalated list |
| CHAT-E2E-021 | Admin resolves escalation | 200, status=CLOSED |

### Security E2E

| ID | Scenario | Expected Result |
|----|----------|-----------------|
| CHAT-E2E-030 | Admin endpoints return 401 without token | Unauthorized |
| CHAT-E2E-031 | Admin endpoints return 200 with valid token | Authorized |
| CHAT-E2E-032 | Invalid token returns 401 | Rejected |

---

## Coverage Requirements

| Component | Minimum Coverage |
|-----------|------------------|
| Controller | 90% |
| Service | 85% |
| Transformer | 95% |
| DTO | 70% |
| **Overall** | **80%** |
