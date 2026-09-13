# Java Test Case Writing Guidelines

## 1. Purpose & Core Philosophy

Ask one question per method:

> "Does this method make a decision, transform data, or talk to something external?"

- **Yes** → it needs a test.
- **No** (it just stores/returns a value) → it doesn't.

## 2. Priority Tiers

| Tier | What It Covers | Test? |
|------|-----------------|-------|
| **P0 – Always test** | Business rules, pricing/discount logic, validation, state transitions, security checks (auth/authz), financial calculations, anything with `if/else`, `switch`, loops, or thrown exceptions | Yes, exhaustively (all branches) |
| **P1 – Test the contract** | Public service/controller methods that orchestrate collaborators (e.g., `OrderService.placeOrder`) | Yes — happy path + each validation failure + verify key collaborator interactions |
| **P2 – Test at the boundary only** | Utility/helper methods reused across the app (validators, formatters, parsers) | Yes, with parameterized tests covering edge cases |
| **P3 – Covered indirectly** | Private methods, package-private helpers called only from an already-tested public method | No separate test — exercised through the public API test |
| **P4 – Skip** | Getters/setters, constructors with no logic, plain DTOs/POJOs, Lombok/IDE-generated `toString()`/`equals()`/`hashCode()`, enums with no behavior, `main()`/bootstrap classes, auto-generated code (MapStruct, Lombok, protobuf) | No |
| **P5 – Integration test instead** | Real DB repositories, real HTTP clients, file I/O, third-party SDK wrappers | Yes, but as integration tests (e.g., Testcontainers), not unit tests with everything mocked |

## 3. Test Structure & Naming Conventions

Use descriptive method names stating **method under test → condition → expected result**, following **Given–When–Then** or **Arrange–Act–Assert**.

```java
@Test
void calculateDiscount_whenCustomerIsPremium_shouldApplyTenPercent() {
    // given
    Customer customer = new Customer(CustomerType.PREMIUM);

    // when
    double discount = discountService.calculateDiscount(customer, 100.0);

    // then
    assertEquals(10.0, discount, 0.001);
}
```

- Avoid generic names like `test1()`, `testMethod()`.
- Follow consistent suffix/prefix conventions (`*Test`, `*Tests`, `should...`, `given...`) (`S3577`).

### One Logical Assertion Focus per Test

Keep each test focused on a single behavior; use multiple assertions only if they validate one logical outcome.

```java
@Test
void createUser_shouldReturnUserWithGeneratedId() {
    User user = userService.createUser("john@test.com");

    assertNotNull(user.getId());
    assertEquals("john@test.com", user.getEmail());
}
```

## 4. Code Coverage Alignment

**Cover all branches, not just happy paths:**

- Positive case
- Negative/exception case
- Boundary/edge case
- Null/empty input case

**Use parameterized tests** to cover multiple inputs without duplicating code:

```java
@ParameterizedTest
@ValueSource(strings = {"", " ", "invalid-email"})
void validateEmail_shouldThrowException_forInvalidInputs(String email) {
    assertThrows(InvalidEmailException.class,
        () -> emailValidator.validate(email));
}
```

## 5. Avoiding Code Smells in Test Code

- **No hardcoded `Thread.sleep()`** in tests. Use `Awaitility` or mocked clocks instead.
- **No `System.out.println`** in tests — use logging or assertion failure messages.
- **No magic numbers** — extract to named constants.
- **Avoid catching generic `Exception`** in test code unless necessary.
- **No duplicated test setup code** — extract to `@BeforeEach` or builder/factory methods.

## 6. Mocking Best Practices (Mockito)

- Mock only external dependencies (DB, APIs, other services), not the class under test.
- Avoid over-mocking/verifying implementation details (leads to brittle tests flagged by maintainability metrics).
- Use `@ExtendWith(MockitoExtension.class)` so mocks are reset/closed properly between tests, avoiding state leakage (test isolation).
- Don't mock value objects/DTOs — construct them directly; mocking data-only classes adds noise without value.

## 7. Test Isolation & Determinism

