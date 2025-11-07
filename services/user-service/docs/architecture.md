# System Architecture

## Overview

The User Service follows a **layered architecture** pattern with clear separation of concerns, designed for scalability, maintainability, and testability. The service is built as a cloud-native microservice with stateless application instances backed by persistent storage.

## Architectural Principles

1. **Separation of Concerns** - Clear boundaries between layers (API, business logic, data access)
2. **Stateless Application** - All state stored in PostgreSQL and Redis for horizontal scalability
3. **API-First Design** - OpenAPI specification drives implementation and documentation
4. **Security by Default** - Authentication and authorization enforced at multiple layers
5. **Fail-Fast Validation** - Input validation at API boundary before processing
6. **Idempotency** - Critical operations designed to be safely retried
7. **Observable System** - Comprehensive logging, metrics, and tracing

## Layered Architecture

```mermaid
graph TB
    subgraph "API Layer"
        Routes[Express Routes]
        Middleware[Middleware Stack]
        Validation[Request Validation]
    end
    
    subgraph "Service Layer"
        AuthService[Authentication Service]
        UserService[User Service]
        RoleService[Role Service]
        AdminService[Admin Service]
    end
    
    subgraph "Data Access Layer"
        UserRepo[User Repository]
        RoleRepo[Role Repository]
        SessionRepo[Session Repository]
        Prisma[Prisma ORM]
    end
    
    subgraph "Infrastructure Layer"
        PG[(PostgreSQL)]
        Redis[(Redis)]
        Email[Email Client]
        Audit[Audit Client]
    end
    
    Routes --> Middleware
    Middleware --> Validation
    Validation --> AuthService
    Validation --> UserService
    Validation --> RoleService
    Validation --> AdminService
    
    AuthService --> UserRepo
    AuthService --> SessionRepo
    UserService --> UserRepo
    RoleService --> RoleRepo
    AdminService --> UserRepo
    
    UserRepo --> Prisma
    RoleRepo --> Prisma
    SessionRepo --> Prisma
    
    Prisma --> PG
    SessionRepo --> Redis
    AuthService --> Email
    AdminService --> Audit
    
    style Routes fill:#4A90E2
    style Middleware fill:#4A90E2
    style Validation fill:#4A90E2
    style PG fill:#336791
    style Redis fill:#DC382D
```

## Component Architecture

### 1. API Layer

The API layer handles HTTP requests, routing, and response formatting.

**Components:**
- **Express Routes** - RESTful endpoint definitions
- **Middleware Stack** - Request processing pipeline
- **Request Validation** - Schema-based validation using Joi/Zod
- **Error Handler** - Centralized error handling and formatting
- **Response Formatter** - Consistent API response structure

**Middleware Pipeline:**

```mermaid
sequenceDiagram
    participant Client
    participant CORS
    participant Helmet
    participant RateLimit
    participant Auth
    participant Validate
    participant Handler
    participant ErrorHandler
    
    Client->>CORS: HTTP Request
    CORS->>Helmet: Set security headers
    Helmet->>RateLimit: Check rate limits
    
    alt Rate limit exceeded
        RateLimit-->>Client: 429 Too Many Requests
    else Within limit
        RateLimit->>Auth: Verify JWT token
        
        alt Token invalid
            Auth-->>Client: 401 Unauthorized
        else Token valid
            Auth->>Validate: Validate request body
            
            alt Validation failed
                Validate-->>Client: 400 Bad Request
            else Validation passed
                Validate->>Handler: Process request
                
                alt Business logic error
                    Handler->>ErrorHandler: Handle error
                    ErrorHandler-->>Client: Error response
                else Success
                    Handler-->>Client: Success response
                end
            end
        end
    end
```

### 2. Service Layer

The service layer contains business logic and orchestrates data access operations.

**Services:**

```mermaid
classDiagram
    class AuthenticationService {
        +register(userData) User
        +login(credentials) AuthTokens
        +refreshToken(refreshToken) AccessToken
        +logout(userId) void
        +verifyToken(token) TokenPayload
        -hashPassword(password) string
        -generateTokens(user) AuthTokens
    }
    
    class UserService {
        +getUserById(userId) User
        +updateUser(userId, data) User
        +deleteUser(userId) void
        +listUsers(filters) UserList
        +changePassword(userId, passwords) void
        -validatePassword(password) boolean
    }
    
    class RoleService {
        +listRoles() Role[]
        +getUserRoles(userId) Role[]
        +assignRole(userId, roleId) void
        +revokeRole(userId, roleId) void
        -checkPermission(userId, permission) boolean
    }
    
    class AdminService {
        +suspendUser(userId, reason) User
        +activateUser(userId) User
        +auditUserActions(userId) AuditLog[]
        -notifyAuditService(event) void
    }
    
    AuthenticationService --> UserService
    AdminService --> UserService
    AdminService --> RoleService
```

