# Convention Examples

> **Note**: This example uses `type: convention`. Other valid types: `rule`, `pattern`, `guide`, `documentation`, `reference`, `style`, `environment`. Type is auto-detected from content.

Reference example for Comprehensive tier conventions.

---

## Comprehensive Tier Example (150-300 lines)

**File:** `.conventions/typescript/patterns/error-handling.md`

```markdown
---
type: convention
version: 2.1
scope: typescript
category: patterns
tags: [errors, exceptions, try-catch, async, logging]
---

# Error Handling Patterns

Comprehensive guidelines for error handling in TypeScript, covering synchronous operations, async/await patterns, custom error classes, error boundaries, and logging practices.

## Format

- **Try-catch blocks**: Required for all async operations and error-prone code
- **Custom error classes**: Extend Error for domain-specific errors
- **Error messages**: User-friendly messages with technical details in metadata
- **Logging**: Structured error logging with context
- **Error propagation**: Fail fast, handle at appropriate level
- **Type safety**: Use typed error handling where possible

## Synchronous Error Handling

### Basic Patterns

Use try-catch for operations that may throw:

```typescript
function parseUserInput(input: string): UserData {
  try {
    return JSON.parse(input) as UserData;
  } catch (error) {
    throw new ValidationError('Invalid user data format', {
      cause: error,
      input: input.slice(0, 100) // First 100 chars for debugging
    });
  }
}
```

### Validation Errors

```typescript
function validateAge(age: number): void {
  if (age < 0 || age > 150) {
    throw new ValidationError('Age must be between 0 and 150', {
      provided: age,
      field: 'age'
    });
  }
}
```

## Asynchronous Error Handling

### Async/Await Pattern

Always wrap async operations in try-catch:

```typescript
async function fetchUserProfile(userId: string): Promise<UserProfile> {
  try {
    const response = await api.get(`/users/${userId}`);
    return response.data;
  } catch (error) {
    if (error instanceof NetworkError) {
      logger.error('Network error fetching user profile', {
        userId,
        error: error.message
      });
      throw new ServiceUnavailableError('Unable to fetch user profile', {
        cause: error,
        retry: true
      });
    }

    if (error instanceof NotFoundError) {
      logger.warn('User not found', { userId });
      throw new UserNotFoundError(`User ${userId} not found`);
    }

    // Unexpected errors
    logger.error('Unexpected error fetching user profile', {
      userId,
      error
    });
    throw error;
  }
}
```

### Promise Rejection Handling

```typescript
// Good: Explicit error handling
async function processUserData(userId: string): Promise<void> {
  const profile = await fetchUserProfile(userId);
  const settings = await fetchUserSettings(userId);

  await updateUserCache(profile, settings);
}

// Bad: Unhandled promise rejection
function processUserDataBad(userId: string): void {
  fetchUserProfile(userId); // Promise not awaited or handled
}
```

## Custom Error Classes

### Base Custom Error

```typescript
export class AppError extends Error {
  public readonly statusCode: number;
  public readonly isOperational: boolean;
  public readonly metadata?: Record<string, unknown>;

  constructor(
    message: string,
    statusCode: number = 500,
    isOperational: boolean = true,
    metadata?: Record<string, unknown>
  ) {
    super(message);
    this.name = this.constructor.name;
    this.statusCode = statusCode;
    this.isOperational = isOperational;
    this.metadata = metadata;

    Error.captureStackTrace(this, this.constructor);
  }
}
```

### Domain-Specific Errors

```typescript
export class ValidationError extends AppError {
  constructor(message: string, metadata?: Record<string, unknown>) {
    super(message, 400, true, metadata);
  }
}

export class UserNotFoundError extends AppError {
  constructor(message: string) {
    super(message, 404, true, { domain: 'user' });
  }
}

