---
name: testing-core
description: "Comprehensive guidance for software testing strategies including unit testing, integration testing, end-to-end testing, TDD, BDD, mocking, and test architecture across languages. Use when designing test strategy, writing unit/integration/E2E tests, implementing TDD/BDD, or setting up test infrastructure for any language. Keywords: write unit test, integration test, test coverage, mock dependency, Jest test, pytest, TDD, test fixture."
allowed-tools: Read,Glob,Grep,Bash,Edit,Write
user-invocable: true
---


<!-- generated-by: claude-global-library/tools/project_sync -->
<!-- source: skills/testing-core/SKILL.md -- edit the library, then re-run sync_project.py -->

# Testing Core Skill Instructions

### Skill Name
testing-core

### Description
Comprehensive guidance for software testing strategies across languages and frameworks. Covers unit testing, integration testing, end-to-end testing, TDD, BDD, mocking patterns, test data management, performance testing, and test architecture for building reliable, maintainable test suites.

---

## Mathematical Foundations

### M1: Test Coverage Mathematics
```
Statement coverage:    covered_statements / total_statements
Branch coverage:       covered_branches / total_branches (stricter)
MC/DC coverage:        each condition independently affects decision (aviation standard)
Path coverage:         covered_paths / total_paths = O(2^decisions) — infeasible for large code

Cyclomatic complexity = minimum tests for 100% branch coverage:
  V(G) = E - N + 2P  (edges - nodes + 2 × components)
  Simplified: V(G) = decision_points + 1
  V(G) = 10 → minimum 10 test cases for full branch coverage

Coverage diminishing returns:
  0→80% coverage: catches ~70% of latent bugs
  80→90% coverage: catches additional ~10% of bugs  (high ROI)
  90→100% coverage: catches additional ~5% of bugs   (diminishing ROI)
  Research finding: 100% coverage does NOT mean 0 bugs
```

### M2: Test Pyramid Cost Ratio
```
Test execution time benchmarks:
  Unit test:           ~1ms  (in-memory, no I/O)
  Integration test:    ~100ms–10s (DB, network I/O)
  E2E/UI test:         ~5s–60s (browser automation)

Recommended test distribution for 10-minute CI budget:
  Unit tests:          ~70% of tests, consume ~1% of CI time
  Integration tests:   ~20% of tests, consume ~19% of CI time
  E2E tests:           ~10% of tests, consume ~80% of CI time

  For 10min budget, E2E at 30s each: max 16 E2E tests
  For 10min budget, integration at 500ms: max 1200 integration tests
  For 10min budget, unit at 1ms: max 600,000 unit tests

ROI formula:
  test_ROI = (bugs_caught × avg_bug_fix_cost) / (test_write_time + test_run_time × runs)
  Unit tests have highest ROI: fast to write, fast to run, catch regressions early
```

### M3: Statistical Significance for Flaky Test Detection
```
Flaky test: test that fails non-deterministically
  Flakiness rate p = failures / total_runs

  Sample size for p% flakiness detection at 95% confidence:
    n ≥ log(0.05) / log(1-p)
    p=0.01 (1% flaky):  n ≥ 299 runs
    p=0.05 (5% flaky):  n ≥ 59 runs
    p=0.10 (10% flaky): n ≥ 29 runs
    p=0.30 (30% flaky): n ≥ 9 runs

  Practical: rerun test 5 times; if any failure → suspect flaky
  False alarm rate at 5 reruns: P(at least 1 failure for 1% truly-flaky) = 1-(0.99)^5 = 4.9%

Flakiness cost:
  developer_investigation_time × flaky_alerts_per_day = wasted_hours
  At 5 flaky alerts/day × 20 min each = 100 min/day waste
```

### M4: Mutation Testing Score
```
Mutation testing: artificially introduce bugs and check if tests catch them

Mutation score = caught_mutants / total_mutants

  Killed mutant:  at least 1 test fails when mutation introduced → good
  Surviving mutant: all tests pass with mutation → test gap found

Mutation score vs line coverage correlation:
  80% line coverage: typically ~50–60% mutation score
  90% line coverage: typically ~70–80% mutation score
  Good mutation score: > 80% (industry target)

Mutation testing cost:
  Time = n_mutants × avg_test_suite_time
  n_mutants ≈ 5–10 × lines_of_code (each operator produces multiple mutants)
  For 1000 LOC, 10s test suite: 5000–10000 × 10s = 50,000–100,000s ≈ 14–28 hours
  → Run only on changed files (incremental mutation testing)
```

### M5: Property-Based Testing Shrinking
```
Property-based testing (QuickCheck, Hypothesis, fast-check):
  Generates n random inputs, verifies property holds for each
  n = 100 (default) → P(finding edge case) depends on input space distribution

  Shrinking: when failure found, minimize failing input
    Shrinking cost: O(log input_size) steps typically
    Shrink step: try smaller input, verify still fails, repeat

  Coverage of input space:
    Purely random: uniform distribution → misses edge cases
    Guided generation: focuses on boundary values (0, max, -1, empty, etc.)
    P(boundary found) = 1 - (1 - boundary_fraction)^n

  For string inputs: boundary_fraction ≈ 0.01%  → need n=46,050 to find with 99% confidence
  Hypothesis solves this with explicit boundary generators
```