### 3. Data Access Layer

The data access layer abstracts database operations using the Repository pattern.

**Repository Structure:**

```mermaid
classDiagram
    class BaseRepository~T~ {
        <<abstract>>
        +findById(id) T
        +findAll(filters) T[]
        +create(data) T
        +update(id, data) T
        +delete(id) void
    }
    
    class UserRepository {
        +findByEmail(email) User
        +findWithRoles(userId) User
        +searchUsers(query) User[]
        +updateLastLogin(userId) void
    }
    
    class RoleRepository {
        +findByName(name) Role
        +findUserRoles(userId) Role[]
        +assignRoleToUser(userId, roleId) void
        +revokeRoleFromUser(userId, roleId) void
    }
    
    class SessionRepository {
        +createSession(userId, refreshToken) Session
        +getSession(refreshToken) Session
        +deleteSession(refreshToken) void
        +deleteUserSessions(userId) void
    }
    
    BaseRepository <|-- UserRepository
    BaseRepository <|-- RoleRepository
    BaseRepository <|-- SessionRepository
```

## Data Flow Architecture

### Registration Flow

```mermaid
sequenceDiagram
    participant Client
    participant API
    participant AuthService
    participant UserRepo
    participant DB
    participant Email
    participant Audit
    
    Client->>API: POST /auth/register
    API->>API: Validate request
    API->>AuthService: register(userData)
    
    AuthService->>UserRepo: findByEmail(email)
    UserRepo->>DB: SELECT * FROM users WHERE email = ?
    DB-->>UserRepo: null (user not found)
    UserRepo-->>AuthService: null
    
    AuthService->>AuthService: hashPassword(password)
    AuthService->>UserRepo: create(user)
    UserRepo->>DB: INSERT INTO users ...
    DB-->>UserRepo: User created
    UserRepo-->>AuthService: User object
    
    AuthService->>AuthService: generateTokens(user)
    AuthService->>UserRepo: createSession(userId, refreshToken)
    UserRepo->>DB: Store session
    
    par Async operations
        AuthService->>Email: sendWelcomeEmail(user)
        AuthService->>Audit: logRegistration(user)
    end
    
    AuthService-->>API: AuthResponse
    API-->>Client: 201 Created + tokens
```

### Authentication Flow

```mermaid
sequenceDiagram
    participant Client
    participant API
    participant AuthService
    participant UserRepo
    participant DB
    participant Redis
    
    Client->>API: POST /auth/login
    API->>API: Validate credentials
    API->>AuthService: login(email, password)
    
    AuthService->>UserRepo: findByEmail(email)
    UserRepo->>DB: SELECT * FROM users WHERE email = ?
    DB-->>UserRepo: User object
    UserRepo-->>AuthService: User object
    
    AuthService->>AuthService: comparePassword(inputPassword, hashedPassword)
    
    alt Password valid
        AuthService->>AuthService: generateTokens(user)
        
        AuthService->>UserRepo: createSession(userId, refreshToken)
        UserRepo->>DB: INSERT INTO sessions
        
        AuthService->>Redis: SET session:userId:token with TTL
        Redis-->>AuthService: OK
        
        AuthService->>UserRepo: updateLastLogin(userId)
        UserRepo->>DB: UPDATE users SET last_login = NOW()
        
        AuthService-->>API: AuthResponse with tokens
        API-->>Client: 200 OK + tokens
    else Password invalid
        AuthService-->>API: Invalid credentials error
        API-->>Client: 401 Unauthorized
    end
```

### Authorization Flow

