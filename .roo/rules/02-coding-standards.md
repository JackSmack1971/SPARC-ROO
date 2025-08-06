# Project Coding Standards
## Universal Standards for All SPARC Development

### File Organization and Structure

#### The 500-Line Sacred Boundary
Every source file must remain under 500 lines of code. This is not a suggestion—it's a fundamental architectural principle that enables:
- **Cognitive Clarity**: Human developers can hold an entire file in working memory
- **Collaborative Accessibility**: AI modes can process complete files within context windows
- **Modular Excellence**: Natural boundaries emerge that promote single responsibility
- **Quality Maintenance**: Smaller files are easier to test, review, and refactor

**When approaching 500 lines:**
1. **Feel the Weight**: Does this file still have a single, clear purpose?
2. **Find Natural Boundaries**: Where do concerns separate naturally?
3. **Extract with Purpose**: Create new modules that solve complete problems
4. **Maintain Cohesion**: Ensure extracted modules have clear, logical responsibilities

#### Directory and File Naming Conventions
```
# Directories: kebab-case with clear hierarchical meaning
src/user-management/authentication/
src/data-processing/validation/
tests/integration/api-endpoints/

# Files: kebab-case with descriptive, action-oriented names
user-authentication-service.js
password-validation-utils.js
integration-test-helpers.js

# Test files: mirror source structure with .test/.spec suffix
user-service.js → user-service.test.js
auth-middleware.js → auth-middleware.spec.js
```

### Code Organization Within Files

#### The SPARC File Structure Pattern
Every source file should follow this organizational pattern:

```javascript
// 1. HEADER: Imports and dependencies (max 50 lines)
import { authService } from '../auth/auth-service.js';
import { validateInput } from '../utils/validation.js';
import { logger } from '../shared/logger.js';

// 2. CONSTANTS: Configuration and magic numbers (max 30 lines)
const MAX_LOGIN_ATTEMPTS = 3;
const SESSION_TIMEOUT_MS = 30 * 60 * 1000; // 30 minutes
const SUPPORTED_AUTH_METHODS = ['password', 'oauth', 'token'];

// 3. TYPES/INTERFACES: Data structure definitions (max 50 lines)
/**
 * @typedef {Object} UserAuthRequest
 * @property {string} email - User email address
 * @property {string} password - User password
 * @property {string} [method='password'] - Authentication method
 */

// 4. CORE FUNCTIONS: Primary business logic (max 300 lines)
export async function authenticateUser(authRequest) {
  // Implementation here
}

export function validateAuthRequest(request) {
  // Implementation here
}

// 5. UTILITIES: Helper functions specific to this module (max 70 lines)
function hashPassword(password) {
  // Implementation here
}

function generateSessionToken() {
  // Implementation here
}
```

### Naming Conventions and Clarity

#### Variables and Functions
```javascript
// USE: Descriptive, intention-revealing names
const authenticatedUsers = new Map();
const maxRetryAttempts = 3;
function validateUserCredentials(email, password) { }
function calculatePasswordStrength(password) { }

// AVOID: Abbreviated or cryptic names
const authUsers = new Map();
const maxRetry = 3;
function validate(e, p) { }
function calc(pwd) { }
```

#### Constants and Configuration
```javascript
// USE: SCREAMING_SNAKE_CASE for constants
const DATABASE_CONNECTION_TIMEOUT = 5000;
const MAX_FILE_UPLOAD_SIZE_MB = 10;
const SUPPORTED_IMAGE_FORMATS = ['jpg', 'png', 'webp'];

// USE: Grouped configuration objects
const AUTH_CONFIG = {
  maxLoginAttempts: 3,
  sessionTimeoutMs: 30 * 60 * 1000,
  passwordMinLength: 8,
  enableTwoFactor: true
};
```

#### Classes and Types
```javascript
// USE: PascalCase for classes and types
class UserAuthenticationService { }
class PasswordValidator { }

// USE: Descriptive interface/type names
interface DatabaseConnectionConfig { }
type UserPermissionLevel = 'read' | 'write' | 'admin';
```

### Error Handling and Resilience

#### Comprehensive Error Strategies
Every function must handle errors gracefully and provide meaningful feedback:

