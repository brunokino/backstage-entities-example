# Backstage Entities Example

This repository contains example Backstage entities for demonstration purposes, showcasing best practices for documenting microservices, APIs, and organizational structure.

## Repository Structure

```
backstage-entities-example/
├── catalog-info.yaml           # Root location entity
├── org/                        # Organizational entities
│   ├── groups/                # Team/group definitions
│   │   └── platform-team.yaml
│   └── users/                 # User definitions
│       ├── alice-martinez.yaml
│       ├── bob-chen.yaml
│       └── carol-johnson.yaml
└── services/                   # Service components
    └── user-service/          # User management service
        ├── catalog-info.yaml  # Service component definition
        ├── openapi.yaml       # OpenAPI specification
        ├── mkdocs.yml         # TechDocs configuration
        └── docs/              # Technical documentation
            ├── index.md
            ├── architecture.md
            ├── api-reference.md
            ├── authentication.md
            ├── local-development.md
            └── deployment.md
```

## Getting Started

### Prerequisites

- A running Backstage instance
- Access to register entities via Backstage UI or API

### Registering Entities

1. **Register the root location:**
   ```
   https://github.com/brunokino/backstage-entities-example/blob/main/catalog-info.yaml
   ```

2. Backstage will automatically discover and register all entities in this repository.

### Viewing TechDocs

The user-service includes comprehensive TechDocs documentation. Once registered, navigate to the service in Backstage and click on the "Docs" tab to view the technical documentation.

## Example Entities

### Organization

- **Group:** `platform-team` - The engineering team responsible for platform services
- **Users:** Alice Martinez (Team Lead), Bob Chen (Senior Engineer), Carol Johnson (DevOps Engineer)

### Services

- **user-service** - RESTful API for user management with authentication, authorization, and user lifecycle operations

## Company Context

All entities in this repository are part of **Kinoshita Labs**, a fictional technology company used for demonstration purposes.

## Documentation Standards

All documentation follows these principles:

- ✅ Production-quality content suitable for real-world use
- ✅ Comprehensive technical details with architecture diagrams
- ✅ Real API specifications with proper schemas
- ✅ Security and authentication patterns
- ✅ Local development and deployment guides
- ✅ Mermaid diagrams for visual understanding

## Contributing

This is a demonstration repository. Feel free to fork and adapt these examples for your own organization.

## License

MIT License - Free to use and modify for your own purposes.