### M6: India Software Quality Certification Math — STQC, BIS, and NASSCOM Standards

India has a structured software quality certification ecosystem for government and regulated-sector software, with mandatory testing requirements carrying financial and contractual penalties for non-compliance.

STQC (Standardisation Testing and Quality Certification) certification framework:
- STQC provides software product testing under IS/ISO/IEC 25010 (SQuaRE) framework
- Required for: all e-governance software (MeitY mandate), healthcare IT systems (CDSCO medical device software), financial sector software (RBI technology guidelines)

STQC testing coverage requirements:
```
Minimum coverage for STQC certification:
  statement_coverage ≥ 85% (government software)
  branch_coverage ≥ 80%
  requirement_coverage = 100% (all functional requirements must have test cases)

STQC audit cost model:
  test_effort_person_days = SLOC / 1000 × complexity_factor × 15
  complexity_factor: simple (1.0), moderate (1.5), complex (2.5)
  STQC rate: ₹8,000-12,000 per person-day
  certification_cost = test_effort_person_days × ₹10,000 (median)
```

BIS IS 9000 (Software Quality Assurance): India's national standard for software quality assurance practices. Maps to ISO 9001 adapted for software — requires documented test plan, test cases, and defect tracking for BIS IS mark certification. Mandatory for software in BIS-regulated product categories (electronic goods, medical devices).

India-specific defect density benchmarks by sector:
```
Industry defect density benchmarks (defects per KLOC, post-UAT):
  RBI fintech (banking/payments software): ≤ 0.10 defects/KLOC
  CDSCO medical device software (Class B/C): ≤ 0.05 defects/KLOC (ISO 14971 risk-aligned)
  STQC e-governance: ≤ 0.20 defects/KLOC
  General enterprise (India average): 1.0-2.5 defects/KLOC (NASSCOM benchmark)
  NASSCOM BFSI testing standard: ≤ 0.15 defects/KLOC for critical financial systems

Defect detection efficiency (DDE) formula:
  DDE = defects_found_in_testing / total_defects_in_system
  India BFSI target DDE ≥ 0.92 (92% of defects caught before production)
  DDE = 1 - (production_defects / (test_defects + production_defects))

Penalty model for non-compliant government software:
  liquidated_damages = contract_value × penalty_rate × delay_weeks
  typical penalty_rate: 0.5-1.0% per week of delay due to quality failures
  cap: usually 10-15% of contract value (NIC/government tender standard terms)
```

NASSCOM BFSI testing compliance framework: testing of banking software must include regulatory scenario coverage — all RBI circulars affecting the system must have corresponding test cases, verified by internal audit before submission to RBI for approval of new banking technology systems.

---

## Testing Fundamentals

### 1. Testing Pyramid
- **Unit tests (base)**: largest number, fastest execution, test individual functions/classes in isolation
- **Integration tests (middle)**: moderate number, test interactions between components, databases, APIs
- **End-to-end tests (top)**: fewest number, slowest execution, test complete user flows through the full system
- The pyramid shape reflects the ideal ratio: many unit tests, fewer integration tests, fewest E2E tests
- Anti-pattern: ice cream cone (many E2E, few unit) leads to slow, flaky, expensive test suites
- Complement with exploratory testing and manual testing for edge cases automation misses

### 2. Test Types
- **Unit tests**: test a single function, method, or class in isolation from dependencies
- **Integration tests**: test interactions between two or more components (API + database, service + message queue)
- **End-to-end (E2E) tests**: test complete user workflows through the full application stack
- **Smoke tests**: quick sanity checks that critical paths work after deployment
- **Regression tests**: verify that previously working functionality still works after changes
- **Acceptance tests**: verify business requirements are met from the user's perspective
- **Load/performance tests**: verify the system meets performance requirements under expected load

### 3. Test-Driven Development (TDD)
- **Red**: write a failing test that defines the desired behavior
- **Green**: write the minimum code to make the test pass
- **Refactor**: improve the code structure while keeping all tests green
- TDD drives design decisions: code is designed for testability from the start
- Benefits: higher test coverage, better design, living documentation, confidence in refactoring
- Start with the simplest test case and incrementally add complexity

### 4. Behavior-Driven Development (BDD)
- Extend TDD with natural language specifications using Given-When-Then format
- Specifications written in collaboration with stakeholders, developers, and testers
- Tools: Cucumber (Java/JS), Behave (Python), SpecFlow (.NET)
- Feature files serve as living documentation that is both human-readable and executable
- Focus on behavior from the user's perspective rather than implementation details

### 5. Testing Economics
- Cost of finding bugs increases exponentially the later they're discovered (requirements -> design -> code -> testing -> production)
- Automated tests have high upfront cost but pay off through repeated execution
- ROI of testing: measure by bugs caught, regression prevention, and developer confidence
- Over-testing: diminishing returns beyond 80-90% meaningful coverage
- Under-testing: technical debt that compounds with every change

### 6. When to Test What
- **Always unit test**: business logic, algorithms, data transformations, validation rules
- **Always integration test**: database queries, API endpoints, message handling, authentication
- **Always E2E test**: critical user flows (login, checkout, data submission)
- **Consider not testing**: trivial getters/setters, framework code, generated code, one-off scripts
- **Test boundaries**: focus tests on interfaces between components, not internal implementation

