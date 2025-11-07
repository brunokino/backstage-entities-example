# Testing Strategy

## Overview

The User Service implements a comprehensive testing strategy with multiple layers of tests to ensure reliability, correctness, and maintainability. We follow the **testing pyramid** approach with emphasis on unit tests, supported by integration and end-to-end tests.

## Testing Pyramid

```mermaid
graph TB
    E2E[End-to-End Tests<br/>~5%<br/>Complete user flows]
    Integration[Integration Tests<br/>~25%<br/>Component interactions]
    Unit[Unit Tests<br/>~70%<br/>Individual functions]
    
    E2E --> Integration
    Integration --> Unit
    
    style E2E fill:#FF6B6B
    style Integration fill:#4ECDC4
    style Unit fill:#95E1D3
```

## Test Layers

### 1. Unit Tests (~70%)

Test individual functions, methods, and classes in isolation.

**Focus Areas:**
- Service layer business logic
- Utility functions
- Validators
- Data transformations
- Error handling

**Example:**

```typescript
// tests/unit/services/auth.service.test.ts
import { AuthService } from '../../../src/services/auth.service';
import { UserRepository } from '../../../src/repositories/user.repository';
import bcrypt from 'bcrypt';
import jwt from 'jsonwebtoken';

jest.mock('../../../src/repositories/user.repository');
jest.mock('bcrypt');
jest.mock('jsonwebtoken');

describe('AuthService', () => {
  let authService: AuthService;
  let userRepository: jest.Mocked<UserRepository>;

  beforeEach(() => {
    userRepository = new UserRepository() as jest.Mocked<UserRepository>;
    authService = new AuthService(userRepository);
  });

  describe('register', () => {
    it('should create a new user with hashed password', async () => {
      const userData = {
        email: 'test@example.com',
        password: 'SecurePass123!',
        firstName: 'Test',
        lastName: 'User',
      };

      const hashedPassword = 'hashed_password';
      (bcrypt.hash as jest.Mock).mockResolvedValue(hashedPassword);
      
      userRepository.findByEmail.mockResolvedValue(null);
      userRepository.create.mockResolvedValue({
        id: 'user-id',
        email: userData.email,
        passwordHash: hashedPassword,
        firstName: userData.firstName,
        lastName: userData.lastName,
        status: 'active',
        createdAt: new Date(),
      });

      const result = await authService.register(userData);

      expect(bcrypt.hash).toHaveBeenCalledWith(userData.password, 12);
      expect(userRepository.create).toHaveBeenCalledWith({
        email: userData.email,
        passwordHash: hashedPassword,
        firstName: userData.firstName,
        lastName: userData.lastName,
      });
      expect(result.email).toBe(userData.email);
    });

    it('should throw ConflictError if email already exists', async () => {
      userRepository.findByEmail.mockResolvedValue({
        id: 'existing-id',
        email: 'test@example.com',
      });

      await expect(
        authService.register({
          email: 'test@example.com',
          password: 'pass',
          firstName: 'Test',
          lastName: 'User',
        })
      ).rejects.toThrow('User with this email already exists');
    });
  });

  describe('login', () => {
    it('should return tokens for valid credentials', async () => {
      const user = {
        id: 'user-id',
        email: 'test@example.com',
        passwordHash: 'hashed_password',
      };

      userRepository.findByEmail.mockResolvedValue(user);
      (bcrypt.compare as jest.Mock).mockResolvedValue(true);
      (jwt.sign as jest.Mock).mockReturnValue('mock_token');

      const result = await authService.login('test@example.com', 'password');

      expect(result).toHaveProperty('accessToken');
      expect(result).toHaveProperty('refreshToken');
    });

    it('should throw error for invalid password', async () => {
      const user = {
        id: 'user-id',
        email: 'test@example.com',
        passwordHash: 'hashed_password',
      };

      userRepository.findByEmail.mockResolvedValue(user);
      (bcrypt.compare as jest.Mock).mockResolvedValue(false);

      await expect(
        authService.login('test@example.com', 'wrong_password')
      ).rejects.toThrow('Invalid email or password');
    });
  });
});
```

### 2. Integration Tests (~25%)

Test interactions between components, including database operations.

