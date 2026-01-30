Here’s a focused test plan to cover messaging + delivery/seen:

### 1) Unit-ish (service/repo) with Jest + supertest not required
- `sendMessage`:
  - Inserts message; returns payload
  - Marks delivered when `isUserConnected` returns true (mock to true/false)
- `markConversationSeen`:
  - Marks only messages not sent by caller; sets `read_at`
- `repository`:
  - `markDelivered` sets `delivered_at`
  - `getUndeliveredMessages` returns only messages with `delivered_at IS NULL`

### 2) HTTP Integration (supertest)
Tests run against express app (no SSE yet):
- **Send**: `POST /messages` → 201, body matches input, `delivered_at` null when no stream.
- **List messages**: `GET /conversations/:id/messages` → returns newest-first, marks read_at for caller.
- **Seen**: `POST /conversations/:id/seen` → read_at set for messages from other user; not for caller’s own.
- **Auth guard**: 401/403 on unauthorized or foreign conversation.

### 3) SSE Flow (integration)
Use `eventsource` or `@microsoft/fetch-event-source` in tests, or manual fetch with `node-fetch` + `eventsource` polyfill.
- **Delivery on connect**:
  1) User B opens `/conversations/:id/stream` (keep connection open).
  2) User A sends message → expect SSE event; DB should have `delivered_at` set for that message.
- **Retroactive delivery**:
  1) User A sends message while B offline.
  2) B opens stream → DB updates `delivered_at` for undelivered messages; SSE notifies only new ones (optional assert via repo).
- **Delete broadcast**:
  1) Send, then DELETE `/messages/:id` as sender.
  2) Expect SSE event with `{ deleted: true }`.

### 4) Suggested test cases (HTTP)
- `POST /messages` (new conversation via listing_id) → creates convo, message, delivered_at null.
- `POST /messages` (existing conversation) → delivered_at set if `isUserConnected` mocked true.
- `GET /conversations` → includes `unread_count`, `last_message`.
- `GET /conversations/:id/messages?limit=2&offset=0` → pagination + marks read_at.
- `POST /conversations/:id/seen` → read_at updated; idempotent.
- `DELETE /messages/:id` → soft delete; 404 if not owner.

### 5) Minimal tooling setup
- Use existing Jest config in apps/api (supertest for HTTP).
- For SSE tests, add dev deps: `eventsource` (or `@microsoft/fetch-event-source`).
- Spin app with `app` export (no need to start server): `await request(app).get(...)`.

### 6) Example SSE test sketch (pseudo)
```typescript
import EventSource from 'eventsource';
import request from 'supertest';
import app from '../../src/main/app';

test('marks delivered when receiver online', async () => {
  const convoId = '...'; // seeded or created via POST /messages
  const receiverToken = '...'; // mock auth header

  const es = new EventSource(`${base}/api/v1/conversations/${convoId}/stream`, {
    headers: { Authorization: `Bearer ${receiverToken}` },
  });

  const received = new Promise((resolve, reject) => {
    es.onmessage = (e) => resolve(JSON.parse(e.data));
    es.onerror = reject;
  });

  await request(app)
    .post('/api/v1/messages')
    .set('Authorization', `Bearer ${senderToken}`)
    .send({ conversation_id: convoId, body: 'hi' })
    .expect(201);

  const event = await received;
  expect(event.body).toBe('hi');

  // Optionally, query DB to assert delivered_at is not null
});
```

### 7) Prioritize for MVP
1) HTTP integration (send/list/seen/delete) — fastest coverage.
2) One happy-path SSE test (online delivery) — proves real-time + delivered_at.
3) Optional retroactive delivery test.

If you want, I can add a test suite scaffold with one HTTP spec and one SSE spec to get you started.