---

## Unit Testing

### 1. Isolation
- Each unit test should test one thing and one thing only
- Dependencies should be replaced with test doubles (mocks, stubs, fakes)
- No database, network, filesystem, or external service calls in unit tests
- Tests should run in any order without affecting each other
- Isolation enables fast execution (1000+ tests per second) and reliable results

### 2. Assertions
- Use specific assertions: `assertEqual`, `assertTrue`, `assertContains` over generic `assert`
- One logical assertion per test (may involve multiple assert statements for one concept)
- Custom assertion messages for clear failure diagnostics: `assertEqual(result, expected, "Tax calculation failed for income > 100k")`
- Assertion libraries: AssertJ (Java), assertpy (Python), Chai (JS), Hamcrest (cross-language)
- Fluent assertions improve readability: `assertThat(result).isEqualTo(expected).isNotNull()`

### 3. Test Structure: Arrange-Act-Assert (AAA)
- **Arrange**: set up test data, create objects, configure mocks
- **Act**: execute the code under test (single method call or operation)
- **Assert**: verify the result matches expectations
- Keep each section visually separated with blank lines or comments
- Avoid multiple Act sections in a single test; split into separate tests

### 4. Test Naming Conventions
- Name should describe the scenario and expected outcome
- Patterns: `test_methodName_condition_expectedResult`, `should_returnTrue_when_inputIsValid`
- Java: `@DisplayName("should calculate tax correctly for income above threshold")`
- Python: `def test_calculate_tax_above_threshold_returns_higher_rate():`
- Good names eliminate the need to read the test body to understand intent

### 5. Parameterized Tests
- Run the same test logic with different input-output pairs
- JUnit 5: `@ParameterizedTest` with `@ValueSource`, `@CsvSource`, `@MethodSource`
- pytest: `@pytest.mark.parametrize("input,expected", [(...), (...)])`
- Jest: `test.each([[input1, expected1], [input2, expected2]])`
- Reduces test code duplication while increasing coverage across edge cases

### 6. Test Doubles
- **Mock**: records interactions and verifies expected calls were made
- **Stub**: returns predefined responses without recording interactions
- **Spy**: wraps a real object, records calls, and delegates to the real implementation
- **Fake**: lightweight implementation of a dependency (in-memory database, fake HTTP server)
- Choose the simplest test double that achieves the testing goal

### 7. Code Coverage Metrics
- **Line coverage**: percentage of code lines executed during tests
- **Branch coverage**: percentage of conditional branches (if/else) exercised
- **Function/method coverage**: percentage of functions called during tests
- 80% line coverage is a reasonable target; 100% is rarely worth the effort
- Coverage measures quantity, not quality; high coverage with weak assertions provides false confidence

### 8. Mutation Testing
- Introduces small changes (mutations) to source code and checks if tests catch them
- Mutations: change operators (+/-), swap conditions (>/< ), remove method calls, return null
- "Killed" mutations (tests fail) indicate strong tests; "survived" mutations indicate test gaps
- Tools: PIT (Java), mutmut (Python), Stryker (JS/TS)
- More meaningful than code coverage for measuring test effectiveness

---

## Integration Testing

### 1. Testing Boundaries
- Focus on the interface between components: does component A correctly communicate with component B?
- Test serialization/deserialization across boundaries (JSON, XML, Protobuf)
- Verify error handling at integration points: timeouts, malformed responses, connection failures
- Contract testing ensures both sides of an integration agree on the interface
- Integration tests should be independent of unit tests; both are needed

### 2. Database Integration Tests
- Test actual SQL queries against a real (or containerized) database
- Use migrations to set up schema before tests; rollback or recreate between tests
- Test repositories/DAOs with real database operations: CRUD, transactions, constraints
- Verify query performance with EXPLAIN plans for critical queries
- Tools: H2 (in-memory SQL for Java), SQLite (lightweight), Testcontainers (real database in Docker)

### 3. API Integration Tests
- Test REST/GraphQL endpoints with actual HTTP requests
- Verify request/response format, status codes, headers, and error responses
- Test authentication and authorization: valid tokens, expired tokens, missing permissions
- Use HTTP client libraries or framework test utilities (MockMvc, TestClient, supertest)
- Test API versioning and backward compatibility

### 4. Message Queue Testing
- Verify message production and consumption across queues (Kafka, RabbitMQ, SQS)
- Test message serialization, routing, and error handling (dead letter queues)
- Use embedded brokers or Testcontainers for isolated queue testing
- Verify exactly-once processing, ordering guarantees, and retry behavior
- Test consumer idempotency: same message processed twice should not cause issues

### 5. External Service Testing
- Test interactions with third-party APIs and services
- Use contract tests to verify your client matches the service's API contract
- Mock external services for speed and reliability; test against real services periodically
- Tools: WireMock (HTTP mocking), LocalStack (AWS service emulation), MockServer
- Test failure modes: service unavailable, slow response, unexpected response format