**Focus Areas:**
- Repository + Database
- Service + Repository
- API routes + Services
- External service integrations

**Example:**

```typescript
// tests/integration/repositories/user.repository.test.ts
import { PrismaClient } from '@prisma/client';
import { UserRepository } from '../../../src/repositories/user.repository';

const prisma = new PrismaClient();
const userRepository = new UserRepository(prisma);

describe('UserRepository Integration', () => {
  beforeAll(async () => {
    await prisma.$connect();
  });

  afterAll(async () => {
    await prisma.user.deleteMany({});
    await prisma.$disconnect();
  });

  afterEach(async () => {
    await prisma.user.deleteMany({});
  });

  describe('create', () => {
    it('should create a user in the database', async () => {
      const userData = {
        email: 'test@example.com',
        passwordHash: 'hashed_password',
        firstName: 'Test',
        lastName: 'User',
      };

      const user = await userRepository.create(userData);

      expect(user.id).toBeDefined();
      expect(user.email).toBe(userData.email);
      expect(user.firstName).toBe(userData.firstName);

      // Verify in database
      const dbUser = await prisma.user.findUnique({
        where: { id: user.id },
      });
      expect(dbUser).not.toBeNull();
      expect(dbUser!.email).toBe(userData.email);
    });

    it('should throw error for duplicate email', async () => {
      const userData = {
        email: 'test@example.com',
        passwordHash: 'hashed_password',
        firstName: 'Test',
        lastName: 'User',
      };

      await userRepository.create(userData);

      await expect(userRepository.create(userData)).rejects.toThrow();
    });
  });

  describe('findByEmail', () => {
    it('should find user by email', async () => {
      const userData = {
        email: 'test@example.com',
        passwordHash: 'hashed_password',
        firstName: 'Test',
        lastName: 'User',
      };

      await userRepository.create(userData);

      const user = await userRepository.findByEmail('test@example.com');

      expect(user).not.toBeNull();
      expect(user!.email).toBe('test@example.com');
    });

    it('should return null for non-existent email', async () => {
      const user = await userRepository.findByEmail('nonexistent@example.com');
      expect(user).toBeNull();
    });
  });
});
```

### 3. End-to-End Tests (~5%)

Test complete user flows through the API.

**Focus Areas:**
- Registration → Login → API access
- User creation → Role assignment → Permission check
- Password change → Re-authentication

**Example:**

```typescript
// tests/e2e/auth.flow.test.ts
import request from 'supertest';
import { app } from '../../src/app';
import { PrismaClient } from '@prisma/client';

const prisma = new PrismaClient();

describe('Authentication Flow E2E', () => {
  beforeAll(async () => {
    await prisma.$connect();
  });

  afterAll(async () => {
    await prisma.user.deleteMany({});
    await prisma.$disconnect();
  });

  it('should complete full authentication flow', async () => {
    // Step 1: Register
    const registerResponse = await request(app)
      .post('/api/v1/auth/register')
      .send({
        email: 'e2e@example.com',
        password: 'SecurePass123!',
        firstName: 'E2E',
        lastName: 'Test',
      })
      .expect(201);

    expect(registerResponse.body).toHaveProperty('accessToken');
    expect(registerResponse.body).toHaveProperty('refreshToken');
    expect(registerResponse.body.user.email).toBe('e2e@example.com');

    const { accessToken, refreshToken } = registerResponse.body;

    // Step 2: Access protected endpoint
    const userResponse = await request(app)
      .get('/api/v1/users')
      .set('Authorization', `Bearer ${accessToken}`)
      .expect(200);

    expect(userResponse.body.data).toBeInstanceOf(Array);

    // Step 3: Logout
    await request(app)
      .post('/api/v1/auth/logout')
      .set('Authorization', `Bearer ${accessToken}`)
      .expect(204);

    // Step 4: Login again
    const loginResponse = await request(app)
      .post('/api/v1/auth/login')
      .send({
        email: 'e2e@example.com',
        password: 'SecurePass123!',
      })
      .expect(200);

    expect(loginResponse.body).toHaveProperty('accessToken');
    expect(loginResponse.body.accessToken).not.toBe(accessToken); // New token

    // Step 5: Refresh token
    const refreshResponse = await request(app)
      .post('/api/v1/auth/refresh')
      .send({ refreshToken: loginResponse.body.refreshToken })
      .expect(200);

    expect(refreshResponse.body).toHaveProperty('accessToken');
  });
});
```

