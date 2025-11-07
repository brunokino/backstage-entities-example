# Authentication & Authorization

## Overview

The User Service implements a robust authentication and authorization system based on **JWT (JSON Web Tokens)** for stateless authentication and **RBAC (Role-Based Access Control)** for fine-grained authorization.

## Authentication Flow

### Registration & Login Flow

```mermaid
sequenceDiagram
    participant User
    participant Client
    participant API
    participant AuthService
    participant Database
    participant Redis
    
    rect rgb(200, 220, 240)
        Note over User,Redis: Registration Flow
        User->>Client: Enter registration details
        Client->>API: POST /auth/register
        API->>AuthService: Validate & create user
        AuthService->>Database: Store user (hashed password)
        AuthService->>AuthService: Generate JWT tokens
        AuthService->>Redis: Store session
        AuthService-->>Client: Return tokens + user data
        Client->>Client: Store tokens securely
    end
    
    rect rgb(220, 240, 220)
        Note over User,Redis: Login Flow
        User->>Client: Enter credentials
        Client->>API: POST /auth/login
        API->>AuthService: Authenticate
        AuthService->>Database: Verify credentials
        AuthService->>AuthService: Compare password hash
        AuthService->>AuthService: Generate JWT tokens
        AuthService->>Redis: Store session
        AuthService->>Database: Update last_login
        AuthService-->>Client: Return tokens + user data
        Client->>Client: Store tokens securely
    end
```

## Token Architecture

The User Service uses a **two-token system**:

1. **Access Token** - Short-lived JWT for API authentication
2. **Refresh Token** - Long-lived token for obtaining new access tokens

### Access Token (JWT)

**Lifetime:** 1 hour (3600 seconds)  
**Storage:** Client-side (memory or secure HTTP-only cookie)  
**Purpose:** Authenticate API requests

**JWT Structure:**

```json
{
  "header": {
    "alg": "HS256",
    "typ": "JWT"
  },
  "payload": {
    "sub": "550e8400-e29b-41d4-a716-446655440000",
    "email": "john.doe@example.com",
    "roles": ["user", "moderator"],
    "permissions": ["users:read", "users:write"],
    "iat": 1704624000,
    "exp": 1704627600
  },
  "signature": "..."
}
```

**Claims:**

| Claim | Description | Example |
|-------|-------------|---------|
| `sub` | Subject (User ID) | `550e8400-e29b-41d4-a716-446655440000` |
| `email` | User email address | `john.doe@example.com` |
| `roles` | Array of user roles | `["user", "moderator"]` |
| `permissions` | Array of permissions | `["users:read", "users:write"]` |
| `iat` | Issued at (Unix timestamp) | `1704624000` |
| `exp` | Expiration (Unix timestamp) | `1704627600` |

### Refresh Token

**Lifetime:** 15 days (1,296,000 seconds)  
**Storage:** Database + Redis cache  
**Purpose:** Obtain new access tokens without re-authentication

**Characteristics:**
- Opaque token (random string, not JWT)
- Stored in database with user association
- Can be invalidated server-side
- Single-use or limited reuse

## Token Management Flow

### Using Access Tokens

```bash
# Include access token in Authorization header
curl -X GET https://user-service.kinoshita-labs.com/api/v1/users/me \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
```

### Refreshing Tokens

```mermaid
sequenceDiagram
    participant Client
    participant API
    participant AuthService
    participant Redis
    participant Database
    
    Client->>API: POST /auth/refresh (with refresh token)
    API->>AuthService: Validate refresh token
    
    alt Token in Redis cache
        AuthService->>Redis: Validate token
        Redis-->>AuthService: Session data
    else Token not in cache
        AuthService->>Database: Lookup session
        Database-->>AuthService: Session data
        AuthService->>Redis: Cache session
    end
    
    alt Token valid
        AuthService->>AuthService: Generate new access token
        AuthService-->>API: New access token
        API-->>Client: 200 OK + new access token
    else Token invalid/expired
        AuthService-->>API: Error
        API-->>Client: 401 Unauthorized
        Client->>Client: Redirect to login
    end
```

### Logout Flow

```mermaid
sequenceDiagram
    participant Client
    participant API
    participant AuthService
    participant Redis
    participant Database
    
    Client->>API: POST /auth/logout (with access token)
    API->>AuthService: Invalidate session
    AuthService->>Redis: Delete cached session
    AuthService->>Database: Delete refresh token
    AuthService-->>API: Success
    API-->>Client: 204 No Content
    Client->>Client: Clear stored tokens
```

## Password Security

### Password Requirements

