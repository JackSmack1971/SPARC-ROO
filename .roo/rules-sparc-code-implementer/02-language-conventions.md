# Language Conventions and Code Standards

## Philosophy of Code Aesthetics in SPARC

### The Art of Readable Implementation
Code is poetry written for both machines and humans. In SPARC methodology, **code aesthetics matter as much as functionality** because beautiful, consistent code enables rapid comprehension, reduces cognitive load, and accelerates the development cycle through all phases.

**Core aesthetic principles:**
- **Clarity over cleverness**: Prefer explicit, readable code over compact but obscure implementations
- **Consistency over personal preference**: Follow established patterns to reduce mental context switching
- **Intention-revealing names**: Variable and function names should tell a story about business logic
- **Vertical density**: Related concepts should be grouped together with appropriate whitespace separation
- **Horizontal simplicity**: Avoid deeply nested structures; prefer flat, linear code flow

### Language-Specific Philosophy for TypeScript/JavaScript

**TypeScript as the Foundation Language**
TypeScript serves as our primary implementation language because it bridges the gap between rapid prototyping and enterprise-grade reliability. Every piece of production code must leverage TypeScript's type system to prevent runtime errors and enable confident refactoring.

**Key TypeScript Principles:**
```typescript
// GOOD: Explicit types that tell a story
interface UserRegistrationRequest {
  readonly email: EmailAddress;
  readonly password: SecurePassword;
  readonly profile: UserProfile;
  readonly agreedToTerms: boolean;
  readonly registrationSource: 'web' | 'mobile' | 'api';
}

// AVOID: Generic types that hide business meaning
interface UserData {
  email: string;
  password: string;
  profile: any;
  terms: boolean;
  source: string;
}
```

**Functional Programming Bias**
While TypeScript supports multiple paradigms, SPARC methodology favors **functional programming patterns** because they enhance testability, reduce side effects, and align with security-by-design principles.

```typescript
// PREFERRED: Pure functions with explicit dependencies
const calculateUserSubscriptionCost = (
  user: User,
  plan: SubscriptionPlan,
  discounts: readonly Discount[]
): SubscriptionCost => {
  const baseCost = plan.monthlyPrice;
  const totalDiscount = discounts.reduce((sum, discount) => 
    sum + calculateDiscount(discount, baseCost), 0
  );
  
  return {
    baseCost,
    discountAmount: totalDiscount,
    finalCost: Math.max(0, baseCost - totalDiscount),
    currency: plan.currency
  };
};

// AVOID: Methods with hidden dependencies and side effects
class UserService {
  calculateCost(userId: string): number {
    const user = this.userRepository.findById(userId); // Hidden dependency
    const plan = this.getCurrentPlan(user); // Hidden state
    this.auditLog.record('cost_calculation', userId); // Side effect
    return plan.price - this.getDiscount(user); // Implicit business logic
  }
}
```

## TypeScript/JavaScript Conventions

### Type System Architecture

**Domain-Driven Type Definitions**
All types should reflect business domain concepts rather than technical implementation details:

```typescript
// Domain types that reflect business reality
type EmailAddress = string & { readonly __brand: 'EmailAddress' };
type UserId = string & { readonly __brand: 'UserId' };
type SecurePassword = string & { readonly __brand: 'SecurePassword' };

// Brand type creation helpers
const createEmailAddress = (email: string): EmailAddress => {
  if (!isValidEmail(email)) {
    throw new Error(`Invalid email format: ${email}`);
  }
  return email as EmailAddress;
};

const createUserId = (id: string): UserId => {
  if (!UUID_REGEX.test(id)) {
    throw new Error(`Invalid user ID format: ${id}`);
  }
  return id as UserId;
};

// Usage in business logic
interface User {
  readonly id: UserId;
  readonly email: EmailAddress;
  readonly profile: UserProfile;
  readonly createdAt: Timestamp;
  readonly lastLoginAt: Timestamp | null;
}
```

**Error Handling Type Patterns**
Implement Result types for explicit error handling:

