# Local Development Setup

## Prerequisites

Before setting up the User Service locally, ensure you have the following installed:

| Tool | Version | Installation |
|------|---------|--------------|
| **Node.js** | 20 LTS or higher | [nodejs.org](https://nodejs.org/) |
| **npm** | 9.x or higher | Included with Node.js |
| **PostgreSQL** | 15 or higher | [postgresql.org](https://www.postgresql.org/) |
| **Redis** | 7.x or higher | [redis.io](https://redis.io/) |
| **Git** | Latest | [git-scm.com](https://git-scm.com/) |

### Verify Prerequisites

```bash
# Check Node.js version
node --version
# Expected: v20.x.x or higher

# Check npm version
npm --version
# Expected: 9.x.x or higher

# Check PostgreSQL
psql --version
# Expected: psql (PostgreSQL) 15.x or higher

# Check Redis
redis-cli --version
# Expected: redis-cli 7.x.x or higher

# Check Git
git --version
# Expected: git version 2.x.x
```

## Quick Start

### 1. Clone the Repository

```bash
git clone https://github.com/brunokino/backstage-entities-example.git
cd backstage-entities-example/services/user-service
```

### 2. Install Dependencies

```bash
npm install
```

This will install all production and development dependencies defined in `package.json`.

### 3. Set Up Environment Variables

Create a `.env` file in the project root:

```bash
cp .env.example .env
```

Edit `.env` with your local configuration:

```bash
# Application
NODE_ENV=development
PORT=3000
API_VERSION=v1

# Database
DATABASE_URL="postgresql://postgres:password@localhost:5432/user_service_dev?schema=public"

# Redis
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_PASSWORD=
REDIS_DB=0

# JWT
JWT_SECRET=your-super-secret-jwt-key-change-in-production
JWT_ACCESS_EXPIRATION=3600
JWT_REFRESH_EXPIRATION=1296000

# External Services
EMAIL_SERVICE_URL=http://localhost:3001/api/v1
AUDIT_SERVICE_URL=http://localhost:3002/api/v1

# Rate Limiting
RATE_LIMIT_WINDOW_MS=900000
RATE_LIMIT_MAX_ANONYMOUS=100
RATE_LIMIT_MAX_AUTHENTICATED=1000

# Logging
LOG_LEVEL=debug
LOG_FORMAT=json

# CORS
CORS_ORIGIN=http://localhost:3000,http://localhost:3001
```

**Important:** Never commit the `.env` file. It's included in `.gitignore`.

### 4. Set Up Database

#### Start PostgreSQL

**Using Docker:**

```bash
docker run --name postgres-user-service \
  -e POSTGRES_USER=postgres \
  -e POSTGRES_PASSWORD=password \
  -e POSTGRES_DB=user_service_dev \
  -p 5432:5432 \
  -d postgres:15
```

**Using local PostgreSQL:**

```bash
# Create database
createdb user_service_dev

# Or using psql
psql -U postgres -c "CREATE DATABASE user_service_dev;"
```

#### Run Migrations

```bash
# Generate Prisma client
npm run prisma:generate

# Run migrations
npm run migrate:dev
```

This will:
1. Create all necessary database tables
2. Set up indexes
3. Seed initial data (default roles)

#### Verify Database Setup

```bash
# Check tables
npm run prisma:studio

# Or use psql
psql -U postgres -d user_service_dev -c "\dt"
```

Expected tables:
- `users`
- `roles`
- `permissions`
- `user_roles`
- `role_permissions`
- `sessions`
- `audit_logs`
- `_prisma_migrations`

### 5. Set Up Redis

**Using Docker:**

```bash
docker run --name redis-user-service \
  -p 6379:6379 \
  -d redis:7-alpine
```

**Using local Redis:**

```bash
# Start Redis server
redis-server

# Verify connection
redis-cli ping
# Expected: PONG
```

### 6. Start Development Server

```bash
npm run dev
```

The service will start on `http://localhost:3000` (or the port specified in `.env`).

**Expected output:**

```
[INFO] Starting User Service...
[INFO] Environment: development
[INFO] Database connected: postgresql://localhost:5432/user_service_dev
[INFO] Redis connected: localhost:6379
[INFO] Server listening on http://localhost:3000
[INFO] API Documentation: http://localhost:3000/api-docs
```

### 7. Verify Setup

**Health check:**

```bash
curl http://localhost:3000/api/v1/health
```

**Expected response:**

```json
{
  "status": "healthy",
  "timestamp": "2025-01-07T10:30:00Z",
  "version": "1.0.0",
  "uptime": 10
}
```

**Access Swagger UI:**

Open your browser and navigate to: `http://localhost:3000/api-docs`

## Development Workflow

### Project Structure

```
user-service/
├── src/
│   ├── config/           # Configuration files
│   ├── controllers/      # Route controllers
│   ├── middleware/       # Express middleware
│   ├── routes/           # API route definitions
│   ├── services/         # Business logic
│   ├── repositories/     # Data access layer
│   ├── models/           # Type definitions
│   ├── utils/            # Utility functions
│   ├── validators/       # Request validators
│   ├── app.ts            # Express app setup
│   └── server.ts         # Server entry point
├── prisma/
│   ├── schema.prisma     # Database schema
│   ├── migrations/       # Database migrations
│   └── seed.ts           # Seed data
├── tests/
│   ├── unit/             # Unit tests
│   ├── integration/      # Integration tests
│   └── e2e/              # End-to-end tests
├── docs/                 # TechDocs documentation
├── .env.example          # Environment variables template
├── .eslintrc.json        # ESLint configuration
├── .prettierrc           # Prettier configuration
├── tsconfig.json         # TypeScript configuration
├── jest.config.js        # Jest configuration
├── package.json          # Dependencies and scripts
└── README.md             # Project README
```

### Available npm Scripts

```bash
# Development
npm run dev              # Start development server with hot reload
npm run build            # Build TypeScript to JavaScript
npm run start            # Start production server
npm run clean            # Clean build directory

# Database
npm run prisma:generate  # Generate Prisma client
npm run migrate:dev      # Run migrations (dev)
npm run migrate:deploy   # Run migrations (production)
npm run prisma:studio    # Open Prisma Studio (database GUI)
npm run seed             # Seed database with initial data

# Testing
npm test                 # Run all tests
npm run test:unit        # Run unit tests
npm run test:integration # Run integration tests
npm run test:e2e         # Run end-to-end tests
npm run test:watch       # Run tests in watch mode
npm run test:coverage    # Generate coverage report

# Code Quality
npm run lint             # Run ESLint
npm run lint:fix         # Fix ESLint errors
npm run format           # Format code with Prettier
npm run format:check     # Check code formatting
npm run type-check       # Type check without building

# Other
npm run docs:serve       # Serve TechDocs locally
```

### Making Changes

#### 1. Create a Feature Branch

```bash
git checkout -b feature/add-user-search
```

#### 2. Make Changes

Edit files in `src/` directory. The development server will automatically reload.

#### 3. Run Tests

```bash
# Run affected tests
npm test

# Run tests with coverage
npm run test:coverage
```

#### 4. Check Code Quality

```bash
# Lint code
npm run lint

# Format code
npm run format

# Type check
npm run type-check
```

#### 5. Commit Changes

```bash
git add .
git commit -m "feat(users): add user search functionality"
```

**Commit message format:** `type(scope): description`

Types: `feat`, `fix`, `docs`, `style`, `refactor`, `test`, `chore`

### Database Changes

#### Creating a Migration

```bash
# Make changes to prisma/schema.prisma
# Then create migration
npm run migrate:dev -- --name add_user_status_field
```

#### Applying Migrations

```bash
# Development
npm run migrate:dev

# Production
npm run migrate:deploy
```

#### Resetting Database

```bash
# WARNING: This will delete all data
npm run prisma:reset
```

## Testing Locally

### Manual API Testing

#### Using cURL

**Register a user:**

```bash
curl -X POST http://localhost:3000/api/v1/auth/register \
  -H "Content-Type: application/json" \
  -d '{
    "email": "test@example.com",
    "password": "TestPass123!",
    "firstName": "Test",
    "lastName": "User"
  }'
```

**Login:**

```bash
# Save response to variable
RESPONSE=$(curl -X POST http://localhost:3000/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{
    "email": "test@example.com",
    "password": "TestPass123!"
  }')

# Extract access token
TOKEN=$(echo $RESPONSE | jq -r '.accessToken')

# Use token
curl -X GET http://localhost:3000/api/v1/users \
  -H "Authorization: Bearer $TOKEN"
```

#### Using Postman

1. Import OpenAPI specification: `http://localhost:3000/api-docs.json`
2. Set environment variable: `baseUrl = http://localhost:3000/api/v1`
3. Create requests or use auto-generated collection

#### Using Swagger UI

Navigate to `http://localhost:3000/api-docs` and test APIs interactively.

### Automated Testing

```bash
# Run all tests
npm test

# Run specific test file
npm test -- user.service.test.ts

# Run tests in watch mode
npm run test:watch

# Generate coverage report
npm run test:coverage
```

**Coverage thresholds:**
- Statements: 80%
- Branches: 80%
- Functions: 80%
- Lines: 80%

## Troubleshooting

### Common Issues

#### Port Already in Use

**Error:**

```
Error: listen EADDRINUSE: address already in use :::3000
```

**Solution:**

```bash
# Find process using port 3000
lsof -i :3000

# Kill process
kill -9 <PID>

# Or change port in .env
PORT=3001
```

#### Database Connection Failed

**Error:**

```
Error: Connection refused (postgresql://localhost:5432/user_service_dev)
```

**Solution:**

```bash
# Check if PostgreSQL is running
pg_isready

# Start PostgreSQL
# macOS (Homebrew)
brew services start postgresql@15

# Linux (systemd)
sudo systemctl start postgresql

# Docker
docker start postgres-user-service
```

#### Redis Connection Failed

**Error:**

```
Error: Redis connection to localhost:6379 failed
```

**Solution:**

```bash
# Check if Redis is running
redis-cli ping

# Start Redis
# macOS (Homebrew)
brew services start redis

# Linux (systemd)
sudo systemctl start redis

# Docker
docker start redis-user-service
```

#### Prisma Client Not Generated

**Error:**

```
Error: Cannot find module '@prisma/client'
```

**Solution:**

```bash
npm run prisma:generate
```

#### Migration Failed

**Error:**

```
Error: Migration failed to apply
```

**Solution:**

```bash
# Reset database (WARNING: deletes all data)
npm run prisma:reset

# Or manually fix and reapply
npm run prisma:migrate resolve --applied <migration-name>
npm run migrate:dev
```

## Development Tips

### Hot Reload

The development server uses `nodemon` and `tsx` for automatic reloading. Changes to TypeScript files will trigger a server restart.

### Debugging

#### VS Code Launch Configuration

Create `.vscode/launch.json`:

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "type": "node",
      "request": "launch",
      "name": "Debug User Service",
      "runtimeExecutable": "npm",
      "runtimeArgs": ["run", "dev"],
      "skipFiles": ["<node_internals>/**"],
      "console": "integratedTerminal",
      "env": {
        "NODE_ENV": "development"
      }
    }
  ]
}
```

#### Using Node Debugger

```bash
# Start with debugger
node --inspect-brk -r tsx/register src/server.ts

# Connect with Chrome DevTools
# Open: chrome://inspect
```

### Database GUI

**Prisma Studio:**

```bash
npm run prisma:studio
```

Opens at `http://localhost:5555` with interactive database explorer.

**pgAdmin / DBeaver / TablePlus:**

Connect with:
- Host: localhost
- Port: 5432
- Database: user_service_dev
- Username: postgres
- Password: (from .env)

### Logging

Adjust log level in `.env`:

```bash
LOG_LEVEL=debug    # Most verbose: debug, info, warn, error
LOG_FORMAT=pretty  # or 'json' for structured logging
```

### Environment-Specific Configuration

Create multiple `.env` files:

```bash
.env                # Default (development)
.env.test           # Testing
.env.staging        # Staging
.env.production     # Production (never commit)
```

Load specific env:

```bash
NODE_ENV=test npm test
```

## Next Steps

Once your local development environment is set up:

1. **Read the Architecture docs** - [architecture.md](architecture.md)
2. **Explore the API** - [api-reference.md](api-reference.md)
3. **Understand Authentication** - [authentication.md](authentication.md)
4. **Learn Testing strategies** - [testing.md](testing.md)

---

**Last Updated**: 2025-01-07  
**Document Owner**: Platform Team

