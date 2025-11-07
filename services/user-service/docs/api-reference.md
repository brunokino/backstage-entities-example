# API Reference

## Base URL

The User Service API is available at the following base URLs:

| Environment | Base URL | Purpose |
|-------------|----------|---------|
| Production | `https://user-service.kinoshita-labs.com/api/v1` | Production environment |
| Staging | `https://user-service-staging.kinoshita-labs.com/api/v1` | Testing and validation |
| Local | `http://localhost:3000/api/v1` | Local development |

## Interactive Documentation

The User Service provides interactive API documentation through Swagger UI:

- **Production**: https://user-service.kinoshita-labs.com/api-docs
- **Staging**: https://user-service-staging.kinoshita-labs.com/api-docs
- **Local**: http://localhost:3000/api-docs

## API Endpoints Overview

### Health Check

| Method | Endpoint | Description | Authentication |
|--------|----------|-------------|----------------|
| GET | `/health` | Service health status | No |

### Authentication

| Method | Endpoint | Description | Authentication |
|--------|----------|-------------|----------------|
| POST | `/auth/register` | Register new user | No |
| POST | `/auth/login` | Authenticate user | No |
| POST | `/auth/refresh` | Refresh access token | No |
| POST | `/auth/logout` | Invalidate refresh token | Yes |

### User Management

| Method | Endpoint | Description | Authentication |
|--------|----------|-------------|----------------|
| GET | `/users` | List users (paginated) | Yes |
| GET | `/users/{userId}` | Get user by ID | Yes |
| PATCH | `/users/{userId}` | Update user profile | Yes |
| DELETE | `/users/{userId}` | Delete user account | Yes |
| PUT | `/users/{userId}/password` | Change user password | Yes |

### Role Management

| Method | Endpoint | Description | Authentication |
|--------|----------|-------------|----------------|
| GET | `/roles` | List all available roles | Yes |
| GET | `/users/{userId}/roles` | Get user's assigned roles | Yes |
| POST | `/users/{userId}/roles` | Assign role to user | Yes |
| DELETE | `/users/{userId}/roles/{roleId}` | Revoke role from user | Yes |

### Administrative Operations

| Method | Endpoint | Description | Authentication |
|--------|----------|-------------|----------------|
| POST | `/admin/users/{userId}/suspend` | Suspend user account | Yes (Admin) |
| POST | `/admin/users/{userId}/activate` | Activate suspended account | Yes (Admin) |

## Detailed Endpoint Documentation

### Health Check

#### GET /health

Check the health status of the service.

**Request:**

```bash
curl -X GET https://user-service.kinoshita-labs.com/api/v1/health
```

**Response (200 OK):**

```json
{
  "status": "healthy",
  "timestamp": "2025-01-07T10:30:00Z",
  "version": "1.0.0",
  "uptime": 86400
}
```

---

### Authentication Endpoints

#### POST /auth/register

Register a new user account.

**Request:**

```bash
curl -X POST https://user-service.kinoshita-labs.com/api/v1/auth/register \
  -H "Content-Type: application/json" \
  -d '{
    "email": "john.doe@example.com",
    "password": "SecurePass123!",
    "firstName": "John",
    "lastName": "Doe",
    "phoneNumber": "+1234567890"
  }'
```

**Response (201 Created):**

```json
{
  "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "refreshToken": "dGhpcyBpcyBhIHJlZnJlc2ggdG9rZW4=",
  "expiresIn": 3600,
  "tokenType": "Bearer",
  "user": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "email": "john.doe@example.com",
    "firstName": "John",
    "lastName": "Doe",
    "phoneNumber": "+1234567890",
    "status": "active",
    "roles": [
      {
        "id": "660e8400-e29b-41d4-a716-446655440001",
        "name": "user",
        "description": "Standard user role"
      }
    ],
    "createdAt": "2025-01-07T10:30:00Z",
    "updatedAt": "2025-01-07T10:30:00Z"
  }
}
```

**Validation Rules:**
- Email: Must be valid email format
- Password: Minimum 8 characters with uppercase, lowercase, number, and special character
- First Name: 1-50 characters
- Last Name: 1-50 characters
- Phone Number: Optional, E.164 format

---

#### POST /auth/login

Authenticate with email and password.

**Request:**

```bash
curl -X POST https://user-service.kinoshita-labs.com/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{
    "email": "john.doe@example.com",
    "password": "SecurePass123!"
  }'
```

**Response (200 OK):**

