# 📅 Day 1 & 2 Implementation Guide - StuFlux Setup (CORRECTED)

## 🎯 Overview
This guide covers the first two days of Sprint 1, focusing on **monorepo setup** and development environment configuration. We'll be setting up the hybrid architecture (Next.js + Express) in a single repository with proper workspace management.

## 📋 Day 1 Tasks

### ~~1. Monorepo Setup (1-2 hours)~~
```bash
# Create monorepo structure (matches corrected plan)
mkdir stuflux && cd stuflux

# Initialize root workspace
npm init -y
```

#### ~~Create Workspace Structure~~
```bash
# Create monorepo directories
mkdir -p apps/web apps/api packages/types

# Initialize pnpm workspace (better than npm for monorepos)
echo 'packages:
  - "apps/*"
  - "packages/*"' > pnpm-workspace.yaml
```

#### ~~Frontend (Next.js) Setup~~
```bash
cd apps/web
# Initialize Next.js 14 project with TypeScript
npx create-next-app@latest . --typescript --tailwind --eslint --app --src-dir --import-alias "@/*"
```

#### ~~Backend (Express) Setup~~
```bash
cd ../api
# Initialize Node project
pnpm init

# Install modern dependencies (updated packages)
pnpm install express @types/express typescript
pnpm install -D tsx nodemon @types/node

# No need for ts-node-dev (deprecated) - use tsx instead
# No need for manual TypeScript init - will customize
```

#### ~~Root Repository Setup~~
```bash
cd ../..
# Initialize Git at root level (monorepo)
git init
echo "node_modules/
.env*
!.env.example
dist/
.next/
coverage/
.turbo/
*.log" > .gitignore

git add .
git commit -m "Initial commit: Monorepo setup with Next.js + Express"
```

### 2. Development Environment Setup (2-3 hours)

#### Frontend Configuration (apps/web)
Create the following files in `apps/web`:

~~**.env.example** (Updated for hybrid architecture)~~
```env
# Database (for Server Components - hybrid approach)
DATABASE_URL=postgresql://user:password@host:5432/dbname

# Clerk Authentication
NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=pk_test_...
CLERK_SECRET_KEY=sk_test_...

# API Configuration
NEXT_PUBLIC_API_URL=http://localhost:4000/api/v1
NEXT_PUBLIC_APP_URL=http://localhost:3000


# Cloudinary (client-side uploads)
NEXT_PUBLIC_CLOUDINARY_CLOUD_NAME=your_cloud_name
NEXT_PUBLIC_CLOUDINARY_API_KEY=your_api_key

# Development
NODE_ENV=development
```

~~**✅ tsconfig.json** (Next.js 14 default - perfect as-is!)~~
```json
{
  "compilerOptions": {
    "target": "ES2017",
    "lib": ["dom", "dom.iterable", "esnext"],
    "allowJs": true,
    "skipLibCheck": true,
    "strict": true,
    "noEmit": true,
    "esModuleInterop": true,
    "module": "esnext",
    "moduleResolution": "bundler",
    "resolveJsonModule": true,
    "isolatedModules": true,
    "jsx": "react-jsx",
    "incremental": true,
    "plugins": [
      {
        "name": "next"
      }
    ],
    "paths": {
      "@/*": ["./src/*"]
    }
  },
  "include": [
    "next-env.d.ts",
    "**/*.ts",
    "**/*.tsx",
    ".next/types/**/*.ts",
    ".next/dev/types/**/*.ts",
    "**/*.mts"
  ],
  "exclude": ["node_modules"]
}
```

#### ~~Backend Configuration (apps/api)~~
Create the following files in `apps/api`:

**.env.example**
```env
# Server Configuration
PORT=4000
NODE_ENV=development

# Database
DATABASE_URL=postgresql://user:password@host:5432/dbname

# Clerk Authentication
CLERK_SECRET_KEY=sk_test_...

# Cloudinary (server-side signatures)
CLOUDINARY_CLOUD_NAME=your_cloud_name
CLOUDINARY_API_KEY=your_api_key
CLOUDINARY_API_SECRET=your_api_secret

# CORS
ALLOWED_ORIGINS=http://localhost:3000,https://your-staging.vercel.app
```

~~**tsconfig.json** (Modern Express config)~~
```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "node",
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "allowSyntheticDefaultImports": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "removeComments": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "**/*.test.ts", "dist"]
}
```

