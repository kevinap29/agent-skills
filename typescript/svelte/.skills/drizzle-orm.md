# Drizzle ORM Skill (TypeScript)

## Description

Expert skill for using Drizzle ORM in SvelteKit and TypeScript projects. Covers schema definition, migrations with `drizzle-kit`, type-safe queries, joins, and integration with SvelteKit server logic. Use this skill when the user needs to perform database operations, define table schemas, or handle migrations in a TypeScript-first manner.

## Installation

```bash
npm i drizzle-orm @libsql/client # For SQLite/LibSQL
npm i -D drizzle-kit
```

## Schema Definition (`src/lib/server/db/schema.ts`)

```ts
import { sqliteTable, text, integer } from 'drizzle-orm/sqlite-core';

export const users = sqliteTable('users', {
  id: integer('id').primaryKey({ autoIncrement: true }),
  name: text('name').notNull(),
  email: text('email').notNull().unique(),
  createdAt: integer('created_at', { mode: 'timestamp' }).default(new Date())
});

export const posts = sqliteTable('posts', {
  id: integer('id').primaryKey({ autoIncrement: true }),
  title: text('title').notNull(),
  content: text('content').notNull(),
  authorId: integer('author_id').references(() => users.id)
});
```

## Database Connection (`src/lib/server/db/index.ts`)

```ts
import { drizzle } from 'drizzle-orm/libsql';
import { createClient } from '@libsql/client';
import { env } from '$env/dynamic/private';
import * as schema from './schema';

const client = createClient({ url: env.DATABASE_URL });
export const db = drizzle(client, { schema });
```

## Queries

### Insert

```ts
await db.insert(users).values({
  name: 'John Doe',
  email: 'john@example.com'
});
```

### Select with Relations

```ts
const allPosts = await db.query.posts.findMany({
  with: {
    author: true
  },
  where: (posts, { eq }) => eq(posts.id, 1)
});
```

### Update & Delete

```ts
await db.update(users)
  .set({ name: 'Updated Name' })
  .where(eq(users.id, 1));

await db.delete(users).where(eq(users.id, 1));
```

## Migrations (`drizzle.config.ts`)

```ts
import { defineConfig } from 'drizzle-kit';

export default defineConfig({
  schema: './src/lib/server/db/schema.ts',
  out: './drizzle',
  dialect: 'sqlite', // or 'postgresql', 'mysql'
  dbCredentials: {
    url: process.env.DATABASE_URL!
  }
});
```

```bash
# Generate migration
npx drizzle-kit generate

# Push to DB (for dev)
npx drizzle-kit push
```

## Drizzle Tips

- **Relational Queries**: Use the `db.query` syntax for easy fetching of related data without manual joins.
- **Zod Integration**: Use `drizzle-zod` to automatically generate Zod schemas from your Drizzle tables.
- **Type Inference**: Use `typeof users.$inferSelect` and `typeof users.$inferInsert` to get TypeScript types for your entities.
- **Transactions**: Use `await db.transaction(async (tx) => { ... })` for atomic operations.
- **SQL Template Tags**: Use the `sql` tag for raw queries when needed, while still maintaining type safety for parameters.