- **Minimum length:** 8 characters
- **Complexity:** Must contain:
  - At least one uppercase letter (A-Z)
  - At least one lowercase letter (a-z)
  - At least one number (0-9)
  - At least one special character (!@#$%^&*()_+-=[]{}|;:,.<>?)

**Validation Regex:**

```regex
^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)(?=.*[@$!%*?&])[A-Za-z\d@$!%*?&]{8,}$
```

### Password Hashing

The User Service uses **bcrypt** for password hashing:

- **Algorithm:** bcrypt
- **Cost Factor:** 12 (2^12 = 4096 iterations)
- **Salt:** Automatically generated per password

**Example Implementation:**

```typescript
import bcrypt from 'bcrypt';

// Hash password
const saltRounds = 12;
const hashedPassword = await bcrypt.hash(plainPassword, saltRounds);

// Verify password
const isValid = await bcrypt.compare(plainPassword, hashedPassword);
```

**Why bcrypt?**
- Deliberately slow (prevents brute-force attacks)
- Adaptive (cost factor can be increased as hardware improves)
- Automatic salt generation
- Industry-standard and battle-tested

## Authorization (RBAC)

### Permission Model

The User Service implements a hierarchical permission model:

```mermaid
graph TB
    User[User] --> UserRole[User-Role Assignment]
    UserRole --> Role[Role]
    Role --> RolePerm[Role-Permission Assignment]
    RolePerm --> Permission[Permission]
    
    Permission --> Resource[Resource]
    Permission --> Action[Action]
    
    style User fill:#4A90E2
    style Role fill:#50C878
    style Permission fill:#FF6B6B
```

### Default Roles

| Role | Description | Permissions |
|------|-------------|-------------|
| **user** | Standard user | `users:read`, `profile:write` |
| **moderator** | Content moderator | All user permissions + `users:suspend`, `content:moderate` |
| **admin** | System administrator | `*:*` (all permissions) |

### Permission Format

Permissions follow the format: `resource:action`

**Examples:**
- `users:read` - Read user data
- `users:write` - Create or update users
- `users:delete` - Delete users
- `roles:manage` - Manage role assignments
- `*:*` - All permissions (admin)

### Authorization Flow

```mermaid
sequenceDiagram
    participant Client
    participant Middleware
    participant AuthService
    participant RoleService
    participant Handler
    
    Client->>Middleware: Request + JWT
    Middleware->>AuthService: Verify token
    
    alt Token valid
        AuthService-->>Middleware: User + Roles
        Middleware->>RoleService: Check permission
        
        RoleService->>RoleService: Load user roles
        RoleService->>RoleService: Check required permission
        
        alt Has permission
            RoleService-->>Middleware: Authorized
            Middleware->>Handler: Execute
            Handler-->>Client: 200 OK + data
        else No permission
            RoleService-->>Middleware: Forbidden
            Middleware-->>Client: 403 Forbidden
        end
    else Token invalid
        AuthService-->>Middleware: Unauthorized
        Middleware-->>Client: 401 Unauthorized
    end
```

### Protecting Endpoints

**Example Authorization Middleware:**

```typescript
import { Request, Response, NextFunction } from 'express';

// Require authentication
export const requireAuth = async (
  req: Request,
  res: Response,
  next: NextFunction
) => {
  const token = req.headers.authorization?.replace('Bearer ', '');
  
  if (!token) {
    return res.status(401).json({
      error: 'Unauthorized',
      message: 'No authentication token provided'
    });
  }
  
  try {
    const payload = await verifyToken(token);
    req.user = payload;
    next();
  } catch (error) {
    return res.status(401).json({
      error: 'Unauthorized',
      message: 'Invalid or expired token'
    });
  }
};

// Require specific permission
export const requirePermission = (permission: string) => {
  return async (req: Request, res: Response, next: NextFunction) => {
    const hasPermission = await checkUserPermission(
      req.user.id,
      permission
    );
    
    if (!hasPermission) {
      return res.status(403).json({
        error: 'Forbidden',
        message: `Insufficient permissions. Required: ${permission}`
      });
    }
    
    next();
  };
};

// Usage
app.delete(
  '/users/:id',
  requireAuth,
  requirePermission('users:delete'),
  deleteUserHandler
);
```

## Security Best Practices

### Token Storage

**✅ Recommended:**
- Store access token in memory (React state, Vue store)
- Store refresh token in HTTP-only, Secure cookie
- Never store tokens in localStorage (XSS vulnerable)

**❌ Not Recommended:**
- localStorage (vulnerable to XSS attacks)
- sessionStorage (vulnerable to XSS attacks)
- Regular cookies without HttpOnly flag

### Token Transmission

**Always use HTTPS in production:**
```bash
# ✅ Secure
https://user-service.kinoshita-labs.com/api/v1/auth/login

# ❌ Insecure (development only)
http://localhost:3000/api/v1/auth/login
```

### Token Validation

The service performs comprehensive token validation:

1. **Signature verification** - Ensure token hasn't been tampered
2. **Expiration check** - Reject expired tokens
3. **Issuer validation** - Verify token issued by this service
4. **Session verification** - Check if session still exists in database/cache

### Session Management

**Session Invalidation Triggers:**
- User initiates logout
- Password change
- Role change
- Account suspension
- Admin-triggered logout
- Token expiration

**Example: Logout all sessions**

```typescript
// Logout user from all devices
async function logoutAllSessions(userId: string): Promise<void> {
  // Delete all refresh tokens for user
  await database.sessions.deleteMany({
    where: { userId }
  });
  
  // Clear all cached sessions
  const keys = await redis.keys(`session:${userId}:*`);
  if (keys.length > 0) {
    await redis.del(...keys);
  }
}
```

## API Authentication Examples

### JavaScript/TypeScript (Fetch API)

```typescript
// Login and store tokens
async function login(email: string, password: string) {
  const response = await fetch('https://user-service.kinoshita-labs.com/api/v1/auth/login', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({ email, password }),
  });
  
  const data = await response.json();
  
  // Store tokens securely
  sessionStorage.setItem('accessToken', data.accessToken);
  // Refresh token should be stored in HTTP-only cookie by server
  
  return data;
}

// Make authenticated request
async function getUser(userId: string) {
  const accessToken = sessionStorage.getItem('accessToken');
  
  const response = await fetch(
    `https://user-service.kinoshita-labs.com/api/v1/users/${userId}`,
    {
      headers: {
        'Authorization': `Bearer ${accessToken}`,
      },
    }
  );
  
  if (response.status === 401) {
    // Token expired, refresh it
    await refreshAccessToken();
    // Retry request
    return getUser(userId);
  }
  
  return response.json();
}