~~**package.json** (Modern scripts with tsx)~~~~
```json
{
    "name": "@stuflux/api",
    "version": "1.0.0",
    "description": "",
    "main": "index.js",
    "type": "module",
    "scripts": {
        "dev": "tsx watch src/index.ts",
        "build": "tsc",
        "start": "node dist/index.js",
        "test": "jest",
        "type-check": "tsc --noEmit",
        "lint": "eslint src/**/*.ts",
        "clean": "rm -rf dist"
    },
    "keywords": [],
    "author": "",
    "license": "ISC",
    "packageManager": "pnpm@8.15.0",
    "dependencies": {
        "express": "^5.1.0",
        "typescript": "^5.9.3"
    },
    "devDependencies": {
        "@types/express": "^5.0.5",
        "@types/node": "^20.19.24",
        "nodemon": "^3.1.10",
        "tsx": "^4.20.6"
    }
}

```

### 3. Project Structure Setup (2-3 hours)

#### ~~Complete Monorepo Structure (matches corrected plan)~~
```bash
# From project root, create the full structure
mkdir -p apps/web/src/{app,components,lib,types}
mkdir -p apps/api/src/{routes,middleware,utils}
mkdir -p packages/types

# Create workspace package for shared types
cd packages/types
pnpm init
echo '{
  "name": "@stuflux/types",
  "version": "1.0.0",
  "type": "module",
  "main": "index.ts",
  "exports": {
    ".": "./index.ts"
  }
}' > package.json

# Create initial shared types
echo "export interface User {
  id: string;
  clerk_id: string;
  email: string;
  name?: string;
}

export interface Listing {
  id: string;
  title: string;
  description: string;
  daily_rate: number;
  city: string;
}" > index.ts
```

#### ~~Root Package.json (Workspace management)~~
```json
{
  "name": "stuflux",
  "version": "1.0.0",
  "main": "index.js",
  
  "private": true,
  "type": "module",
  "engines": {
    "node": ">=18.0.0",
    "pnpm": ">=8.0.0"
  },
  "scripts": {
    "dev": "concurrently \"pnpm -F web dev\" \"pnpm -F api dev\"",
    "build": "pnpm -F web build && pnpm -F api build",
    "test": "pnpm -r test",
    "lint": "pnpm -r lint",
    "type-check": "pnpm -r type-check",
    "clean": "pnpm -r clean && rm -rf node_modules"
  },
  "devDependencies": {
    "concurrently": "^8.2.2",
    "@types/node": "^20.0.0",
    "typescript": "^5.3.0"
  },
  "packageManager": "pnpm@8.15.0",
  "keywords": [],
  "author": "",
  "license": "ISC",
  "description": ""
}

```

## 📋 Day 2 Tasks

### 1. Git Configuration & Branch Strategy (1-2 hours)

#### Frontend & Backend Repositories
Create the following files in both repositories:

~~**.gitignore**~~
```gitignore
# --------------------
# Dependencies
# --------------------
node_modules/
.pnp
.pnp.js
.pnp.*
.yarn/*
!.yarn/patches
!.yarn/plugins
!.yarn/releases
!.yarn/versions
.pnpm-debug.log*

# --------------------
# Testing
# --------------------
/coverage

# --------------------
# Production / Build
# --------------------
/build
/dist
/.next/
/out/

# --------------------
# Environment files
# --------------------
.env*
!.env.example

# --------------------
# Debug logs
# --------------------
npm-debug.log*
yarn-debug.log*
yarn-error.log*

# --------------------
# System files
# --------------------
.DS_Store
*.pem
*.swp
*.swo

# --------------------
# IDE / Editor
# --------------------
.idea/
.vscode/

# --------------------
# Vercel
# --------------------
.vercel

# --------------------
# TypeScript
# --------------------
*.tsbuildinfo
next-env.d.ts

```

Create branch protection rules in GitHub:[[Dev Guide]]
1. `main` - Requires pull request reviews
2. `develop` - Base branch for feature development
3. Feature branches pattern: `feature/*`

### 2. Essential Package Installation (2-3 hours)

#### ~~Frontend Packages (apps/web)~~
```bash
cd apps/web

# ✅ MODERN PACKAGES (2025 standards)
pnpm install @clerk/nextjs @clerk/themes
pnpm install sharp  # @next/bundle-analyzer only if needed

# Database client (for Server Components)
pnpm install pg @types/pg

# HTTP client for API calls
pnpm install ky  # Modern alternative to axios

# Form handling and validation
pnpm install react-hook-form zod @hookform/resolvers

# UI Components (shadcn/ui approach) - UPDATED
pnpm install @radix-ui/react-select @radix-ui/react-dialog @radix-ui/react-toast
pnpm install class-variance-authority clsx tailwind-merge
pnpm install lucide-react

# Image handling
pnpm install cloudinary browser-image-compression

# Development tools
pnpm install -D prettier prettier-plugin-tailwindcss
pnpm install -D @types/pg eslint-config-next
```

