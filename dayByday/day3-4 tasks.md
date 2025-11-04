# 📅 Day 3 & 4 Implementation Guide - StuFlux Authentication & Database (Drizzle Edition)

## 🎯 Overview
This guide covers Days 3-4 of Sprint 1, focusing on **Express API completion**, **Clerk Authentication integration**, and **PostgreSQL + Drizzle ORM setup**. Building on the monorepo foundation from Day 1-2.

## ✅ 2025 Compatibility Check

### **Prerequisites Verified:**
- ✅ **Node.js 20+**: Required for latest Express 5.x and Clerk SDK
- ✅ **TypeScript 5.6+**: ESM modules with strict type checking
- ✅ **pnpm 8.15+**: Modern package manager with workspace support
- ✅ **Express 5.1.0**: Latest major version with async/await improvements
- ✅ **Clerk SDK**: `@clerk/backend@^1.15.0` with latest JWT verification
- ✅ **PostgreSQL 15+**: Neon default version for optimal performance

### **Breaking Changes Addressed:**
- ✅ **Express 5.x**: Updated middleware patterns and error handling
- ✅ **Clerk Auth**: Proper JWT verification with `verifyToken` (not deprecated methods)
- ✅ **ESM Modules**: `.js` extensions in TypeScript imports for Node.js compatibility
- ✅ **Error Handling**: Type-safe error catching with proper `any` annotations

## 📋 Day 3 Tasks - Express API + Clerk Authentication

### 1. Express API Structure Completion (2-3 hours)

#### ~~Backend Directory Structure Setup~~
```bash
cd apps/api

# Create remaining API structure (building on Day 2)
mkdir -p migrations seeds tests

# Create essential files
touch src/routes/{health.ts,listings.ts,uploads.ts}
touch src/middleware/{auth.ts,errors.ts,logger.ts}
touch src/utils/cloudinary.ts
touch src/db.ts
```

#### ~~Updated Express Index (src/index.ts) - Enhanced from Day 2~~
```typescript
import express from 'express';
import cors from 'cors';
import helmet from 'helmet';
import morgan from 'morgan';
import compression from 'compression';
import dotenv from 'dotenv';

// Import routes
import healthRoutes from './routes/health.js';
import listingsRoutes from './routes/listings.js';
import uploadsRoutes from './routes/uploads.js';

// Import middleware
import { errorHandler } from './middleware/errors.js';
import { requestLogger } from './middleware/logger.js';

// Load environment variables
dotenv.config();
																		
const app : Application = express();
const PORT = process.env.PORT || 4000;

// Trust proxy for Railway/Heroku deployment
app.set('trust proxy', 1);

// Security middleware (Enhanced)
app.use(helmet({
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      styleSrc: ["'self'", "'unsafe-inline'"],
      scriptSrc: ["'self'"],
      imgSrc: ["'self'", "data:", "https:", "https://res.cloudinary.com"],
      connectSrc: ["'self'", "https://api.clerk.com"],
    }
  },
  crossOriginEmbedderPolicy: false, // Required for Cloudinary uploads
}));

// CORS configuration (Production ready)
const allowedOrigins = process.env.ALLOWED_ORIGINS?.split(',') || [
  'http://localhost:3000',
  'https://stuflux.vercel.app' // Add your Vercel domain
];

app.use(cors({
  origin: (origin, callback) => {
    // Allow requests with no origin (mobile apps, Postman, etc.)
    if (!origin) return callback(null, true);
    
    if (allowedOrigins.includes(origin)) {
      callback(null, true);
    } else {
      callback(new Error('Not allowed by CORS'));
    }
  },
  credentials: true,
  methods: ['GET', 'POST', 'PUT', 'DELETE', 'PATCH'],
  allowedHeaders: ['Content-Type', 'Authorization', 'X-Requested-With']
}));

// Parsing & compression
app.use(compression());
app.use(express.json({ limit: '10mb' }));
app.use(express.urlencoded({ extended: true, limit: '10mb' }));

// Logging
app.use(requestLogger);

// API Routes (v1 namespace)
app.use('/api/v1/health', healthRoutes);
app.use('/api/v1/listings', listingsRoutes);
app.use('/api/v1/uploads', uploadsRoutes);

// Root route
app.get('/', (req, res) => {
  res.json({ 
    name: 'StuFlux API',
    version: '1.0.0',
    status: 'running',
    docs: '/api/v1/health'
  });
});

// 404 handler
app.use('*', (req, res) => {
  res.status(404).json({ 
    error: 'Route not found',
    path: req.originalUrl,
    method: req.method
  });
});

// Global error handler (must be last)
app.use(errorHandler);

// Graceful shutdown
const server = app.listen(PORT, () => {
  console.log(`🚀 StuFlux API running on http://localhost:${PORT}`);
  console.log(`📊 Environment: ${process.env.NODE_ENV || 'development'}`);
});

