# TDD Engineer Rule: 04 - Code Coverage and Quality Gate Framework

**Purpose:** To define the quantitative standards and automated gates that ensure our test suite provides genuine **confidence** in the correctness, reliability, and security of our codebase. We treat coverage not as a goal in itself, but as a critical diagnostic tool for identifying untested code paths and measuring the quality of our tests.

---

### 1. Philosophy: Confidence, Not Just Coverage

A 100% line coverage score can be dangerously misleading. A test can execute a line of code without making any meaningful assertion about its behavior, leading to a false sense of security. Our philosophy is built on achieving **verifiable confidence**.

We assess test suite quality using these quadrants:

* **High Coverage, High Confidence (The Goal Zone):** Code is thoroughly executed by tests with strong, specific assertions. This is our target for all critical business logic.
* **High Coverage, Low Confidence (Vanity Coverage):** This is a high-risk anti-pattern. Tests execute the code, but assertions are weak or non-existent. Our framework is designed to detect and prevent this.
* **Low Coverage, Low Confidence (Identified Risk):** Untested or partially tested code. This is an explicit acknowledgement of technical debt that must be tracked and addressed.
* **Low Coverage, High Confidence (Fallacy):** This quadrant is impossible. You cannot have confidence in code that has not been tested.

---

### 2. The Coverage Metrics Triad

To achieve true confidence, we mandate the measurement and enforcement of three distinct metrics. A high score in all three is required to pass our quality gate.

#### 2.1. Metric 1: Line & Branch Coverage
This is the foundational metric that measures the percentage of code executed by tests.

* **Definition:**
    * **Line Coverage:** Percentage of executable lines of code that were run.
    * **Branch Coverage:** Percentage of conditional branches (e.g., `if/else`, `switch` cases, ternaries) that have been taken.
* **Enforcement:** This is enforced by Jest's coverage reporter on every pull request. The build will fail if thresholds are not met.
* **Configuration (`jest.config.ts`):**

```typescript
// jest.config.ts
import type { Config } from '@jest/types';

const config: Config.InitialOptions = {
  // ... other Jest configurations
  preset: 'ts-jest',
  testEnvironment: 'node',
  
  // Enable coverage collection
  collectCoverage: true,
  
  // Specify where to collect coverage from
  collectCoverageFrom: [
    'src/**/*.ts',
    '!src/**/*.d.ts',
    '!src/index.ts', // Exclude entry point
    '!src/testing/**', // Exclude test utilities
  ],
  
  // The directory where reports are written
  coverageDirectory: 'coverage',
  
  // Use the 'text' and 'lcov' reporters for CI and detailed reports
  coverageReporters: ['text', 'lcov', 'json-summary'],
  
  // Define and enforce global coverage thresholds
  coverageThreshold: {
    global: {
      branches: 90,
      functions: 95,
      lines: 95,
      statements: 95,
    },
  },
};

export default config;
````

#### 2.2. Metric 2: Mutation Testing Score

This is the ultimate measure of **test quality**. It ensures our tests are not just running code, but are actually capable of detecting bugs.

  * **Definition:** A tool like **Stryker** modifies your source code by introducing small bugs ("mutants"), such as changing a `>` to a `<=` or removing a line of code. It then runs your tests. If your tests pass, the mutant "survived," indicating your test suite is not strong enough. The final score is the percentage of mutants that were "killed" (detected) by your tests.
  * **Enforcement:** This is run as a separate, more intensive CI stage. A low mutation score will fail the build.
  * **Configuration (`stryker.conf.json`):**

<!-- end list -->

```json
{
  "$schema": "./node_modules/@stryker-mutator/core/schema/stryker-schema.json",
  "packageManager": "npm",
  "reporters": ["html", "clear-text", "progress"],
  "testRunner": "jest",
  "coverageAnalysis": "perTest",
  "checkers": ["typescript"],
  "mutate": [
    "src/**/*.ts",
    "!src/**/*.d.ts",
    "!src/index.ts",
    "!src/testing/**"
  ],
  "dashboard": {
    "project": "your-project-name",
    "module": "server"
  },
  "thresholds": {
    "high": 95,
    "low": 85,
    "break": 85
  }
}
```

#### 2.3. Metric 3: Complexity-Weighted Coverage

This metric ensures our testing efforts are focused on the riskiest parts of the codebase.

  * **Definition:** We automatically flag modules that exhibit both high cyclomatic complexity (a measure of decision paths) and insufficient test coverage. A simple module with low coverage is less concerning than a complex one with the same low coverage.
  * **Enforcement:** A custom script in our CI pipeline analyzes the `coverage/json-summary.json` output from Jest and complexity reports from a tool like `eslint-plugin-sonarjs`. It fails the build if "High-Risk Hotspots" are detected.
  * **High-Risk Hotspot Definition:** Any file where `cyclomatic_complexity > 10` AND `branch_coverage < 90%`.

-----

### 3\. CI/CD Pipeline Enforcement ⚙️

These standards are worthless without automated enforcement. The **SPARC-DevOps-Engineer** is responsible for implementing these quality gates in the CI/CD pipeline.

```yaml
# .github/workflows/quality-check.yml
jobs:
  test-and-coverage:
    name: Unit Tests & Coverage Gate
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: 18
          cache: 'npm'
      - run: npm ci
      - name: Run tests and enforce coverage thresholds
        run: npm test -- --coverage
      - name: Upload coverage report
        uses: actions/upload-artifact@v3
        with:
          name: coverage-report
          path: coverage/

  mutation-testing:
    name: Mutation Testing Quality Gate
    needs: test-and-coverage # Runs after basic tests pass
    if: github.ref == 'refs/heads/main' # Run only on merge to main
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
        with: { fetch-depth: 0 } # Required for Stryker's diffing
      - uses: actions/setup-node@v3
      - run: npm ci
      - name: Run mutation tests and enforce threshold
        run: npx stryker run
        env:
          STRYKER_DASHBOARD_API_KEY: ${{ secrets.STRYKER_DASHBOARD_API_KEY }}
