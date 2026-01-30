import request from 'supertest';

import EventSource from 'eventsource';

import { jest, describe, beforeAll, afterAll, beforeEach, it, expect } from '@jest/globals';

import { authHeader } from '../helpers/auth.js';

import { resetDb, createUser, createCategory, createListing, createConversation, createMessage, findMessages, findMessageById } from '../helpers/db.js';

  

// Mock Clerk + user sync to avoid network calls

jest.unstable_mockModule('@clerk/backend', () => ({

verifyToken: jest.fn(async (token: string) => ({

sub: token,

sid: `${token}-sid`,

})),

createClerkClient: jest.fn(() => ({

users: { getUser: jest.fn() },

})),

}));

  

jest.unstable_mockModule('../../src/modules/users/service.js', () => ({

ensureUserSynced: jest.fn(),

getProfile: jest.fn(async () => null),

updateProfile: jest.fn(async () => null),

}));

  

// Import app after mocks are registered

const { default: app } = await import('../../src/main/app.js');

const { closePool } = await import('../../db/index.js');

  

// Allow global cleanup for server/pool to avoid open handles

let globalServer: any;

  

describe('Messaging HTTP flows', () => {

let categoryId: number;

let ownerId: string;

let renterId: string;

let listingId: string;

  

beforeAll(async () => {

await resetDb();

});

  

beforeEach(async () => {

await resetDb();

const category = await createCategory();

categoryId = category.id;

const owner = await createUser({ display_name: 'Owner' });

const renter = await createUser({ display_name: 'Renter' });

ownerId = owner.id;

renterId = renter.id;

const listing = await createListing(ownerId, categoryId);

listingId = listing.id;

});

  

it('creates a message for a new conversation', async () => {

const res = await request(app)

.post('/api/v1/messages')

.set(authHeader(renterId))

.send({ listing_id: listingId, body: 'Hi there' });

  

expect(res.status).toBe(201);

expect(res.body.data).toMatchObject({ body: 'Hi there', sender_id: renterId, conversation_id: expect.any(String) });

  

const convoRes = await request(app)

.get('/api/v1/conversations')

.set(authHeader(ownerId));

  

expect(convoRes.status).toBe(200);

const [convo] = convoRes.body.data;

expect(convo).toMatchObject({ listing_id: listingId, renter_id: renterId, owner_id: ownerId });

expect(convo.last_message?.body).toBe('Hi there');

expect(convo.unread_count).toBe(1);

});

  

it('leaves delivered_at null when receiver is offline', async () => {

const res = await request(app)

.post('/api/v1/messages')

.set(authHeader(renterId))

.send({ listing_id: listingId, body: 'Hi there' });

  

expect(res.status).toBe(201);

const messageId = res.body.data.id;

  

const dbMsg = await findMessageById(messageId);

expect(dbMsg?.delivered_at).toBeNull();

});

  

it('lists messages and marks them read for the viewer', async () => {

const convo = await createConversation(listingId, renterId, ownerId);

await createMessage(convo.id, renterId, 'First');

await createMessage(convo.id, renterId, 'Second');

  

const res = await request(app)

.get(`/api/v1/conversations/${convo.id}/messages`)

.set(authHeader(ownerId));

  

expect(res.status).toBe(200);

expect(res.body.data).toHaveLength(2);

  

const dbMsgs = await findMessages(convo.id);

dbMsgs

.filter((m) => m.sender_id !== ownerId)

.forEach((m) => {

expect(m.read_at).not.toBeNull();

});

});

  

it('marks conversation seen explicitly', async () => {

const convo = await createConversation(listingId, renterId, ownerId);

await createMessage(convo.id, renterId, 'Ping');

  

const res = await request(app)

.post(`/api/v1/conversations/${convo.id}/seen`)

.set(authHeader(ownerId));

  

expect(res.status).toBe(200);

expect(res.body.data).toMatchObject({ conversation_id: convo.id, status: 'seen' });

  

const msgs = await findMessages(convo.id);

expect(msgs.every((m) => m.read_at !== null || m.sender_id === ownerId)).toBe(true);

});

  

it('soft deletes own message and hides it from fetches', async () => {

const convo = await createConversation(listingId, renterId, ownerId);

const msg = await createMessage(convo.id, renterId, 'To delete');

  

const delRes = await request(app)

.delete(`/api/v1/messages/${msg.id}`)

.set(authHeader(renterId));

  

expect(delRes.status).toBe(200);

expect(delRes.body.data.deleted_at).not.toBeNull();

  

const listRes = await request(app)

.get(`/api/v1/conversations/${convo.id}/messages`)

.set(authHeader(ownerId));

  

expect(listRes.status).toBe(200);

const bodies = listRes.body.data.map((m: any) => m.body);

expect(bodies).not.toContain('To delete');

});

  

it('forbids non-participants from accessing messages', async () => {

const convo = await createConversation(listingId, renterId, ownerId);

await createMessage(convo.id, renterId, 'Secret');

const outsider = (await createUser({ display_name: 'Intruder' })).id;

  

const res = await request(app)

.get(`/api/v1/conversations/${convo.id}/messages`)

.set(authHeader(outsider));

  

expect(res.status).toBe(403);

});

  

});

  