// Handle graceful shutdown
process.on('SIGTERM', () => {
  console.log('⏱️  SIGTERM received, shutting down gracefully');
  server.close(() => {
    console.log('💤 Process terminated');
    process.exit(0);
  });
});

export default app;
```

#### ~~Error Handling Middleware (src/middleware/errors.ts)~~
```typescript
import { Request, Response, NextFunction } from 'express';
import { ZodError } from 'zod';

export interface ApiError extends Error {
  statusCode?: number;
  code?: string;
}

export class AppError extends Error implements ApiError {
  statusCode: number;
  code: string;
  isOperational: boolean;

  constructor(message: string, statusCode: number = 500, code: string = 'INTERNAL_ERROR') {
    super(message);
    this.statusCode = statusCode;
    this.code = code;
    this.isOperational = true;

    Error.captureStackTrace(this, this.constructor);
  }
}

export const errorHandler = (
  err: ApiError | ZodError | Error,
  req: Request,
  res: Response,
  next: NextFunction
) => {
  let statusCode = 500;
  let message = 'Internal server error';
  let code = 'INTERNAL_ERROR';

  // Zod validation errors
  if (err instanceof ZodError) {
    statusCode = 400;
    code = 'VALIDATION_ERROR';
    message = 'Validation failed';
    
    return res.status(statusCode).json({
      error: {
        code,
        message,
        details: err.errors.map(error => ({
          field: error.path.join('.'),
          message: error.message
        }))
      }
    });
  }

  // Custom app errors
  if (err instanceof AppError) {
    statusCode = err.statusCode;
    message = err.message;
    code = err.code;
  }

  // Log error (in production, use proper logging service)
  console.error(`❌ Error ${code}:`, {
    message: err.message,
    stack: err.stack,
    url: req.url,
    method: req.method,
    ip: req.ip,
    timestamp: new Date().toISOString()
  });

  res.status(statusCode).json({
    error: {
      code,
      message,
      timestamp: new Date().toISOString(),
      ...(process.env.NODE_ENV === 'development' && { 
        stack: err.stack,
        details: err.message 
      })
    }
  });
};

// Async error wrapper with proper typing
export const asyncHandler = (fn: (req: Request, res: Response, next: NextFunction) => Promise<any>) => {
  return (req: Request, res: Response, next: NextFunction) => {
    Promise.resolve(fn(req, res, next)).catch(next);
  };
};
```

#### ~~Logger Middleware (src/middleware/logger.ts)~~
```typescript
import { Request, Response, NextFunction } from 'express';

export const requestLogger = (req: Request, res: Response, next: NextFunction) => {
  const start = Date.now();

  // Log request
  console.log(`📥 ${req.method} ${req.path}`, {
    timestamp: new Date().toISOString(),
    ip: req.ip,
    userAgent: req.get('User-Agent')?.substring(0, 100) || 'Unknown',
    contentLength: req.get('Content-Length') || '0'
  });

  // Log response when finished
  res.on('finish', () => {
    const duration = Date.now() - start;
    const status = res.statusCode;
    const emoji = status >= 400 ? '❌' : status >= 300 ? '⚠️' : '✅';
    
    console.log(`📤 ${emoji} ${status} ${req.method} ${req.path} - ${duration}ms`);
  });

  next();
};
```

### 2. Clerk Authentication Setup (3-4 hours)

#### Frontend Clerk Integration (apps/web)

~~**Updated .env.example** (apps/web)~~
```env
# Database (for Server Components - hybrid approach)
DATABASE_URL=postgresql://user:password@host:5432/dbname