### 6. Testcontainers
- Library for running Docker containers as test dependencies (databases, message brokers, caches)
- Available for Java, Python, Node.js, Go, .NET, and Rust
- `@Testcontainers` annotation (Java) or `with` context manager (Python) for lifecycle management
- Provides pre-built modules for PostgreSQL, MySQL, MongoDB, Redis, Kafka, Elasticsearch
- Containers start fresh for each test class/module ensuring isolation

### 7. In-Memory Databases
- H2 (Java): in-memory SQL database compatible with PostgreSQL and MySQL modes
- SQLite: lightweight file-based or in-memory database for testing
- Pros: very fast, no Docker needed, simple setup
- Cons: behavior differences from production database (dialect quirks, missing features)
- Best for unit tests of repository logic; use Testcontainers for integration tests

---

## End-to-End Testing

### 1. E2E Frameworks
- **Cypress**: JavaScript-based, excellent developer experience, automatic waiting, time travel debugging
- **Playwright**: Microsoft's cross-browser tool, supports Chromium/Firefox/WebKit, multi-language (JS/Python/Java/.NET)
- **Selenium**: oldest and most widely used, supports all browsers and languages, WebDriver protocol
- **Puppeteer**: Chrome/Chromium-only, Node.js, good for Chrome-specific testing and PDF generation
- Choose based on: browser support needs, language preference, team experience, CI/CD integration

### 2. Page Object Model
- Encapsulate page structure and interactions in reusable page classes
- Each page class contains selectors (locators) and action methods (click, fill, assert)
- Tests interact with page objects, not raw selectors: `loginPage.enterUsername("user")`
- Benefits: single place to update when UI changes, readable test code, reduced duplication
- Organize page objects hierarchically: base page -> specific pages -> page components

### 3. Test Data Management
- Create test data before each test and clean up after; never depend on pre-existing data
- Use factories or builders to generate test data programmatically
- Isolate test data per test to prevent interference between parallel tests
- Seed databases with known state before test suite execution
- Environment-specific test data: separate data for local, CI, staging environments

### 4. Flaky Test Prevention
- **Wait strategies**: use explicit waits for dynamic content instead of fixed sleep
- **Isolation**: ensure tests don't depend on state from other tests or previous runs
- **Deterministic data**: avoid random data that changes test behavior
- **Retry policies**: retry flaky tests once in CI, but investigate and fix the root cause
- **Stability metrics**: track flake rate per test and quarantine consistently flaky tests

### 5. CI/CD Integration
- Run E2E tests in CI pipelines with headless browser mode
- Parallelize test execution across multiple containers for faster feedback
- Capture screenshots and videos on test failure for debugging
- Fail the pipeline on E2E test failures for critical flows; warn for non-critical
- Schedule full E2E suites nightly; run smoke tests on every commit

### 6. Visual Regression Testing
- Capture screenshots of pages/components and compare against baseline images
- Detect unintended visual changes: layout shifts, color changes, missing elements
- Tools: Percy, Chromatic, BackstopJS, Playwright visual comparison
- Manage visual baselines: approve intentional changes, flag unexpected ones
- Set tolerance thresholds for acceptable pixel differences (anti-aliasing, rendering)

### 7. Cross-Browser Testing
- Test on multiple browsers: Chrome, Firefox, Safari, Edge (at minimum)
- Mobile browser testing: responsive design on iOS Safari and Chrome Android
- Browser-specific issues: CSS rendering differences, JavaScript API availability
- Cloud testing platforms: BrowserStack, Sauce Labs, LambdaTest for broad coverage
- Prioritize browser coverage based on your user analytics data

---

## Mocking and Test Doubles

### 1. Mocks vs Stubs vs Spies vs Fakes
- **Mock**: verifies behavior - "was this method called with these arguments?"
- **Stub**: provides canned responses - "when called, return this value"
- **Spy**: wraps real implementation, records calls for later verification
- **Fake**: working implementation unsuitable for production (in-memory database, fake SMTP server)
- **Dummy**: placeholder objects passed but never used (satisfying parameter requirements)

### 2. When to Mock
- Mock external dependencies: APIs, databases, file systems, clocks, random generators
- Mock at boundaries: interfaces between your code and external systems
- Do not mock the code under test; mock its dependencies
- Prefer real implementations for fast, deterministic dependencies
- If mocking makes the test harder to understand, reconsider the approach

### 3. Mocking Frameworks
- **Java**: Mockito (`when(mock.method()).thenReturn(value)`), MockK (Kotlin-friendly)
- **Python**: `unittest.mock` (`@patch`, `Mock()`, `MagicMock()`), pytest-mock
- **JavaScript**: Jest (`jest.fn()`, `jest.mock()`), Sinon.js, Vitest
- **Go**: gomock, testify/mock, interface-based mocking
- **Kotlin**: MockK (`every { mock.method() } returns value`, `coEvery` for coroutines)

### 4. Mock Verification
- Verify method was called: `verify(mock).method(argument)`
- Verify call count: `verify(mock, times(3)).method(any())`
- Verify call order: `inOrder(mock1, mock2).verify(...)` (Mockito)
- Argument capture: `ArgumentCaptor` (Java), `call_args` (Python) for inspecting arguments
- Over-verification: avoid verifying every interaction; focus on meaningful behavior

