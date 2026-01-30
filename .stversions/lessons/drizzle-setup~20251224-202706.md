# Drizzle ORM Setup for StuFlux

## Overview
This guide explains how to set up Drizzle ORM with PostgreSQL in a Next.js + Express monorepo.

## 1. Install Dependencies

```bash
# Backend (apps/api)
cd apps/api
pnpm add drizzle-orm pg
pnpm add -D drizzle-kit @types/pg

# Frontend (apps/web)
cd ../web
pnpm add drizzle-orm pg
pnpm add -D drizzle-kit @types/pg
```

## 2. Configure Drizzle

Create `apps/api/drizzle.config.ts`:
```typescript
import type { Config } from 'drizzle-kit';

export default {
  schema: './src/db/schema.ts',
  out: './src/db/migrations',
  driver: 'pg',
  dbCredentials: {
    connectionString: process.env.DATABASE_URL!,
  },
  verbose: true,
  strict: true,
} satisfies Config;
```

## 3. Define Schema

Create `apps/api/src/db/schema.ts`:
```typescript
import { pgTable, uuid, text, integer, boolean, jsonb, timestamp, serial } from 'drizzle-orm/pg-core';

export const users = pgTable('users', {
  id: uuid('id').defaultRandom().primaryKey(),
  clerk_id: text('clerk_id').unique().notNull(),
  email: text('email'),
  name: text('name'),
  avatar_url: text('avatar_url'),
  city: text('city'),
  phone: text('phone'),
  created_at: timestamp('created_at', { withTimezone: true }).defaultNow(),
  updated_at: timestamp('updated_at', { withTimezone: true }).defaultNow()
});

export const categories = pgTable('categories', {
  id: serial('id').primaryKey(),
  name: text('name').notNull(),
  slug: text('slug').unique().notNull(),
  description: text('description'),
  icon: text('icon'),
  created_at: timestamp('created_at', { withTimezone: true }).defaultNow()
});

export const listings = pgTable('listings', {
  id: uuid('id').defaultRandom().primaryKey(),
  owner_id: uuid('owner_id').notNull().references(() => users.id, { onDelete: 'cascade' }),
  title: text('title').notNull(),
  description: text('description').notNull(),
  category_id: integer('category_id').notNull().references(() => categories.id),
  daily_rate: integer('daily_rate').notNull(),
  city: text('city').notNull(),
  address: text('address'),
  latitude: text('latitude'),
  longitude: text('longitude'),
  status: text('status', { enum: ['draft', 'active', 'inactive', 'archived'] }).default('draft'),
  specs: jsonb('specs').default({}),
  min_rental_days: integer('min_rental_days').default(1),
  max_rental_days: integer('max_rental_days').default(30),
  delivery_available: boolean('delivery_available').default(false),
  delivery_fee: integer('delivery_fee').default(0),
  security_deposit: integer('security_deposit').default(0),
  booking_count: integer('booking_count').default(0),
  view_count: integer('view_count').default(0),
  created_at: timestamp('created_at', { withTimezone: true }).defaultNow(),
  updated_at: timestamp('updated_at', { withTimezone: true }).defaultNow()
});
```

## 4. Setup Database Clients

### Backend (apps/api/src/db/index.ts)
```typescript
import { drizzle } from 'drizzle-orm/node-postgres';
import { Pool } from 'pg';
import * as schema from './schema';

const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
  ssl: process.env.NODE_ENV === 'production' ? { rejectUnauthorized: false } : false,
});

export const db = drizzle(pool, { schema });

export async function testConnection() {
  try {
    await db.execute(sql`SELECT NOW()`);
    console.log('✅ Database connected successfully');
    return true;
  } catch (error: any) {
    console.error('❌ Database connection failed:', error?.message || error);
    return false;
  }
}
```

### Frontend (apps/web/src/lib/db.ts)
```typescript
import { drizzle } from 'drizzle-orm/node-postgres';
import { Pool } from 'pg';
import * as schema from '../../../api/src/db/schema';

let db: ReturnType<typeof drizzle>;

export function getDb() {
  if (!db) {
    const pool = new Pool({
      connectionString: process.env.DATABASE_URL,
      ssl: process.env.NODE_ENV === 'production' ? { rejectUnauthorized: false } : false,
    });
    db = drizzle(pool, { schema });
  }
  return db;
}
```

## 5. Migrations

Add to `apps/api/package.json`:
```json
{
  "scripts": {
    "db:generate": "drizzle-kit generate:pg",
    "db:push": "drizzle-kit push:pg",
    "db:studio": "drizzle-kit studio",
    "db:seed": "tsx src/db/seed.ts",
    "db:migrate": "tsx src/db/migrate.ts"
  }
}
```

Generate and run migrations:
```bash
pnpm db:generate  # Generate migration files
pnpm db:push     # Push changes to database
pnpm db:studio   # Open Drizzle Studio UI (optional)
```

## 6. Usage Examples

### In Express API Routes:
```typescript
import { db } from '../db';
import { listings } from '../db/schema';
import { eq } from 'drizzle-orm';

router.get('/listings', async (req, res) => {
  const allListings = await db
    .select()
    .from(listings)
    .where(eq(listings.status, 'active'));
  res.json(allListings);
});
```

### In Next.js Server Components:
```typescript
import { getDb } from '@/lib/db';
import { listings } from '../../../api/src/db/schema';
import { eq } from 'drizzle-orm';

export default async function ListingsPage() {
  const db = getDb();
  const allListings = await db
    .select()
    .from(listings)
    .where(eq(listings.status, 'active'));
  
  return (
    <div>
      {allListings.map(listing => (
        <div key={listing.id}>{listing.title}</div>
      ))}
    </div>
  );
}
```

## Benefits of Using Drizzle
1. Type-safe queries and schema
2. No need for manual SQL writing
3. Automatic migration generation
4. Visual database explorer (drizzle-kit studio)
5. Great TypeScript integration
6. Lightweight and fast

## Common Operations

### Insert:
```typescript
const newUser = await db.insert(users).values({
  clerk_id: 'user_123',
  email: 'user@example.com',
}).returning();
```

### Select with Relations:
```typescript
const listingsWithOwners = await db
  .select()
  .from(listings)
  .leftJoin(users, eq(listings.owner_id, users.id));
```

### Update:
```typescript
await db
  .update(listings)
  .set({ status: 'active' })
  .where(eq(listings.id, listingId));
```

### Delete:
```typescript
await db
  .delete(listings)
  .where(eq(listings.id, listingId));
```