# Clerk Authentication (CORRECTED - Latest API)
NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=pk_test_...
CLERK_SECRET_KEY=sk_test_...
NEXT_PUBLIC_CLERK_SIGN_IN_URL=/sign-in
NEXT_PUBLIC_CLERK_SIGN_UP_URL=/sign-up
NEXT_PUBLIC_CLERK_AFTER_SIGN_IN_URL=/dashboard
NEXT_PUBLIC_CLERK_AFTER_SIGN_UP_URL=/dashboard

# API Configuration
NEXT_PUBLIC_API_URL=http://localhost:4000/api/v1
NEXT_PUBLIC_APP_URL=http://localhost:3000

# Cloudinary (client-side uploads)
NEXT_PUBLIC_CLOUDINARY_CLOUD_NAME=your_cloud_name
NEXT_PUBLIC_CLOUDINARY_API_KEY=your_api_key

# Development
NODE_ENV=development
```

~~**Root Layout with Clerk Provider** (apps/web/src/app/layout.tsx)~~
```typescript
import { clerkAppearance } from '@/lib/clerk-theme';
import { ClerkProvider } from '@clerk/nextjs';
import { Inter } from 'next/font/google';
import './globals.css';

const inter = Inter({ subsets: ['latin'] });

export const metadata = {
  title: 'StuFlux - Peer-to-Peer Rental Platform',
  description: 'Rent anything from your neighbors - cameras, tools, vehicles & more',
};

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <ClerkProvider
      publishableKey={process.env.NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY!}
      appearance={clerkAppearance}
    >
      <html lang="en">
        <body className={inter.className}>
          <div className="min-h-screen bg-gray-50">
            {children}
          </div>
        </body>
      </html>
    </ClerkProvider>
  );
}
```

**Authentication Pages** (apps/web/src/app/sign-in/[[...sign-in]]/page.tsx)
```typescript
import { SignIn } from '@clerk/nextjs';

export default function SignInPage() {
  return (
    <div className="flex min-h-screen items-center justify-center bg-gray-50 py-12 px-4 sm:px-6 lg:px-8">
      <div className="w-full max-w-md space-y-8">
        <div>
          <h2 className="mt-6 text-center text-3xl font-bold tracking-tight text-gray-900">
            Sign in to StuFlux
          </h2>
          <p className="mt-2 text-center text-sm text-gray-600">
            Access your rental dashboard
          </p>
        </div>
        <SignIn />
      </div>
    </div>
  );
}
```

~~**Sign Up Page** (apps/web/src/app/sign-up/[[...sign-up]]/page.tsx)~~
```typescript
import { SignUp } from '@clerk/nextjs';

export default function SignUpPage() {
  return (
    <div className="flex min-h-screen items-center justify-center bg-gray-50 py-12 px-4 sm:px-6 lg:px-8">
      <div className="w-full max-w-md space-y-8">
        <div>
          <h2 className="mt-6 text-center text-3xl font-bold tracking-tight text-gray-900">
            Join StuFlux
          </h2>
          <p className="mt-2 text-center text-sm text-gray-600">
            Start renting and earning today
          </p>
        </div>
        <SignUp />
      </div>
    </div>
  );
}
```

~~**Clerk HTTP Client for API Calls** (apps/web/src/lib/api-client.ts)~~
```typescript
import ky from 'ky';
import { useAuth } from '@clerk/nextjs';

const API_BASE_URL = process.env.NEXT_PUBLIC_API_URL || 'http://localhost:4000/api/v1';

// Create base client
const apiClient = ky.create({
  prefixUrl: API_BASE_URL,
  timeout: 10000,
  retry: {
    limit: 2,
    methods: ['get'],
  },
});

// Hook for authenticated API calls
export function useApiClient() {
  const { getToken } = useAuth();

  const authenticatedClient = apiClient.extend({
    hooks: {
      beforeRequest: [
        async (request) => {
          const token = await getToken();
          if (token) {
            request.headers.set('Authorization', `Bearer ${token}`);
          }
        },
      ],
    },
  });

  return authenticatedClient;
}