```typescript
// Result type for operations that can fail
type Result<T, E = Error> = 
  | { readonly success: true; readonly data: T }
  | { readonly success: false; readonly error: E };

// Usage in service functions
const registerUser = async (
  request: UserRegistrationRequest
): Promise<Result<User, RegistrationError>> => {
  try {
    const validatedRequest = await validateRegistrationRequest(request);
    const hashedPassword = await hashPassword(validatedRequest.password);
    const user = await userRepository.create({
      ...validatedRequest,
      password: hashedPassword
    });
    
    return { success: true, data: user };
  } catch (error) {
    if (error instanceof ValidationError) {
      return { success: false, error: new RegistrationError('VALIDATION_FAILED', error.message) };
    }
    if (error instanceof DuplicateEmailError) {
      return { success: false, error: new RegistrationError('EMAIL_EXISTS', 'Email already registered') };
    }
    
    return { success: false, error: new RegistrationError('INTERNAL_ERROR', 'Registration failed') };
  }
};

// Caller handling
const handleRegistration = async (req: Request, res: Response) => {
  const result = await registerUser(req.body);
  
  if (result.success) {
    res.status(201).json({ user: result.data });
  } else {
    const statusCode = getStatusCodeForError(result.error);
    res.status(statusCode).json({ error: result.error.message });
  }
};
```

**Configuration and Environment Types**
All configuration must be type-safe and validated at startup:

```typescript
// Configuration schema with runtime validation
interface AppConfig {
  readonly server: {
    readonly port: number;
    readonly host: string;
    readonly cors: {
      readonly origins: readonly string[];
      readonly credentials: boolean;
    };
  };
  readonly database: {
    readonly url: string;
    readonly maxConnections: number;
    readonly ssl: boolean;
  };
  readonly auth: {
    readonly jwtSecret: string;
    readonly tokenExpiryHours: number;
    readonly refreshTokenExpiryDays: number;
  };
  readonly external: {
    readonly paymentService: {
      readonly apiKey: string;
      readonly baseUrl: string;
      readonly timeoutMs: number;
    };
  };
}

// Runtime validation using Zod or similar
import { z } from 'zod';

const configSchema = z.object({
  server: z.object({
    port: z.number().min(1).max(65535),
    host: z.string().min(1),
    cors: z.object({
      origins: z.array(z.string().url()),
      credentials: z.boolean()
    })
  }),
  database: z.object({
    url: z.string().url(),
    maxConnections: z.number().positive(),
    ssl: z.boolean()
  }),
  auth: z.object({
    jwtSecret: z.string().min(32),
    tokenExpiryHours: z.number().positive(),
    refreshTokenExpiryDays: z.number().positive()
  }),
  external: z.object({
    paymentService: z.object({
      apiKey: z.string().min(1),
      baseUrl: z.string().url(),
      timeoutMs: z.number().positive()
    })
  })
});

// Configuration loading with validation
const loadConfig = (): AppConfig => {
  const rawConfig = {
    server: {
      port: parseInt(process.env.PORT || '3000'),
      host: process.env.HOST || 'localhost',
      cors: {
        origins: (process.env.CORS_ORIGINS || '').split(','),
        credentials: process.env.CORS_CREDENTIALS === 'true'
      }
    },
    // ... other config sections
  };

  const result = configSchema.safeParse(rawConfig);
  
  if (!result.success) {
    throw new Error(`Configuration validation failed: ${result.error.message}`);
  }
  
  return result.data;
};
```

### Function and Variable Naming Conventions

**Intention-Revealing Function Names**
Function names should clearly communicate both the action and the business context:

```typescript
// GOOD: Names that reveal business intent
const calculateMonthlySubscriptionCost = (plan: SubscriptionPlan): Money => { /* */ };
const validateUserEmailAddress = (email: string): ValidationResult => { /* */ };
const generateSecurePasswordHash = (password: string): Promise<HashedPassword> => { /* */ };
const sendWelcomeEmailToNewUser = (user: User): Promise<EmailDeliveryResult> => { /* */ };

// AVOID: Generic names that hide business meaning
const calculate = (plan: any): number => { /* */ };
const validate = (input: string): boolean => { /* */ };
const hash = (data: string): Promise<string> => { /* */ };
const send = (user: any): Promise<any> => { /* */ };
```

**Variable Naming Patterns**
Variables should follow consistent patterns that indicate their purpose and scope:

```typescript
// Constants: SCREAMING_SNAKE_CASE for module-level constants
const MAX_LOGIN_ATTEMPTS = 5;
const DEFAULT_SESSION_TIMEOUT_MINUTES = 30;
const SUPPORTED_IMAGE_FORMATS = ['jpg', 'png', 'webp'] as const;

// Collections: Plural nouns with descriptive qualifiers
const activeUserSessions = new Map<UserId, UserSession>();
const pendingEmailVerifications = new Set<EmailAddress>();
const failedLoginAttemptsByEmail = new Map<EmailAddress, number>();

// Booleans: is/has/can/should prefixes
const isUserEmailVerified = user.emailVerifiedAt !== null;
const hasUserExceededLoginAttempts = getLoginAttempts(user.email) >= MAX_LOGIN_ATTEMPTS;
const canUserAccessPremiumFeatures = user.subscriptionTier === 'premium';
const shouldSendPasswordResetEmail = isUserEmailVerified && !hasUserExceededLoginAttempts;

// Async operations: Descriptive action verbs
const userRegistrationPromise = registerNewUser(registrationData);
const emailDeliveryPromise = sendVerificationEmail(user.email);
const passwordHashingPromise = generateSecureHash(password);

// Event handlers: on + PastTense action
const onUserRegistrationCompleted = (user: User) => { /* */ };
const onPaymentProcessingFailed = (error: PaymentError) => { /* */ };
const onPasswordResetRequested = (email: EmailAddress) => { /* */ };
```

**Class and Interface Naming**
Follow consistent patterns that reflect architectural roles:

```typescript
// Domain entities: Business nouns
class User { /* */ }
class SubscriptionPlan { /* */ }
class PaymentTransaction { /* */ }

// Value objects: Descriptive compound nouns
class EmailAddress { /* */ }
class MonetaryAmount { /* */ }
class DateTimeRange { /* */ }

// Services: Domain + Service suffix
class UserRegistrationService { /* */ }
class PaymentProcessingService { /* */ }
class EmailDeliveryService { /* */ }

// Repositories: Entity + Repository suffix
interface UserRepository { /* */ }
interface SubscriptionPlanRepository { /* */ }
interface PaymentTransactionRepository { /* */ }

// Events: Entity + PastTense action + Event suffix
class UserRegisteredEvent { /* */ }
class PaymentProcessedEvent { /* */ }
class PasswordResetRequestedEvent { /* */ }

// Commands: Action + Entity + Command suffix
class RegisterUserCommand { /* */ }
class ProcessPaymentCommand { /* */ }
class ResetPasswordCommand { /* */ }

// Queries: Get/Find + Entity + Query suffix
class GetUserByIdQuery { /* */ }
class FindActiveSubscriptionsQuery { /* */ }
class SearchUsersQuery { /* */ }
```

### Code Structure and Organization Patterns

**File Organization Within 500-Line Principle**
Every source file must remain under 500 lines to maintain cognitive clarity and enable rapid comprehension:

```typescript
// GOOD: Focused file with single responsibility
// src/domain/user/UserRegistrationService.ts (~ 200 lines)
export class UserRegistrationService {
  constructor(
    private readonly userRepository: UserRepository,
    private readonly emailService: EmailDeliveryService,
    private readonly passwordHasher: PasswordHasher
  ) {}

  async registerUser(request: UserRegistrationRequest): Promise<Result<User, RegistrationError>> {
    // Implementation focused on single business capability
  }

  private async validateRegistrationRequest(request: UserRegistrationRequest): Promise<void> {
    // Focused validation logic
  }

  private async checkEmailAvailability(email: EmailAddress): Promise<void> {
    // Single-purpose helper method
  }
}

// If file exceeds 500 lines, split by responsibility:
// src/domain/user/UserRegistrationService.ts        (Core service)
// src/domain/user/UserRegistrationValidator.ts      (Validation logic)
// src/domain/user/UserRegistrationNotifier.ts       (Notification logic)
```

**Module Boundary Patterns**
Organize code into cohesive modules with clear boundaries:

```typescript
// src/domain/user/index.ts - Public API for user domain
export { User } from './User';
export { UserRepository } from './UserRepository';
export { UserRegistrationService } from './UserRegistrationService';
export { UserService } from './UserService';
export type { 
  UserRegistrationRequest,
  UserProfile,
  UserPreferences 
} from './types';

// Internal implementation details not exported
// - UserRegistrationValidator (internal to service)
// - UserPasswordHasher (internal utility)
// - UserEmailVerifier (internal process)
```

