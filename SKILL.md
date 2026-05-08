---
name: quarkus-testing
description: >
  Use when writing or modifying tests in a Quarkus project — using @QuarkusTest, @QuarkusIntegrationTest,
  REST Assured, @InjectMock, QuarkusMock.installMockForType, QuarkusMock.installMockForInstance, Dev Services,
  or Continuous Testing. Trigger when naming an integration test class, deciding between unit and integration
  tests, mocking CDI beans, deciding whether different test methods need different mock behaviors, or setting
  up PostgreSQL for tests. Excludes pure-domain tests (aggregates and value objects without Quarkus) — those
  are plain JUnit tests with no framework.
---

# Quarkus Testing Skill

Conventions for tests that involve Quarkus — `@QuarkusTest`, the integration-test companion pattern, REST
Assured idioms, and CDI bean mocking. The goal: tests that are fast, deterministic, and unambiguous about what
they're actually exercising.

---

## When to Use

- Writing or editing a test class annotated with `@QuarkusTest` or `@QuarkusIntegrationTest`.
- Naming a new IT class — use the `*IT extends *Test` companion pattern.
- Deciding whether to use `@InjectMock`, `QuarkusMock.installMockForType`, or `QuarkusMock.installMockForInstance`.
- Mocking a CDI bean (application service, repository).
- Hitting an endpoint with REST Assured.
- Reviewing a test that mocks a `Service` and names the variable `mockRepository`, or that uses both
  `@InjectMock` and `QuarkusMock.installMockForType` for the same bean.

**Out of scope**: pure-JVM aggregate / value-object tests (no Quarkus, just JUnit + assertions — those are
plain unit tests), end-to-end tests against a deployed environment, performance tests.

---

## Core Rules

1. **Use `@QuarkusTest` for JVM tests, `@QuarkusIntegrationTest` for packaged-mode tests.** Quarkus boots once
   per test profile; tests share the same CDI container.
2. **Use the companion pattern for integration tests:** `class FooIT extends FooTest {}`. The IT class adds
   `@QuarkusIntegrationTest` and inherits every test method, so the same suite runs against both the JVM build
   and the packaged artifact. Match this everywhere.
3. **`@InjectMock` is always the default.** Declare `@InjectMock OrderService orders;` and stub in `@BeforeEach`
   with `Mockito.when(...)`. The mock is class-scoped and visible to every test method. Always start here.
   Escalate to `QuarkusMock.installMockForType` only for the specific cases in Rule 4.
4. **Use `QuarkusMock.installMockForType` only for these three cases.** It replaces the bean globally for the
   test's duration; reach for it when `@InjectMock` can't do the job.
   - **(a) Programmatic control.** You need to construct the mock or fake yourself in a setup method — typically
     a hand-rolled stateful fake (a real class, not a Mockito-generated mock) whose construction or wiring is
     too complex for field injection. Install in `@BeforeAll` for the class lifetime, or `@BeforeEach` per test.
     For a non-static `@BeforeAll` method, the test class needs `@TestInstance(Lifecycle.PER_CLASS)`.
   - **(b) Normal-scoped bean replacement.** The bean is `@ApplicationScoped` or `@RequestScoped` and the mock
     must be visible to *other* beans that inject the same type — not just to the field in your test class.
     `installMockForType` swaps the delegate globally so every injection point sees the mock.
   - **(c) Dynamic mocking within a test.** You need to change the mock implementation mid-execution.
     Pair with `QuarkusMock.installMockForInstance` (Rule 5).
5. **`QuarkusMock.installMockForInstance` is only used in conjunction with `installMockForType`.** When an
   individual test method needs to override the already-installed type mock, build a per-test mock and call
   `QuarkusMock.installMockForInstance(perTestMock, installedDefault)`. The override applies for the duration
   of that test method only. Don't use `installMockForInstance` standalone — it overrides nothing if no type
   mock is registered.
6. **`QuarkusMock` is incompatible with parallel test execution.** Both `installMockForType` and
   `installMockForInstance` replace beans globally for the test's duration, which causes race conditions when
   tests run in parallel. If the project enables JUnit parallelism (`junit.jupiter.execution.parallel.enabled`),
   stick to `@InjectMock` for these tests.