```json
{
  "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "refreshToken": "dGhpcyBpcyBhIHJlZnJlc2ggdG9rZW4=",
  "expiresIn": 3600,
  "tokenType": "Bearer",
  "user": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "email": "john.doe@example.com",
    "firstName": "John",
    "lastName": "Doe",
    "status": "active",
    "roles": [
      {
        "id": "660e8400-e29b-41d4-a716-446655440001",
        "name": "user",
        "description": "Standard user role"
      }
    ],
    "lastLoginAt": "2025-01-07T10:30:00Z"
  }
}
```

---

#### POST /auth/refresh

Get a new access token using a refresh token.

**Request:**

```bash
curl -X POST https://user-service.kinoshita-labs.com/api/v1/auth/refresh \
  -H "Content-Type: application/json" \
  -d '{
    "refreshToken": "dGhpcyBpcyBhIHJlZnJlc2ggdG9rZW4="
  }'
```

**Response (200 OK):**

```json
{
  "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "expiresIn": 3600,
  "tokenType": "Bearer"
}
```

---

#### POST /auth/logout

Invalidate the current refresh token.

**Request:**

```bash
curl -X POST https://user-service.kinoshita-labs.com/api/v1/auth/logout \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
```

**Response (204 No Content)**

---

### User Management Endpoints

#### GET /users

List users with pagination and filtering.

**Request:**

```bash
curl -X GET 'https://user-service.kinoshita-labs.com/api/v1/users?page=1&limit=20&search=john&status=active' \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
```

**Query Parameters:**

| Parameter | Type | Description | Default |
|-----------|------|-------------|---------|
| page | integer | Page number (1-indexed) | 1 |
| limit | integer | Items per page (1-100) | 20 |
| search | string | Search by name or email | - |
| role | string | Filter by role name | - |
| status | string | Filter by status (active/inactive/suspended) | - |

**Response (200 OK):**

```json
{
  "data": [
    {
      "id": "550e8400-e29b-41d4-a716-446655440000",
      "email": "john.doe@example.com",
      "firstName": "John",
      "lastName": "Doe",
      "phoneNumber": "+1234567890",
      "status": "active",
      "roles": [
        {
          "id": "660e8400-e29b-41d4-a716-446655440001",
          "name": "user",
          "description": "Standard user role"
        }
      ],
      "createdAt": "2025-01-01T10:00:00Z",
      "updatedAt": "2025-01-07T10:30:00Z",
      "lastLoginAt": "2025-01-07T09:00:00Z"
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 150,
    "totalPages": 8
  }
}
```

---

#### GET /users/{userId}

Get detailed information about a specific user.

**Request:**

```bash
curl -X GET https://user-service.kinoshita-labs.com/api/v1/users/550e8400-e29b-41d4-a716-446655440000 \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
```

**Response (200 OK):**

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "email": "john.doe@example.com",
  "firstName": "John",
  "lastName": "Doe",
  "phoneNumber": "+1234567890",
  "status": "active",
  "roles": [
    {
      "id": "660e8400-e29b-41d4-a716-446655440001",
      "name": "user",
      "description": "Standard user role",
      "permissions": ["users:read", "profile:write"]
    }
  ],
  "createdAt": "2025-01-01T10:00:00Z",
  "updatedAt": "2025-01-07T10:30:00Z",
  "lastLoginAt": "2025-01-07T09:00:00Z"
}
```

---

#### PATCH /users/{userId}

Update user profile information (partial update).

**Request:**

```bash
curl -X PATCH https://user-service.kinoshita-labs.com/api/v1/users/550e8400-e29b-41d4-a716-446655440000 \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..." \
  -H "Content-Type: application/json" \
  -d '{
    "firstName": "Jonathan",
    "phoneNumber": "+9876543210"
  }'
```

**Response (200 OK):**

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "email": "john.doe@example.com",
  "firstName": "Jonathan",
  "lastName": "Doe",
  "phoneNumber": "+9876543210",
  "status": "active",
  "roles": [...],
  "createdAt": "2025-01-01T10:00:00Z",
  "updatedAt": "2025-01-07T11:00:00Z",
  "lastLoginAt": "2025-01-07T09:00:00Z"
}
```

---

#### PUT /users/{userId}/password

Change user password.

**Request:**

```bash
curl -X PUT https://user-service.kinoshita-labs.com/api/v1/users/550e8400-e29b-41d4-a716-446655440000/password \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..." \
  -H "Content-Type: application/json" \
  -d '{
    "currentPassword": "SecurePass123!",
    "newPassword": "NewSecurePass456!"
  }'
```

**Response (204 No Content)**

---

### Role Management Endpoints

#### GET /roles

List all available roles in the system.

**Request:**

```bash
curl -X GET https://user-service.kinoshita-labs.com/api/v1/roles \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
```

**Response (200 OK):**