// Server-side API client (for Server Components)
export async function createServerApiClient(token?: string) {
  return apiClient.extend({
    hooks: {
      beforeRequest: [
        async (request) => {
          if (token) {
            request.headers.set('Authorization', `Bearer ${token}`);
          }
        },
      ],
    },
  });
}

export default apiClient;
```

#### Backend Clerk Authentication (apps/api)

~~**Updated .env.example** (apps/api)~~
```env
# Server Configuration
PORT=4000
NODE_ENV=development

# Database
DATABASE_URL=postgresql://user:password@host:5432/dbname

# Clerk Authentication (CORRECTED - Backend SDK)
CLERK_SECRET_KEY=sk_test_...
CLERK_PUBLISHABLE_KEY=pk_test_...

# Cloudinary (server-side signatures)
CLOUDINARY_CLOUD_NAME=your_cloud_name
CLOUDINARY_API_KEY=your_api_key
CLOUDINARY_API_SECRET=your_api_secret

# CORS
ALLOWED_ORIGINS=http://localhost:3000,https://stuflux.vercel.app

# Logging
LOG_LEVEL=debug
```

~~**Clerk Authentication Middleware** (src/middleware/auth.ts) - CORRECTED~~
```typescript
import { Request, Response, NextFunction } from 'express';
import { createClerkClient, verifyToken } from '@clerk/backend';
import { AppError } from './errors.js';

const clerkClient = createClerkClient({
  secretKey: process.env.CLERK_SECRET_KEY!,
});

export interface AuthenticatedRequest extends Request {
  auth?: {
    userId: string;
    sessionId: string;
    claims: any;
  };
}

export const requireAuth = async (
  req: AuthenticatedRequest,
  res: Response,
  next: NextFunction
) => {
  try {
    const authHeader = req.headers.authorization;
    
    if (!authHeader?.startsWith('Bearer ')) {
      throw new AppError('No authorization token provided', 401, 'UNAUTHORIZED');
    }

    const token = authHeader.substring(7); // Remove 'Bearer ' prefix

    // Verify the JWT token with Clerk
    const payload = await verifyToken(token, {
      secretKey: process.env.CLERK_SECRET_KEY!,
    });

    // Add user info to request
    req.auth = {
      userId: payload.sub,
      sessionId: payload.sid as string,
      claims: payload
    };

    next();
  } catch (error: any) {
    console.error('🔒 Auth verification failed:', error.message);
    
    if (error instanceof AppError) {
      next(error);
    } else {
      next(new AppError('Invalid authentication token', 401, 'INVALID_TOKEN'));
    }
  }
};

// Optional auth - doesn't fail if no token
export const optionalAuth = async (
  req: AuthenticatedRequest,
  res: Response,
  next: NextFunction
) => {
  try {
    const authHeader = req.headers.authorization;
    
    if (authHeader?.startsWith('Bearer ')) {
      const token = authHeader.substring(7);
      const payload = await verifyToken(token, {
        secretKey: process.env.CLERK_SECRET_KEY!,
      });

      req.auth = {
        userId: payload.sub,
        sessionId: payload.sid as string,
        claims: payload
      };
    }

    next();
  } catch (error: any) {
    // Silently continue without auth
    console.warn('⚠️ Optional auth failed:', error?.message || 'Unknown error');
    next();
  }
};

// Get user details from Clerk
export const getUserDetails = async (userId: string) => {
  try {
    const user = await clerkClient.users.getUser(userId);
    return {
      id: user.id,
      email: user.emailAddresses[0]?.emailAddress,
      name: `${user.firstName || ''} ${user.lastName || ''}`.trim(),
      avatar: user.imageUrl,
      createdAt: new Date(user.createdAt)
    };
  } catch (error: any) {
    console.error('Failed to fetch user details:', error?.message || error);
    throw new AppError('Failed to fetch user details', 500, 'USER_FETCH_ERROR');
  }
};
```

### 3. API Routes Implementation (2-3 hours)

#### ~~Health Check Route (src/routes/health.ts)~~
```typescript
import { Router } from 'express';
import { asyncHandler } from '../middleware/errors.js';
import { testConnection } from '../db/index.js';