export class ServiceUnavailableError extends AppError {
  constructor(message: string, metadata?: Record<string, unknown>) {
    super(message, 503, true, metadata);
  }
}
```

## Error Logging

### Structured Logging

```typescript
function logError(error: Error, context?: Record<string, unknown>): void {
  const errorLog = {
    message: error.message,
    name: error.name,
    stack: error.stack,
    timestamp: new Date().toISOString(),
    ...context,
    ...(error instanceof AppError && { metadata: error.metadata })
  };

  if (error instanceof AppError && error.statusCode < 500) {
    logger.warn('Client error', errorLog);
  } else {
    logger.error('Server error', errorLog);
  }
}
```

### Context Enrichment

```typescript
async function processOrder(orderId: string): Promise<void> {
  try {
    const order = await fetchOrder(orderId);
    await validateOrder(order);
    await chargePayment(order);
    await shipOrder(order);
  } catch (error) {
    logError(error, {
      operation: 'processOrder',
      orderId,
      stage: determineStage(error)
    });
    throw error;
  }
}
```

## Allowed

```typescript
// Example 1: Comprehensive async error handling
async function createUser(userData: CreateUserDTO): Promise<User> {
  try {
    // Validation
    validateUserData(userData);

    // Check existing
    const existing = await userRepository.findByEmail(userData.email);
    if (existing) {
      throw new ValidationError('Email already in use', {
        email: userData.email
      });
    }

    // Create user
    const user = await userRepository.create(userData);
    logger.info('User created successfully', { userId: user.id });

    return user;
  } catch (error) {
    if (error instanceof ValidationError) {
      logger.warn('User creation validation failed', {
        email: userData.email,
        error: error.message
      });
      throw error;
    }

    logger.error('User creation failed', {
      email: userData.email,
      error
    });
    throw new ServiceError('Failed to create user', { cause: error });
  }
}

// Example 2: Error boundary with cleanup
async function withTransaction<T>(
  operation: () => Promise<T>
): Promise<T> {
  const transaction = await db.startTransaction();

  try {
    const result = await operation();
    await transaction.commit();
    return result;
  } catch (error) {
    await transaction.rollback();
    logger.error('Transaction failed, rolled back', { error });
    throw error;
  }
}

// Example 3: Typed error handling
type Result<T, E = Error> =
  | { success: true; data: T }
  | { success: false; error: E };

async function safelyFetchUser(
  userId: string
): Promise<Result<User, AppError>> {
  try {
    const user = await fetchUser(userId);
    return { success: true, data: user };
  } catch (error) {
    if (error instanceof AppError) {
      return { success: false, error };
    }
    return {
      success: false,
      error: new ServiceError('Unknown error', { cause: error })
    };
  }
}
```

## Forbidden

```typescript
// Bad: Empty catch block
try {
  await riskyOperation();
} catch (error) {
  // Silent failure - never do this
}

// Bad: Catching without handling
try {
  await operation();
} catch (error) {
  console.log(error); // Just logging is insufficient
}

// Bad: Losing error context
try {
  await operation();
} catch (error) {
  throw new Error('Operation failed'); // Lost original error
}

// Bad: String errors
throw 'Something went wrong'; // Use Error objects

// Bad: No async error handling
async function badFunction() {
  const data = await fetchData(); // No try-catch
  processData(data);
}
```

## Rules

1. **Always use try-catch for async operations**: Every async function should handle potential errors

2. **Create custom error classes**: Extend Error for domain-specific errors with metadata

3. **Log errors with context**: Include relevant information (user ID, operation, timestamp)

4. **Preserve error chains**: Use `cause` option to maintain error context

5. **User-friendly messages**: Error messages should be clear for users, technical details in metadata

6. **Fail fast principle**: Validate inputs early, throw errors as soon as issues detected

7. **Handle errors at appropriate level**: Don't catch errors unless you can meaningfully handle them

8. **Type-safe error handling**: Use TypeScript features for typed error handling where possible

9. **No silent failures**: Never catch errors without proper handling or logging

10. **Cleanup in finally**: Use finally blocks for cleanup (close connections, release resources)

## Summary

| Aspect | Requirement | Example |
|--------|-------------|---------|
| Async operations | Always use try-catch | `try { await fn() } catch (e) { }` |
| Custom errors | Extend Error class | `class MyError extends Error` |
| Error messages | User-friendly + metadata | `new Error('msg', { metadata })` |
| Logging | Structured with context | `logger.error('msg', { ctx })` |
| Propagation | Preserve error chain | `{ cause: originalError }` |
| Cleanup | Use finally blocks | `finally { cleanup() }` |
```

---

## Convention Guidelines

All conventions are Comprehensive tier with:
- **Lines**: 150-300 (strict range)
- **Rules**: 7+ detailed guidelines
- **Examples**: 4+ code pairs (Allowed/Forbidden)
- **Subsections**: 2+ domain-specific areas
- **Summary**: Required table format

## Quality Checklist

Good convention examples should:
- [ ] Use realistic, production-quality code
- [ ] Include clear, helpful comments
- [ ] Show both correct and incorrect patterns
- [ ] Explain the "why", not just the "what"
- [ ] Provide sufficient context
- [ ] Use consistent formatting
- [ ] Stay within tier line limits (±20%)