```mermaid
sequenceDiagram
    participant Client
    participant Middleware
    participant AuthService
    participant Redis
    participant RoleService
    participant DB
    participant Handler
    
    Client->>Middleware: GET /admin/users (with JWT)
    Middleware->>AuthService: verifyToken(jwt)
    
    AuthService->>AuthService: Decode and validate JWT
    
    alt Token valid
        AuthService->>Redis: GET session:userId:token
        
        alt Session exists in cache
            Redis-->>AuthService: Session data
        else Session not in cache
            AuthService->>DB: SELECT * FROM sessions WHERE token = ?
            DB-->>AuthService: Session data
            AuthService->>Redis: Cache session
        end
        
        AuthService-->>Middleware: TokenPayload(userId, roles)
        
        Middleware->>RoleService: checkPermission(userId, 'admin:users:read')
        RoleService->>DB: Check user roles and permissions
        DB-->>RoleService: Permissions
        
        alt Has permission
            RoleService-->>Middleware: Authorized
            Middleware->>Handler: Execute handler
            Handler-->>Client: 200 OK + data
        else No permission
            RoleService-->>Middleware: Forbidden
            Middleware-->>Client: 403 Forbidden
        end
    else Token invalid
        AuthService-->>Middleware: Invalid token
        Middleware-->>Client: 401 Unauthorized
    end
```

## Database Schema

### Entity Relationship Diagram

```mermaid
erDiagram
    USERS ||--o{ USER_ROLES : has
    ROLES ||--o{ USER_ROLES : assigned_to
    ROLES ||--o{ ROLE_PERMISSIONS : has
    PERMISSIONS ||--o{ ROLE_PERMISSIONS : granted_via
    USERS ||--o{ SESSIONS : owns
    USERS ||--o{ AUDIT_LOGS : generates
    
    USERS {
        uuid id PK
        string email UK
        string password_hash
        string first_name
        string last_name
        string phone_number
        enum status
        timestamp created_at
        timestamp updated_at
        timestamp last_login_at
    }
    
    ROLES {
        uuid id PK
        string name UK
        string description
        timestamp created_at
    }
    
    USER_ROLES {
        uuid user_id FK
        uuid role_id FK
        timestamp assigned_at
    }
    
    PERMISSIONS {
        uuid id PK
        string name UK
        string description
        string resource
        string action
    }
    
    ROLE_PERMISSIONS {
        uuid role_id FK
        uuid permission_id FK
        timestamp granted_at
    }
    
    SESSIONS {
        uuid id PK
        uuid user_id FK
        string refresh_token UK
        string ip_address
        string user_agent
        timestamp expires_at
        timestamp created_at
    }
    
    AUDIT_LOGS {
        uuid id PK
        uuid user_id FK
        string action
        string resource
        jsonb metadata
        string ip_address
        timestamp created_at
    }
```

### Indexing Strategy

| Table | Index | Columns | Type | Purpose |
|-------|-------|---------|------|---------|
| users | idx_users_email | email | UNIQUE | Fast email lookup for authentication |
| users | idx_users_status | status | BTREE | Filter users by status |
| users | idx_users_created_at | created_at | BTREE | Time-based queries |
| sessions | idx_sessions_user_id | user_id | BTREE | Find all user sessions |
| sessions | idx_sessions_expires_at | expires_at | BTREE | Cleanup expired sessions |
| sessions | idx_sessions_refresh_token | refresh_token | UNIQUE | Fast token lookup |
| user_roles | idx_user_roles_user_id | user_id | BTREE | Get user's roles |
| user_roles | idx_user_roles_role_id | role_id | BTREE | Get role's users |
| audit_logs | idx_audit_user_id | user_id | BTREE | User activity history |
| audit_logs | idx_audit_created_at | created_at | BTREE | Time-based audit queries |

## Caching Strategy

### Redis Cache Architecture

```mermaid
graph LR
    subgraph "Cache Layers"
        L1[Session Cache]
        L2[User Profile Cache]
        L3[Role/Permission Cache]
        L4[Rate Limit Counters]
    end
    
    subgraph "TTL Configuration"
        T1[Sessions: 15 days]
        T2[Profiles: 1 hour]
        T3[Roles: 6 hours]
        T4[Rate Limits: 15 minutes]
    end
    
    L1 --- T1
    L2 --- T2
    L3 --- T3
    L4 --- T4
```

**Cache Keys:**
- `session:{userId}:{tokenId}` - User session with refresh token
- `user:profile:{userId}` - User profile data
- `user:roles:{userId}` - User's roles and permissions
- `ratelimit:{identifier}:{endpoint}` - Rate limit counters

