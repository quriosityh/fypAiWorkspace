Alright, this is about **not writing garbage test setup over and over**. Test data helpers are how you keep tests readable, fast, and not cringe.

Let’s break it down.

---

## 1️⃣ Factories (the backbone of sane tests)

### What a factory is

A **factory** is just a function that:

* Inserts a valid row into the DB
* Uses **sensible defaults**
* Lets you override only what you care about

Instead of this mess 👇

```ts
await db.insert(users).values({
  email: 'a@b.com',
  passwordHash: 'x',
  role: 'USER',
  createdAt: new Date(),
  ...
});
```

You do this 👇

```ts
const user = await createUser();
```

### Why this matters

✅ Tests stay short and readable
✅ Schema changes = update factory once
✅ No copy-paste garbage
✅ You focus on **behavior**, not setup

---

## 2️⃣ Shared DB client (non-negotiable)

Factories must use:

* **The same DB client / pool**
* The same connection your app uses in tests

Why?

* Transactions (if you add them later)
* Predictable cleanup
* No phantom data issues

🚨 Never create a new DB connection inside a factory.

---

## 3️⃣ Sensible defaults (this is an art)

Your defaults should be:

* **Valid**
* **Minimal**
* **Production-realistic**

Example:

```ts
export async function createUser(overrides = {}) {
  const [user] = await db.insert(users).values({
    email: faker.internet.email(),
    role: 'USER',
    isVerified: true,
    ...overrides,
  }).returning();

  return user;
}
```

Good defaults:

* Verified user
* Normal role
* No edge cases unless you ask for them

Edge cases belong **in the test**, not the factory.

---

## 4️⃣ Composable factories (power move)

Factories should build on each other.

```ts
const user = await createUser();
const listing = await createListing({ ownerId: user.id });
const conversation = await createConversation({
  buyerId: user.id,
  listingId: listing.id,
});
```

This mirrors real data relationships and makes intent obvious.

---

## 5️⃣ Factories vs fixtures (why factories win)

| Fixtures               | Factories        |
| ---------------------- | ---------------- |
| Static                 | Dynamic          |
| Brittle                | Flexible         |
| Hard to reuse          | Easy to compose  |
| Painful schema updates | One update, done |

Fixtures die fast. Factories scale.

---

## 6️⃣ Seeding lookup data once per suite

### When seeds make sense

Some tables:

* `categories`
* `roles`
* `statuses`
* `countries`

These:

* Rarely change
* Are read-only
* Needed by many routes

👉 Seed them **once per test suite**, not before every test.

### How to do it

```ts
beforeAll(async () => {
  await runSeeds(); // db/seeds
});
```

### Why not truncate these?

* Slower
* Risky
* Tests shouldn’t mutate lookup tables anyway

Rule:

> **Factories = mutable data**
> **Seeds = immutable reference data**

---

## 7️⃣ How this fits with TRUNCATE strategy

With option 1 (TRUNCATE between tests):

* You **do truncate**:

  * users
  * listings
  * conversations
  * messages
  * bookings

* You **do NOT truncate**:

  * categories
  * static lookup tables

That’s clean separation of concerns.

---

## 8️⃣ Folder structure (clean & obvious)

```txt
tests/
  helpers/
    db.ts          // truncate + migrations
    factories/
      user.ts
      listing.ts
      conversation.ts
      message.ts
    seeds.ts       // calls db/seeds
```

If a new dev opens this, they instantly get it. That’s the goal.

---

## Bottom line (no fluff)

* Factories make tests readable
* Defaults prevent setup fatigue
* Overrides give precision
* Seeds handle shared reference data
* TRUNCATE keeps state clean

If your tests read like a story instead of a SQL script,
**you’re doing it right.**
