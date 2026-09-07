# EduEasy Chatbot API Architecture

> Technical architecture overview for epy-chatbot-api

---

## Technology Stack

| Component | Technology | Version |
|-----------|------------|---------|
| Framework | Spring Boot | 3.4.x |
| Language | Java | 21 |
| Build Tool | Maven | 3.9+ |
| API Spec | OpenAPI | 3.0.3 |
| Security | OAuth2 JWT | Azure AD |
| Messaging | RabbitMQ | via common-starter-communication |
| Database | Oracle | EDU_PAY schema |
| Parent POM | common-starter-parent | 2.0.30 |

---

## Layer Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      PRESENTATION LAYER                      │
│  ┌───────────────┐ ┌──────────────┐ ┌────────────────────┐ │
│  │ ChatController │ │FaqController │ │EscalationController│ │
│  │               │ │              │ │StudentStatusCtrl   │ │
│  └───────┬───────┘ └──────┬───────┘ └────────┬───────────┘ │
└──────────┼────────────────┼───────────────────┼─────────────┘
           │                │                   │
           ▼                ▼                   ▼
┌─────────────────────────────────────────────────────────────┐
│                       BUSINESS LAYER                         │
│  ┌────────────────┐ ┌──────────────┐ ┌──────────────────┐  │
│  │ ChatbotService │ │  FaqService  │ │ChatEscalation    │  │
│  │                │ │              │ │Service            │  │
│  └────────┬───────┘ └──────┬───────┘ └────────┬─────────┘  │
│  ┌────────────────┐ ┌──────────────┐ ┌──────────────────┐  │
│  │ChatSession     │ │ChatMessage   │ │IntentDetection   │  │
│  │Service         │ │Service       │ │Service            │  │
│  └────────┬───────┘ └──────┬───────┘ └──────────────────┘  │
│  ┌────────────────┐ ┌──────────────┐                        │
│  │StudentStatus   │ │Communication │                        │
│  │Service         │ │Service       │                        │
│  └────────┬───────┘ └──────┬───────┘                        │
└───────────┼────────────────┼────────────────────────────────┘
            │                │
            ▼                ▼
┌─────────────────────────────────────────────────────────────┐
│                    PERSISTENCE / INTEGRATION                 │
│  ┌──────────────────┐  ┌──────────────────────────────────┐ │
│  │ JPA Repositories │  │    RabbitMQ (Email Events)       │ │
│  │ - FaqRepository  │  │    via CommunicationService      │ │
│  │ - ChatSession    │  │    → edueasy-comms-api              │ │
│  │ - ChatMessage    │  └──────────────────────────────────┘ │
│  │ - Student (R/O)  │                                       │
│  │ - Application    │                                       │
│  │ - Payment (R/O)  │                                       │
│  └──────────────────┘                                       │
└─────────────────────────────────────────────────────────────┘
```

---

## Component Responsibilities

### Controller Layer
- Accept HTTP requests and validate request format
- Delegate to Service layer
- Return appropriate HTTP responses
- `@PreAuthorize` on admin endpoints
- **No business logic**

### Service Layer
- Implement business rules (intent detection, FAQ matching, escalation triggers)
- Coordinate session lifecycle and conversation state management
- Publish email events via `CommunicationService` → RabbitMQ
- Handle exceptions with `ExceptionUtil`
- Concrete `@Service` classes (no interface pattern)

### Transformer Layer
- Convert between DTOs and Entities
- Handle null safety and enum mapping
- **Never modify audit fields** (Oracle DB triggers manage them)

### Repository Layer
- JPA CRUD operations via Spring Data
- Custom queries for FAQ keyword search
- Read-only access to cross-schema EDU_PAY tables
- `@Immutable` entities for cross-schema reads

### Communication Layer
- `ChatEscalationEmailTemplate` — escalation notification to admin
- `RecurringFaqAlertEmailTemplate` — recurring unanswered question alert
- Published via `CommunicationService` → `CommonPublisherUtil.send()` → RabbitMQ

---

## Security Architecture

```
┌──────────────┐    ┌──────────────┐    ┌──────────────────────┐
│   Client     │───▶│  Azure AD    │───▶│  EduEasy Chatbot API │
│  (Browser/   │    │  (OAuth2)    │    │                      │
│   Mobile)    │    └──────────────┘    │  SecurityConfig:     │
└──────────────┘                        │  - Public: sessions, │
                                        │    FAQs read         │
       ┌────────────────────────────────│  - Admin: FAQs CRUD, │
       │ Guest: No JWT needed           │    escalations       │
       │ Student: JWT with student:read │  - Student: me/status│
       │ Admin: JWT with admin role     └──────────────────────┘
       └──────────────────────────────────────────────────────┘
```

### Authentication Flow
1. **Guest users** access chat and FAQ read endpoints without authentication
2. **Students** request JWT from Azure AD with `student:read` scope
3. **Admins** request JWT with `edueasy-edueasy-chatbot.admin` role
4. API validates JWT signature and claims via `JwtAuthenticationConverter`
5. `SecurityConfig` enforces role-based access per endpoint

---

## Data Flow

### Chat Message Flow
```
POST /api/v1/chat/sessions/{token}/messages
        │
        ▼
┌─────────────────┐
│  ChatController  │ ← Validates message format
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ ChatbotService   │ ← Orchestrates message processing
└────────┬────────┘
         │
    ┌────┼────────────────────────┐
    ▼    ▼                        ▼