7. **Never declare the same bean with both `@InjectMock` and `QuarkusMock.installMockForType`.** That's the
   cargo-cult pattern: two mechanisms install the same mock, one always wins, the other is dead code that
   misleads readers. Pick the right mechanism per Rules 3-4 and use only that one. (This rule does *not*
   forbid the legitimate `installMockForType` + `installMockForInstance` pairing from Rule 5, where the two
   work together by design.)
8. **Don't mock domain types.** Aggregates, value objects, and entities should be exercised with real instances
   — that's what makes the test useful. Mock only collaborators (other application services, external clients).
9. **Use REST Assured for HTTP assertions.** `given().when().get(...).then().statusCode(...).body(...)`.
   Assert on status code, then on body shape. Hamcrest matchers (`is`, `equalTo`, `hasSize`) for body content.
10. **Don't test the framework.** Don't write a test that asserts Hibernate persists a field, Jackson serializes
    a record, or Quarkus injects a bean. Test *your* logic.
11. **Know whether the project skips integration tests by default.** Many Maven Quarkus projects set
    `<skipITs>true</skipITs>`, so `./mvnw verify` runs only unit tests; ITs run with `-DskipITs=false` or
    under the `native` profile. Check `pom.xml` before assuming.
12. **Endpoint paths in tests must match `@Path` on the resource.** If a test hits `/api/orders` and the
    resource is `@Path("/orders")`, the test is wrong — fix the test, not the resource. Match what the resource
    actually declares.

---

## Canonical Examples

### REST resource test with `@InjectMock`

```java
package com.example.orders.interfaces.rest;

import com.example.orders.application.OrderApplicationService;
import com.example.orders.application.OrderDTO;
import io.quarkus.test.InjectMock;
import io.quarkus.test.junit.QuarkusTest;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.mockito.Mockito;

import java.util.List;
import java.util.Optional;

import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.*;

@QuarkusTest
class OrdersResourceTest {

    @InjectMock
    OrderApplicationService orders;

    @BeforeEach
    void setUp() {
        Mockito.when(orders.findAll()).thenReturn(List.of(
            new OrderDTO(1L, "first", 9.99),
            new OrderDTO(2L, "second", 19.99)
        ));
        Mockito.when(orders.findById(1L)).thenReturn(Optional.of(new OrderDTO(1L, "first", 9.99)));
        Mockito.when(orders.findById(99L)).thenReturn(Optional.empty());
    }

    @Test
    void listOrdersReturnsAll() {
        given()
            .when().get("/orders")
            .then()
                .statusCode(200)
                .body("size()", is(2));
    }

    @Test
    void getOrderReturnsBodyWhenFound() {
        given()
            .when().get("/orders/1")
            .then()
                .statusCode(200)
                .body("id", equalTo(1));
    }

    @Test
    void getOrderReturns404WhenMissing() {
        given()
            .when().get("/orders/99")
            .then()
                .statusCode(404);
    }
}
```

### Integration-test companion

```java
package com.example.orders.interfaces.rest;

import io.quarkus.test.junit.QuarkusIntegrationTest;

@QuarkusIntegrationTest
class OrdersResourceIT extends OrdersResourceTest {
    // Same tests, run against the packaged artifact in `target/`.
}
```

### Repository test against Dev Services PostgreSQL

```java
package com.example.orders.infrastructure;

import com.example.orders.domain.Money;
import com.example.orders.domain.Order;
import io.quarkus.test.TestTransaction;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import java.math.BigDecimal;
import java.util.Currency;
import java.util.UUID;

import static org.junit.jupiter.api.Assertions.*;

@QuarkusTest
class OrderRepositoryTest {

    @Inject OrderRepository orders;

    @Test
    @TestTransaction
    void persistsAndFindsById() {
        UUID id = UUID.randomUUID();
        Order order = Order.place(id, new Money(new BigDecimal("9.99"), Currency.getInstance("USD")));
        orders.persist(order);

        Order found = orders.findById(id).orElseThrow();
        assertEquals(id, found.id);
        assertEquals(0L, found.version);
    }
}
```

`@TestTransaction` rolls back at the end of each test, keeping the database clean between tests.