// Refresh access token
async function refreshAccessToken() {
  const response = await fetch(
    'https://user-service.kinoshita-labs.com/api/v1/auth/refresh',
    {
      method: 'POST',
      credentials: 'include', // Include HTTP-only cookie
    }
  );
  
  const data = await response.json();
  sessionStorage.setItem('accessToken', data.accessToken);
}
```

### Python (Requests)

```python
import requests
from datetime import datetime, timedelta

class UserServiceClient:
    def __init__(self, base_url):
        self.base_url = base_url
        self.access_token = None
        self.refresh_token = None
        self.token_expiry = None
    
    def login(self, email, password):
        response = requests.post(
            f"{self.base_url}/auth/login",
            json={"email": email, "password": password}
        )
        response.raise_for_status()
        
        data = response.json()
        self.access_token = data['accessToken']
        self.refresh_token = data['refreshToken']
        self.token_expiry = datetime.now() + timedelta(seconds=data['expiresIn'])
        
        return data
    
    def _get_headers(self):
        if datetime.now() >= self.token_expiry:
            self._refresh_token()
        
        return {
            'Authorization': f'Bearer {self.access_token}'
        }
    
    def _refresh_token(self):
        response = requests.post(
            f"{self.base_url}/auth/refresh",
            json={"refreshToken": self.refresh_token}
        )
        response.raise_for_status()
        
        data = response.json()
        self.access_token = data['accessToken']
        self.token_expiry = datetime.now() + timedelta(seconds=data['expiresIn'])
    
    def get_user(self, user_id):
        response = requests.get(
            f"{self.base_url}/users/{user_id}",
            headers=self._get_headers()
        )
        response.raise_for_status()
        return response.json()

# Usage
client = UserServiceClient('https://user-service.kinoshita-labs.com/api/v1')
client.login('john.doe@example.com', 'SecurePass123!')
user = client.get_user('550e8400-e29b-41d4-a716-446655440000')
```

### cURL

```bash
# Login
TOKEN_RESPONSE=$(curl -X POST https://user-service.kinoshita-labs.com/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"john.doe@example.com","password":"SecurePass123!"}')

# Extract access token
ACCESS_TOKEN=$(echo $TOKEN_RESPONSE | jq -r '.accessToken')

# Make authenticated request
curl -X GET https://user-service.kinoshita-labs.com/api/v1/users/me \
  -H "Authorization: Bearer $ACCESS_TOKEN"
```

## Troubleshooting

### Common Authentication Errors

| Error | Status | Cause | Solution |
|-------|--------|-------|----------|
| Token missing | 401 | No Authorization header | Include `Authorization: Bearer <token>` |
| Token expired | 401 | Access token expired | Refresh token or re-login |
| Token invalid | 401 | Malformed or tampered token | Re-login to get valid token |
| Session not found | 401 | Session deleted (logout/password change) | Re-login |
| Insufficient permissions | 403 | User lacks required permission | Contact admin for role assignment |
| Account suspended | 403 | User account suspended | Contact administrator |

### Debug Authentication Issues

**Check token validity:**

```bash
# Decode JWT (using jwt.io or jwt-cli)
jwt decode eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...

# Check expiration
date -d @1704627600  # Unix timestamp from 'exp' claim
```

**Verify session exists:**

```bash
# Check Redis cache
redis-cli GET "session:550e8400-e29b-41d4-a716-446655440000:token123"

# Check database
SELECT * FROM sessions WHERE user_id = '550e8400-e29b-41d4-a716-446655440000';
```

---

**Relevant Source Files:**
- [services/user-service/src/middleware/auth.ts](https://github.com/brunokino/backstage-entities-example)
- [services/user-service/src/services/auth.service.ts](https://github.com/brunokino/backstage-entities-example)
- [services/user-service/src/utils/jwt.ts](https://github.com/brunokino/backstage-entities-example)

**Last Updated**: 2025-01-07  
**Document Owner**: Platform Team