### 5. Over-Mocking Anti-Pattern
- Tests that mock everything become fragile and test mock configuration, not real behavior
- Symptoms: tests break on refactoring even when behavior is unchanged
- Tests that are harder to read than the code they test
- Solution: mock at the boundary, use real implementations for internal dependencies
- Integration tests complement unit tests by testing real interactions

### 6. Contract-Based Mocking
- Generate mocks from API contracts (OpenAPI/Swagger, GraphQL schema, Pact)
- Consumer-driven contracts: consumers define expected interactions, providers verify compliance
- Pact: record consumer expectations, replay against provider, verify compatibility
- Ensures mock behavior matches the real service's behavior
- Catches integration issues before deployment without running both services together

---

## Test Data Management

### 1. Test Fixtures
- Reusable test data configurations loaded before test execution
- JSON/YAML/SQL files containing predefined test data sets
- pytest fixtures: `@pytest.fixture` with scope control (function, class, module, session)
- JUnit: `@BeforeEach`, `@BeforeAll` for fixture setup; `@AfterEach`, `@AfterAll` for cleanup
- Keep fixtures minimal: only the data needed for the specific test

### 2. Factories and Builders
- Programmatically generate test data with sensible defaults and customizable fields
- Factory pattern: `UserFactory.create(name="Alice", role="admin")` with default values for unspecified fields
- Builder pattern: `User.builder().name("Alice").role("admin").build()` for complex objects
- Libraries: FactoryBot (Ruby), factory_boy (Python), Fishery (TypeScript), Instancio (Java)
- Override only the fields relevant to the test; use defaults for everything else

### 3. Database Seeding
- Populate databases with initial data before test execution
- Migration-based seeding: run data migrations alongside schema migrations
- Script-based seeding: SQL or ORM-based scripts that insert known data
- Seed data should be idempotent: running twice produces the same state
- Environment-specific seeds: minimal for unit tests, realistic for E2E tests

### 4. Data Cleanup Strategies
- **Transaction rollback**: wrap each test in a transaction and rollback after (fastest, cleanest)
- **Truncate tables**: clear all data between tests (PostgreSQL `TRUNCATE ... CASCADE`)
- **Delete specific records**: remove only test-created data using known IDs or markers
- **Recreate database**: drop and recreate the entire database (slowest, most thorough)
- Choose based on speed requirements and isolation needs

### 5. Test Isolation
- Each test must be independent: pass/fail regardless of execution order or other tests
- No shared mutable state between tests (global variables, class-level state, shared database rows)
- Parallel test execution requires strict isolation; shared resources need synchronization
- Detect isolation violations by running tests in random order
- Unique identifiers in test data prevent collisions between parallel tests

### 6. Fake Data Generators
- **Faker** (Python/JS/Java/Ruby): generates realistic fake names, addresses, emails, phone numbers
- Locale support: generate data appropriate for specific countries and languages
- Seeded randomness: `Faker.seed(42)` for reproducible fake data across test runs
- Custom providers: extend Faker with domain-specific data generators
- Use fake data for realistic test scenarios without using real PII

---

## API Testing

### 1. REST API Testing
- Test all HTTP methods: GET, POST, PUT, PATCH, DELETE for each endpoint
- Verify response status codes: 200, 201, 204, 400, 401, 403, 404, 500
- Validate response body structure and content against expected schema
- Test query parameters, path parameters, headers, and request body validation
- Tools: REST Assured (Java), requests + pytest (Python), supertest (Node.js), httpx (Python async)

### 2. Contract Testing (Pact)
- Consumer writes a contract defining expected interactions with the provider
- Provider verifies it can fulfill all consumer contracts
- Contracts stored in a Pact Broker for sharing between consumer and provider teams
- Catches breaking changes before deployment without running full integration tests
- Best for: microservice architectures with many service-to-service interactions

### 3. Schema Validation
- Validate API responses against JSON Schema or OpenAPI specifications
- Catch schema drift: response structure changes that break consumers
- Tools: jsonschema (Python), Ajv (JS), SchemaValidator (Java)
- Generate schemas from code (code-first) or validate code against schemas (schema-first)
- Automate schema validation in CI to catch breaking changes early

### 4. Response Assertion
- Assert specific field values, types, and presence/absence in responses
- Deep equality for complex nested objects; partial matching for large responses
- Assert response headers: Content-Type, Cache-Control, CORS headers
- Assert response time within acceptable bounds for performance-critical endpoints
- Use JSON path expressions for asserting deeply nested values

### 5. Authentication in Tests
- Test with valid tokens, expired tokens, invalid tokens, and missing tokens
- Verify role-based access: each endpoint accessible only to authorized roles
- Generate test tokens programmatically rather than using hardcoded values
- Test token refresh flows and session management
- Security testing: verify that unauthenticated requests are properly rejected

### 6. API Mocking (WireMock/MSW)
- **WireMock**: Java-based HTTP mock server for simulating external APIs
- **MSW (Mock Service Worker)**: intercepts HTTP requests at the network level in browser and Node.js
- Define request matchers and response templates for predictable testing
- Record real API interactions and replay them as mocks (record-and-replay)
- Simulate error conditions: timeouts, 500 errors, malformed responses

