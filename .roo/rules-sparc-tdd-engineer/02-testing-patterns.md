# TDD Engineer Rule: 02 - Test-Driven Development & Quality Assurance Framework

**Purpose:** To establish a comprehensive, automated, and design-centric framework for testing. In the SPARC methodology, testing is not a final validation step but a core **design activity** that drives quality, security, and architectural integrity from the outset. Tests are the executable, living specifications of the system.

---

### 1. Philosophy: Testing as Design

The **SPARC-TDD-Engineer** operates on the principle that a test is the first-use case of any piece of code. This "test-first" mindset drives the **Specification** and **Refinement** phases of SPARC.

* **Tests as Executable Specifications:** A test suite defines what the system *does*, not how it does it. It is the most accurate and reliable form of documentation.
* **The Testing Pyramid as a SPARC Contract:**
    * **Unit Tests (TDD Engineer):** Fast, isolated tests verifying a single class or function. These form the base of our quality pyramid.
    * **Integration Tests (TDD Engineer):** Verifying interactions between internal components (e.g., service-to-database, service-to-cache).
    * **Contract Tests (TDD Engineer & Architect):** Guaranteeing that service-to-service APIs adhere to agreed-upon contracts, preventing breaking changes in a microservices environment.
    * **End-to-End (E2E) Tests (DevOps Engineer):** Verifying complete user flows in a production-like environment. They are slow, expensive, and used sparingly for critical paths only. 
* **Shift-Left Security:** The TDD-Engineer is the first line of defense. We write tests that actively try to break our code by probing for common security vulnerabilities.

### 2. The Test Specification Framework

To ensure consistency and clarity, all tests follow a Behavior-Driven Development (BDD) structure. This is enforced through a standard testing framework.

```typescript
// src/testing/TestSpecification.ts
import { ZodSchema } from 'zod';

/**
 * Defines the contract for a single test case, structured in BDD format.
 */
export interface TestCase<TContext, TResult> {
  readonly description: string;
  readonly context?: TContext;
  readonly given: (context: TContext) => Promise<{ sut: any; dependencies: any }>;
  readonly when: (sut: any, dependencies: any) => Promise<TResult>;
  readonly then: (result: TResult, context: TContext, dependencies: any) => Promise<void>;
  readonly securityChecks?: SecurityCheck[];
}

/**
 * Represents a security check to be performed within a test case.
 */
export interface SecurityCheck {
  type: 'authorization' | 'inputValidation' | 'resourceOwnership';
  description: string;
  isViolationExpected: boolean;
  getPayload?: (context: any) => any;
}

/**

 * A lightweight test runner to execute a suite of test cases.
 */
export class TestSuiteRunner {
  public static async run<TContext, TResult>(suiteName: string, cases: TestCase<TContext, TResult>[]) {
    describe(suiteName, () => {
      for (const testCase of cases) {
        it(testCase.description, async () => {
          // ARRANGE
          const { sut, dependencies } = await testCase.given(testCase.context);

          // ACT
          const result = await testCase.when(sut, dependencies);

          // ASSERT
          await testCase.then(result, testCase.context, dependencies);
        });
      }
    });
  }
}
````

### 3\. Core Testing Patterns

#### 3.1. Unit Testing Pattern

Unit tests are focused on a single module ("System Under Test" or SUT) in complete isolation. All external dependencies (classes, APIs, databases) **MUST** be mocked.

```typescript
// Example: src/services/UserService.test.ts
import { UserService } from './UserService';
import { UserRepository } from '../repositories/UserRepository';
import { TestSuiteRunner, TestCase } from '../testing/TestSpecification';
import { mock } from 'jest-mock-extended'; // A type-safe mocking library

// Define the context for this test suite
interface UserTestContext {
  userId: string;
  expectedUser: { id: string; name: string; email: string };
}

// Mock the dependencies
const mockUserRepo = mock<UserRepository>();

const testCases: TestCase<UserTestContext, any>[] = [
  {
    description: 'should return a user when a valid ID is provided',
    context: {
      userId: 'user-123',
      expectedUser: { id: 'user-123', name: 'Jane Doe', email: 'jane.doe@example.com' },
    },
    given: async (ctx) => {
      // ARRANGE: Set up the mock to return a specific user
      mockUserRepo.findById.calledWith(ctx.userId).mockResolvedValue(ctx.expectedUser);
      const sut = new UserService(mockUserRepo);
      return { sut, dependencies: { mockUserRepo } };
    },
    when: async (sut, _) => {
      // ACT: Call the method under test
      return sut.getUserById('user-123');
    },
    then: async (result, ctx, deps) => {
      // ASSERT: Verify the outcome
      expect(deps.mockUserRepo.findById).toHaveBeenCalledWith('user-123');
      expect(result).toEqual(ctx.expectedUser);
    },
  },
  {
    description: 'should throw a NotFoundError for a non-existent user ID',
    context: { userId: 'user-non-existent' },
    given: async (ctx) => {
      mockUserRepo.findById.calledWith(ctx.userId).mockResolvedValue(null);
      const sut = new UserService(mockUserRepo);
      return { sut, dependencies: { mockUserRepo } };
    },
    when: async (sut, _) => {
      return sut.getUserById('user-non-existent');
    },
    then: async (result, _ctx, _deps) => {
      // When expecting an error, the 'when' block is wrapped in expect().rejects
      await expect(result).rejects.toThrow('User not found');
    },
  },
];

TestSuiteRunner.run('UserService Unit Tests', testCases);
```

#### 3.2. Integration Testing Pattern

Integration tests verify the interaction between two or more components. We use `testcontainers` to spin up live dependencies (like a database) in Docker for each test run, ensuring high-fidelity testing without mocks.

```typescript
// src/repositories/UserRepository.integration.test.ts
import { PrismaClient } from '@prisma/client';
import { PostgreSqlContainer, StartedPostgreSqlContainer } from 'testcontainers';
import { exec } from 'node:child_process';
import { UserRepository } from './UserRepository';