**Dependency Injection Patterns**
Use constructor injection with explicit interfaces:

```typescript
// Interface definition
interface UserService {
  registerUser(request: UserRegistrationRequest): Promise<Result<User, RegistrationError>>;
  authenticateUser(credentials: LoginCredentials): Promise<Result<AuthSession, AuthError>>;
  updateUserProfile(userId: UserId, updates: ProfileUpdates): Promise<Result<User, UpdateError>>;
}

// Implementation with injected dependencies
class DefaultUserService implements UserService {
  constructor(
    private readonly userRepository: UserRepository,
    private readonly emailService: EmailDeliveryService,
    private readonly auditLogger: AuditLogger,
    private readonly eventBus: EventBus
  ) {}

  async registerUser(request: UserRegistrationRequest): Promise<Result<User, RegistrationError>> {
    // Implementation with clear dependency usage
    const validationResult = await this.validateRequest(request);
    if (!validationResult.success) {
      await this.auditLogger.logFailedRegistration(request.email, validationResult.error);
      return { success: false, error: validationResult.error };
    }

    const user = await this.userRepository.create(validationResult.data);
    await this.emailService.sendWelcomeEmail(user);
    await this.eventBus.publish(new UserRegisteredEvent(user));
    
    return { success: true, data: user };
  }
}

// Dependency injection container configuration
const container = new Container();
container.bind<UserRepository>('UserRepository').to(PostgresUserRepository);
container.bind<EmailDeliveryService>('EmailDeliveryService').to(SendGridEmailService);
container.bind<UserService>('UserService').to(DefaultUserService);
```

### Import/Export Strategies for Modular Architecture

**Barrel Exports for Clean APIs**
Use index files to create clean public APIs for each module:

```typescript
// src/domain/index.ts - Domain layer public API
export * from './user';
export * from './subscription';
export * from './payment';
export * from './notification';

// src/infrastructure/index.ts - Infrastructure layer public API
export * from './database';
export * from './email';
export * from './logging';
export * from './monitoring';

// src/application/index.ts - Application layer public API
export * from './services';
export * from './handlers';
export * from './queries';
export * from './commands';
```

**Import Organization Standards**
Organize imports in consistent groups with clear separation:

```typescript
// External library imports first
import express from 'express';
import { z } from 'zod';
import { inject, injectable } from 'inversify';

// Internal framework/shared imports
import { Logger } from '@/shared/logging';
import { ValidationError } from '@/shared/errors';
import { Result } from '@/shared/types';

// Domain imports
import { User, UserRepository } from '@/domain/user';
import { EmailService } from '@/domain/notification';

// Application layer imports
import { RegisterUserCommand } from '@/application/commands';
import { UserRegistrationHandler } from '@/application/handlers';

// Infrastructure imports (typically only in composition root)
import { PostgresUserRepository } from '@/infrastructure/database';
import { SendGridEmailService } from '@/infrastructure/email';

// Types-only imports when appropriate
import type { Request, Response } from 'express';
import type { UserRegistrationRequest } from '@/domain/user/types';
```

**Circular Dependency Prevention**
Organize layers to prevent circular dependencies:

```typescript
// GOOD: Clear layer separation
// Domain layer - no dependencies on other layers
export class User {
  // Pure domain logic, no infrastructure concerns
}

// Application layer - depends only on domain
export class UserService {
  constructor(
    private readonly userRepository: UserRepository // Domain interface
  ) {}
}

// Infrastructure layer - implements domain interfaces
export class PostgresUserRepository implements UserRepository {
  // Database-specific implementation
}

// Composition root - wires everything together
const userRepository = new PostgresUserRepository(database);
const userService = new UserService(userRepository);

// AVOID: Circular dependencies
// Domain depending on infrastructure
export class User {
  constructor(private readonly db: PostgresConnection) {} // ❌ Wrong layer
}
```

## Security-Integrated Coding Patterns

### Input Validation and Sanitization
All external input must be validated and sanitized at the application boundary:

```typescript
// Input validation with security-first approach
const validateUserRegistrationInput = (rawInput: unknown): Result<UserRegistrationRequest, ValidationError> => {
  const schema = z.object({
    email: z.string()
      .email('Invalid email format')
      .max(254, 'Email too long') // RFC 5321 limit
      .refine(email => !isDisposableEmail(email), 'Disposable emails not allowed'),
    
    password: z.string()
      .min(12, 'Password must be at least 12 characters')
      .max(128, 'Password too long') // Prevent DoS
      .refine(password => isStrongPassword(password), 'Password not strong enough'),
    
    name: z.string()
      .min(1, 'Name required')
      .max(100, 'Name too long')
      .refine(name => !containsHtmlOrScript(name), 'Name contains invalid characters'),
    
    dateOfBirth: z.string()
      .datetime('Invalid date format')
      .refine(date => isValidAge(date), 'Must be 13 or older'),
      
    agreedToTerms: z.literal(true, { 
      errorMap: () => ({ message: 'Must agree to terms and conditions' })
    })
  });

  const result = schema.safeParse(rawInput);
  
  if (!result.success) {
    return {
      success: false,
      error: new ValidationError('INPUT_VALIDATION_FAILED', result.error.issues)
    };
  }

  return { success: true, data: result.data };
};

// Additional security validation helpers
const isStrongPassword = (password: string): boolean => {
  const hasLowercase = /[a-z]/.test(password);
  const hasUppercase = /[A-Z]/.test(password);
  const hasNumbers = /\d/.test(password);
  const hasSpecialChars = /[!@#$%^&*(),.?":{}|<>]/.test(password);
  const noCommonPatterns = !isCommonPassword(password);
  
  return hasLowercase && hasUppercase && hasNumbers && hasSpecialChars && noCommonPatterns;
};

const containsHtmlOrScript = (input: string): boolean => {
  const htmlPattern = /<[^>]*>/;
  const scriptPattern = /<script[\s\S]*?>[\s\S]*?<\/script>/i;
  return htmlPattern.test(input) || scriptPattern.test(input);
};
```

### Authentication and Authorization Patterns
Implement security checks at every service boundary:

```typescript
// Authentication context type
interface AuthContext {
  readonly userId: UserId;
  readonly email: EmailAddress;
  readonly roles: readonly Role[];
  readonly permissions: readonly Permission[];
  readonly sessionId: SessionId;
  readonly issuedAt: Timestamp;
  readonly expiresAt: Timestamp;
}

// Authorization decorator for service methods
const requirePermission = (permission: Permission) => {
  return (target: any, propertyKey: string, descriptor: PropertyDescriptor) => {
    const originalMethod = descriptor.value;
    
    descriptor.value = async function(authContext: AuthContext, ...args: any[]) {
      if (!authContext.permissions.includes(permission)) {
        throw new AuthorizationError(`Missing required permission: ${permission}`);
      }
      
      if (authContext.expiresAt < new Date()) {
        throw new AuthenticationError('Session expired');
      }
      
      return originalMethod.apply(this, [authContext, ...args]);
    };
  };
};

// Usage in service methods
class UserService {
  @requirePermission('user:read')
  async getUserProfile(authContext: AuthContext, userId: UserId): Promise<Result<UserProfile, ServiceError>> {
    // Additional authorization check for data access
    if (authContext.userId !== userId && !authContext.roles.includes('admin')) {
      throw new AuthorizationError('Cannot access other user profiles');
    }
    
    const user = await this.userRepository.findById(userId);
    if (!user) {
      return { success: false, error: new NotFoundError('User not found') };
    }
    
    return { success: true, data: user.profile };
  }

  @requirePermission('user:write')
  async updateUserProfile(
    authContext: AuthContext, 
    userId: UserId, 
    updates: ProfileUpdates
  ): Promise<Result<User, ServiceError>> {
    // Validate user can modify this profile
    if (authContext.userId !== userId && !authContext.roles.includes('admin')) {
      throw new AuthorizationError('Cannot modify other user profiles');
    }
    
    // Sanitize updates to prevent injection attacks
    const sanitizedUpdates = sanitizeProfileUpdates(updates);
    
    const result = await this.userRepository.update(userId, sanitizedUpdates);
    
    // Audit significant profile changes
    await this.auditLogger.logProfileUpdate(authContext.userId, userId, sanitizedUpdates);
    
    return result;
  }
}
```

### Data Privacy and Protection Patterns
Implement privacy-by-design in all data handling:

```typescript
// Personal data classification types
type PersonalData = string & { readonly __brand: 'PersonalData' };
type SensitiveData = string & { readonly __brand: 'SensitiveData' };

// Data masking utilities
const maskEmail = (email: EmailAddress): string => {
  const [localPart, domain] = email.split('@');
  const maskedLocal = localPart.charAt(0) + '*'.repeat(localPart.length - 2) + localPart.charAt(localPart.length - 1);
  return `${maskedLocal}@${domain}`;
};

const maskPhoneNumber = (phone: string): string => {
  return phone.replace(/\d(?=\d{4})/g, '*');
};

// Data access logging
class DataAccessLogger {
  async logPersonalDataAccess(
    authContext: AuthContext,
    dataSubject: UserId,
    dataType: string,
    purpose: string
  ): Promise<void> {
    await this.auditLogger.log({
      event: 'PERSONAL_DATA_ACCESS',
      accessor: authContext.userId,
      subject: dataSubject,
      dataType,
      purpose,
      timestamp: new Date(),
      sessionId: authContext.sessionId
    });
  }
}

// GDPR-compliant user data export
class UserDataExportService {
  async exportUserData(authContext: AuthContext, userId: UserId): Promise<UserDataExport> {
    // Verify user can access this data
    if (authContext.userId !== userId && !authContext.roles.includes('data-protection-officer')) {
      throw new AuthorizationError('Cannot export other user data');
    }
    
    await this.dataAccessLogger.logPersonalDataAccess(
      authContext, 
      userId, 
      'FULL_EXPORT', 
      'GDPR_DATA_EXPORT'
    );
    
    const [user, transactions, communications, activities] = await Promise.all([
      this.userRepository.findById(userId),
      this.transactionRepository.findByUserId(userId),
      this.communicationRepository.findByUserId(userId),
      this.activityRepository.findByUserId(userId)
    ]);
    
    return {
      exportedAt: new Date(),
      exportedBy: authContext.userId,
      userData: user,
      transactionHistory: transactions,
      communicationHistory: communications,
      activityLog: activities
    };
  }
}
```

## Memory Bank Integration and Decision Capture

### Code Pattern Documentation Template
When establishing new coding patterns, document them in `memory-bank/systemPatterns.md`:

```markdown
## Coding Pattern: [Pattern Name]
**Established**: [YYYY-MM-DD]
**Context**: [When and why this pattern was adopted]
**Code Implementer**: [Mode/Human identifier]

### Pattern Definition
[Clear description of the coding pattern]

### TypeScript Implementation
```typescript
// Example implementation showing the pattern
```

### Usage Guidelines
- **When to use**: [Specific scenarios where this pattern applies]
- **When not to use**: [Scenarios where different patterns are better]
- **Security considerations**: [Any security implications]
- **Performance implications**: [Performance characteristics]

### Related Patterns
- **Complements**: [Patterns that work well together]
- **Conflicts with**: [Patterns to avoid combining]
- **Supersedes**: [Older patterns this replaces]

### Quality Metrics
- **Testability**: [How this pattern affects testing]
- **Maintainability**: [Long-term maintenance considerations]
- **Performance**: [Expected performance characteristics]
```

### Language Decision Logging
Document significant language and framework decisions in `memory-bank/decisionLog.md`:

```markdown
## Language Decision: [Technology/Pattern Choice]
**Date**: [YYYY-MM-DD]
**Phase**: Implementation
**Decision Type**: Language Convention
**Implementer**: [Mode/Human identifier]

### Context
- **Problem**: [What coding challenge needed solving]
- **Constraints**: [Technical/business limitations]
- **Requirements**: [Must-have vs nice-to-have features]

### Options Considered
1. **Option A**: [Approach name - e.g., "Class-based architecture"]
   - **Benefits**: [Advantages of this approach]
   - **Drawbacks**: [Disadvantages and limitations]
   - **Example**: [Code snippet showing usage]

2. **Option B**: [Alternative approach]
   - **Benefits**: [Advantages of this approach]
   - **Drawbacks**: [Disadvantages and limitations]
   - **Example**: [Code snippet showing usage]

### Decision
**Chosen**: [Selected approach]
**Rationale**: [Why this option was selected]

### Implementation Details
- **Coding standard**: [Specific coding rules to follow]
- **Testing approach**: [How this pattern affects testing]
- **Security measures**: [Security considerations for this pattern]
- **Documentation requirements**: [What needs to be documented]

### Success Criteria
- **Code quality**: [Specific quality metrics]
- **Developer experience**: [Ease of use and understanding]
- **Performance**: [Expected performance outcomes]
- **Maintainability**: [Long-term maintenance goals]

### Review Schedule
- **Next Review**: [Date for pattern review]
- **Success Metrics**: [How we'll measure pattern success]
- **Failure Triggers**: [Conditions that would require pattern change]
```

