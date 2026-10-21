# Workflow: Writing Java Unit Tests

## Scope

Write unit tests for the Spring Boot application in `order-service/` using
**JUnit 5 + Mockito + AssertJ**. Only create or edit files under
`order-service/src/test/`. Never modify production code under `src/main/`; if
you find a bug or an untestable design, report it to the user instead.

## Steps

1. **Identify the class under test.** Read the production class completely
   (under `order-service/src/main/java/...`) and note every public method,
   branch, thrown exception, and collaborator.
2. **Read the related existing tests first.** Before writing anything, read the
   tests under `order-service/src/test/` that cover the same layer or neighbors:
   - Use `glob` on `order-service/src/test/**/*.java` to list them.
   - Read the test for the same class (if it exists) and at least one test from
     the same layer (service, controller, repository).
   - Read `order-service/src/test/resources/` for test properties.
3. **Extract the conventions** listed in `2_conventions.md` from those files
   (assertion library, mocking style, naming pattern, fixture setup). If what
   you see differs from the defaults in this mode, **follow the repo**.
4. **List the cases to cover** before coding: happy path, empty/absent results,
   each branch, each exception, boundary values, and verification of side
   effects on collaborators. Skip cases that the existing test already covers.
5. **Write the tests.**
   - Existing test class: add new `@Test` methods to it; do not rewrite or
     reformat existing tests.
   - No test class yet: create `<ClassName>Test.java` in the mirrored package
     under `src/test/java`.
6. **Run and verify.** If command access is unavailable, say so and tell the
   user the command to run:
   `cd order-service && mvn test -Dtest=<ClassName>Test`
   Report any failure honestly and fix the test, not the production code.

## Completion Criteria

- Every public method and branch of the targeted class has a test.
- New tests match the style of the neighboring tests.
- No production files were changed.
- You summarized which cases were added and any production issues noticed.
