# prisma — prisma

# Prisma Database Migrations

This module handles database schema migrations for the Papermark application using Prisma ORM. It provides both manual migration procedures and an automated shell script to synchronize the database schema with changes in `schema.prisma`.

## Overview

Prisma migrations allow you to evolve your database schema over time in a controlled, versioned manner. When you modify the `schema.prisma` file, you need to generate and apply a migration to update your database structure without losing data or creating drift between your schema definition and actual database state.

## Key Components

### `schema.prisma`

The Prisma schema file defines your database models, relations, and configuration. Changes to this file trigger the need for migrations.

### `migrations/` Directory

Contains timestamped migration folders (e.g., `20240408000000_add_model/`) each with a `migration.sql` file that captures the SQL changes to apply.

### `add-migration.sh`

An automated script that creates a new migration folder, generates the migration SQL diff, and applies the migration in one command.

## Usage

### Automated Migration (Recommended)

Use the `add-migration.sh` script for a streamlined workflow:

```bash
./add-migration.sh --name <migration_name> --user <database_user>
```

**Parameters:**

| Parameter | Required | Description |
|-----------|----------|-------------|
| `--name` | Yes | Descriptive name for the migration (e.g., `add_user_table`) |
| `--user` | Yes | PostgreSQL database user for the shadow database connection |

**Example:**

```bash
./add-migration.sh --name add_user_preferences --user papermark_user
```

The script:

1. Generates a timestamped migration name (format: `YYYYMMDD000000_<name>`)
2. Creates the migration directory under `prisma/migrations/`
3. Runs `prisma migrate diff` to generate the SQL changes
4. Executes `prisma migrate resolve --applied` to mark the migration as applied

### Manual Migration Process

For manual control, follow these steps:

1. **Create the migration folder:**

   ```bash
   mkdir -p prisma/migrations/YYYYMMDD000000_<migration_name>
   ```

2. **Generate the migration SQL:**

   ```bash
   prisma migrate diff \
       --from-migrations prisma/migrations \
       --to-schema-datamodel prisma/schema.prisma \
       --shadow-database-url "postgresql://<USER>@localhost:5432/papermark-shadow-db" \
       --script > prisma/migrations/YYYYMMDD000000_<migration_name>/migration.sql
   ```

3. **Apply the migration:**

   ```bash
   prisma migrate resolve --applied YYYYMMDD000000_<migration_name>
   ```

## Prerequisites

### 1. Global Prisma Installation

The migration workflow requires Prisma to be installed globally, not via `npx`:

```bash
npm install -g prisma
```

Using `npx prisma` will not work correctly for these migration operations.

### 2. Shadow Database

A shadow database is required for Prisma Migrate to detect schema differences. This must be a separate database instance to prevent accidental data loss during migration generation.

**Connection format:**

```
postgresql://<USER>@localhost:5432/papermark-shadow-db
```

Ensure the shadow database is accessible and separate from your primary development database.

## Understanding Migration Drift

When Prisma reports "Drift detected," it means your database schema has diverged from the migration history. This can occur when:

- Manual database changes were made outside of Prisma migrations
- A previous migration failed mid-execution
- The migration history table became corrupted

The `add-migration.sh` script uses `prisma migrate diff` with the `--from-migrations` flag, which compares against your recorded migration history rather than assuming the database is in sync.

## Error Handling

The script validates required parameters and exits with descriptive error messages:

- Missing `--name`: Prompts with usage instructions
- Missing `--user`: Prompts with usage instructions
- Directory creation failure: Reports failure and exits
- Migration generation failure: Reports failure and exits
- Migration application failure: Reports failure and exits

## Relationship to Schema Changes

This module is the bridge between code changes in `schema.prisma` and database schema updates. Any time you add, modify, or remove models, fields, relations, or enums in the schema file, you must run a migration to apply those changes to the actual database.