---

## Performance Testing

### 1. Load Testing
- Simulate expected production load to verify the system handles it without degradation
- Gradually ramp up concurrent users/requests to find the performance curve
- Measure: response time (p50, p95, p99), throughput (requests/second), error rate
- Tools: k6 (JavaScript DSL, modern), JMeter (GUI, widely used), Locust (Python, distributed)
- Run load tests against production-like environments, not development machines

### 2. Stress Testing
- Push the system beyond expected capacity to find breaking points
- Identify: at what load does the system start degrading? at what load does it fail?
- Observe failure modes: graceful degradation vs catastrophic failure
- Verify recovery: does the system recover automatically after load subsides?
- Document stress test results for capacity planning and alerting thresholds

### 3. Spike Testing
- Simulate sudden, dramatic traffic increases (flash sales, viral content, traffic spikes)
- Measure: how quickly the system scales up, response time during the spike, error rate
- Test auto-scaling behavior: do new instances come online fast enough?
- Verify: no data loss or corruption during spike handling
- Important for: e-commerce, media, event-driven applications

### 4. Endurance Testing
- Run the system under sustained moderate load for extended periods (hours to days)
- Detect: memory leaks, connection pool exhaustion, disk space growth, log file bloat
- Monitor resource usage trends: CPU, memory, disk, network over time
- Identify issues that only manifest under prolonged operation
- Schedule endurance tests weekly or before major releases

### 5. Performance Baselines
- Establish baseline performance metrics for key operations and endpoints
- Compare new releases against baselines to detect performance regressions
- Automated performance regression detection in CI/CD pipelines
- Track baselines over time to observe long-term performance trends
- Alert when performance degrades beyond defined thresholds (e.g., p95 > 500ms)

### 6. Performance Budgets
- Define maximum acceptable response times for each endpoint or user flow
- Budget examples: page load < 3s, API response < 200ms, database query < 50ms
- Enforce budgets in CI: fail the build if a performance budget is exceeded
- Budget allocation: if total page load budget is 3s, allocate budget per component
- Review and adjust budgets based on user experience requirements

---

## Testing Patterns

### 1. Given-When-Then
- **Given**: the preconditions and initial state (test setup)
- **When**: the action or event being tested (single operation)
- **Then**: the expected outcome (assertions)
- Natural language mapping for BDD-style test specifications
- Improves test readability for non-technical stakeholders

### 2. Test Fixtures
- Shared setup code that creates a known testing state
- Fixture scope: per-test (isolate), per-class (share within class), per-module (share within module)
- Composable fixtures: combine small fixtures into larger test configurations
- Parameterized fixtures: generate fixture variants for different test scenarios
- Document fixture purpose and the state they create

### 3. Setup and Teardown
- `setUp`/`tearDown` (per test), `setUpClass`/`tearDownClass` (per class)
- Setup: create test data, initialize dependencies, configure mocks
- Teardown: clean up resources, close connections, restore state
- Ensure teardown runs even on test failure (use `finally` or framework guarantees)
- Prefer setup in individual tests for clarity; use class/module setup for expensive operations

### 4. Test Categories and Tags
- Categorize tests: unit, integration, e2e, slow, fast, smoke, regression
- Tag tests for selective execution: `@Tag("slow")`, `@pytest.mark.slow`, `describe.skip()`
- CI configuration: run fast tests on every commit, slow tests on merge or nightly
- Team conventions: agree on tag taxonomy and when to use each tag
- Filter tests by tag: `pytest -m "not slow"`, `mvn test -Dgroups="unit"`

### 5. Conditional Tests
- Skip tests based on environment: `@pytest.mark.skipif(sys.platform == "win32")`
- Skip tests for missing dependencies: `@unittest.skipUnless(has_redis, "Redis not available")`
- Environment-specific tests: run database tests only when database is available
- Feature flags: enable/disable test groups based on feature availability
- Always provide a reason for conditional skips for maintainability

### 6. Retry Flaky Tests
- Configure automatic retry for known-flaky tests: `@pytest.mark.flaky(reruns=2)`
- Retry is a temporary measure; always investigate and fix the root cause
- Track flake rate per test: tests with high flake rate need priority fixing
- Maximum retries: 2-3; more retries mask real issues
- Distinguish between environment-flaky (timing, network) and code-flaky (race conditions)

---

## CI/CD Integration

### 1. Test Automation in Pipelines
- Run tests automatically on every push, pull request, and merge
- Pipeline stages: lint -> unit tests -> integration tests -> E2E tests -> deploy
- Fast feedback: fail fast on unit tests before running slow integration tests
- Parallel stages for independent test types to reduce total pipeline time
- Gate deployments on test passage: no deploy if tests fail

### 2. Parallel Test Execution
- Split test suites across multiple runners/containers for faster execution
- Strategies: split by file, split by test count, split by historical duration
- Tools: pytest-xdist (Python), Maven Surefire parallel (Java), Jest --workers (JS)
- Ensure tests are independent for safe parallel execution (no shared state)
- Monitor for interference: tests that pass individually but fail in parallel

