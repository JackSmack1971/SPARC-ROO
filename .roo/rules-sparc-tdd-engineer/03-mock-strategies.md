# TDD Engineer Rule: 03 - Advanced Mocking and Isolation Framework

**Purpose:** To establish a formal framework for isolating components under test. Effective isolation using **Test Doubles** (mocks, stubs, fakes, etc.) is the cornerstone of fast, deterministic, and maintainable unit tests. This framework defines not just the *what*, but the *why* and *when* for each isolation strategy.

---

### 1. Philosophy: Isolation as a Reflection of Architecture

Mocking is not a testing trick; it is the practical application of good architectural principles, specifically the **Dependency Inversion Principle**.

* **Interfaces are Contracts for Mocking:** A component's dependencies should be defined by abstract interfaces, not concrete implementations. The **SPARC-Architect** designs these interfaces. The **SPARC-TDD-Engineer** uses these interfaces as "seams" to inject test doubles, allowing a component's internal logic to be tested in complete isolation from its collaborators. If a component is difficult to test or mock, it is a strong signal of a design flaw (tight coupling).
* **The Fidelity-Isolation Trade-off:** The choice of a test double is a deliberate trade-off. We balance **Test Fidelity** (how closely the double behaves like the real component) against **Test Isolation & Speed** (how fast and deterministic the test is). 
* **Behavior vs. State Verification:** Our strategy is guided by what we are trying to prove.
    * **State Verification:** We check that the System Under Test (SUT) produces the correct output or changes its state correctly based on inputs. This typically uses **Stubs** and **Fakes**.
    * **Behavior Verification:** We check that the SUT calls its dependencies correctly. This uses **Mocks** and **Spies**.

---

### 2. Test Double Taxonomy and Implementation

We use a clear, consistent classification of test doubles. The `jest-mock-extended` library is preferred for its strong type safety.

#### 2.1. Stub
A **Stub** provides canned responses to method calls. It's used when the test needs a dependency to return specific data to proceed. It does not care about how or if it's called.

* **Use Case:** Providing data from a repository, configuration values, or a fixed response from an external API client.

```typescript
// Example: Stubbing a configuration service
import { IConfigService } from '../interfaces/IConfigService';
import { mock } from 'jest-mock-extended';

// Create a stub for the config service
const configStub = mock<IConfigService>();

// Program the stub to return a specific value for a method call
configStub.get.calledWith('Feature.NewCheckout.IsEnabled').mockReturnValue(true);
configStub.get.calledWith('API.Timeout.ms').mockReturnValue(5000);

// Now, any service that depends on IConfigService can be tested with predictable feature flags.
const myService = new MyService(configStub);
````

#### 2.2. Spy

A **Spy** is a Stub that also records information about how it was called (e.g., number of calls, arguments passed). It is used to verify **indirect outputs** of the SUT. An indirect output is when the SUT doesn't return a value but instead triggers an action in a dependency.

  * **Use Case:** Verifying that a logger was called, an email was sent, or an analytics event was dispatched.

<!-- end list -->

```typescript
// Example: Spying on a notification service
import { NotificationService } from '../services/NotificationService';

// Real services can be spied on to observe their behavior
const notificationService = new NotificationService();
const emailSpy = jest.spyOn(notificationService, 'sendEmail');

const user = { email: 'test@example.com', name: 'Test User' };
await registerUser(user, notificationService); // Function that uses the service

// Assert that the spy was called correctly
expect(emailSpy).toHaveBeenCalledTimes(1);
expect(emailSpy).toHaveBeenCalledWith({
  to: 'test@example.com',
  subject: 'Welcome!',
  body: 'Hello Test User, welcome to our platform.',
});
```

#### 2.3. Mock

A **Mock** is an object with pre-programmed expectations. Unlike a Spy, a Mock will cause the test to fail if it is not used in the exact way it expects. Mocks are about **strict behavior verification**.

  * **Use Case:** Verifying a critical interaction protocol, such as a payment gateway where methods must be called in a specific order (`authorize`, `capture`) and exactly once.

<!-- end list -->

```typescript
import { IPaymentGateway } from '../interfaces/IPaymentGateway';
import { mock } from 'jest-mock-extended';

// Create a strict mock for the payment gateway
const paymentMock = mock<IPaymentGateway>();

// The test will now fail if authorize is not called, or if capture is called before authorize.
// This is often handled by the mocking library's setup or verification methods.
// In jest-mock-extended, the assertions serve this purpose.

const order = { amount: 100, creditCardToken: 'tok_visa' };
const paymentProcessor = new PaymentProcessor(paymentMock);

await paymentProcessor.process(order);

// Assert the strict behavioral contract
expect(paymentMock.authorize).toHaveBeenCalledWith(order.amount, order.creditCardToken);
expect(paymentMock.capture).toHaveBeenCalledTimes(1);
```

#### 2.4. Fake

A **Fake** is a full, working implementation of an interface, but it takes a shortcut that makes it unsuitable for production (e.g., using an in-memory database). Fakes provide the highest fidelity while maintaining isolation from external systems.

  * **Use Case:** Replacing a database repository, a file system, or a complex external API client for more realistic integration-style unit tests.

<!-- end list -->

```typescript
// Example: A Fake repository for testing services without a real database
import { IUserRepository, User } from '../interfaces/IUserRepository';