┌────────────┐ ┌──────────────┐ ┌─────────────────┐
│IntentDetect│ │ChatMessage   │ │ChatSession      │
│Service     │ │Service       │ │Service           │
│            │ │(persist msg) │ │(update state)    │
└────┬───────┘ └──────────────┘ └─────────────────┘
     │
     ├── FAQ match (confidence ≥ 0.7) → Return FAQ answer
     ├── Low confidence → Return suggestions
     ├── Status intent → Prompt for reference / lookup
     └── Escalation → Publish email via RabbitMQ
```

### Escalation Flow
```
Escalation Trigger (user request / low confidence / sensitive topic)
        │
        ▼
┌────────────────────┐
│ChatEscalationService│ ← Sets session status=ESCALATED
└────────┬───────────┘
         │
         ▼
┌────────────────────────┐
│CommunicationService    │ ← Builds email event
│→ ChatEscalationEmail   │
│  Template              │
└────────┬───────────────┘
         │
         ▼
┌────────────────────────┐
│CommonPublisherUtil.send│ ← Publishes to RabbitMQ
│→ SEND_MAIL routing key │
└────────┬───────────────┘
         │
         ▼
┌────────────────────────┐
│edueasy-comms-api          │ ← Consumes and sends email
└────────────────────────┘
```

---

## Event Architecture

```
┌──────────────────────┐    ┌─────────────────┐    ┌──────────────────┐
│  CommunicationService│───▶│   RabbitMQ      │───▶│  edueasy-comms-api  │
│  (Email Templates)   │    │   SEND_MAIL     │    │  (Email Sender)  │
└──────────────────────┘    └─────────────────┘    └──────────────────┘
```

### Events Published

| Event | Routing Key | Trigger |
|-------|-------------|---------|
| ChatEscalationEmailTemplate | SEND_MAIL | Session escalated to admin |
| RecurringFaqAlertEmailTemplate | SEND_MAIL | Same unmatched intent 3+ times in 7 days |

---

## Cross-Schema Database Design

```
┌─── Chatbot Schema (Read-Write) ───┐    ┌─── EDU_PAY Schema (Read-Only) ───┐
│ EPY_FAQ                            │    │ EPY_STUDENT                       │
│ EPY_FAQ_CATEGORY                   │    │ EPY_APPLICATION                   │
│ EPY_CHAT_SESSION                   │    │ EPY_APPLICATION_CHOICE            │
│ EPY_CHAT_MESSAGE                   │    │ EPY_PAYMENT                       │
│ EPY_CHAT_STATUS                    │    │ EPY_INSTITUTION                   │
└────────────────────────────────────┘    │ EPY_PROGRAMME                     │
                                          │ EPY_SPONSORSHIP                   │
                                          │ EPY_APPLICATION_STATUS            │
                                          │ EPY_PAYMENT_STATUS                │
                                          │ EPY_PAYMENT_METHOD                │
                                          │ EPY_PLAN_TYPE                     │
                                          └────────────────────────────────────┘
```

---

## Package Structure

```
za.co.edueasy.api.chatbot/
├── Application.java
├── config/
│   ├── JacksonConfig.java
│   ├── OpenApiConfig.java
│   ├── SecurityConfig.java
│   └── WebConfig.java
├── controller/
│   ├── ChatController.java
│   ├── FaqController.java
│   ├── EscalationController.java
│   └── StudentStatusController.java
├── dto/
│   ├── ChatSessionDTO.java
│   ├── ChatMessageDTO.java
│   ├── FaqDTO.java
│   ├── StudentStatusDTO.java
│   ├── SendMessageRequestDTO.java
│   ├── SendMessageResponseDTO.java
│   └── code/
│       ├── ChatStatusCode.java
│       ├── ConversationStateCode.java
│       ├── FaqCategoryCode.java
│       └── MessageTypeCode.java
├── entity/
│   ├── FaqEntity.java
│   ├── FaqCategoryEntity.java
│   ├── ChatSessionEntity.java
│   ├── ChatMessageEntity.java
│   ├── ChatStatusEntity.java
│   ├── StudentEntity.java (read-only)
│   ├── ApplicationEntity.java (read-only)
│   ├── ApplicationChoiceEntity.java (read-only)
│   ├── PaymentEntity.java (read-only)
│   ├── InstitutionEntity.java (read-only)
│   ├── ProgrammeEntity.java (read-only)
│   └── SponsorshipEntity.java (read-only)
├── repository/
│   ├── FaqRepository.java
│   ├── ChatSessionRepository.java
│   ├── ChatMessageRepository.java
│   └── (read-only repositories)
├── service/
│   ├── ChatbotService.java
│   ├── ChatSessionService.java
│   ├── ChatMessageService.java
│   ├── FaqService.java
│   ├── IntentDetectionService.java
│   ├── ChatEscalationService.java
│   └── StudentStatusService.java
├── transformer/
│   ├── FaqTransformer.java
│   ├── ChatSessionTransformer.java
│   ├── ChatMessageTransformer.java
│   └── StudentStatusTransformer.java
├── communication/
│   ├── ChatEscalationEmailTemplate.java
│   └── RecurringFaqAlertEmailTemplate.java
└── util/
    └── (utility classes)
```

---

## Configuration

See [setup-and-configuration-guide.md](setup-and-configuration-guide.md) for environment-specific settings.