const router = Router();

router.get('/', asyncHandler(async (req, res) => {
  const healthCheck = {
    status: 'ok',
    timestamp: new Date().toISOString(),
    uptime: process.uptime(),
    version: process.env.npm_package_version || '1.0.0',
    environment: process.env.NODE_ENV || 'development',
    memory: {
      used: Math.round((process.memoryUsage().heapUsed / 1024 / 1024) * 100) / 100,
      total: Math.round((process.memoryUsage().heapTotal / 1024 / 1024) * 100) / 100
    }
  };

  res.json(healthCheck);
}));

// Database health check (will add in Day 4)
router.get('/db', asyncHandler(async (req, res) => {
  const isConnected = await testConnection();
  
  res.json({ 
    status: isConnected ? 'ok' : 'error',
    database: {
      connected: isConnected,
      type: 'PostgreSQL',
      provider: 'Neon',
      timestamp: new Date().toISOString()
    }
  });
}));

export default router;
```

#### ~~Basic Listings Route (src/routes/listings.ts) - Skeleton~~
```typescript
import { Router } from 'express';
import { z } from 'zod';
import { requireAuth, optionalAuth, type AuthenticatedRequest } from '../middleware/auth.js';
import { asyncHandler, AppError } from '../middleware/errors.js';

const router = Router();

// Validation schemas
const createListingSchema = z.object({
  title: z.string().min(3).max(200),
  description: z.string().min(10).max(2000),
  category_id: z.number().int().positive(),
  daily_rate: z.number().int().positive(),
  city: z.string().min(2).max(100),
  address: z.string().optional(),
  latitude: z.number().optional(),
  longitude: z.number().optional(),
  min_rental_days: z.number().int().positive().default(1),
  max_rental_days: z.number().int().positive().default(30),
  delivery_available: z.boolean().default(false),
  delivery_fee: z.number().int().min(0).default(0),
  security_deposit: z.number().int().min(0).default(0),
  specs: z.record(z.any()).default({})
});

// GET /api/v1/listings - Browse listings (public)
router.get('/', optionalAuth, asyncHandler(async (req: AuthenticatedRequest, res) => {
  // TODO: Implement in Day 7 (Database + Schema)
  res.json({
    message: 'Listings endpoint ready - will be implemented with database in Day 7',
    authenticated: !!req.auth,
    timestamp: new Date().toISOString()
  });
}));

// GET /api/v1/listings/:id - Get single listing (public)
router.get('/:id', optionalAuth, asyncHandler(async (req: AuthenticatedRequest, res) => {
  const { id } = req.params;
  
  // TODO: Implement in Day 7
  res.json({
    message: `Listing ${id} endpoint ready - will be implemented with database`,
    id,
    authenticated: !!req.auth
  });
}));

// POST /api/v1/listings - Create listing (auth required)
router.post('/', requireAuth, asyncHandler(async (req: AuthenticatedRequest, res) => {
  const validatedData = createListingSchema.parse(req.body);
  
  // TODO: Implement in Day 7
  res.status(201).json({
    message: 'Create listing endpoint ready',
    userId: req.auth!.userId,
    data: validatedData
  });
}));

// PUT /api/v1/listings/:id - Update listing (auth required)
router.put('/:id', requireAuth, asyncHandler(async (req: AuthenticatedRequest, res) => {
  const { id } = req.params;
  const validatedData = createListingSchema.partial().parse(req.body);
  
  // TODO: Implement in Day 7
  res.json({
    message: `Update listing ${id} endpoint ready`,
    userId: req.auth!.userId,
    data: validatedData
  });
}));

export default router;
```

#### ~~Basic Uploads Route (src/routes/uploads.ts) - Skeleton~~
```typescript
import { Router } from 'express';
import { requireAuth, type AuthenticatedRequest } from '../middleware/auth.js';
import { asyncHandler } from '../middleware/errors.js';

const router = Router();