describe('Messaging SSE flows', () => {

let baseUrl: string;

let categoryId: number;

let ownerId: string;

let renterId: string;

let listingId: string;

let conversationId: string;

  

beforeAll(async () => {

await resetDb();

globalServer = app.listen(0);

const address = globalServer.address();

baseUrl = `http://127.0.0.1:${address.port}`;

});

  

beforeEach(async () => {

await resetDb();

const category = await createCategory();

categoryId = category.id;

const owner = await createUser({ display_name: 'Owner' });

const renter = await createUser({ display_name: 'Renter' });

ownerId = owner.id;

renterId = renter.id;

const listing = await createListing(ownerId, categoryId);

listingId = listing.id;

const convo = await createConversation(listingId, renterId, ownerId);

conversationId = convo.id;

});

  

afterAll(async () => {

await closePool();

globalServer?.close();

});

  

it('pushes new messages over SSE to connected participant', async () => {

const stream = new EventSource(`${baseUrl}/api/v1/conversations/${conversationId}/stream`, {

headers: authHeader(ownerId),

});

  

const received = new Promise((resolve, reject) => {

const timer = setTimeout(() => reject(new Error('No SSE payload received')), 5000);

stream.onmessage = (event) => {

clearTimeout(timer);

stream.close();

resolve(JSON.parse(event.data));

};

stream.onerror = (err) => {

clearTimeout(timer);

stream.close();

reject(err);

};

});

  

await request(baseUrl)

.post('/api/v1/messages')

.set(authHeader(renterId))

.send({ conversation_id: conversationId, body: 'Live hello' });

  

const payload: any = await received;

expect(payload.body).toBe('Live hello');

expect(payload.sender_id).toBe(renterId);

expect(payload.conversation_id).toBe(conversationId);

expect(payload.delivered_at).toBeTruthy();

  

const dbMsg = await findMessageById(payload.id);

expect(dbMsg?.delivered_at).not.toBeNull();

});

  

it('retroactively marks delivered_at when receiver connects after messages were sent', async () => {

const msg = await createMessage(conversationId, renterId, 'Offline hello');

  

const stream = new EventSource(`${baseUrl}/api/v1/conversations/${conversationId}/stream`, {

headers: authHeader(ownerId),

});

  

await new Promise((resolve, reject) => {

const timer = setTimeout(() => reject(new Error('No connect')), 3000);

stream.onopen = () => {

clearTimeout(timer);

resolve(null);

};

stream.onerror = reject;

});

  

const dbMsg = await findMessageById(msg.id);

expect(dbMsg?.delivered_at).not.toBeNull();

  

stream.close();

});

});

Here’s a pragmatic testing plan for the API backend, grounded in what’s in the repo and the project spec.