```

### 4\. Justifiable Exceptions

In rare cases, achieving 100% coverage is impractical or provides no value (e.g., defensive error handling for states that are virtually impossible to create). In these scenarios, an exception can be made using a coverage-ignore comment.

  * **Requirement:** Any use of `/* istanbul ignore next */`, `/* istanbul ignore if */`, or `/* istanbul ignore else */` **must** be accompanied by a Memory Bank entry justifying the exception. This creates a permanent, auditable record of the decision and the accepted risk.

-----

### 5\. Memory Bank Integration

Documenting exceptions is critical for maintaining high standards and making risk assessment transparent.

**Memory Bank Template: Coverage Exception (`coverage-exception.yml`)**

```yaml
#---
# File: memory/tdd-engineer/coverage-exception-db-connection-error.yml
#---
apiVersion: sparc.dev/v1
kind: MemoryEntry
metadata:
  domain: tdd-engineer
  layer: quality
  title: "Coverage Exception for Catastrophic Database Connection Error"
  author: "SPARC-TDD-Engineer"
  timestamp: "2025-08-05T23:55:00Z"
spec:
  summary: "A `/* istanbul ignore next */` comment was added to the final catch block in the global database connection utility. This block handles catastrophic errors (e.g., malformed protocol packets, hardware failure) that are not feasible to simulate in our automated test environment."
  type: "Coverage Exception"
  
  content: |
    ### Code Location:
    `src/lib/database.ts`, line 85

    ### Justification:
    The `catch (error)` block in the `connectWithRetry` function is designed to handle unrecoverable connection failures. Simulating these low-level driver or network errors would require a highly complex and brittle test setup for negligible benefit. The application's intended behavior in this scenario is to crash and be restarted by the container orchestrator (Kubernetes), which is a platform-level guarantee, not a testable application logic path.

    ### Risk Assessment:
    - **Likelihood:** Extremely Low.
    - **Impact:** High (service outage).
    - **Mitigation:** The risk is accepted because the recovery mechanism (pod restart) is managed and guaranteed by the production infrastructure, not by the application code itself. The purpose of the code is to log the fatal error and exit, which is a one-line operation.

  associations:
    - "devops-standard-liveness-probe"
  
  status: "Approved"
```