```javascript
// Pattern 1: Validation with detailed error context
function validateEmailAddress(email) {
  if (!email) {
    throw new ValidationError('Email address is required', {
      field: 'email',
      code: 'MISSING_EMAIL',
      context: { received: email }
    });
  }
  
  if (!email.includes('@')) {
    throw new ValidationError('Email address must contain @ symbol', {
      field: 'email',
      code: 'INVALID_EMAIL_FORMAT',
      context: { received: email }
    });
  }
  
  return true;
}

// Pattern 2: Async operations with proper error propagation
async function fetchUserData(userId) {
  try {
    const userData = await database.users.findById(userId);
    
    if (!userData) {
      throw new NotFoundError(`User not found`, {
        userId,
        operation: 'fetchUserData',
        timestamp: new Date().toISOString()
      });
    }
    
    return userData;
  } catch (error) {
    // Log error with context but don't expose internal details
    logger.error('Failed to fetch user data', {
      userId,
      error: error.message,
      stack: error.stack
    });
    
    // Re-throw with appropriate error type
    if (error instanceof NotFoundError) {
      throw error;
    }
    
    throw new DatabaseError('Failed to retrieve user information', {
      originalError: error.message,
      operation: 'fetchUserData'
    });
  }
}
```

#### Error Classification System
```javascript
// Define clear error hierarchies
class AppError extends Error {
  constructor(message, context = {}) {
    super(message);
    this.name = this.constructor.name;
    this.context = context;
    this.timestamp = new Date().toISOString();
  }
}

class ValidationError extends AppError { }
class NotFoundError extends AppError { }
class AuthenticationError extends AppError { }
class DatabaseError extends AppError { }
class NetworkError extends AppError { }
```

### Performance and Optimization Guidelines

#### Memory Management
```javascript
// USE: Efficient data structures and cleanup
class UserSessionManager {
  constructor() {
    this.activeSessions = new Map();
    this.cleanupInterval = setInterval(() => {
      this.removeExpiredSessions();
    }, 60000); // Cleanup every minute
  }
  
  destroy() {
    clearInterval(this.cleanupInterval);
    this.activeSessions.clear();
  }
  
  removeExpiredSessions() {
    const now = Date.now();
    for (const [sessionId, session] of this.activeSessions.entries()) {
      if (session.expiresAt < now) {
        this.activeSessions.delete(sessionId);
      }
    }
  }
}

// AVOID: Memory leaks and unbounded growth
class BadSessionManager {
  constructor() {
    this.allSessions = []; // Never cleaned up!
    setInterval(() => {
      // This interval never gets cleared
      console.log('Session count:', this.allSessions.length);
    }, 1000);
  }
}
```

#### Async Operation Patterns
```javascript
// USE: Proper Promise handling and concurrency control
async function processUserBatch(userIds, options = {}) {
  const { batchSize = 10, maxConcurrency = 3 } = options;
  const results = [];
  
  for (let i = 0; i < userIds.length; i += batchSize) {
    const batch = userIds.slice(i, i + batchSize);
    
    // Process batch with controlled concurrency
    const batchPromises = batch.map(async (userId) => {
      try {
        return await processUser(userId);
      } catch (error) {
        logger.error(`Failed to process user ${userId}`, error);
        return { userId, error: error.message };
      }
    });
    
    const batchResults = await Promise.allSettled(batchPromises);
    results.push(...batchResults);
    
    // Optional: Add delay between batches to avoid overwhelming systems
    if (i + batchSize < userIds.length) {
      await new Promise(resolve => setTimeout(resolve, 100));
    }
  }
  
  return results;
}
```

### Documentation and Comments

#### Self-Documenting Code Priority
Code should be written to minimize the need for comments by being inherently clear:

```javascript
// GOOD: Clear function names and structure
function calculateMonthlySubscriptionRevenue(subscriptions, month, year) {
  const targetMonth = new Date(year, month - 1);
  
  return subscriptions
    .filter(sub => isActiveInMonth(sub, targetMonth))
    .reduce((total, sub) => total + sub.monthlyAmount, 0);
}

function isActiveInMonth(subscription, targetMonth) {
  const subStart = new Date(subscription.startDate);
  const subEnd = subscription.endDate ? new Date(subscription.endDate) : null;
  
  return subStart <= targetMonth && (subEnd === null || subEnd >= targetMonth);
}

// AVOID: Comments explaining unclear code
function calc(subs, m, y) {
  // Get target month
  const tm = new Date(y, m - 1);
  
  // Filter active subscriptions and sum amounts
  return subs
    .filter(s => {
      // Check if subscription is active in target month
      const start = new Date(s.start);
      const end = s.end ? new Date(s.end) : null;
      return start <= tm && (end === null || end >= tm);
    })
    .reduce((t, s) => t + s.amount, 0); // Add up all amounts
}
```