### 3. Test Reporting
- Generate test reports in standard formats: JUnit XML, HTML, JSON
- CI integration: display test results in pull request checks and dashboards
- Historical reporting: track test pass rates, execution time, and flake rates over time
- Failure reports: clear error messages, stack traces, and screenshots for E2E failures
- Tools: Allure, ReportPortal, built-in CI reporting (GitHub Actions, GitLab CI)

### 4. Code Coverage Gates
- Set minimum coverage thresholds: fail CI if coverage drops below target
- Coverage targets: 80% line coverage as a reasonable baseline
- Incremental coverage: require new code to meet higher coverage than overall threshold
- Tools: JaCoCo (Java), coverage.py (Python), Istanbul/c8 (JavaScript), go test -cover
- Coverage reports in pull requests: show which new lines are untested

### 5. Test Environments
- **Local**: developer machine, fast iteration, may lack dependencies
- **CI**: automated pipeline, controlled environment, all dependencies available
- **Staging**: production-like environment for pre-deployment testing
- **Preview**: per-PR environments for E2E testing against feature branches
- Environment parity: minimize differences between test and production environments

---

## Language-Specific Testing

### 1. Java: JUnit 5 / TestNG / AssertJ
- **JUnit 5**: standard Java testing framework, `@Test`, `@BeforeEach`, `@ParameterizedTest`, `@Nested`
- **TestNG**: alternative with data providers, test groups, and parallel execution built in
- **AssertJ**: fluent assertion library: `assertThat(result).isNotNull().hasSize(3).contains("expected")`
- **Mockito**: de facto mocking framework: `when(mock.method()).thenReturn(value)`, `verify(mock)`
- Build tools: Maven Surefire/Failsafe plugins, Gradle test task

### 2. Python: pytest / unittest
- **pytest**: modern, concise, powerful fixtures, parametrize, plugins ecosystem
- **unittest**: built-in, class-based, `TestCase` with `setUp`/`tearDown`
- **pytest fixtures**: dependency injection for test setup, `@pytest.fixture(scope="module")`
- **pytest-cov**: coverage reporting integrated with pytest
- **pytest-mock**: thin wrapper around `unittest.mock` for cleaner mock syntax with pytest

### 3. JavaScript: Jest / Vitest / Mocha
- **Jest**: batteries-included (assertions, mocking, coverage), default for React projects
- **Vitest**: Vite-native, Jest-compatible API, faster execution, modern ESM support
- **Mocha**: flexible, requires separate assertion (Chai) and mocking (Sinon) libraries
- **Testing Library**: DOM testing utilities for React, Vue, Angular (`@testing-library/*`)
- TypeScript support: ts-jest, Vitest native, ts-mocha

### 4. Kotlin: MockK / Turbine
- **MockK**: Kotlin-native mocking library with coroutine support
- `every { mock.method() } returns value`, `coEvery` for suspend functions
- **Turbine**: testing library for Kotlin Flow, `flow.test { awaitItem() shouldBe expected }`
- JUnit 5 integration with Kotlin-specific extensions
- Kotest: Kotlin-native testing framework with property-based testing

### 5. Go: testing Package
- Built-in `testing` package: `func TestMethodName(t *testing.T) { ... }`
- Table-driven tests: idiomatic Go pattern using slices of test cases
- `t.Run()` for subtests, `t.Parallel()` for parallel execution
- Testify: assertion and mocking library (`assert.Equal`, `mock.Mock`)
- Benchmarks: `func BenchmarkMethod(b *testing.B)` for performance testing

### 6. Swift: XCTest
- Apple's testing framework for iOS, macOS, watchOS, tvOS
- `XCTestCase` with `setUp()`, `tearDown()`, `func testMethodName()`
- Async testing: `func testAsync() async throws { ... }`
- UI testing: `XCUIApplication`, `XCUIElement` for iOS UI automation
- Combine testing: expectations for publisher outputs

---

## Test Architecture

### 1. Test Project Structure
- Mirror source code directory structure in test directories
- `src/main/java/com/app/Service.java` -> `src/test/java/com/app/ServiceTest.java`
- Separate directories for unit, integration, and E2E tests
- Shared test utilities in a common test library
- Test resources (fixtures, data files) co-located with test code

### 2. Shared Test Utilities
- Common base test classes with shared setup and helper methods
- Test data builders and factories in a shared module
- Custom assertions for domain-specific validation
- Test configuration (database URLs, API keys) centralized and environment-aware
- Avoid circular dependencies between test utilities and test code

### 3. Test Configuration
- Environment-specific configuration: different databases for local vs CI vs staging
- Configuration hierarchy: defaults < environment variables < test-specific overrides
- Sensitive test credentials managed through environment variables or secrets managers
- Logging configuration: verbose for local debugging, minimal for CI
- Timeout configuration: shorter timeouts in CI to fail fast

### 4. Environment-Specific Tests
- Tag tests that require specific infrastructure: `@requires_database`, `@requires_redis`
- Skip environment-dependent tests when the dependency is unavailable
- Docker Compose or Testcontainers for consistent environments across machines
- Document environment requirements for each test category
- CI pipeline configured with all required services for full test execution

### 5. Test Documentation
- Document testing strategy: what is tested at each level, coverage expectations
- Test naming convention documented and enforced
- Troubleshooting guide for common test failures and environment issues
- New developer onboarding: how to run tests, set up environment, write new tests
- Keep test documentation close to the code (README in test directory, not a separate wiki)