- Tests must not depend on execution order.
- Tests must not depend on shared mutable static state.
- Avoid real network/database/file I/O in unit tests — use test doubles, or an embedded test database (e.g., H2) for integration tests only.
- Clean up resources in `@AfterEach`/`@AfterAll` to prevent leaks flagged by Sonar reliability checks.
- **Flaky test handling:** a test that fails intermittently must be fixed or quarantined immediately, not silenced with `@RepeatedTest` retries or increased timeouts as a permanent fix.

## 8. Suggested Test Package Structure

```
src/test/java/com/example/app/
 ├── unit/
 │    ├── service/
 │    │    ├── DiscountServiceTest.java
 │    │    └── OrderServiceTest.java
 │    ├── controller/
 │    │    └── DiscountControllerTest.java
 │    └── util/
 │         └── EmailValidatorTest.java
 ├── integration/
 │    ├── repository/
 │    │    └── OrderRepositoryIT.java
 │    ├── client/
 │    │    └── PaymentGatewayClientIT.java
 │    ├── controller/
 │    │    └── DiscountControllerIT.java
 │    └── context/
 │         └── ApplicationContextLoadIT.java
 └── testutils/
      ├── TestDataFactory.java
      ├── CustomerTestDataBuilder.java
      └── OrderTestDataBuilder.java
```

**Naming convention drives tooling, not just folders:**

- `*Test.java` → picked up by **Surefire**, runs in the `test` phase (fast, every commit) — everything under `unit/`
- `*IT.java` → picked up by **Failsafe**, runs in the `integration-test`/`verify` phase (slower, real infra) — everything under `integration/`

Placing a file in `integration/` but naming it `*Test.java` (or vice versa) is a common mistake — the *suffix*, not the folder, is what Surefire/Failsafe actually filter on. Keep both consistent.

`testutils/` holds shared builders/factories used by **both** unit and integration tests — never test-framework-specific code (no stray `@Test` methods here).

## 9. What to Test vs. What to Mock

This is the recurring question when writing a test for *any* method: given its dependencies, what does *this* test need to prove, and what gets proven elsewhere?

### The Rule

> **Mock the collaborator's existence, not its correctness.** Your test proves *your* class uses the collaborator correctly. The collaborator's *own* test proves it computes the right thing.

For every method under test, split its dependencies into two buckets:

| Bucket | Definition | Handling |
|--------|-------------|----------|
| **Collaborators** | Other services/repositories/clients/gateways — anything with its own logic and its own test class | Mock them. Stub only the return values/exceptions needed for *this* scenario. |
| **Data/value objects** | DTOs, domain entities, value objects passed in or returned | Construct them for real (via builders/factories) — never mock a data-only object. |

### What This Method's Test Should Assert

For the method under test itself, walk through:

1. **Its own branching/validation** — every `if/else`, guard clause, null check it contains directly.
2. **How it reacts to each collaborator outcome** — success, each checked exception the collaborator can throw, and (where relevant) an unexpected/runtime exception.
3. **What it passes to each collaborator** — `verify(collaborator).method(expectedArgs)`, using an argument captor if the exact object matters.
4. **What it returns/does with the collaborator's result** — does it transform, wrap, or forward the value correctly?
5. **Order/count of interactions, if the order matters** — e.g., "payment must be charged before the confirmation email is sent" — `InOrder`, `times(1)`, `verifyNoMoreInteractions()` where meaningful.

