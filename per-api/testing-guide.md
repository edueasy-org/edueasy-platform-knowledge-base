# EduEasy Chatbot API Testing Guide

> Test execution, coverage, and best practices

---

## Test Structure

```
src/test/java/za/co/edueasy/api/chatbot/
├── controller/
│   ├── ChatControllerTest.java
│   ├── FaqControllerTest.java
│   ├── EscalationControllerTest.java
│   └── StudentStatusControllerTest.java
├── service/
│   ├── ChatbotServiceTest.java
│   ├── ChatSessionServiceTest.java
│   ├── ChatMessageServiceTest.java
│   ├── FaqServiceTest.java
│   ├── IntentDetectionServiceTest.java
│   ├── ChatEscalationServiceTest.java
│   └── StudentStatusServiceTest.java
├── transformer/
│   ├── FaqTransformerTest.java
│   ├── ChatSessionTransformerTest.java
│   ├── ChatMessageTransformerTest.java
│   └── StudentStatusTransformerTest.java
├── dto/
│   ├── FaqDTOValidationTest.java
│   └── ChatMessageDTOValidationTest.java
├── communication/
│   ├── ChatEscalationEmailTemplateTest.java
│   └── RecurringFaqAlertEmailTemplateTest.java
├── data/
│   └── (enum tests)
└── integration/
    ├── IntegrationTestSecurityConfig.java
    ├── ChatSecurityIntegrationTest.java
    ├── FaqSecurityIntegrationTest.java
    ├── EscalationSecurityIntegrationTest.java
    ├── StudentStatusSecurityIntegrationTest.java
    └── InputSecurityIntegrationTest.java
```

---

## Running Tests

### All Unit Tests

```bash
mvn test
```

### All Tests (Unit + Integration)

```bash
mvn verify
```

### Integration Tests Only

```bash
mvn verify -DskipUTs=true
```

### Specific Test Class

```bash
mvn test -Dtest=ChatControllerTest
```

### Specific Test Method

```bash
mvn test -Dtest=ChatControllerTest#whenCreateSession_thenReturns201
```

### With Coverage Report

```bash
mvn test jacoco:report
```

---

## Test Categories

| Category | Count | Run Command | Location |
|----------|-------|-------------|----------|
| Unit Tests | 140 | `mvn test` | `**/controller/`, `**/service/`, `**/transformer/`, `**/dto/`, `**/communication/`, `**/data/` |
| Integration Tests | 35 | `mvn verify -DskipUTs=true` | `**/integration/` |
| **Total** | **175** | `mvn clean verify` | All |

### Surefire / Failsafe Split

- **Surefire** (unit tests): Excludes `*IntegrationTest.java`, `*SecurityTest.java`, `*E2ETest.java`
- **Failsafe** (integration tests): Includes `*IntegrationTest.java`, `*SecurityTest.java`

---

## Coverage

### Generate Report

```bash
mvn test jacoco:report
```

### View Report

Open `target/site/jacoco/index.html`

### Coverage Targets

| Layer | Target |
|-------|--------|
| Controller | 90% |
| Service | 85% |
| Transformer | 95% |
| DTO | 70% |
| **Overall** | **80%** |

---

## Test Naming Convention

### Format
```
[CHAT-###] Should {expected behavior} when {condition}
```

### Prefixes

| Category | Prefix | Range |
|----------|--------|-------|
| Controller Happy | CHAT-0XX | 001-099 |
| Service Happy | CHAT-1XX | 100-199 |
| Transformer | CHAT-2XX | 200-299 |
| Enums | CHAT-3XX | 300-399 |
| DTO Validation | CHAT-4XX | 400-499 |
| Controller Negative | CHATN-0XX | 001-099 |
| Integration | CHATI-0XX | 001-099 |
| Authentication | CHATAI-0XX | 001-099 |
| Authorization | CHATAuI-0XX | 001-099 |

---

## Test Patterns

### Arrange-Act-Assert

```java
@Test
@DisplayName("[CHAT-001] Should create session when valid request")
void whenCreateSession_thenReturns201() {
    // Arrange
    ChatSessionDTO dto = ChatSessionTestFixture.validDTO();
    when(chatSessionService.createSession(any())).thenReturn(dto);

    // Act
    ResponseEntity<ChatSessionDTO> result = chatController.createSession(null);

    // Assert
    assertThat(result.getStatusCode()).isEqualTo(HttpStatus.CREATED);
    verify(chatSessionService).createSession(any());
}
```

### Integration Test Pattern

```java
@WebMvcTest(FaqController.class)
@Import({SecurityConfig.class, IntegrationTestSecurityConfig.class})
@ActiveProfiles("test")
class FaqSecurityIntegrationTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private FaqService faqService;

    @Test
    @DisplayName("[CHATI-001] Should return 401 for unauthenticated admin request")
    void whenNoToken_thenReturns401() throws Exception {
        mockMvc.perform(post("/api/v1/chat/faqs"))
            .andExpect(status().isUnauthorized());
    }

    @Test
    @DisplayName("[CHATI-002] Should allow admin access with correct role")
    void whenAdminRole_thenReturns200() throws Exception {
        mockMvc.perform(get("/api/v1/chat/faqs/drafts")
                .with(jwt().authorities(new SimpleGrantedAuthority("ROLE_edueasy-edueasy-chatbot.admin"))))
            .andExpect(status().isOk());
    }
}
```

---

## E2E Tests

### Location

```
e2e-tests/
├── tests/
│   ├── smoke/
│   │   └── api-health.spec.ts
│   ├── chat/
│   │   └── chat-session.spec.ts
│   ├── faq/
│   │   └── faq-crud.spec.ts
│   ├── escalation/
│   │   └── escalation-admin.spec.ts
│   └── security/
│       └── auth-security.spec.ts
├── api-clients/
│   ├── BaseApiClient.ts
│   ├── ChatApiClient.ts
│   ├── FaqApiClient.ts
│   └── EscalationApiClient.ts
├── fixtures/
│   └── chatbot-fixtures.ts
└── helpers/
    ├── auth-helper.ts
    ├── api-debug-helper.ts
    ├── api-matchers.ts
    └── test-cleanup-helper.ts
```

### Running E2E Tests

```bash
cd e2e-tests

# Install dependencies
npm install

# Run against local
npm test

# Run against specific environment
npm run test:dev
npm run test:test
npm run test:staging

# Run smoke tests only
npm run test:smoke

# View report
npm run report
```

---

## Common Issues

| Issue | Solution |
|-------|----------|
| Mock not returning value | Check `when()` matcher matches actual call |
| NullPointerException | Ensure `@Mock` annotations and `@ExtendWith(MockitoExtension.class)` |
| 401 in integration test | Use `.with(jwt().authorities(...))` with `SimpleGrantedAuthority` |
| 403 in integration test | Use `ROLE_` prefix: `new SimpleGrantedAuthority("ROLE_edueasy-edueasy-chatbot.admin")` |
| Coverage below target | Check for untested branches in jacoco report |
| Stale failsafe results | Use `mvn clean verify` instead of `mvn verify` |

---

## Related Documentation

- [API Specification](api-specification.md)
- [Test Cases Specification](test-cases-specification.md)
