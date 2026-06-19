# prisma

# Prisma Database Migrations

This module group manages database schema evolution for Papermark using Prisma ORM with PostgreSQL.

## How the Sub-Modules Fit Together

The prisma module operates as a two-layer system:

1. **[Schema Definition](prisma-schema.md)** — Defines *what* the database should look like across multiple domain-specific files (`schema.prisma`, `team.prisma`, `document.prisma`, `link.prisma`, `dataroom.prisma`, `conversation.prisma`)
2. **[Migrations](prisma-migrations.md)** — Contains the SQL scripts that *apply* those changes to the live database

```mermaid
flow LR
    A[schema.prisma<br/>domain files] --> B[Migration Files]
    B --> C[(PostgreSQL<br/>Database)]
```

When you modify any schema file, Prisma generates a new migration. That migration becomes part of the versioned history in the `migrations` folder, ensuring every schema change is tracked and can be reproduced.

## Key Workflow

**Development cycle:**

1. Modify a schema file (e.g., add a field to `document.prisma`)
2. Run `prisma migrate dev` — Prisma compares your schema against the current database state
3. Prisma generates a new timestamped migration file with the necessary SQL
4. Apply migrations in staging/production with `prisma migrate deploy`

## Core Entities

The schema defines the complete data model for Papermark's document sharing platform:

- **Users & Teams** — User accounts, OAuth accounts, team membership with roles
- **Documents** — Document storage, versioning, and storage configuration
- **Views & Viewers** — Tracking for document views with authentication options
- **Links** — Sharing links with presets, password protection, and expiration
- **Datarooms** — Virtual data rooms with folders, branding, and request lists
- **Conversations** — Chat threads attached to documents or datarooms
- **Workflows** — Task and request management with automation triggers

## Related Modules

- [Schema Definition](prisma-schema.md) — Domain-organized schema files
- [Migrations](prisma-migrations.md) — Versioned migration history (75+ migrations since 2023)