### Implementation Progress Tracking
Track coding pattern adoption in `memory-bank/progress.md`:

```markdown
## Code Implementation Progress

### Active Coding Tasks
- **User Service Refactoring**: Converting to functional patterns (85% complete)
- **Type Safety Migration**: Adding branded types for domain objects (60% complete)
- **Security Integration**: Implementing input validation patterns (40% complete)

### Coding Standards Adoption
- ✅ **Function naming conventions**: Adopted across all new code
- ✅ **Import organization**: Enforced via ESLint rules
- ⚠️ **Error handling patterns**: 70% adoption, legacy code needs migration
- ⚠️ **Security validation**: Active implementation in progress
- ❌ **Performance monitoring**: Code instrumentation pending

### Technical Debt Registry
- **Priority High**: Legacy authentication code lacks type safety
- **Priority Medium**: Inconsistent error handling in payment service
- **Priority Low**: Code comments need standardization

### Learning and Improvement
- **New patterns identified**: Result type for error handling proving effective
- **Performance improvements**: Functional patterns reducing memory usage by 15%
- **Developer feedback**: Team prefers explicit typing over inference
- **Security wins**: Input validation catching 23% more invalid requests
```

### Code Quality Enforcement Configuration
Document quality standards in workspace configuration:

```typescript
// .eslintrc.js - Enforcing SPARC language conventions
module.exports = {
  extends: [
    '@typescript-eslint/recommended',
    '@typescript-eslint/recommended-requiring-type-checking'
  ],
  rules: {
    // Naming conventions aligned with SPARC methodology
    '@typescript-eslint/naming-convention': [
      'error',
      {
        selector: 'interface',
        format: ['PascalCase'],
        custom: {
          regex: '^I[A-Z]',
          match: false // Prevent Hungarian notation
        }
      },
      {
        selector: 'typeAlias',
        format: ['PascalCase']
      },
      {
        selector: 'enum',
        format: ['PascalCase']
      },
      {
        selector: 'function',
        format: ['camelCase'],
        leadingUnderscore: 'forbid',
        trailingUnderscore: 'forbid'
      },
      {
        selector: 'variable',
        format: ['camelCase', 'UPPER_CASE'],
        leadingUnderscore: 'forbid',
        trailingUnderscore: 'forbid'
      }
    ],

    // File size enforcement for 500-line principle
    'max-lines': ['error', { max: 500, skipBlankLines: true, skipComments: true }],
    
    // Function complexity limits
    'complexity': ['error', { max: 10 }],
    'max-depth': ['error', { max: 4 }],
    'max-nested-callbacks': ['error', { max: 3 }],
    
    // Security-oriented rules
    '@typescript-eslint/no-explicit-any': 'error',
    '@typescript-eslint/no-unsafe-assignment': 'error',
    '@typescript-eslint/no-unsafe-call': 'error',
    '@typescript-eslint/no-unsafe-member-access': 'error',
    '@typescript-eslint/no-unsafe-return': 'error',
    
    // Functional programming preferences
    'prefer-const': 'error',
    'no-var': 'error',
    '@typescript-eslint/prefer-readonly': 'error',
    '@typescript-eslint/prefer-readonly-parameter-types': 'warn'
  }
};

// .prettierrc.js - Code formatting standards
module.exports = {
  semi: true,
  trailingComma: 'es5',
  singleQuote: true,
  printWidth: 100,
  tabWidth: 2,
  useTabs: false,
  bracketSpacing: true,
  arrowParens: 'avoid'
};
```

This comprehensive language convention guide ensures that all code implementation in the SPARC methodology maintains consistency, security, and clarity while supporting rapid development cycles and confident refactoring.