## Testing Tools & Configuration

### Jest Configuration

```javascript
// jest.config.js
module.exports = {
  preset: 'ts-jest',
  testEnvironment: 'node',
  roots: ['<rootDir>/tests'],
  testMatch: ['**/__tests__/**/*.ts', '**/?(*.)+(spec|test).ts'],
  transform: {
    '^.+\\.ts$': 'ts-jest',
  },
  coverageDirectory: 'coverage',
  collectCoverageFrom: [
    'src/**/*.ts',
    '!src/**/*.d.ts',
    '!src/**/*.interface.ts',
    '!src/server.ts',
  ],
  coverageThresholds: {
    global: {
      branches: 80,
      functions: 80,
      lines: 80,
      statements: 80,
    },
  },
  setupFilesAfterEnv: ['<rootDir>/tests/setup.ts'],
};
```

### Test Setup

```typescript
// tests/setup.ts
import { PrismaClient } from '@prisma/client';

const prisma = new PrismaClient();

// Global test setup
beforeAll(async () => {
  // Connect to test database
  await prisma.$connect();
});

afterAll(async () => {
  // Cleanup and disconnect
  await prisma.$disconnect();
});

// Set test timeout
jest.setTimeout(10000);
```

## Running Tests

### Command Line

```bash
# Run all tests
npm test

# Run specific test suite
npm test -- auth.service.test.ts

# Run tests in watch mode
npm run test:watch

# Run tests with coverage
npm run test:coverage

# Run only unit tests
npm run test:unit

# Run only integration tests
npm run test:integration

# Run only E2E tests
npm run test:e2e

# Run tests matching pattern
npm test -- --testNamePattern="should create user"
```

### Coverage Reports

```bash
npm run test:coverage
```

Generates coverage report in `coverage/` directory:

- **HTML Report**: `coverage/lcov-report/index.html`
- **Terminal Output**: Summary in console

**Coverage Thresholds:**
- Statements: 80%
- Branches: 80%
- Functions: 80%
- Lines: 80%

## Test Data Management

### Fixtures

```typescript
// tests/fixtures/users.ts
export const mockUsers = {
  validUser: {
    email: 'john.doe@example.com',
    password: 'SecurePass123!',
    firstName: 'John',
    lastName: 'Doe',
  },
  adminUser: {
    email: 'admin@example.com',
    password: 'AdminPass123!',
    firstName: 'Admin',
    lastName: 'User',
    roles: ['admin'],
  },
};

export const mockTokens = {
  validAccessToken: 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...',
  expiredAccessToken: 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...',
};
```

### Database Seeding for Tests

```typescript
// tests/helpers/seed.ts
import { PrismaClient } from '@prisma/client';
import bcrypt from 'bcrypt';

export async function seedTestDatabase(prisma: PrismaClient) {
  // Create default roles
  const userRole = await prisma.role.create({
    data: {
      name: 'user',
      description: 'Standard user role',
    },
  });

  const adminRole = await prisma.role.create({
    data: {
      name: 'admin',
      description: 'Administrator role',
    },
  });

  // Create test users
  const hashedPassword = await bcrypt.hash('TestPass123!', 12);

  const testUser = await prisma.user.create({
    data: {
      email: 'test@example.com',
      passwordHash: hashedPassword,
      firstName: 'Test',
      lastName: 'User',
      userRoles: {
        create: {
          roleId: userRole.id,
        },
      },
    },
  });

  return { userRole, adminRole, testUser };
}
```

## Mocking Strategies

### Mocking External Services

```typescript
// tests/mocks/email.service.ts
export class MockEmailService {
  async sendEmail(to: string, subject: string, body: string) {
    // Mock implementation
    console.log(`[MOCK] Sending email to ${to}: ${subject}`);
    return { success: true };
  }
}

// Use in tests
jest.mock('../../../src/services/email.service', () => ({
  EmailService: MockEmailService,
}));
```

### Mocking Redis