What it should **not** assert: the internal correctness of the collaborator (that's the collaborator's own test's job), or implementation details of *how* the collaborator does its work.

### Worked Example

```java
public class OrderService {
    private final DiscountService discountService;
    private final PaymentService paymentService;

    public OrderConfirmation placeOrder(Order order) {
        if (order.getItems().isEmpty()) {
            throw new InvalidOrderException("Order must contain at least one item");
        }
        double discount = discountService.calculateDiscount(order.getCustomer(), order.getTotal());
        double finalAmount = order.getTotal() - discount;
        paymentService.processPayment(order.getCustomer(), finalAmount);
        return new OrderConfirmation(order.getId(), finalAmount);
    }
}
```

**`OrderServiceTest`** (this class's responsibility — mock both collaborators):

```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {

    @Mock private DiscountService discountService;
    @Mock private PaymentService paymentService;
    @InjectMocks private OrderService orderService;

    @Test
    void placeOrder_whenItemsEmpty_shouldThrowWithoutCallingCollaborators() {
        Order order = OrderTestDataBuilder.anOrder().withNoItems().build();

        assertThrows(InvalidOrderException.class, () -> orderService.placeOrder(order));

        verifyNoInteractions(discountService, paymentService);
    }

    @Test
    void placeOrder_shouldChargeAmountAfterApplyingDiscount() {
        Order order = OrderTestDataBuilder.anOrder().withTotal(200.0).build();
        when(discountService.calculateDiscount(order.getCustomer(), 200.0)).thenReturn(20.0);

        OrderConfirmation confirmation = orderService.placeOrder(order);

        assertThat(confirmation.getFinalAmount()).isEqualTo(180.0);
        verify(paymentService).processPayment(order.getCustomer(), 180.0);
    }

    @Test
    void placeOrder_whenPaymentFails_shouldPropagateException() {
        Order order = OrderTestDataBuilder.anOrder().withTotal(100.0).build();
        when(discountService.calculateDiscount(any(), anyDouble())).thenReturn(0.0);
        doThrow(new PaymentDeclinedException("card declined"))
            .when(paymentService).processPayment(any(), anyDouble());

        assertThrows(PaymentDeclinedException.class, () -> orderService.placeOrder(order));
    }
}
```

Note what this test does **not** do: it never checks *how* `discountService` arrives at `20.0` — that belongs to `DiscountServiceTest`. `OrderServiceTest` only checks that `OrderService` calls it correctly and reacts correctly to what it returns or throws.

### Quick Reference: Mock vs. Don't Mock

| Situation | Mock It? |
|-----------|----------|
| Another service/component with its own business logic | ✅ Yes |
| Repository/DAO, HTTP client, external SDK | ✅ Yes |
| A collaborator whose *exception* the method under test needs to react to | ✅ Yes — stub `thenThrow(...)` |
| DTO / domain entity / value object passed as an argument | ❌ No — construct it via a builder/factory |
| The class under test itself | ❌ Never — if you're mocking the class you're supposedly testing, the test proves nothing |
| A static utility with no state (e.g., `Math.max`, a pure formatter) | ❌ No — call it directly |
| A clock/time source (`Instant.now()`, `LocalDate.now()`) | ✅ Yes — inject a `Clock` and mock/fix it for determinism |

## 10. Integration Tests

Anything that talks to a **real** external system instead of a mock or in-memory stand-in:

- A real database (embedded test DB like H2, or a real Postgres/MySQL via Testcontainers)
- A real HTTP call to another service (or a sandbox/WireMock endpoint standing in for it)
- Real file I/O, message queues, caches
- A full Spring context wired together for real (`@SpringBootTest`)

### Example — Repository Integration Test

```java
@Testcontainers
@SpringBootTest
class OrderRepositoryIT {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16");

    @DynamicPropertySource
    static void registerDatasource(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired private OrderRepository orderRepository;

    @Test
    void save_shouldPersistOrderAndGenerateId() {
        Order order = OrderTestDataBuilder.anOrder().build();

        Order saved = orderRepository.save(order);

        assertThat(saved.getId()).isNotNull();
        assertThat(orderRepository.findById(saved.getId())).isPresent();
    }
}
```

### What to Actually Write as Integration Tests (Not Everything)

Keep this deliberately small — it's the top of the pyramid, not the bulk of the suite:

| Item | Why It Needs to Be Real, Not Mocked |
|------|----------------------------------------|
| Repository query correctness (custom `@Query`, joins, pagination) | SQL correctness can't be verified against a mock |
| Real HTTP client vs. a sandbox/WireMock endpoint | Verifies the actual request/response contract, serialization, timeouts |
| Full Spring context load (`@SpringBootTest`, no args) | Smoke test that all beans wire up — catches missing `@Bean`, circular deps, misconfigured properties |
| A handful of true end-to-end happy paths | Confidence that the layers actually connect in production-like conditions |

## 11. Async Testing

**The trap:** asserting on the future object itself, or not blocking for the result, lets the test pass before the async code actually runs.

```java
public CompletableFuture<Double> calculateDiscountAsync(Customer customer, double amount) {
    return CompletableFuture.supplyAsync(() -> discountService.calculateDiscount(customer, amount));
}
```

```java
@Test
void calculateDiscountAsync_whenPremiumCustomer_shouldCompleteWithTenPercent() {
    Customer customer = new Customer(CustomerType.PREMIUM);

    CompletableFuture<Double> result = service.calculateDiscountAsync(customer, 100.0);

    // .join() blocks and unwraps — this is what actually waits for the async work
    assertThat(result.join()).isEqualTo(10.0);
}

@Test
void calculateDiscountAsync_whenUnderlyingCallFails_shouldCompleteExceptionally() {
    Customer customer = new Customer(CustomerType.PREMIUM);
    when(discountService.calculateDiscount(any(), anyDouble()))
        .thenThrow(new IllegalArgumentException("bad amount"));

    CompletableFuture<Double> result = service.calculateDiscountAsync(customer, -50.0);

    assertThatThrownBy(result::join)
        .hasCauseInstanceOf(IllegalArgumentException.class);
}
```

For code that polls or eventually settles (not a `Future` you can `.join()` directly — e.g., checking a status flag another thread will set), use **Awaitility** instead of `Thread.sleep()`:

```java
await().atMost(2, SECONDS).untilAsserted(() ->
    assertThat(orderStatusRepository.findById(orderId).getStatus()).isEqualTo(CONFIRMED)
);
```

## 12. Concurrency / Thread Safety

**What needs this kind of test:** shared mutable state accessed by multiple threads — counters, caches, in-memory registries, anything with a race-condition risk.

**The trap:** a single-threaded test of a thread-safety bug will pass every time — the bug only manifests under concurrent access. Don't rely on incidental timing; force concurrent execution deliberately.

```java
public class InMemoryRateLimiter {
    private final AtomicInteger counter = new AtomicInteger(0);
    private final int limit;

    public boolean tryAcquire() {
        return counter.incrementAndGet() <= limit;
    }
}
```

```java
@Test
void tryAcquire_underConcurrentAccess_shouldNeverAllowMoreThanLimit() throws InterruptedException {
    int limit = 10;
    int threads = 50;
    InMemoryRateLimiter rateLimiter = new InMemoryRateLimiter(limit);

    ExecutorService executor = Executors.newFixedThreadPool(threads);
    CountDownLatch readyLatch = new CountDownLatch(threads);
    CountDownLatch startLatch = new CountDownLatch(1);
    AtomicInteger successCount = new AtomicInteger(0);

    for (int i = 0; i < threads; i++) {
        executor.submit(() -> {
            readyLatch.countDown();
            try {
                startLatch.await(); // all threads fire at once, maximizing race exposure
                if (rateLimiter.tryAcquire()) {
                    successCount.incrementAndGet();
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });
    }

    readyLatch.await();      // wait until every thread is ready
    startLatch.countDown();  // release them simultaneously
    executor.shutdown();
    assertThat(executor.awaitTermination(5, SECONDS)).isTrue();

    assertThat(successCount.get()).isEqualTo(limit);
}
```

Key pattern elements — reuse this shape for any thread-safety test:

- **`CountDownLatch` "ready" gate** — ensures all threads are queued up before any starts, instead of relying on thread-pool scheduling luck.
- **A shared `AtomicInteger`/`AtomicBoolean`** to collect results without introducing a second race condition in the test itself.
- **`executor.awaitTermination(...)`** — always assert this returned `true`; otherwise the test can finish and assert before all threads actually completed, silently passing.
- **Run it more than once locally / in CI with `@RepeatedTest(20)`** while stabilizing a new concurrency test — a flaky failure that only shows up 1-in-20 runs is the actual bug, not a false negative to suppress.

## 13. Recommended Tools

| Tool | Purpose |
|------|---------|
| **JUnit 5** | Test framework |
| **Mockito** | Mocking framework |
| **AssertJ** | Fluent assertions (improves readability, reduces assertion-related smells) |
| **JaCoCo** | Coverage report generation (feeds into SonarQube) |
| **SonarLint** | IDE plugin for real-time Sonar rule feedback before commit |
| **Awaitility** | Asynchronous/eventual-consistency assertions instead of `Thread.sleep()` |
| **Testcontainers** | P5 integration tests against real DB/queue/HTTP dependencies |