```json
{
  "data": [
    {
      "id": "660e8400-e29b-41d4-a716-446655440001",
      "name": "user",
      "description": "Standard user role",
      "permissions": ["users:read", "profile:write"]
    },
    {
      "id": "660e8400-e29b-41d4-a716-446655440002",
      "name": "moderator",
      "description": "Content moderator role",
      "permissions": ["users:read", "users:suspend", "content:moderate"]
    },
    {
      "id": "660e8400-e29b-41d4-a716-446655440003",
      "name": "admin",
      "description": "Administrator with full system access",
      "permissions": ["*:*"]
    }
  ]
}
```

---

#### POST /users/{userId}/roles

Assign a role to a user.

**Request:**

```bash
curl -X POST https://user-service.kinoshita-labs.com/api/v1/users/550e8400-e29b-41d4-a716-446655440000/roles \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..." \
  -H "Content-Type: application/json" \
  -d '{
    "roleId": "660e8400-e29b-41d4-a716-446655440002"
  }'
```

**Response (201 Created):**

```json
{
  "id": "660e8400-e29b-41d4-a716-446655440002",
  "name": "moderator",
  "description": "Content moderator role",
  "permissions": ["users:read", "users:suspend", "content:moderate"]
}
```

---

## Request/Response Formats

### Standard Response Structure

All successful responses follow this structure:

```json
{
  "data": {},  // or []
  "pagination": {},  // Optional, for paginated endpoints
  "meta": {}  // Optional metadata
}
```

### Error Response Structure

All error responses follow this structure:

```json
{
  "error": "ErrorType",
  "message": "Human-readable error message",
  "statusCode": 400,
  "details": [
    {
      "field": "email",
      "message": "Must be a valid email address"
    }
  ],
  "timestamp": "2025-01-07T10:30:00Z",
  "path": "/api/v1/users"
}
```

### Common HTTP Status Codes

| Status Code | Meaning | Usage |
|-------------|---------|-------|
| 200 | OK | Successful GET, PATCH requests |
| 201 | Created | Successful POST requests (resource created) |
| 204 | No Content | Successful DELETE or PUT with no response body |
| 400 | Bad Request | Invalid request format or validation failure |
| 401 | Unauthorized | Missing or invalid authentication token |
| 403 | Forbidden | Insufficient permissions for the operation |
| 404 | Not Found | Requested resource does not exist |
| 409 | Conflict | Resource already exists (e.g., duplicate email) |
| 429 | Too Many Requests | Rate limit exceeded |
| 500 | Internal Server Error | Unexpected server error |
| 503 | Service Unavailable | Service temporarily down (maintenance) |

## Rate Limiting

Rate limits are applied per IP address (anonymous) or per user (authenticated).

| User Type | Limit | Window | Header |
|-----------|-------|--------|--------|
| Anonymous | 100 requests | 15 minutes | `X-RateLimit-Limit: 100` |
| Authenticated | 1000 requests | 15 minutes | `X-RateLimit-Limit: 1000` |

**Rate Limit Headers:**

```
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 847
X-RateLimit-Reset: 1704624000
```

When rate limit is exceeded:

```json
{
  "error": "RateLimitExceeded",
  "message": "Too many requests. Please try again later.",
  "statusCode": 429,
  "retryAfter": 900,
  "timestamp": "2025-01-07T10:30:00Z"
}
```

## Pagination

All list endpoints support pagination using query parameters:

| Parameter | Type | Description | Default | Range |
|-----------|------|-------------|---------|-------|
| page | integer | Page number (1-indexed) | 1 | 1-∞ |
| limit | integer | Items per page | 20 | 1-100 |

**Pagination Response:**

```json
{
  "data": [...],
  "pagination": {
    "page": 2,
    "limit": 20,
    "total": 150,
    "totalPages": 8
  }
}
```

## Filtering & Searching

List endpoints support filtering and searching:

**User Search:**

```bash
# Search by name or email
GET /users?search=john

# Filter by status
GET /users?status=active

# Filter by role
GET /users?role=admin

# Combine filters
GET /users?status=active&role=moderator&search=john
```

## OpenAPI Specification

The complete OpenAPI 3.0 specification is available at:

- **YAML**: [services/user-service/openapi.yaml](https://github.com/brunokino/backstage-entities-example/blob/main/services/user-service/openapi.yaml)
- **Swagger UI**: https://user-service.kinoshita-labs.com/api-docs

You can import this specification into tools like Postman, Insomnia, or generate client SDKs using OpenAPI Generator.

---

**Relevant Source Files:**
- [services/user-service/openapi.yaml](https://github.com/brunokino/backstage-entities-example/blob/main/services/user-service/openapi.yaml)
- [services/user-service/src/routes/](https://github.com/brunokino/backstage-entities-example)

**Last Updated**: 2025-01-07  
**Document Owner**: Platform Team