```typescript
// tests/mocks/redis.ts
export class MockRedisClient {
  private store: Map<string, string> = new Map();

  async get(key: string): Promise<string | null> {
    return this.store.get(key) || null;
  }

  async set(key: string, value: string, options?: any): Promise<void> {
    this.store.set(key, value);
  }

  async del(key: string): Promise<void> {
    this.store.delete(key);
  }

  async keys(pattern: string): Promise<string[]> {
    return Array.from(this.store.keys()).filter(key =>
      key.startsWith(pattern.replace('*', ''))
    );
  }

  clear() {
    this.store.clear();
  }
}
```

### Mocking Time

```typescript
import { advanceTo, clear } from 'jest-date-mock';

describe('Token expiration', () => {
  beforeEach(() => {
    advanceTo(new Date('2025-01-07T10:00:00Z'));
  });

  afterEach(() => {
    clear();
  });

  it('should expire token after 1 hour', () => {
    const token = generateToken();

    // Advance time by 1 hour
    advanceTo(new Date('2025-01-07T11:00:01Z'));

    expect(isTokenExpired(token)).toBe(true);
  });
});
```

## CI/CD Integration

### GitHub Actions Workflow

```yaml
# .github/workflows/test.yml
name: Tests

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

jobs:
  test:
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: user_service_test
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

      redis:
        image: redis:7-alpine
        ports:
          - 6379:6379
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - uses: actions/checkout@v3

      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run migrations
        run: npm run migrate:test
        env:
          DATABASE_URL: postgresql://postgres:postgres@localhost:5432/user_service_test

      - name: Run tests
        run: npm run test:coverage
        env:
          NODE_ENV: test
          DATABASE_URL: postgresql://postgres:postgres@localhost:5432/user_service_test
          REDIS_HOST: localhost
          REDIS_PORT: 6379

      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          files: ./coverage/lcov.info
```

## Best Practices

### 1. Test Independence

Each test should be independent and not rely on other tests.

```typescript
// ❌ Bad: Tests depend on execution order
it('creates user', async () => {
  user = await userService.create({...});
});

it('updates user', async () => {
  await userService.update(user.id, {...}); // Depends on previous test
});

// ✅ Good: Each test is independent
it('creates user', async () => {
  const user = await userService.create({...});
  expect(user).toBeDefined();
});

it('updates user', async () => {
  const user = await userService.create({...}); // Create own data
  await userService.update(user.id, {...});
});
```

### 2. Descriptive Test Names

Use clear, descriptive test names that explain what is being tested.

```typescript
// ❌ Bad
it('works', () => { ... });
it('test 1', () => { ... });

// ✅ Good
it('should create user with valid data', () => { ... });
it('should throw ConflictError when email already exists', () => { ... });
it('should return 401 when token is expired', () => { ... });
```

### 3. Arrange-Act-Assert Pattern

Structure tests with clear AAA pattern.

```typescript
it('should authenticate user with valid credentials', async () => {
  // Arrange
  const email = 'test@example.com';
  const password = 'SecurePass123!';
  await createTestUser(email, password);

  // Act
  const result = await authService.login(email, password);

  // Assert
  expect(result).toHaveProperty('accessToken');
  expect(result).toHaveProperty('refreshToken');
  expect(result.user.email).toBe(email);
});
```

### 4. Test Edge Cases

Test not only happy paths but also edge cases and error conditions.

```typescript
describe('User creation', () => {
  it('should create user with valid data', async () => { ... });
  it('should throw error when email is missing', async () => { ... });
  it('should throw error when email is invalid', async () => { ... });
  it('should throw error when password is too short', async () => { ... });
  it('should throw error when email already exists', async () => { ... });
});
```

### 5. Clean Up After Tests

Always clean up test data to prevent test pollution.

```typescript
afterEach(async () => {
  await prisma.user.deleteMany({});
  await prisma.session.deleteMany({});
  redisClient.clear();
});
```

## Performance Testing

While not part of regular test suite, performance testing is important:

```bash
# Using Artillery
artillery quick --count 100 --num 10 http://localhost:3000/api/v1/health

# Using Apache Bench
ab -n 1000 -c 10 http://localhost:3000/api/v1/health
```

---

**Last Updated**: 2025-01-07  
**Document Owner**: Platform Team

