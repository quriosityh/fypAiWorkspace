**What this file does**  
service.ts defines the handler for Clerk webhooks. It listens for events from Clerk (like `user.created` or `user.updated`) and then syncs the corresponding user into your local `users` table by calling `ensureUserSynced`.

**How it’s called (scenario)**  
- You configure Clerk to send webhooks to your backend endpoint `/api/v1/webhooks/clerk` (mounted in the router).  
- When a user signs up or updates their profile in Clerk, Clerk sends a POST to that endpoint with the event payload.  
- This handler reads `event.type` and `event.data.id` (Clerk user id); if the type is `user.created` or `user.updated`, it upserts the user locally so your DB stays in sync with Clerk’s email/avatar/name.  
- It replies `200 { received: true }` after processing.

**Note**  
It currently assumes a trusted environment; signature verification (Svix) should be added to validate that the request actually comes from Clerk before trusting the payload. Also ensure the route receives the raw body if required for signature verification.