**Cache Invalidation:**
- **Session cache**: Invalidated on logout or password change
- **User profile**: Invalidated on profile update
- **Roles cache**: Invalidated on role assignment/revocation
- **Rate limit**: Expires automatically based on TTL

## Scalability Considerations

### Horizontal Scaling

The User Service is designed to scale horizontally:

```mermaid
graph TB
    subgraph "Load Balancer"
        LB[Nginx / ALB]
    end
    
    subgraph "Auto Scaling Group"
        Pod1[Instance 1]
        Pod2[Instance 2]
        Pod3[Instance 3]
        PodN[Instance N]
    end
    
    subgraph "Shared State"
        PG[(PostgreSQL Primary)]
        PGR[(PostgreSQL Replica)]
        Redis[(Redis Cluster)]
    end
    
    LB --> Pod1
    LB --> Pod2
    LB --> Pod3
    LB --> PodN
    
    Pod1 --> PG
    Pod2 --> PG
    Pod3 --> PGR
    PodN --> PGR
    
    Pod1 --> Redis
    Pod2 --> Redis
    Pod3 --> Redis
    PodN --> Redis
```

**Scaling Triggers:**
- CPU utilization > 70%
- Memory utilization > 80%
- Request rate > 1000 req/s per instance
- Response time p95 > 300ms

### Performance Optimizations

1. **Connection Pooling** - PostgreSQL connection pool (min: 10, max: 30)
2. **Redis Pipelining** - Batch multiple Redis operations
3. **Database Read Replicas** - Read-heavy operations use replicas
4. **Prepared Statements** - Prisma uses prepared statements
5. **Response Compression** - Gzip compression for responses > 1KB
6. **Query Optimization** - Indexed queries and efficient JOINs

## Security Architecture

### Defense in Depth

```mermaid
graph TB
    subgraph "Perimeter Security"
        WAF[Web Application Firewall]
        DDoS[DDoS Protection]
    end
    
    subgraph "Transport Security"
        TLS[TLS 1.3]
        Cert[Certificate Management]
    end
    
    subgraph "Application Security"
        Auth[JWT Authentication]
        RBAC[Role-Based Access Control]
        RateLimit[Rate Limiting]
        Validation[Input Validation]
    end
    
    subgraph "Data Security"
        Encryption[Encryption at Rest]
        Hashing[Password Hashing]
        Sanitization[SQL Injection Prevention]
    end
    
    WAF --> TLS
    DDoS --> TLS
    TLS --> Auth
    Cert --> TLS
    Auth --> RBAC
    RBAC --> RateLimit
    RateLimit --> Validation
    Validation --> Encryption
    Encryption --> Hashing
    Hashing --> Sanitization
```

### Security Layers

1. **Network Layer** - VPC isolation, security groups, firewall rules
2. **Transport Layer** - TLS 1.3, strong cipher suites
3. **Application Layer** - JWT authentication, RBAC, rate limiting
4. **Data Layer** - Encryption at rest, password hashing (bcrypt)
5. **Audit Layer** - Comprehensive audit logging

## Failure Scenarios & Recovery

### Circuit Breaker Pattern

```mermaid
stateDiagram-v2
    [*] --> Closed
    Closed --> Open: Failure threshold exceeded
    Open --> HalfOpen: After timeout period
    HalfOpen --> Closed: Success
    HalfOpen --> Open: Failure
    
    note right of Closed
        Normal operation
        All requests pass through
    end note
    
    note right of Open
        Fast fail
        Return cached response or error
    end note
    
    note right of HalfOpen
        Test with limited requests
        Evaluate service recovery
    end note
```

### Graceful Degradation

| Dependency | Failure Mode | Degradation Strategy |
|------------|--------------|----------------------|
| PostgreSQL Primary | Connection timeout | Use read replica for read operations |
| Redis Cache | Connection failure | Bypass cache, direct database queries |
| Email Service | API timeout | Queue emails for retry, continue request |
| Audit Service | Unavailable | Log locally, sync when available |

---

**Relevant Source Files:**
- [services/user-service/src/middleware/](https://github.com/brunokino/backstage-entities-example)
- [services/user-service/src/services/](https://github.com/brunokino/backstage-entities-example)
- [services/user-service/src/repositories/](https://github.com/brunokino/backstage-entities-example)
- [services/user-service/prisma/schema.prisma](https://github.com/brunokino/backstage-entities-example)

**Last Updated**: 2025-01-07  
**Document Owner**: Platform Team

