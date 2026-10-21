# Patterns and Pitfalls

## Template: service unit test

```java
package com.example.orders.service;

import com.example.orders.model.Order;
import com.example.orders.repository.OrderRepository;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import java.math.BigDecimal;
import java.util.Optional;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;
import static org.mockito.Mockito.when;

@ExtendWith(MockitoExtension.class)
class ExampleServiceTest {

    @Mock
    private OrderRepository orderRepository;

    @InjectMocks
    private OrderService orderService;

    private Order testOrder;

    @BeforeEach
    void setUp() {
        testOrder = new Order();
        testOrder.setId(1L);
        testOrder.setCustomerName("John Doe");
        testOrder.setProduct("Widget");
        testOrder.setAmount(new BigDecimal("29.99"));
        testOrder.setStatus("PENDING");
    }

    @Test
    void methodName_condition_expectedResult() {
        when(orderRepository.findById(1L)).thenReturn(Optional.of(testOrder));

        Optional<Order> result = orderService.getOrderById(1L);

        assertThat(result).isPresent();
    }
}
```

## Covering state-machine / branching logic

For code like `OrderService.validateStatusTransition` (a private method), test
it **through the public method** (`updateOrderStatus`). Cover each branch:
terminal source status, `CANCELLED` from a non-terminal status, each valid
transition, each invalid transition, and the unknown-status default branch.
Prefer one `@Test` per behavior; use `@ParameterizedTest` + `@CsvSource` only
when many near-identical cases exist and only if the repo already uses it or the
user agrees (it ships with `spring-boot-starter-test` via `junit-jupiter-params`).

## Not-found paths

When production code throws on a missing entity (e.g.
`new RuntimeException("Order not found: " + id)`), assert both the type and the
message with `assertThatThrownBy`. If the production exception is overly
generic, mention it to the user rather than changing it.

## Pitfalls

- **Do not** use `@SpringBootTest` for a unit test; it loads the whole context
  and needs a database. Use `MockitoExtension` (services) or `@WebMvcTest`
  (controllers).
- **Do not** mix `@MockBean` into pure Mockito tests, or `@Mock` into
  `@WebMvcTest` tests.
- **Do not** leave unused `when(...)` stubs; `MockitoExtension` will fail the
  test with `UnnecessaryStubbingException`.
- **Do not** assert on log output or private methods directly.
- **Do not** use `Thread.sleep`, real clocks, random values, or network/DB
  access in unit tests.
- **Do not** edit or reformat existing tests unless the user asks.
- **Do not** weaken an assertion to make a failing test pass; a failing test may
  reveal a real production bug — report it.