#### ~~Backend Packages (apps/api)~~
```bash
cd ../api

# ✅ MODERN EXPRESS STACK
pnpm install express@^5.1.0 cors helmet morgan compression
pnpm install @clerk/backend  # Correct Clerk package
pnpm install pg cloudinary
pnpm install zod dotenv  # Validation + env management

# Type definitions (Updated for Express 5.x)
pnpm install -D @types/express@^5.0.5 @types/cors @types/morgan @types/pg

# Modern development tools
pnpm install -D tsx nodemon prettier eslint @typescript-eslint/parser @typescript-eslint/eslint-plugin
pnpm install -D jest supertest @types/jest @types/supertest

# Remove deprecated packages:
# ❌ ts-node-dev (use tsx instead)
# ❌ ts-node (use tsx instead)
```

#### ~~Shared Dependencies (packages/types)~~
```bash
cd ../../packages/types
pnpm install -D typescript
```

### 2. Modern Configuration Files (2-3 hours)

#### Frontend Configuration (apps/web)

~~**next.config.js** (Security & Performance)~~
```javascript
/** @type {import('next').NextConfig} */
const nextConfig = {
  experimental: {
    // ✅ Allow native Node modules in Server Components (e.g., 'pg')
    serverComponentsExternalPackages: ['pg'],

    // ✅ Type-safe routing (Next.js 14+)
    typedRoutes: true,

    // ✅ Enable React Compiler (Next.js 15+)
    reactCompiler: true,

    // ✅ Optimize imports for popular libraries
    optimizePackageImports: ['lucide-react', 'clsx', 'react-hook-form'],
  },

  images: {
    // ✅ Whitelisted remote image sources
    remotePatterns: [
      { protocol: 'https', hostname: 'res.cloudinary.com' },
      { protocol: 'https', hostname: 'images.clerk.dev' },
      { protocol: 'https', hostname: 'avatars.githubusercontent.com' }, // Optional for GitHub avatars
    ],
    formats: ['image/avif', 'image/webp'], // ✅ Modern, efficient formats
  },

  compiler: {
    // ✅ Strip console logs only in production
    removeConsole: process.env.NODE_ENV === 'production',
  },

  poweredByHeader: false, // ✅ Hide “X-Powered-By: Next.js”
  reactStrictMode: true,  // ✅ Extra React checks in development

  async headers() {
    // ✅ Security headers applied globally
    return [
      {
        source: '/(.*)',
        headers: [
          { key: 'X-Frame-Options', value: 'DENY' },
          { key: 'X-Content-Type-Options', value: 'nosniff' },
          { key: 'Referrer-Policy', value: 'origin-when-cross-origin' },
          { key: 'Permissions-Policy', value: 'camera=(), microphone=(), geolocation=()' },
          { key: 'Cross-Origin-Opener-Policy', value: 'same-origin' },
          { key: 'Cross-Origin-Embedder-Policy', value: 'require-corp' },
        ],
      },
    ];
  },
};

export default nextConfig;


```

~~**prettier.config.js**~~
```javascript
/** @type {import('prettier').Config} */
export default {
  plugins: ['prettier-plugin-tailwindcss'],

  // ✅ Code style preferences
  semi: false, // No semicolons
  singleQuote: true, // Prefer single quotes
  tabWidth: 2, // 2 spaces per tab
  trailingComma: 'es5', // Add trailing commas where valid in ES5
  printWidth: 100, // Good balance between readability and width
  bracketSpacing: true, // Add spaces inside object braces
  arrowParens: 'avoid', // Avoid parentheses for single-arg arrow functions

  // ✅ Optional modern flags
  endOfLine: 'lf', // Consistent line endings (important for Linux)
  useTabs: false, // Always use spaces for indentation
}

```

~~**tailwind.config.js** (Modern design system - 2025)~~
```javascript
/** @type {import('tailwindcss').Config} */
module.exports = {
  darkMode: ["class"],
  content: [
    './src/**/*.{ts,tsx}',
  ],
  theme: {
    extend: {
      colors: {
        primary: {
          50: "#f0f9ff",
          500: "#3b82f6",
          600: "#2563eb",
          900: "#1e3a8a"
        }
      },
      fontFamily: {
        sans: ['var(--font-geist-sans)', 'system-ui', 'sans-serif']
      },
      animation: {
        "fade-in": "fadeIn 0.5s ease-in-out",
        "slide-up": "slideUp 0.3s ease-out"
      }
    },
  },
  plugins: [
    // Note: @tailwindcss/forms and @tailwindcss/typography are deprecated
    // Use native Tailwind utilities instead
  ],
}
```

#### Backend Configuration (apps/api)