- Current state: Jest + ts-jest ESM setup and Supertest wiring are ready; see package.json and apps/api/jest.config.ts. Global setup only stubs console; no DB orchestration. Messaging tests are just TODO scaffolds (apps/api/tests/integration/messages.spec.ts). Auth middleware verifies Clerk tokens and calls ensureUserSynced (apps/api/src/infra/http/middleware/auth.ts), so external calls must be avoided in tests.

- Testing levels to target:
  - Integration-first for HTTP routes with Supertest against a real Postgres instance (best fidelity for Drizzle and SQL constraints).
  - Light service-level tests only where logic is branchy (e.g., messaging service validation/permission paths).
  - Contract tests between API and Next.js are optional later; for now, keep response shapes asserted in integration tests.

- Database strategy (decide one): [[testing db strategies]]
- 
  1) Dedicated test database URL + run migrations before the suite; TRUNCATE tables between tests.
  2) Same, but wrap each test in BEGIN/ROLLBACK using the Pool (faster, but requires per-test transaction hooks).
  3) Dockerized Postgres via Testcontainers (not yet installed; adds weight).
  Default recommendation: option 1 for simplicity; add a tests/helpers/db.ts to run drizzle migrations (or raw SQL) and TRUNCATE users, listings, conversations, messages, bookings beforeEach.

- Auth strategy:![[Mocking]]
  - `jest.mock('@clerk/backend', () => ({ verifyToken: () => ({ sub: 'user-1', sid: 'sid', ...claims }) }))` and `jest.mock('../../modules/users/service', () => ({ ensureUserSynced: jest.fn() }))` to avoid network.
  - Provide a helper that builds Authorization headers for arbitrary user IDs so you can test owner/renter roles.

- Test data helpers:[[test data helpers]]
  - Factories using the shared db client to insert users, listings, conversations, messages with sensible defaults.
  - Seed categories once per suite if routes depend on them (there are seeds in db/seeds).

- Messaging test focus (per spec and existing code):
  - POST /api/v1/messages with listing_id creates a conversation and message; 201 with message payload; conversation participants set to renter=sender, owner=listing.owner.
  - POST /api/v1/messages with conversation_id appends; forbids non-participants.
  - GET /api/v1/conversations returns last_message and unread_count per conversation.
  - GET /api/v1/conversations/:id/messages respects limit/offset and sets read_at for messages not sent by the requester.
  - DELETE /api/v1/messages/:id soft-deletes own message; ensure deleted_at set and deleted messages are hidden in subsequent fetches (current repository does not filter deleted messages—either adjust expectations or add a filter before asserting).
  - SSE: spin up app.listen(0) in the test, use EventSource (eventsource package) to connect to /api/v1/conversations/:id/stream, send a message as the other user, and assert an event arrives. Add a cleanup hook to close the EventSource and server.
  - Observed gaps to resolve before/while testing: delivered_at is never set anywhere; there is no POST /conversations/:id/seen route in router/controller; deleted messages are not filtered in listMessages. Decide whether to (a) change code to match spec, or (b) adjust test expectations.

- Non-messaging coverage to add soon (aligned with MVP spec):
  - Health endpoints (already partially covered) and DB health.
  - Listings CRUD with auth + validation (Zod) and owner permissions.
  - Bookings optimistic flow: create pending, owner confirm/reject, date blocking checks.
  - Categories read-only list.
  - Uploads/webhooks likely need stubbing/mocking of external services.

- Tooling tweaks:
  - Extend setup.ts to: set NODE_ENV=test defaults, load .env.test, start a DB clean routine, and register global helpers (e.g., makeAuthHeader(userId)).
  - Add a jest global teardown to close the pg Pool (closePool exists in apps/api/db/index.js referenced by the older integration test).

Decisions to confirm before implementing:
1) Which DB strategy (truncate vs transaction vs Testcontainers)?
2) Do we patch the messaging code for delivered_at, deleted filtering, and “seen” route now, or shape tests to current behavior?
3) Are we okay mocking Clerk entirely, or do we want a “real-ish” JWT verification path with a local secret?

If you confirm those, I can sketch the concrete test harness (helpers, mocks, and first messaging integration tests) next.