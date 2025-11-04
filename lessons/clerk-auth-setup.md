# Clerk Authentication Setup: Next.js & Express Backend

## Overview
This lesson explains how to set up authentication using Clerk in a modern Next.js frontend and Express backend. You'll learn the flow, code structure, and how each part works.

---

## 1. Next.js Frontend Setup

### a. Install Clerk
```bash
pnpm add @clerk/nextjs
```

### b. Configure Environment Variables
Add these to `.env.local`:
```
NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=pk_test_...
CLERK_SECRET_KEY=sk_test_...
NEXT_PUBLIC_CLERK_SIGN_IN_URL=/sign-in
NEXT_PUBLIC_CLERK_SIGN_UP_URL=/sign-up
NEXT_PUBLIC_CLERK_AFTER_SIGN_IN_URL=/dashboard
NEXT_PUBLIC_CLERK_AFTER_SIGN_UP_URL=/dashboard
```

### c. Wrap App with ClerkProvider
In `src/app/layout.tsx`:
```tsx
import { ClerkProvider } from '@clerk/nextjs';

export default function RootLayout({ children }) {
  return (
    <ClerkProvider publishableKey={process.env.NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY!}>
      <html lang="en">
        <body>{children}</body>
      </html>
    </ClerkProvider>
  );
}
```
**How it works:**
- Makes Clerk authentication available throughout your app.

### d. Add Sign-In & Sign-Up Pages
`src/app/sign-in/page.tsx`:
```tsx
import { SignIn } from '@clerk/nextjs';
export default function SignInPage() { return <SignIn />; }
```
`src/app/sign-up/page.tsx`:
```tsx
import { SignUp } from '@clerk/nextjs';
export default function SignUpPage() { return <SignUp />; }
```
**How it works:**
- Provides ready-made authentication UI.

### e. Protect Routes (Middleware)
`middleware.ts`:
```ts
import { clerkMiddleware } from '@clerk/nextjs/server';
export default clerkMiddleware();
export const config = { matcher: ['/dashboard/:path*'] };
```
**How it works:**
- Blocks unauthenticated users from protected routes.

### f. Use Auth in Components
```tsx
import { useAuth } from '@clerk/nextjs';
const { userId, isSignedIn } = useAuth();
```
**How it works:**
- Access user info and auth state in your React components.

### g. Authenticated API Calls
`src/lib/api-client.ts`:
```ts
import ky from 'ky';
import { useAuth } from '@clerk/nextjs';
const apiClient = ky.create({ prefixUrl: process.env.NEXT_PUBLIC_API_URL });
export function useApiClient() {
  const { getToken } = useAuth();
  return apiClient.extend({
    hooks: { beforeRequest: [async req => {
      const token = await getToken();
      if (token) req.headers.set('Authorization', `Bearer ${token}`);
    }] }
  });
}
```
**How it works:**
- Sends Clerk JWT in API requests for backend authentication.

---

## 2. Express Backend Setup

### a. Install Clerk Backend SDK
```bash
pnpm add @clerk/backend
```

### b. Configure Environment Variables
Add to `.env`:
```
CLERK_SECRET_KEY=sk_test_...
```

### c. Auth Middleware
`src/middleware/auth.ts`:
```ts
import { verifyToken } from '@clerk/backend';
export async function requireAuth(req, res, next) {
  const authHeader = req.headers.authorization;
  if (!authHeader?.startsWith('Bearer ')) return res.status(401).json({ error: 'No token' });
  const token = authHeader.substring(7);
  try {
    const payload = await verifyToken(token, { secretKey: process.env.CLERK_SECRET_KEY });
    req.auth = { userId: payload.sub };
    next();
  } catch {
    res.status(401).json({ error: 'Invalid token' });
  }
}
```
**How it works:**
- Verifies JWT from frontend, attaches user info to request.

### d. Protect API Routes
`src/routes/listings.ts`:
```ts
import { requireAuth } from '../middleware/auth.js';
router.post('/', requireAuth, async (req, res) => {
  // Only authenticated users can create listings
});
```
**How it works:**
- Only allows access if user is authenticated.

### e. User Sync via Webhook (Optional)
`src/routes/webhooks.ts`:
```ts
router.post('/clerk/user', async (req, res) => {
  // Save Clerk user to DB
});
```
**How it works:**
- Keeps your DB in sync with Clerk users.

---

## 3. Authentication Flow
1. User signs in/up via Clerk UI in Next.js.
2. Clerk issues JWT, stored in browser.
3. Frontend sends JWT in `Authorization` header for API requests.
4. Backend verifies JWT, attaches user info to request.
5. Protected routes only accessible if JWT is valid.
6. Optionally, Clerk webhooks keep user DB in sync.

---

## 4. Summary Table
| Step                | Next.js Frontend                | Express Backend           |
|---------------------|---------------------------------|--------------------------|
| Auth UI             | ClerkProvider, SignIn/SignUp     | N/A                      |
| Route Protection    | Middleware, useAuth             | requireAuth middleware   |
| API Auth            | JWT in Authorization header      | verifyToken              |
| User Sync           | N/A (webhook triggers)           | /webhooks/clerk/user     |

---

## 5. Best Practices
- Always protect sensitive routes with middleware.
- Never expose your Clerk secret key in frontend code.
- Use Clerk webhooks for user sync if you need user data in your DB.
- Test authentication flow end-to-end.

---

## 6. References
- [Clerk Next.js Docs](https://clerk.com/docs/nextjs)
- [Clerk Backend Docs](https://clerk.com/docs/backend)

---

**This lesson covers the complete authentication setup and flow for a modern Next.js + Express app using Clerk.**
