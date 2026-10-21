# Conventions (derived from existing tests in `order-service/src/test/`)

These reflect `OrderServiceTest` and `OrderControllerTest` at the time this mode
was written. **Re-read the existing tests every time** — if they have changed,
the repo wins over this file.

## Stack

- Java 17, Spring Boot 3.2.0, Maven (`spring-boot-starter-test` brings JUnit 5,
  Mockito, AssertJ, Spring Test). No extra test dependencies should be needed.

## Naming

- Test class: `<ClassUnderTest>Test`, package-private (`class OrderServiceTest`),
  in the same package as the class under test.
- Test method: `methodName_condition_expectedResult`, camelCase segments joined
  by underscores, e.g. `updateOrderStatus_invalidTransition_throwsException`,
  `getOrderById_nonExistingId_returnsEmpty`. Very simple cases may use
  `methodName_expectedResult` (e.g. `getAllOrders_returnsAllOrders`).
- No `@DisplayName`; the method name is the description.
- Test methods are package-private `void` (no `public`).

## Service-layer (pure unit) tests

- `@ExtendWith(MockitoExtension.class)` on the class — no Spring context.
- Collaborators: `@Mock private X x;` ; subject: `@InjectMocks private Y y;`
  (constructor injection is used by the production classes).
- Shared fixtures are built in a `@BeforeEach void setUp()` and stored in a
  private field (e.g. `testOrder`). Tests needing a different state mutate the
  fixture or build a local object.
- Stubbing: `when(mock.call(...)).thenReturn(...)` with static imports from
  `org.mockito.Mockito` and `org.mockito.ArgumentMatchers.any`.
- Interaction checks: `verify(mock).call(...)` — used only where the behavior
  *is* the interaction (e.g. `deleteOrder_callsRepository`).
- Strict stubs are on by default with `MockitoExtension`: do not leave unused
  stubs.

## Assertions

- **AssertJ only**: `import static org.assertj.core.api.Assertions.assertThat;`
  and `assertThatThrownBy`. Do not use JUnit `Assertions.*` or Hamcrest.
- Exceptions:
  ```java
  assertThatThrownBy(() -> service.doIt(...))
          .isInstanceOf(IllegalStateException.class)
          .hasMessageContaining("expected text");
  ```
- Optionals: `assertThat(result).isPresent()` / `.isEmpty()`.
- Collections: `assertThat(list).hasSize(n)`.

## Controller-layer tests

- `@WebMvcTest(XController.class)` with `@Autowired MockMvc` and
  `@Autowired ObjectMapper`; the service is a `@MockBean`.
- Requests via `MockMvcRequestBuilders.*`; assertions via
  `MockMvcResultMatchers.*` (`status()`, `jsonPath()`) — this layer's existing
  tests use MockMvc matchers rather than AssertJ. Keep that style in controller
  tests.
- Methods declare `throws Exception`.

## Fixtures

- Entities (`Order`) are created with the no-arg constructor and setters
  (`setId`, `setCustomerName`, `setProduct`, `setAmount`, `setStatus`).
- Money values use `new BigDecimal("29.99")` (string constructor, never double).
- Typical sample data: customer `"John Doe"`, product `"Widget"`, status
  `"PENDING"`. Reuse similar values for consistency.
- Test properties live in `src/test/resources/application-test.properties`.

## Formatting

- 4-space indentation; continuation lines for chained calls indented 8 spaces.
- Imports: project, then third-party, then `java.*`, then `static` imports last.
  Avoid unused imports.