~~**src/index.ts** (Express 5.x compatible)~~
```typescript
import express from 'express';
import cors from 'cors';
import helmet from 'helmet';
import morgan from 'morgan';
import compression from 'compression';
import dotenv from 'dotenv';

// Load environment variables
dotenv.config();

const app = express();
const PORT = process.env.PORT || 4000;

// Security middleware
app.use(helmet({
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      styleSrc: ["'self'", "'unsafe-inline'"],
      scriptSrc: ["'self'"],
      imgSrc: ["'self'", "data:", "https:"],
    }
  }
}));

app.use(cors({
  origin: process.env.ALLOWED_ORIGINS?.split(',') || ['http://localhost:3000'],
  credentials: true
}));

// Parsing & compression
app.use(compression());
app.use(express.json({ limit: '10mb' }));
app.use(express.urlencoded({ extended: true }));
app.use(morgan('combined'));

// Health check
app.get('/api/v1/health', (req, res) => {
  res.json({ 
    status: 'ok', 
    timestamp: new Date().toISOString(),
    version: process.env.npm_package_version || '1.0.0'
  });
});

// 404 handler
app.use('*', (req, res) => {
  res.status(404).json({ error: 'Route not found' });
});

// Error handler
app.use((err: Error, req: express.Request, res: express.Response, next: express.NextFunction) => {
  console.error(err.stack);
  res.status(500).json({ error: 'Internal server error' });
});

app.listen(PORT, () => {
  console.log(`🚀 Server running on http://localhost:${PORT}`);
});

export default app;
```

**src/lib/db.ts** (Modern PostgreSQL setup)
```typescript
import { Pool } from 'pg';

const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
  ssl: process.env.NODE_ENV === 'production' ? { rejectUnauthorized: false } : false,
  max: 20,
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 2000,
});

export default pool;
```

## ✅ End of Day 2 Checklist

### Monorepo Structure
- [ ] Monorepo initialized with `pnpm-workspace.yaml`
- [ ] Next.js app created in `apps/web`
- [ ] Express API created in `apps/api`
- [ ] Shared types package in `packages/types`
- [ ] Root scripts configured for development

### Frontend (apps/web)
- [ ] Next.js 14 project with App Router
- [ ] TypeScript strict mode enabled
- [ ] Modern packages installed (Clerk, Radix UI, etc.)
- [ ] Environment variables configured
- [ ] Development server runs: `pnpm -F web dev`

### Backend (apps/api)
- [ ] Express with TypeScript and modern tooling
- [ ] Security middleware configured (helmet, cors)
- [ ] Database connection ready for Neon
- [ ] Health check endpoint working
- [ ] Development server runs: `pnpm -F api dev`

### Development Workflow
- [ ] Both apps run together: `pnpm dev`
- [ ] Git repository initialized with proper .gitignore
- [ ] Package management works across workspace
- [ ] No deprecated or problematic dependencies

## 🚨 Issues Fixed from Original Guide

### ❌ **Problems in Original:**
1. **Separate repos** instead of monorepo → Fixed with workspace structure
2. **`ts-node-dev`** (deprecated) → Replaced with `tsx`
3. **`axios`** (heavy) → Replaced with `ky` (modern, lighter)
4. **Missing Clerk integration** → Added proper packages
5. **Wrong port numbers** → Standardized (3000/4000)
6. **No shared types** → Added workspace package
7. **Missing database setup** → Added PostgreSQL configuration
8. **Old Tailwind config** → Modern design system approach
9. **Package manager inconsistency** → Standardized on pnpm
10. **Deprecated Tailwind plugins** → Removed @tailwindcss/forms and @tailwindcss/typography

### ✅ **Modern Standards Applied (2025):**
- **pnpm workspaces** for monorepo management
- **tsx** for fast TypeScript execution
- **@clerk/nextjs** + **@clerk/backend** for auth
- **ky** instead of axios for HTTP
- **Sharp** for Next.js image optimization
- **Compression** middleware for Express
- **Typed routes** in Next.js 16
- **Security headers** and helmet configuration
- **ESNext modules** with "type": "module"
- **Node 18+** requirement specified
- **Enhanced CSP** security policies
- **Better error handling** and logging

### ⚠️ **EXPRESS 5.x COMPATIBILITY NOTE:**
- **Express 5.1.0**: Latest major version with breaking changes from v4
- **Enhanced async/await**: Better error handling for async middleware
- **Breaking changes**: Some middleware might need updates
- **Type definitions**: Using `@types/express@^5.0.5` for compatibility
- **Monitor**: Test all middleware during Day 3 setup

## 🎯 Next Steps (Day 3)
1. Set up Clerk Authentication (corrected implementation)
2. Configure Neon database connection
3. Create first API endpoint with proper JWT verification
4. Set up deployment to Vercel + Railway