### 6. Test Maintenance Strategies
- Regular test review: remove obsolete tests, update brittle tests, refactor duplicated setup
- Monitor test execution time and optimize slow tests
- Quarantine consistently failing tests: fix or remove, don't let them rot
- Track test health metrics: pass rate, flake rate, execution time trends
- Budget time for test maintenance in each sprint/iteration

---

## Anti-Patterns to Avoid

- **Chasing 100% coverage as a quality proxy**: M1's own research finding states 100% coverage does NOT mean 0 bugs — the 90→100% band catches only ~5% of latent bugs versus 70% caught in the 0→80% band, so the marginal effort is better spent elsewhere once coverage clears ~90%.
- **Using statement coverage where MC/DC is required**: statement coverage can reach 100% while individual conditions inside a compound decision never get independently exercised — for safety/aviation-grade code, MC/DC (each condition independently affecting the decision) is the correct bar, not the weaker statement metric (M1).
- **Inverting the test pyramid**: over-investing in E2E tests breaks the M2 cost ratio — at a 10-minute CI budget, E2E tests consume ~80% of CI time while representing only ~10% of the suite; a suite skewed toward E2E instead of the recommended ~70/20/10 unit/integration/E2E split burns budget on the slowest, most brittle tier.
- **Declaring a test flaky (or stable) from a single observation**: one failure or one pass provides no statistical evidence at typical flakiness rates — M3's sample-size formula `n ≥ log(0.05)/log(1-p)` requires 29 runs to detect 10% flakiness and 299 runs for 1% flakiness at 95% confidence; a single rerun's false-alarm rate is non-trivial (4.9% at 5 reruns for a truly 1%-flaky test).
- **Trusting line coverage as a proxy for test effectiveness**: M4's correlation table shows 90% line coverage typically corresponds to only 70-80% mutation score — a high line-coverage number can still hide a test suite that doesn't actually verify behavior, which is exactly the gap mutation testing (killed vs. surviving mutants) is designed to expose.
- **Running full mutation testing on every commit**: M4's cost derivation shows 1000 LOC can require 14-28 hours of mutation testing at 5-10× mutants per line of code — running the full suite on every push is infeasible; incremental mutation testing on changed files only is the correct default.
- **Using purely random property-based test generation without boundary guidance**: M5 shows purely random generation needs roughly n=46,050 samples to find a boundary condition at 99% confidence when the boundary fraction is ~0.01% — guided generation that explicitly targets boundary values (0, max, -1, empty) finds the same edge cases in far fewer runs.
- **Applying a flat coverage target across regulatory sectors**: treating STQC's 85% statement / 80% branch requirement as a ceiling instead of a floor ignores that RBI fintech and CDSCO medical-device software carry stricter defect-density targets (≤0.10 and ≤0.05 defects/KLOC respectively, M6) than the general STQC e-governance bar (≤0.20) — the sector determines the actual bar, not a single blanket coverage number.

## Response Rules
- Always recommend the appropriate test type for the specific scenario
- Provide language-specific code examples with correct framework imports and annotations
- Include both positive tests (expected behavior) and negative tests (error handling)
- Recommend test architecture that scales with the project's complexity
- Emphasize test isolation and independence in all testing advice

---

## What Not to Do
- Do not recommend E2E tests for logic that should be unit tested
- Do not mock the code under test; only mock its dependencies
- Do not write tests that depend on execution order or shared mutable state
- Do not ignore flaky tests; fix or quarantine them immediately
- Do not pursue 100% code coverage at the expense of meaningful, maintainable tests
- Do not test framework code or trivial getters/setters

---

## Output Expectations
- Provide complete, runnable test examples with imports, setup, and assertions
- Include test naming conventions appropriate for the language and framework
- Recommend test project structure matching the project's language and build tool
- Offer CI/CD integration configuration for the specific CI platform
- Include test data management strategies appropriate for the test type

---

## Skill Scope

In scope:
- Unit testing, integration testing, and end-to-end testing strategies and implementation
- Test-driven development (TDD) and behavior-driven development (BDD) workflows
- Mocking frameworks and test double patterns across languages
- Test data management, fixtures, factories, and cleanup strategies
- API testing, contract testing, and performance testing
- CI/CD test integration, reporting, and coverage gates
- Language-specific testing frameworks: JUnit, pytest, Jest, MockK, XCTest, Go testing
- Test architecture, project structure, and maintenance strategies

Out of scope:
- Security testing and penetration testing methodologies
- Mobile app testing (Appium, Espresso, XCUITest) in depth beyond XCTest basics
- Test management tools (TestRail, Zephyr, qTest)
- Manual testing processes and test case documentation
- Accessibility testing (a11y) beyond basic automated checks
- Game testing and QA processes

---

## Version
2.0.1 — Added Anti-Patterns to Avoid: coverage-as-quality-proxy, statement coverage where MC/DC required, inverted test pyramid, single-observation flakiness verdicts, line-coverage-as-effectiveness-proxy, full mutation testing on every commit, unguided property-based generation, flat cross-sector coverage targets
2.0.0
