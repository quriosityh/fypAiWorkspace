# 📅 Day 5 & 6 Implementation Guide - Database Setup & First Feature (Listings Browse)

## 🎯 Overview
This guide covers Days 5-6 of Sprint 1, focusing on **PostgreSQL + Drizzle ORM setup**, **database schema implementation**, and **first real feature: Browse Listings**. This builds on your monorepo, Express API, and Clerk auth from Days 1-4.

## 📋 Prerequisites from Days 1-4
- ✅ Monorepo structure with `apps/web`, `apps/api`, `packages/types`
- ✅ Express API running on port 4000
- ✅ Next.js app running on port 3000
- ✅ Clerk authentication integrated in both Next.js and Express
- ✅ TypeScript configuration across workspace

---

## 📅 DAY 5 - DATABASE SETUP (8 hours)

### **GOAL:** Get PostgreSQL running with Drizzle ORM and create your first tables

---

### **Morning Session (4 hours) - Database Foundation**

#### **Task 1: Neon PostgreSQL Setup (45 mins)**

**What you're doing:**
Setting up a cloud PostgreSQL database that both Next.js and Express can connect to.

**Steps:**
1. Go to [neon.tech](https://neon.tech) and sign up (free tier)
2. Create a new project called "stuflux-dev"
3. Copy the connection string (looks like: `postgresql://user:pass@host/db?sslmode=require`)
4. Add to `.env` files in BOTH `apps/web` and `apps/api`:

```bash
# apps/api/.env
DATABASE_URL="postgresql://user:pass@host/db?sslmode=require"
NODE_ENV="development"
PORT=4000

# apps/web/.env.local
DATABASE_URL="postgresql://user:pass@host/db?sslmode=require"
NEXT_PUBLIC_API_URL="http://localhost:4000"
```

**Why both apps need DATABASE_URL:**
- **Express**: Handles complex writes (bookings, messages)
- **Next.js**: Handles simple reads (browsing listings) for speed

---

#### **Task 2: Install Drizzle ORM (30 mins)**

**What you're doing:**
Installing the database toolkit that will replace writing raw SQL.

**Commands:**
```bash
# From workspace root
cd apps/api
pnpm add drizzle-orm postgres
pnpm add -D drizzle-kit tsx

cd ../web
pnpm add drizzle-orm postgres
pnpm add -D drizzle-kit
```

**What each package does:**
- `drizzle-orm`: Type-safe database queries (like Prisma but lighter)
- `postgres`: PostgreSQL client for Node.js
- `drizzle-kit`: CLI tool for migrations
- `tsx`: TypeScript runner for migration scripts

---

#### **Task 3: Drizzle Configuration Files (45 mins)**

**What you're doing:**
Creating configuration so Drizzle knows where your schema is and how to connect.

**File 1: `apps/api/drizzle.config.ts`**
```typescript
import type { Config } from 'drizzle-kit';
import dotenv from 'dotenv';

dotenv.config();

export default {
  schema: './src/db/schema.ts',
  out: './migrations',
  dialect: 'postgresql',
  dbCredentials: {
    url: process.env.DATABASE_URL!,
  },
} satisfies Config;
```

**What this does:**
- `schema`: Where your table definitions live
- `out`: Where migration SQL files are generated
- `dbCredentials`: How to connect to database

**File 2: `apps/api/src/db/schema.ts` (CREATE THIS)**
```typescript
// This file will define your database tables
// You'll fill this in Task 4
```

**File 3: `apps/api/src/db/index.ts` (CREATE THIS)**
```typescript
// This file creates the database connection
// You'll fill this in Task 5
```

---

#### **Task 4: Create Database Schema - Part 1: Users & Categories (90 mins)**

**What you're doing:**
Defining the structure of your database tables in TypeScript (type-safe!).

**File: `apps/api/src/db/schema.ts`**

**IMPORTANT CONCEPTS:**

1. **Primary Keys:** Unique identifier for each row (we use `serial` = auto-incrementing number)
2. **Foreign Keys:** Links between tables (e.g., listing belongs to a user)
3. **Enums:** Predefined values (like listing status: draft/published/rented)
4. **Timestamps:** Track when records are created/updated
5. **RLS (Row-Level Security):** Neon feature to protect data (we'll use later)

**Table 1: Users**
```typescript
import { pgTable, serial, varchar, text, timestamp, boolean } from 'drizzle-orm/pg-core';

// USERS TABLE
// Purpose: Store user profiles (synced from Clerk)
// Why: Clerk handles auth, but we need user data for listings/bookings
export const users = pgTable('users', {
  id: serial('id').primaryKey(),
  clerkId: varchar('clerk_id', { length: 255 }).notNull().unique(), // Links to Clerk
  email: varchar('email', { length: 255 }).notNull().unique(),
  firstName: varchar('first_name', { length: 100 }),
  lastName: varchar('last_name', { length: 100 }),
  avatarUrl: text('avatar_url'),
  phoneNumber: varchar('phone_number', { length: 20 }),
  isPhoneVerified: boolean('is_phone_verified').default(false),
  createdAt: timestamp('created_at').defaultNow().notNull(),
  updatedAt: timestamp('updated_at').defaultNow().notNull(),
});
```

**How this works with Clerk:**
- User signs up via Clerk → webhook triggers
- Express API receives webhook → creates row in `users` table
- Now you have Clerk auth + custom user data in one place

**Table 2: Categories**
```typescript
// CATEGORIES TABLE
// Purpose: Organize listings (Electronics, Books, Sports, etc.)
// Why: Users need to filter by category when browsing
export const categories = pgTable('categories', {
  id: serial('id').primaryKey(),
  name: varchar('name', { length: 100 }).notNull().unique(), // e.g., "Electronics"
  slug: varchar('slug', { length: 100 }).notNull().unique(), // e.g., "electronics" (URL-friendly)
  description: text('description'),
  iconName: varchar('icon_name', { length: 50 }), // e.g., "laptop" for icon component
  isActive: boolean('is_active').default(true),
  createdAt: timestamp('created_at').defaultNow().notNull(),
});
```

**Why `slug` field:**
URL like `/categories/electronics` is better than `/categories/1`

---

### **Afternoon Session (4 hours) - Core Tables**

#### **Task 5: Create Database Schema - Part 2: Listings (90 mins)**

**Table 3: Listings (MOST IMPORTANT TABLE)**

**What you're doing:**
Creating the heart of your platform - the items people rent out.

```typescript
import { pgEnum } from 'drizzle-orm/pg-core';

// ENUMS (Predefined values)
// These prevent typos and give you autocomplete
export const listingStatusEnum = pgEnum('listing_status', [
  'draft',      // Owner is still editing
  'published',  // Live and visible to renters
  'rented',     // Currently rented out
  'inactive',   // Owner paused the listing
]);

export const conditionEnum = pgEnum('item_condition', [
  'new',
  'like_new',
  'good',
  'fair',
  'poor',
]);

// LISTINGS TABLE
export const listings = pgTable('listings', {
  // Identity
  id: serial('id').primaryKey(),
  ownerId: integer('owner_id')
    .references(() => users.id)
    .notNull(), // Foreign key to users table
  
  // Basic Info
  title: varchar('title', { length: 200 }).notNull(),
  description: text('description').notNull(),
  categoryId: integer('category_id')
    .references(() => categories.id)
    .notNull(),
  
  // Pricing
  dailyRate: integer('daily_rate').notNull(), // In paisa (100 paisa = 1 PKR)
  weeklyRate: integer('weekly_rate'), // Optional discount for weekly rentals
  securityDeposit: integer('security_deposit').notNull(),
  
  // Item Details
  condition: conditionEnum('condition').notNull(),
  itemValue: integer('item_value'), // How much the item is worth
  
  // Location (simplified for MVP)
  city: varchar('city', { length: 100 }).notNull(), // e.g., "Lahore"
  area: varchar('area', { length: 100 }), // e.g., "DHA Phase 5"
  
  // Status
  status: listingStatusEnum('status').default('draft').notNull(),
  
  // Metadata
  viewCount: integer('view_count').default(0),
  createdAt: timestamp('created_at').defaultNow().notNull(),
  updatedAt: timestamp('updated_at').defaultNow().notNull(),
  publishedAt: timestamp('published_at'), // When it went live
});
```

**CRITICAL DESIGN DECISIONS:**

1. **Why `dailyRate` is integer (not decimal)?**
   - Storing money as integers avoids floating-point errors
   - 100 paisa = 1 PKR, so 500 PKR = 50000 in database
   - JavaScript: `500 * 100 = 50000` ✅ (no precision loss)

2. **Why separate `weeklyRate`?**
   - Owners can offer discounts: "Rs. 500/day OR Rs. 3000/week (save 500!)"
   - NULL means no weekly discount available

3. **Why `ownerId` references `users.id`?**
   - This creates a relationship: "Each listing BELONGS TO one user"
   - Database enforces: Can't create listing without valid user

---

#### **Task 6: Create Database Schema - Part 3: Listing Photos (45 mins)**

**Table 4: Listing Photos**

**Why separate table?**
- One listing can have multiple photos (1-to-many relationship)
- Need to store order (first photo = cover image)

```typescript
export const listingPhotos = pgTable('listing_photos', {
  id: serial('id').primaryKey(),
  listingId: integer('listing_id')
    .references(() => listings.id, { onDelete: 'cascade' }) // If listing deleted, photos deleted
    .notNull(),
  
  cloudinaryUrl: text('cloudinary_url').notNull(), // Full URL from Cloudinary
  cloudinaryPublicId: varchar('cloudinary_public_id', { length: 255 }), // For deleting later
  
  displayOrder: integer('display_order').notNull(), // 1 = cover photo, 2 = second photo, etc.
  
  createdAt: timestamp('created_at').defaultNow().notNull(),
});
```

**How this works:**
```
Listing ID: 42
├── Photo 1: displayOrder = 1 (cover image)
├── Photo 2: displayOrder = 2
└── Photo 3: displayOrder = 3
```

**What is `onDelete: 'cascade'`?**
If you delete Listing #42, all its photos are automatically deleted from database.

---

#### **Task 7: Create Database Connection (45 mins)**

**What you're doing:**
Creating a reusable connection to query the database.

**File: `apps/api/src/db/index.ts`**

```typescript
import { drizzle } from 'drizzle-orm/postgres-js';
import postgres from 'postgres';
import * as schema from './schema.js';

// Create PostgreSQL connection
const connectionString = process.env.DATABASE_URL!;

if (!connectionString) {
  throw new Error('DATABASE_URL is not defined in environment variables');
}

// Connection pool (reuses connections for performance)
const client = postgres(connectionString, {
  max: 10, // Maximum 10 concurrent connections
  idle_timeout: 20, // Close idle connections after 20 seconds
});

// Drizzle instance (this is what you'll import to query)
export const db = drizzle(client, { schema });

// Export schema for type inference
export * from './schema.js';
```

**How you'll use this:**
```typescript
// In any file:
import { db } from './db/index.js';

// Now you can query:
const allListings = await db.select().from(listings);
```

---

#### **Task 8: Run Your First Migration (30 mins)**

**What you're doing:**
Converting your TypeScript schema into actual PostgreSQL tables.

**Step 1: Generate migration**
```bash
cd apps/api
pnpm drizzle-kit generate:pg
```

**What happens:**
- Drizzle reads `src/db/schema.ts`
- Creates SQL file in `migrations/0000_initial.sql`
- This SQL file creates all your tables

**Step 2: Check the generated SQL (IMPORTANT!)**
```bash
cat migrations/0000_*_initial.sql
```

You should see SQL like:
```sql
CREATE TABLE "users" (
  "id" SERIAL PRIMARY KEY,
  "clerk_id" VARCHAR(255) NOT NULL UNIQUE,
  ...
);

CREATE TABLE "categories" (...);
CREATE TABLE "listings" (...);
CREATE TABLE "listing_photos" (...);
```

**Step 3: Apply migration to database**
```bash
pnpm drizzle-kit push:pg
```

**What happens:**
- Connects to your Neon database
- Runs the SQL to create tables
- Now your database has real tables!

**Step 4: Verify in Neon dashboard**
- Go to neon.tech → your project → SQL Editor
- Run: `SELECT * FROM users;`
- You should see: "0 rows" (table exists but empty!)

---

### **End of Day 5 Checklist:**
- [ ] Neon database created and connection string saved
- [ ] Drizzle installed in both apps
- [ ] Schema file has 4 tables: users, categories, listings, listingPhotos
- [ ] Migration generated and applied
- [ ] Can see tables in Neon dashboard

**If stuck:**
- Check `.env` files have correct `DATABASE_URL`
- Ensure `drizzle.config.ts` points to right schema path
- Check Node version: `node --version` (should be 20+)

---

## 📅 DAY 6 - FIRST FEATURE: BROWSE LISTINGS (8 hours)

### **GOAL:** Build the listings browse page using your hybrid architecture

**ARCHITECTURE REMINDER:**
```
Next.js (apps/web)  → Direct DB read → Display listings (FAST, FREE, CACHED)
Express (apps/api)  → Not used for browsing → Only for complex writes
```

---

### **Morning Session (4 hours) - Seed Data & Next.js Page**

#### **Task 1: Create Seed Data Script (60 mins)**

**What you're doing:**
Adding fake data so you have something to display (like sample listings).

**Why seed data?**
You can't build a listings page with 0 listings! Need realistic test data.

**File: `apps/api/src/db/seed.ts`**

```typescript
import { db } from './index.js';
import { users, categories, listings, listingPhotos } from './schema.js';

async function seed() {
  console.log('🌱 Seeding database...');

  // 1. Create test user (simulate Clerk user)
  const [testUser] = await db.insert(users).values({
    clerkId: 'user_test123',
    email: 'owner@example.com',
    firstName: 'Test',
    lastName: 'Owner',
  }).returning();

  console.log('✅ Created test user:', testUser.id);

  // 2. Create categories
  const categoriesData = [
    { name: 'Electronics', slug: 'electronics', iconName: 'laptop' },
    { name: 'Sports Equipment', slug: 'sports', iconName: 'basketball' },
    { name: 'Books & Media', slug: 'books', iconName: 'book' },
    { name: 'Camping Gear', slug: 'camping', iconName: 'tent' },
  ];

  const createdCategories = await db.insert(categories)
    .values(categoriesData)
    .returning();

  console.log('✅ Created categories:', createdCategories.length);

  // 3. Create sample listings
  const listingsData = [
    {
      ownerId: testUser.id,
      categoryId: createdCategories[0].id, // Electronics
      title: 'Canon EOS 2000D DSLR Camera',
      description: 'Perfect for photography students. Includes 18-55mm lens, battery, charger, and camera bag.',
      dailyRate: 150000, // Rs. 1500/day (stored as paisa)
      weeklyRate: 900000, // Rs. 9000/week
      securityDeposit: 2000000, // Rs. 20,000 deposit
      condition: 'like_new',
      itemValue: 5000000, // Worth Rs. 50,000
      city: 'Lahore',
      area: 'DHA Phase 5',
      status: 'published',
      publishedAt: new Date(),
    },
    {
      ownerId: testUser.id,
      categoryId: createdCategories[1].id, // Sports
      title: 'Professional Badminton Racket Set',
      description: 'Yonex rackets (2 pieces) with shuttlecocks and carrying case.',
      dailyRate: 50000, // Rs. 500/day
      securityDeposit: 500000, // Rs. 5,000
      condition: 'good',
      city: 'Lahore',
      area: 'Gulberg',
      status: 'published',
      publishedAt: new Date(),
    },
    // Add 3-4 more varied examples...
  ];

  const createdListings = await db.insert(listings)
    .values(listingsData)
    .returning();

  console.log('✅ Created listings:', createdListings.length);

  // 4. Add photos to first listing
  await db.insert(listingPhotos).values([
    {
      listingId: createdListings[0].id,
      cloudinaryUrl: 'https://res.cloudinary.com/demo/image/upload/sample.jpg', // Placeholder
      displayOrder: 1,
    },
  ]);

  console.log('✅ Seed completed!');
}

seed()
  .catch((error) => {
    console.error('❌ Seed failed:', error);
    process.exit(1);
  })
  .finally(() => {
    process.exit(0);
  });
```

**Add to `apps/api/package.json` scripts:**
```json
{
  "scripts": {
    "seed": "tsx src/db/seed.ts"
  }
}
```

**Run it:**
```bash
cd apps/api
pnpm seed
```

**Expected output:**
```
🌱 Seeding database...
✅ Created test user: 1
✅ Created categories: 4
✅ Created listings: 2
✅ Seed completed!
```

---

#### **Task 2: Create Shared Types Package (45 mins)**

**What you're doing:**
Creating TypeScript types that both Next.js and Express can use (DRY principle).

**Why?**
Both apps need to know what a "Listing" looks like. Define it once in `packages/types`.

**File: `packages/types/src/listings.ts`**

```typescript
// This matches your database schema but is serializable (no Date objects)
export interface Listing {
  id: number;
  ownerId: number;
  title: string;
  description: string;
  categoryId: number;
  dailyRate: number; // In paisa
  weeklyRate: number | null;
  securityDeposit: number;
  condition: 'new' | 'like_new' | 'good' | 'fair' | 'poor';
  itemValue: number | null;
  city: string;
  area: string | null;
  status: 'draft' | 'published' | 'rented' | 'inactive';
  viewCount: number;
  createdAt: string; // ISO date string
  updatedAt: string;
  publishedAt: string | null;
}

// With relations (when you join tables)
export interface ListingWithPhotos extends Listing {
  photos: ListingPhoto[];
}

export interface ListingWithCategory extends Listing {
  category: Category;
}

export interface ListingPhoto {
  id: number;
  listingId: number;
  cloudinaryUrl: string;
  displayOrder: number;
}

export interface Category {
  id: number;
  name: string;
  slug: string;
  iconName: string | null;
}

// Helper function to format prices
export function formatPrice(paisa: number): string {
  const rupees = paisa / 100;
  return `Rs. ${rupees.toLocaleString('en-PK')}`;
}
```

**Update `packages/types/src/index.ts`:**
```typescript
export * from './listings.js';
export * from './schema.js';
export * from './errors.js';
// ... existing exports
```

---

#### **Task 3: Create Next.js Database Client (30 mins)**

**What you're doing:**
Setting up Drizzle in Next.js (same way as Express).

**File: `apps/web/lib/db.ts` (CREATE THIS DIRECTORY)**

```typescript
import { drizzle } from 'drizzle-orm/postgres-js';
import postgres from 'postgres';

const connectionString = process.env.DATABASE_URL!;

if (!connectionString) {
  throw new Error('DATABASE_URL not found');
}

const client = postgres(connectionString);
export const db = drizzle(client);
```

**Why?**
Next.js Server Components can now query the database directly for fast reads.

---

#### **Task 4: Create Listings Browse Page (90 mins)**

**What you're doing:**
Building your first real feature using the hybrid architecture!

**File: `apps/web/app/listings/page.tsx`**

```typescript
import { db } from '@/lib/db';
import { sql } from 'drizzle-orm';
import { formatPrice } from '@stuflux/types';
import Link from 'next/link';
import Image from 'next/image';

// Server Component (runs on server, no useState/useEffect)
export default async function ListingsPage() {
  // Direct database query (FAST!)
  const listings = await db.execute(sql`
    SELECT 
      l.*,
      c.name as category_name,
      c.slug as category_slug,
      (
        SELECT json_agg(
          json_build_object(
            'id', lp.id,
            'cloudinaryUrl', lp.cloudinary_url,
            'displayOrder', lp.display_order
          )
          ORDER BY lp.display_order
        )
        FROM listing_photos lp
        WHERE lp.listing_id = l.id
      ) as photos
    FROM listings l
    JOIN categories c ON l.category_id = c.id
    WHERE l.status = 'published'
    ORDER BY l.created_at DESC
  `);

  return (
    <div className="container mx-auto px-4 py-8">
      <h1 className="text-3xl font-bold mb-6">Browse Listings</h1>
      
      <div className="grid grid-cols-1 md:grid-cols-3 lg:grid-cols-4 gap-6">
        {listings.rows.map((listing: any) => (
          <ListingCard key={listing.id} listing={listing} />
        ))}
      </div>
    </div>
  );
}

// Client Component for interactivity
'use client';

function ListingCard({ listing }: { listing: any }) {
  const coverPhoto = listing.photos?.[0]?.cloudinaryUrl || '/placeholder.jpg';
  
  return (
    <Link href={`/listings/${listing.id}`}>
      <div className="border rounded-lg overflow-hidden hover:shadow-lg transition">
        <div className="relative h-48">
          <Image
            src={coverPhoto}
            alt={listing.title}
            fill
            className="object-cover"
          />
        </div>
        
        <div className="p-4">
          <span className="text-xs text-gray-500">{listing.category_name}</span>
          <h3 className="font-semibold text-lg truncate">{listing.title}</h3>
          <p className="text-sm text-gray-600">{listing.city}</p>
          
          <div className="mt-2">
            <span className="text-xl font-bold text-blue-600">
              {formatPrice(listing.daily_rate)}/day
            </span>
          </div>
        </div>
      </div>
    </Link>
  );
}
```

**WHAT'S HAPPENING HERE:**

1. **Server Component (default):**
   - Runs on server (not in browser)
   - Can directly query database
   - No loading spinners needed (renders with data)
   - Great for SEO (Google sees the content)

2. **SQL Query Breakdown:**
   ```sql
   SELECT l.*, c.name, ... 
   -- Get listing + category name
   
   (SELECT json_agg(...) FROM listing_photos WHERE ...)
   -- Nest all photos as JSON array
   
   WHERE l.status = 'published'
   -- Only show live listings
   
   ORDER BY l.created_at DESC
   -- Newest first
   ```

3. **Client Component (`'use client'`):**
   - Has interactivity (hover effects, click handlers)
   - Can use React hooks
   - Renders in browser

---

### **Afternoon Session (4 hours) - Individual Listing Page**

#### **Task 5: Create Single Listing View (120 mins)**

**What you're doing:**
When user clicks a listing, show full details with all photos.

**File: `apps/web/app/listings/[id]/page.tsx`**

```typescript
import { db } from '@/lib/db';
import { sql } from 'drizzle-orm';
import { formatPrice } from '@stuflux/types';
import { notFound } from 'next/navigation';
import Image from 'next/image';

// Dynamic route: /listings/42 → params.id = "42"
export default async function ListingDetailPage({
  params,
}: {
  params: { id: string };
}) {
  const listingId = parseInt(params.id);

  // Query database
  const result = await db.execute(sql`
    SELECT 
      l.*,
      c.name as category_name,
      u.first_name || ' ' || u.last_name as owner_name,
      u.avatar_url as owner_avatar,
      (
        SELECT json_agg(
          json_build_object(
            'id', lp.id,
            'url', lp.cloudinary_url,
            'order', lp.display_order
          )
          ORDER BY lp.display_order
        )
        FROM listing_photos lp
        WHERE lp.listing_id = l.id
      ) as photos
    FROM listings l
    JOIN categories c ON l.category_id = c.id
    JOIN users u ON l.owner_id = u.id
    WHERE l.id = ${listingId} AND l.status = 'published'
  `);

  const listing = result.rows[0];

  if (!listing) {
    notFound(); // Shows 404 page
  }

  // Increment view count (async, don't wait)
  db.execute(sql`
    UPDATE listings 
    SET view_count = view_count + 1 
    WHERE id = ${listingId}
  `).catch(console.error);

  return (
    <div className="container mx-auto px-4 py-8">
      <div className="grid grid-cols-1 lg:grid-cols-2 gap-8">
        {/* LEFT: Photos */}
        <div>
          <PhotoGallery photos={listing.photos || []} />
        </div>

        {/* RIGHT: Details */}
        <div>
          <span className="text-sm text-gray-500">{listing.category_name}</span>
          <h1 className="text-3xl font-bold mt-2">{listing.title}</h1>
          
          <div className="mt-4 flex items-center gap-3">
            <Image
              src={listing.owner_avatar || '/default-avatar.png'}
              alt={listing.owner_name}
              width={40}
              height={40}
              className="rounded-full"
            />
            <span className="font-medium">{listing.owner_name}</span>
          </div>

          <div className="mt-6 bg-gray-50 p-4 rounded-lg">
            <div className="text-3xl font-bold text-blue-600">
              {formatPrice(listing.daily_rate)}/day
            </div>
            {listing.weekly_rate && (
              <div className="text-lg text-gray-600 mt-1">
                {formatPrice(listing.weekly_rate)}/week (save{' '}
                {formatPrice(listing.daily_rate * 7 - listing.weekly_rate)}!)
              </div>
            )}
            <div className="text-sm text-gray-500 mt-2">
              Security Deposit: {formatPrice(listing.security_deposit)}
            </div>
          </div>

          <div className="mt-6">
            <h2 className="font-semibold text-lg">Description</h2>
            <p className="mt-2 text-gray-700 whitespace-pre-wrap">
              {listing.description}
            </p>
          </div>

          <div className="mt-6 grid grid-cols-2 gap-4 text-sm">
            <div>
              <span className="text-gray-500">Condition:</span>
              <span className="ml-2 font-medium capitalize">
                {listing.condition.replace('_', ' ')}
              </span>
            </div>
            <div>
              <span className="text-gray-500">Location:</span>
              <span className="ml-2 font-medium">
                {listing.area ? `${listing.area}, ` : ''}{listing.city}
              </span>
            </div>
          </div>

          {/* TODO: Add "Request Booking" button (Day 7-8) */}
          <button 
            className="mt-8 w-full bg-blue-600 text-white py-3 rounded-lg font-semibold hover:bg-blue-700"
            disabled
          >
            Request Booking (Coming Soon)
          </button>
        </div>
      </div>
    </div>
  );
}

// Photo gallery component
'use client';

function PhotoGallery({ photos }: { photos: any[] }) {
  const [selectedIndex, setSelectedIndex] = useState(0);

  if (!photos.length) {
    return <div className="bg-gray-200 h-96 rounded-lg" />;
  }

  return (
    <div>
      {/* Main photo */}
      <div className="relative h-96 rounded-lg overflow-hidden">
        <Image
          src={photos[selectedIndex].url}
          alt="Listing photo"
          fill
          className="object-cover"
        />
      </div>

      {/* Thumbnails */}
      {photos.length > 1 && (
        <div className="mt-4 grid grid-cols-4 gap-2">
          {photos.map((photo, index) => (
            <button
              key={photo.id}
              onClick={() => setSelectedIndex(index)}
              className={`relative h-20 rounded border-2 ${
                index === selectedIndex ? 'border-blue-600' : 'border-gray-200'
              }`}
            >
              <Image
                src={photo.url}
                alt=""
                fill
                className="object-cover"
              />
            </button>
          ))}
        </div>
      )}
    </div>
  );
}
```

**KEY CONCEPTS:**

1. **Dynamic Routes:**
   - File: `[id]/page.tsx` → Matches `/listings/any-id`
   - Access via: `params.id`

2. **404 Handling:**
   - `notFound()` → Shows Next.js 404 page
   - Prevents crashes from invalid IDs

3. **View Count:**
   - Fire-and-forget query (don't await)
   - Doesn't slow down page load

4. **Photo Gallery:**
   - Client component (needs `useState`)
   - Click thumbnail → change main photo

---

#### **Task 6: Add Navigation & Layout (60 mins)**

**What you're doing:**
Adding header/footer so users can navigate between pages.

**File: `apps/web/components/Header.tsx` (CREATE THIS)**

```typescript
import Link from 'next/link';
import { SignInButton, SignedIn, SignedOut, UserButton } from '@clerk/nextjs';

export function Header() {
  return (
    <header className="border-b">
      <div className="container mx-auto px-4 py-4 flex items-center justify-between">
        <Link href="/" className="text-2xl font-bold text-blue-600">
          StuFlux
        </Link>

        <nav className="flex items-center gap-6">
          <Link href="/listings" className="hover:text-blue-600">
            Browse
          </Link>
          
          <SignedIn>
            <Link href="/listings/new" className="hover:text-blue-600">
              List Item
            </Link>
            <Link href="/dashboard" className="hover:text-blue-600">
              Dashboard
            </Link>
            <UserButton afterSignOutUrl="/" />
          </SignedIn>

          <SignedOut>
            <SignInButton mode="modal">
              <button className="bg-blue-600 text-white px-4 py-2 rounded-lg">
                Sign In
              </button>
            </SignInButton>
          </SignedOut>
        </nav>
      </div>
    </header>
  );
}
```

**Update `apps/web/app/layout.tsx`:**
```typescript
import { ClerkProvider } from '@clerk/nextjs';
import { Header } from '@/components/Header';
import './globals.css';

export default function RootLayout({ children }) {
  return (
    <ClerkProvider>
      <html lang="en">
        <body>
          <Header />
          <main className="min-h-screen">{children}</main>
          <footer className="border-t py-8 text-center text-gray-500">
            © 2025 StuFlux - Peer-to-Peer Rentals
          </footer>
        </body>
      </html>
    </ClerkProvider>
  );
}
```

---

#### **Task 7: Test Everything (60 mins)**

**Manual testing checklist:**

1. **Start both servers:**
   ```bash
   # Terminal 1
   cd apps/web
   pnpm dev
   
   # Terminal 2 (not needed yet, but good to test)
   cd apps/api
   pnpm dev
   ```

2. **Test listings page:**
   - Go to `http://localhost:3000/listings`
   - ✅ See grid of listings
   - ✅ See prices formatted correctly
   - ✅ See category names

3. **Test individual listing:**
   - Click a listing
   - ✅ See all photos
   - ✅ See owner name
   - ✅ See full description
   - ✅ Click thumbnails to change main photo

4. **Test navigation:**
   - ✅ Click logo → goes to home
   - ✅ Click "Browse" → goes to listings
   - ✅ Sign in button works (Clerk modal opens)

5. **Test database:**
   - Open new listing page
   - Check view count incremented in database:
   ```sql
   -- In Neon SQL Editor
   SELECT id, title, view_count FROM listings;
   ```

---

### **End of Day 6 Checklist:**
- [ ] Seed script runs and creates sample data
- [ ] Shared types package has listing interfaces
- [ ] Next.js can query database directly
- [ ] Listings browse page shows all published listings
- [ ] Individual listing page shows full details
- [ ] Navigation header works with Clerk auth
- [ ] View count increments when viewing listing

---

## 🎓 LEARNING OUTCOMES

After Day 5-6, you should understand:

### **Database Concepts:**
- ✅ **Schema Design:** How tables relate (foreign keys, enums)
- ✅ **Migrations:** TypeScript → SQL → Live Database
- ✅ **Seed Data:** Creating test data for development
- ✅ **Drizzle ORM:** Type-safe queries vs raw SQL

### **Architecture Patterns:**
- ✅ **Server Components:** When to query DB directly in Next.js
- ✅ **Client Components:** When to use `'use client'` directive
- ✅ **Shared Types:** DRY principle across monorepo
- ✅ **Hybrid Reads:** Next.js for simple reads (listings browse)

### **Next.js Features:**
- ✅ **Dynamic Routes:** `[id]` folder pattern
- ✅ **Image Optimization:** `next/image` component
- ✅ **Data Fetching:** `async` Server Components
- ✅ **Error Handling:** `notFound()` function

---

## 🔧 TROUBLESHOOTING

### **Issue: "DATABASE_URL is not defined"**
**Solution:**
- Check `.env` file exists in correct location
- Restart dev server after adding env variables
- Print the value: `console.log(process.env.DATABASE_URL)`

### **Issue: "Module not found: @stuflux/types"**
**Solution:**
```bash
# From workspace root
pnpm install
# Rebuild types package
cd packages/types
pnpm build
```

### **Issue: "Cannot find table 'listings'"**
**Solution:**
- Run migration: `cd apps/api && pnpm drizzle-kit push:pg`
- Check Neon dashboard: Tables should exist
- Verify DATABASE_URL is correct

### **Issue: "Photos not showing"**
**Solution:**
- Placeholder photos should work (Cloudinary demo images)
- Check `listingPhotos` table has rows: `SELECT * FROM listing_photos;`
- Verify `cloudinaryUrl` is not NULL

---

## 📚 NEXT STEPS (Day 7-8 Preview)

Now that browsing works, you'll build:
- ✅ **Create Listing Form** (Express API endpoint)
- ✅ **Photo Upload to Cloudinary** (client → Cloudinary → API)
- ✅ **Auth Middleware** (protect API routes with Clerk JWT)
- ✅ **User Dashboard** (show "My Listings")

This completes the **Create → Browse** loop!

---

## 💡 TIPS FOR SUCCESS

1. **Don't skip seed data:**
   - Hard to build UI with empty database
   - Add 5-10 varied listings

2. **Test incrementally:**
   - After each task, test it works
   - Don't wait until end of day

3. **Use Neon SQL Editor:**
   - Great for debugging queries
   - Can manually inspect/edit data

4. **Ask questions when stuck:**
   - Database errors can be cryptic
   - Check migration files to understand schema

5. **Keep Express API simple for now:**
   - Days 5-6 focus on Next.js reads
   - Days 7-8 focus on Express writes

**Good luck! 🚀**