export class FakeUserRepository implements IUserRepository {
  private users = new Map<string, User>();
  private nextId = 1;

  async findById(id: string): Promise<User | null> {
    return this.users.get(id) || null;
  }

  async findByEmail(email: string): Promise<User | null> {
    for (const user of this.users.values()) {
      if (user.email === email) return user;
    }
    return null;
  }

  async create(data: Omit<User, 'id'>): Promise<User> {
    const id = `user-${this.nextId++}`;
    const newUser = { id, ...data };
    this.users.set(id, newUser);
    return newUser;
  }
}

// In a test:
const fakeUserRepo = new FakeUserRepository();
const userService = new UserService(fakeUserRepo); // UserService is unaware it's using a Fake

// The test can now verify complex interactions without mocking individual methods
const user = await userService.registerUser({ name: 'Fake User', email: 'fake@test.com' });
const foundUser = await userService.getUserById(user.id);
expect(foundUser).toEqual(user);
```

### 3\. The Mocking Decision Framework

Choose the right test double by answering these questions in order:

| Question                                                          | If Yes...                                                              | If No...                                                    |
| ----------------------------------------------------------------- | ---------------------------------------------------------------------- | ----------------------------------------------------------- |
| 1. Are you verifying an interaction/collaboration (a method call)?| Go to Q2.                                                              | Use a **Stub** to provide values.                           |
| 2. Does the test need to fail if the interaction is wrong/missing?| Use a **Mock** for strict behavioral contracts.                        | Use a **Spy** to observe the interaction without enforcement. |
| 3. Does the dependency require complex state logic for your test? | Create a **Fake** for high-fidelity simulation.                        | Use a **Stub** or **Mock** for simpler interactions.          |

-----

### 4\. Advanced Scenarios and Security

  * **Mocking Time:** For logic involving timeouts, token expirations, or scheduling, never rely on `setTimeout` in tests. Control the clock directly.
    ```typescript
    // Test for a JWT expiration function
    it('should identify an expired token', () => {
        jest.useFakeTimers();
        const now = Date.now();
        const expiredToken = createToken({ payload: {}, expiresIn: '1h' });

        // Advance time by 1 hour and 1 second
        jest.advanceTimersByTime(60 * 60 * 1000 + 1000);
        
        expect(isTokenValid(expiredToken)).toBe(false);
        
        jest.useRealTimers(); // Clean up
    });
    ```
  * **Security Context Mocking:** To test authorization, create a stub that can simulate different user roles.
    ```typescript
    const adminContextStub = mock<ISecurityContext>();
    adminContextStub.getCurrentUser.mockResolvedValue({ id: 'admin-001', roles: ['admin'] });

    const regularUserContextStub = mock<ISecurityContext>();
    regularUserContextStub.getCurrentUser.mockResolvedValue({ id: 'user-123', roles: ['user'] });

    // Test 1: with adminContextStub, the action should succeed
    // Test 2: with regularUserContextStub, the action should throw AuthorizationError
    ```

### 5\. Memory Bank Integration

Decisions on creating complex Fakes or choosing a specific mocking approach for a critical component must be documented.

**Documentation Triggers:**

  * **A Fake is Created:** When a non-trivial Fake (e.g., `FakePaymentGateway`, `FakeS3Client`) is implemented.
  * **Complex Mocking Strategy:** When a component requires a chain of mocks or a particularly clever use of spies to be tested effectively.

**Memory Bank Template: Mocking Decision (`mocking-decision.yml`)**

```yaml
#---
# File: memory/tdd-engineer/mocking-decision-fake-payment-gateway.yml
#---
apiVersion: sparc.dev/v1
kind: MemoryEntry
metadata:
  domain: tdd-engineer
  layer: isolation
  title: "Decision to Implement a Fake Payment Gateway"
  author: "SPARC-TDD-Engineer"
  timestamp: "2025-08-05T23:35:10Z"
spec:
  summary: "To accurately test the `OrderFulfillmentSaga`, we chose to implement a high-fidelity Fake for the `IPaymentGateway` interface instead of using simple stubs. This allows us to simulate success, failure, and refund workflows."
  type: "Mocking Decision"
  
  content: |
    ### Problem:
    The `OrderFulfillmentSaga` orchestrates a multi-step process: `authorizePayment`, `reserveInventory`, `capturePayment`, and `shipOrder`. Simple stubs for the payment gateway could not simulate stateful transitions (e.g., you cannot `capture` a payment that hasn't been `authorized`). This made it difficult to test failure scenarios, such as when payment capture fails after inventory has already been reserved.

    ### Decision:
    We will implement `FakePaymentGateway`, an in-memory Fake that maintains a state machine for payments (`pending`, `authorized`, `captured`, `failed`). 

    ### Rationale:
    1.  **High Fidelity:** The Fake will accurately model the stateful nature of the real payment gateway, allowing us to test complex sagas and compensation logic (e.g., releasing inventory if payment capture fails).
    2.  **Developer Experience:** Developers testing services that use the payment gateway can use the Fake without needing to configure multiple stubs for each test case.
    3.  **Risk Reduction:** Provides much higher confidence in our critical payment processing logic compared to what simple stubs could offer. The higher up-front effort is justified by the reduction in business risk.

  associations:
    - "tdd-pattern-fake"
    - "architect-pattern-saga"
  
  status: "Implemented"
```