### Dynamic mocking pattern — `installMockForType` + `installMockForInstance`

Demonstrates **Rule 4(c) + Rule 5**. Install a default mock in `@BeforeAll`; for individual test methods that need different behavior, swap with `installMockForInstance(perTestMock, defaultMock)`. The same shape is used for **Rule 4(a)** when the default is a hand-rolled fake instead of a Mockito mock.

**Caveat (Rule 6):** This pattern is incompatible with parallel test execution. If the project runs tests in parallel, fall back to `@InjectMock`.

```java
package com.example.orders.interfaces.rest;

import com.example.orders.application.OrderApplicationService;
import com.example.orders.application.OrderNotFoundException;
import io.quarkus.test.junit.QuarkusMock;
import io.quarkus.test.junit.QuarkusTest;
import org.junit.jupiter.api.BeforeAll;
import org.junit.jupiter.api.Test;
import org.mockito.Mockito;

import static io.restassured.RestAssured.given;

@QuarkusTest
class OrdersResourceTest {

    static OrderApplicationService defaultMock;

    @BeforeAll
    static void installDefault() {
        defaultMock = Mockito.mock(OrderApplicationService.class);
        Mockito.doNothing().when(defaultMock).cancel(Mockito.anyLong());
        QuarkusMock.installMockForType(defaultMock, OrderApplicationService.class);
    }

    @Test
    void cancelOrderReturns204WhenSuccessful() {
        // default mock applies — cancel succeeds for any id
        given().when().delete("/orders/1").then().statusCode(204);
    }

    @Test
    void cancelOrderReturns404WhenMissing() {
        OrderApplicationService perTest = Mockito.mock(OrderApplicationService.class);
        Mockito.doThrow(new OrderNotFoundException(99L)).when(perTest).cancel(99L);
        QuarkusMock.installMockForInstance(perTest, defaultMock);

        given().when().delete("/orders/99").then().statusCode(404);
    }
}
```

Key points:

- The default mock is held as a `static` field so per-test methods can pass it to `installMockForInstance` as the existing instance to replace.
- `installMockForInstance(newMock, existingInstance)` swaps the registered mock for the duration of the test method only.
- This pattern is overkill for most tests. Reach for it only when `@InjectMock` + per-call stubbing in `@BeforeEach` genuinely doesn't fit (e.g. tests need to swap the entire mock instance, not just stub different return values per call).

What these examples demonstrate:

- `@InjectMock OrderApplicationService` is a single source of truth for the mock; no `QuarkusMock.installMockForType` line.
- The mocked variable name matches the type being mocked (`orders`, not `mockRepository`).
- REST Assured asserts on status first, body second.
- IT companion is a one-liner — the value is in the inheritance, not the override.
- Repository test uses real Dev Services Postgres + `@TestTransaction` for isolation.
- Endpoint paths in tests match the resource's `@Path`.

---

## Anti-patterns

| Don't | Why it's wrong | Fix |
|---|---|---|
| `@InjectMock OrderService orderService;` *and* `QuarkusMock.installMockForType(mock, OrderService.class)` for the same bean | Two ways of installing the same mock; one always wins, the other is dead code that misleads readers. | Pick `@InjectMock` and stub in `@BeforeEach`. Drop the `installMockForType` call. |
| `OrderService mockOrderRepository = Mockito.mock(OrderService.class)` | Mixing "Service" and "Repository" in the variable name is exactly the layer confusion the type system is supposed to prevent. | Match the variable name to the type: `OrderService orders = Mockito.mock(OrderService.class)`. |
| Mocking `Order` (an aggregate) | The aggregate's invariants are the thing under test. Mocking it bypasses the invariants. | Use a real `Order` instance built with its factory method. Mock only the repository or external client. |
| Test hits `/api/orders` while the resource is `@Path("/orders")` | The test asserts a route that doesn't exist; failure is not informative. | Fix the test to match what the resource's `@Path` actually declares. |
| `@QuarkusTest` test that asserts `entityManager.persist(...)` updates a field | Tests Hibernate, not your code. | Test the behaviour your service exposes — not the ORM's correctness. |
| Integration test class without the companion `extends` | Duplicates every test method, drifts over time. | `class FooIT extends FooTest {}`. |
| Test that creates a real PostgreSQL container manually with `@Testcontainers` while Dev Services is enabled | Two databases compete; flaky and slow. | Let Dev Services manage it. Disable Dev Services explicitly only if you must. |
| Asserting on `body(is("..."))` for JSON | Compares whole-body strings; brittle to whitespace and field order. | Assert on JSON paths: `.body("size()", is(2))`, `.body("id", equalTo(1))`. |