// POST /api/v1/uploads/signature - Get Cloudinary signature (auth required)**Clerk HTTP Client for API Calls** (apps/web/src/lib/api-client.ts)
router.post('/signature', requireAuth, asyncHandler(async (req: AuthenticatedRequest, res) => {
  // TODO: Implement in Day 10-11 (Cloudinary integration)
  res.json({
    message: 'Upload signature endpoint ready - will be implemented with Cloudinary in Day 10-11',
    userId: req.auth!.userId,
    timestamp: new Date().toISOString()
  });
}));

// POST /api/v1/uploads/complete - Complete upload process (auth required)
router.post('/complete', requireAuth, asyncHandler(async (req: AuthenticatedRequest, res) => {
  // TODO: Implement in Day 10-11
  res.json({
    message: 'Upload completion endpoint ready',
    userId: req.auth!.userId
  });
}));

export default router;
```

## 📋 Day 4 Tasks - Database & Drizzle ORM Setup

### 1. Neon PostgreSQL Setup (1-2 hours)

#### Create Neon Database Account
1. **Go to [neon.tech](https://neon.tech)** and create account
2. **Create new project**: 
   - Name: `stuflux-dev`
   - Region: Choose closest to your location
   - PostgreSQL version: 15 (latest stable)
3. **Get connection string** from dashboard
4. **Copy connection string** - it should look like:
   ```
   postgresql://username:password@ep-xyz.region.neon.tech/dbname?sslmode=require
   ```

#### Environment Configuration
**Update apps/web/.env.local**
```env
DATABASE_URL=postgresql://username:password@ep-xyz.region.neon.tech/dbname?sslmode=require
```

**Update apps/api/.env**
```env
DATABASE_URL=postgresql://username:password@ep-xyz.region.neon.tech/dbname?sslmode=require
```

### 2. Drizzle ORM Setup (2-3 hours)

#### Install Dependencies
```bash
# Backend dependencies
cd apps/api
pnpm add drizzle-orm pg
pnpm add -D drizzle-kit @types/pg

# Frontend dependencies
cd ../web
pnpm add drizzle-orm pg
pnpm add -D drizzle-kit @types/pg
```

#### Create Drizzle Config
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

#### Define Database Schema
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

#### Setup Database Client
Create `apps/api/src/db/index.ts`:
```typescript
import { drizzle } from 'drizzle-orm/node-postgres';
import { Pool } from 'pg';
import * as schema from './schema';

const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
  ssl: process.env.NODE_ENV === 'production' ? { rejectUnauthorized: false } : false,
});

export const db = drizzle(pool, { schema });

// Test database connection
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

### 3. Migration Setup (2-3 hours)

#### Create Migration Scripts
Update `apps/api/package.json`:
```json
{
  "scripts": {
    // ...existing scripts...
    "db:generate": "drizzle-kit generate:pg",
    "db:push": "drizzle-kit push:pg",
    "db:studio": "drizzle-kit studio",
    "db:seed": "tsx src/db/seed.ts"
  }
}
```

#### Generate Initial Migration
```bash
cd apps/api
pnpm db:generate
```

#### Run Migration
```bash
pnpm db:push
```

### 4. Next.js Server Components Integration (1-2 hours)

Create `apps/web/src/lib/db.ts`:
```typescript
import { drizzle } from 'drizzle-orm/node-postgres';
import { Pool } from 'pg';
import * as schema from '../../api/src/db/schema';

// Singleton pattern for Next.js
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

Example Server Component usage:
```typescript
import { getDb } from '@/lib/db';
import { listings } from '../../../api/src/db/schema';

export default async function ListingsPage() {
  const db = getDb();
  const allListings = await db.select().from(listings).where(eq(listings.status, 'active'));
  
  return (
    <div>
      {allListings.map(listing => (
        <div key={listing.id}>{listing.title}</div>
      ))}
    </div>
  );
}
```

## ✅ End of Day 4 Checklist

### Database Setup
- [ ] Neon PostgreSQL connected
- [ ] Drizzle ORM installed and configured
- [ ] Schema defined in TypeScript
- [ ] Migrations generated and applied
- [ ] Seed data script created

### Integration
- [ ] Next.js Server Components can query database
- [ ] Express API using Drizzle for queries
- [ ] Type safety across full stack
- [ ] Health checks passing