#### When Comments Are Necessary
Use comments for:

1. **Complex Business Logic**: When domain rules require explanation
```javascript
// Apply the "Rule of 78" interest calculation method for early loan payoffs
// This front-loads interest payments, so early payoffs save less than expected
function calculateRuleOf78Interest(principal, rate, termMonths, payoffMonth) {
  // Implementation requires detailed comment due to complexity
}
```

2. **Performance Optimizations**: When code is non-obvious but necessary
```javascript
// Use binary search instead of linear scan for large sorted datasets
// Performance improvement: O(log n) vs O(n) for 10k+ items
function findUserByIdBinary(users, targetId) {
  // Binary search implementation
}
```

3. **External System Integration**: When APIs have quirks or requirements
```javascript
// Stripe webhook signatures must be verified within 5 minutes of generation
// See: https://stripe.com/docs/webhooks/signatures
function verifyStripeWebhookSignature(payload, signature, timestamp) {
  const fiveMinutesAgo = Date.now() - (5 * 60 * 1000);
  if (timestamp < fiveMinutesAgo) {
    throw new Error('Webhook timestamp too old');
  }
  // Verification logic continues...
}
```

### Security Standards in Code

#### Input Validation and Sanitization
Every external input must be validated and sanitized:

```javascript
import { escape } from 'html-escaper';
import { isEmail, isLength, isAlphanumeric } from 'validator';

function validateAndSanitizeUserInput(rawInput) {
  const validation = {
    email: {
      required: true,
      validator: (value) => isEmail(value),
      sanitizer: (value) => value.toLowerCase().trim(),
      errorMessage: 'Must be a valid email address'
    },
    username: {
      required: true,
      validator: (value) => isLength(value, { min: 3, max: 20 }) && isAlphanumeric(value),
      sanitizer: (value) => value.trim(),
      errorMessage: 'Username must be 3-20 alphanumeric characters'
    },
    bio: {
      required: false,
      validator: (value) => isLength(value, { max: 500 }),
      sanitizer: (value) => escape(value.trim()),
      errorMessage: 'Bio must be less than 500 characters'
    }
  };
  
  const result = {};
  const errors = [];
  
  for (const [field, rules] of Object.entries(validation)) {
    const value = rawInput[field];
    
    if (rules.required && (!value || value === '')) {
      errors.push(`${field} is required`);
      continue;
    }
    
    if (value && !rules.validator(value)) {
      errors.push(rules.errorMessage);
      continue;
    }
    
    result[field] = value ? rules.sanitizer(value) : null;
  }
  
  if (errors.length > 0) {
    throw new ValidationError('Input validation failed', { errors });
  }
  
  return result;
}
```

#### Secure Configuration Management
```javascript
// NEVER hardcode secrets or sensitive configuration
// GOOD: Use environment variables with validation
class DatabaseConfig {
  constructor() {
    this.host = this.getRequiredEnvVar('DB_HOST');
    this.port = parseInt(this.getRequiredEnvVar('DB_PORT'));
    this.database = this.getRequiredEnvVar('DB_NAME');
    this.username = this.getRequiredEnvVar('DB_USERNAME');
    this.password = this.getRequiredEnvVar('DB_PASSWORD');
    
    // Validate configuration
    if (isNaN(this.port) || this.port < 1 || this.port > 65535) {
      throw new Error('DB_PORT must be a valid port number');
    }
  }
  
  getRequiredEnvVar(name) {
    const value = process.env[name];
    if (!value) {
      throw new Error(`Required environment variable ${name} is not set`);
    }
    return value;
  }
  
  // Provide safe representation for logging
  toLogSafeObject() {
    return {
      host: this.host,
      port: this.port,
      database: this.database,
      username: this.username,
      password: '[REDACTED]'
    };
  }
}
```

### Testing Integration Standards

#### Testable Code Design
Write code that enables comprehensive testing:

```javascript
// GOOD: Dependency injection enables testing
class UserService {
  constructor(database, emailService, logger) {
    this.database = database;
    this.emailService = emailService;
    this.logger = logger;
  }
  
  async createUser(userData) {
    try {
      const validatedData = this.validateUserData(userData);
      const user = await this.database.users.create(validatedData);
      
      await this.emailService.sendWelcomeEmail(user.email, user.name);
      this.logger.info('User created successfully', { userId: user.id });
      
      return user;
    } catch (error) {
      this.logger.error('Failed to create user', { userData, error });
      throw error;
    }
  }
  
  validateUserData(userData) {
    // Validation logic that can be tested independently
    return validateAndSanitizeUserInput(userData);
  }
}

// AVOID: Hard dependencies that make testing difficult
class BadUserService {
  async createUser(userData) {
    // Direct database access - hard to mock
    const user = await database.users.create(userData);
    
    // Direct email service access - hard to test
    await sendEmail(user.email, 'Welcome!');
    
    // Direct console access - no testing control
    console.log('User created:', user.id);
    
    return user;
  }
}
```

### Language-Specific Conventions

#### JavaScript/TypeScript Specific Standards
```javascript
// Use modern ES6+ features appropriately
const userPermissions = new Set(['read', 'write']);
const userRoles = new Map([
  ['admin', { permissions: ['read', 'write', 'delete'] }],
  ['user', { permissions: ['read'] }]
]);

// Destructuring for cleaner code
function processUserRequest({ userId, action, permissions = [] }) {
  const { database, logger } = this.dependencies;
  // Implementation
}

// Template literals for readable strings
function generateUserWelcomeMessage(user, organization) {
  return `Welcome to ${organization.name}, ${user.firstName}! 
Your account has been created with ${user.permissions.length} permissions.
Login at: ${organization.loginUrl}`;
}
```

#### Python Specific Standards (if applicable)
```python
# Follow PEP 8 with SPARC adaptations
class UserAuthenticationService:
    """Service for handling user authentication operations.
    
    Maintains the 500-line limit through focused responsibility
    and delegation to helper classes and utility functions.
    """
    
    def __init__(self, database_connector, password_hasher, logger):
        self.database = database_connector
        self.password_hasher = password_hasher
        self.logger = logger
        self.max_login_attempts = 3
    
    def authenticate_user(self, email: str, password: str) -> UserSession:
        """Authenticate user credentials and return session."""
        try:
            validated_email = self._validate_email(email)
            user = self._fetch_user_by_email(validated_email)
            
            if self._verify_password(user, password):
                return self._create_user_session(user)
            else:
                self._handle_failed_login(user)
                raise AuthenticationError("Invalid credentials")
                
        except Exception as error:
            self.logger.error(f"Authentication failed: {error}")
            raise
    
    def _validate_email(self, email: str) -> str:
        """Private helper for email validation."""
        # Implementation
        pass
```

### Continuous Improvement Standards

#### Code Review Integration
Every piece of code should be written with code review in mind:

```javascript
// Include clear commit messages that explain the 'why'
// Good commit: "Add rate limiting to auth endpoint to prevent brute force attacks"
// Bad commit: "Fix auth stuff"

// Structure code for easy review
function authenticateUser(credentials) {
  // Step 1: Validate input (easy to review logic)
  const validatedCredentials = validateCredentials(credentials);
  
  // Step 2: Check rate limiting (clear security concern)
  if (isRateLimited(credentials.ip)) {
    throw new RateLimitError('Too many attempts');
  }
  
  // Step 3: Verify credentials (core authentication logic)
  const user = await verifyUserCredentials(validatedCredentials);
  
  // Step 4: Create session (session management logic)
  return createUserSession(user);
}
```

#### Performance Monitoring Integration
```javascript
// Include performance markers for critical paths
async function processLargeDataset(dataset) {
  const startTime = performance.now();
  
  try {
    const result = await heavyProcessingOperation(dataset);
    
    const duration = performance.now() - startTime;
    logger.info('Dataset processing completed', {
      recordCount: dataset.length,
      durationMs: duration,
      recordsPerSecond: dataset.length / (duration / 1000)
    });
    
    return result;
  } catch (error) {
    const duration = performance.now() - startTime;
    logger.error('Dataset processing failed', {
      recordCount: dataset.length,
      durationMs: duration,
      error: error.message
    });
    throw error;
  }
}
```

These coding standards ensure that every line of code written in the SPARC methodology contributes to a maintainable, secure, and high-quality system that can scale with confidence.