---

## Excuse / Reality

When you catch yourself reasoning around the rules above, look here before you type. The left column is verbatim — what you'll actually say in your head or in Slack. The right column is what defeats it.

| Excuse | Reality |
|---|---|
| "A senior reviewer told me to use both `@InjectMock` and `QuarkusMock.installMockForType` — that's authority." | Cargo-culted defenses against unspecified races aren't authority, they're folklore. If a CDI race exists, name it and file a bug. Otherwise, follow the rule. One source of truth per mock. |
| "Belt-and-braces never hurts — install the mock both ways to be safe." | They install the same mock through different mechanisms. One always wins; the other is dead code. The next test author has to figure out which path is live, and tests are read more than they're written. |
| "Mocking the aggregate lets me control its behavior precisely." | The aggregate's invariants *are* the system under test for the layer below. Mocking it asserts nothing about real correctness — only that your mock returns what you told it to. Mock collaborators, never aggregates. |
| "I'll use a real database for everything; mocks are dishonest." | Integration tests are slow and shared. Mocking the application service for a *resource* test is the right scope. Save Dev Services Postgres for repository tests where the DB *is* the system under test. |
| "Tests are blocking the deploy; just disable the failing ones for now." | A skipped test is a passing test that lies. The deploy is now riskier than if you'd taken five minutes to fix the test. |

---

## Quick Reference

### Annotations cheat sheet

| Annotation | Purpose |
|---|---|
| `@QuarkusTest` | JVM-mode test; boots Quarkus once per test profile. |
| `@QuarkusIntegrationTest` | Packaged-mode test; runs against the `target/` artifact. |
| `@InjectMock` | Replaces a CDI bean with a Mockito mock for the test. |
| `@TestTransaction` | Wraps the test in a transaction that rolls back at the end. |
| `@TestProfile(MyProfile.class)` | Run a subset of tests under a different config profile. |

### REST Assured idiom

```java
given()
    .contentType("application/json")
    .body(requestObject)
.when()
    .post("/orders")
.then()
    .statusCode(201)
    .header("Location", containsString("/orders/"))
    .body("id", notNullValue());
```

### Mocking decision

| You want to... | Use |
|---|---|
| Mock a CDI bean (always start here) | `@InjectMock` field + `Mockito.when(...)` in `@BeforeEach` |
| Install a hand-rolled fake or stateful test double in place of a CDI bean (Rule 4a) | `QuarkusMock.installMockForType(myFake, T.class)` in `@BeforeAll` |
| Mock a normal-scoped bean (`@ApplicationScoped` / `@RequestScoped`) so *other* beans see the mock too (Rule 4b) | `QuarkusMock.installMockForType` in `@BeforeAll` |
| Swap the mock implementation for one test method only (Rule 4c + Rule 5) | `QuarkusMock.installMockForInstance(perTest, installedDefault)` inside the test method |
| Run tests in parallel with mocks (Rule 6) | Stick to `@InjectMock` — `QuarkusMock` is unsafe under parallel execution |
| Test a real CDI bean against a real DB | `@QuarkusTest` + `@Inject` + `@TestTransaction` |
| Test a domain type's invariants | Plain JUnit, no Quarkus annotations |

### Run commands

| Task | Command |
|---|---|
| Unit tests only | `./mvnw test` |
| Single test | `./mvnw test -Dtest=OrdersResourceTest` |
| Single test method | `./mvnw test -Dtest=OrdersResourceTest#getOrderReturns404WhenMissing` |
| Include integration tests | `./mvnw verify -DskipITs=false` |
| Continuous testing in dev mode | `./mvnw quarkus:dev`, then press `r` |
| Native ITs | `./mvnw verify -Dnative` |