describe('UserRepository Integration Tests', () => {
  let dbContainer: StartedPostgreSqlContainer;
  let prisma: PrismaClient;
  let userRepository: UserRepository;

  beforeAll(async () => {
    // Start a real PostgreSQL database in a Docker container
    dbContainer = await new PostgreSqlContainer().start();
    
    // Construct the database URL for the container
    const databaseUrl = dbContainer.getConnectionUri();
    
    // Run Prisma migrations against the test database
    await new Promise((resolve, reject) => {
      exec(`DATABASE_URL=${databaseUrl} npx prisma migrate deploy`, (err, stdout) => {
        if (err) return reject(err);
        resolve(stdout);
      });
    });

    prisma = new PrismaClient({ datasources: { db: { url: databaseUrl } } });
    await prisma.$connect();
    userRepository = new UserRepository(prisma);
  }, 60000); // Increase timeout for container startup

  afterAll(async () => {
    await prisma?.$disconnect();
    await dbContainer?.stop();
  });
  
  beforeEach(async () => {
    // Clean the database before each test
    await prisma.user.deleteMany({});
  });

  it('should create a user and find it by ID', async () => {
    // ARRANGE: Define user data
    const userData = { name: 'John Smith', email: 'john.smith@test.com' };

    // ACT: Call the repository method to create a user
    const createdUser = await userRepository.create(userData);

    // ASSERT: Verify the user was created correctly
    expect(createdUser.id).toBeDefined();
    expect(createdUser.name).toBe(userData.name);

    // ACT 2: Find the user by its new ID
    const foundUser = await userRepository.findById(createdUser.id);
    
    // ASSERT 2: Verify the found user matches the created one
    expect(foundUser).toEqual(createdUser);
  });
});
```

#### 3.3. Security Testing Patterns

Security tests are integrated directly into our unit and integration test suites. We test for negative security outcomes.

```typescript
// src/controllers/DocumentController.security.test.ts
// ... imports and setup for an Express controller test

it('should deny access when a user tries to access a document they do not own', async () => {
    // ARRANGE
    const maliciousUserId = 'user-attacker-456';
    const documentOwnedByAnother = { id: 'doc-123', ownerId: 'user-owner-789' };
    
    // Mock the session to simulate the attacker being logged in
    mockRequest.session = { userId: maliciousUserId };
    
    // Mock the repository to return the document
    mockDocumentRepo.findById.mockResolvedValue(documentOwnedByAnother);
    
    // ACT & ASSERT
    // The controller should throw an AuthorizationError (or return a 403)
    await expect(documentController.getDocument(mockRequest, mockResponse)).rejects.toThrow(AuthorizationError);
});

it('should reject requests with potential NoSQL injection in query parameters', async () => {
    // ARRANGE
    // A payload attempting to use a NoSQL operator to bypass logic
    mockRequest.query = { id: { '$ne': 'some-value' } };

    // ACT & ASSERT
    // Our validation layer (e.g., Zod) should catch this before it reaches the service
    await expect(documentController.getDocument(mockRequest, mockResponse)).rejects.toThrow(ValidationError);
});
```

### 4\. Memory Bank Integration

Key testing strategies and complex test environment setups are documented in the Memory Bank.

**Documentation Triggers:**

  * **New Service Contract:** When a new service-to-service interaction is defined, a `contract-test.yml` is created.
  * **Complex Test Setup:** When a test requires a unique environment (e.g., multiple Docker containers, a message queue), it's documented in a `testing-strategy.yml`.
  * **Novel Security Threat:** When a new potential vulnerability is identified, a `security-test-case.yml` is created to document the threat and the test that covers it.

**Memory Bank Template: Testing Strategy (`testing-strategy.yml`)**

```yaml
#---
# File: memory/tdd-engineer/testing-strategy-order-processing-flow.yml
#---
apiVersion: sparc.dev/v1
kind: MemoryEntry
metadata:
  domain: tdd-engineer
  layer: integration
  title: "Integration Test Strategy for Asynchronous Order Processing"
  author: "SPARC-TDD-Engineer"
  timestamp: "2025-08-05T23:25:00Z"
spec:
  summary: "Defines the integration test setup for the full order processing flow, which involves a REST endpoint, a RabbitMQ message queue, and a PostgreSQL database."
  type: "Testing Strategy"
  
  content: |
    ### Problem:
    The order processing flow is asynchronous. A user hits an API endpoint, which publishes a message to a queue. A worker consumes the message and writes the result to a database. Verifying this entire flow is complex and cannot be done with a simple unit test.

    ### Test Environment Setup:
    1.  **`testcontainers`** will be used to orchestrate the environment.
    2.  A **`PostgreSqlContainer`** will provide the database.
    3.  A **`GenericContainer`** with the `rabbitmq:3-management` image will provide the message queue.
    4.  The test will directly invoke the API controller, then poll the database for the expected result with a timeout.

    ### Test Flow:
    1.  `beforeAll`: Start Postgres and RabbitMQ containers. Run migrations.
    2.  `test`: 
        a. Send a valid POST request to `/api/v1/orders`.
        b. Assert a 202 Accepted response.
        c. Poll `orderRepository.findById()` in a loop with a 5-second timeout.
        d. Once the order appears, assert its status is `PROCESSED`.
    3.  `afterAll`: Stop all containers.

  associations:
    - "tech-constraint-rabbitmq"
    - "tech-constraint-postgresql"
  
  status: "Implemented"
```