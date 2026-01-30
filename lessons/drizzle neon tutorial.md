---
share_link: https://share.note.sx/qm8b5738#JNvTBa7uBrUHLNqTbHoKu2OR/9frUZWaf4lfYE0m9mw
share_updated: 2025-12-17T11:46:25+05:00
---
# Learn TypeScript Fundamentals for Drizzle ORM

I'll teach you TypeScript fundamentally and practically, focusing on exactly what you need to work with Drizzle and Neon DB effectively.

---

## Part 1: TypeScript Essentials (The Bare Minimum)

### 1.1 Basic Types - Your Building Blocks

TypeScript adds **types** to JavaScript. Think of types as labels that tell you what kind of data you're working with.

```typescript
// Basic types - these are your foundation
let name: string = "Ahmed";           // Text
let age: number = 25;                 // Numbers (whole or decimal)
let isActive: boolean = true;         // true/false
let nothing: null = null;             // Explicitly nothing
let notSet: undefined = undefined;    // Not yet defined

// Arrays - lists of items
let names: string[] = ["Ali", "Sara", "Hassan"];
let numbers: number[] = [1, 2, 3, 4];

// When you don't know/care about the type (avoid when possible)
let anything: any = "could be anything";
```

**Key Insight:** TypeScript prevents bugs by catching mistakes *before* your code runs. If you try to assign a number to a string variable, TypeScript will yell at you.

---

### 1.2 Interfaces & Types - Defining Shapes of Data

This is **crucial** for Drizzle. You'll use these to define what your database rows look like.

```typescript
// Interface - defines the "shape" of an object
interface User {
  id: number;
  name: string;
  email: string;
  age: number;
  isActive: boolean;
}

// Now you can create objects that match this shape
const user: User = {
  id: 1,
  name: "Ahmed",
  email: "ahmed@example.com",
  age: 25,
  isActive: true
};

// This would ERROR - missing properties
// const badUser: User = {
//   id: 1,
//   name: "Ahmed"
// };
```

**Type vs Interface:** They're 99% the same. Use either. Drizzle uses `type` more often:

```typescript
// Type - does the same thing
type Product = {
  id: number;
  name: string;
  price: number;
  inStock: boolean;
};

const laptop: Product = {
  id: 101,
  name: "MacBook Pro",
  price: 2500,
  inStock: true
};
```

**Optional Properties:**

```typescript
interface User {
  id: number;
  name: string;
  email: string;
  phone?: string;  // ← The ? means this is optional
}

// This is valid - phone is optional
const user1: User = {
  id: 1,
  name: "Ahmed",
  email: "ahmed@example.com"
};

// This is also valid
const user2: User = {
  id: 2,
  name: "Sara",
  email: "sara@example.com",
  phone: "+92-300-1234567"
};
```

---

### 1.3 Generics - The Magic Boxes (You Don't Write Them, Just Use Them)

Generics are like **template variables for types**. You'll see them in Drizzle but won't write them yourself.

```typescript
// Think of Array<T> as a "box that holds type T"
const numbers: Array<number> = [1, 2, 3];     // Box of numbers
const names: Array<string> = ["Ali", "Sara"];  // Box of strings

// This is the same as:
const numbers2: number[] = [1, 2, 3];

// In Drizzle, you'll see things like:
// Promise<User[]>  ← A promise that will give you an array of Users
// QueryResult<Product>  ← A query result containing a Product
```

**You just need to recognize the pattern:** `SomeThing<TheTypeInside>`

---

### 1.4 Let Your Editor Do The Work (Autocomplete is King)

TypeScript's superpower is **autocomplete**. Once you define types, your editor becomes psychic.

```typescript
interface User {
  id: number;
  name: string;
  email: string;
  createdAt: Date;
}

const user: User = {
  id: 1,
  name: "Ahmed",
  email: "ahmed@example.com",
  createdAt: new Date()
};

// Type "user." and watch the magic:
user.  // ← Your editor shows: id, name, email, createdAt
user.name.  // ← Your editor shows string methods: toLowerCase(), toUpperCase(), etc.
```

**Pro Tip:** If you're stuck, type the variable name, add a dot `.`, and let autocomplete guide you.

---

## Part 2: Drizzle ORM Fundamentals with Neon

Now let's apply TypeScript to Drizzle.

---

### 2.1 How Drizzle Maps TypeScript to SQL

Drizzle lets you define your database schema in TypeScript, and it converts it to SQL automatically.

**The Pattern:**
```
TypeScript Schema Definition → Drizzle → SQL Migration → Neon Database
```

---

### 2.2 Defining Tables and Columns

Here's how you define a table in Drizzle:

```typescript
// schema.ts
import { pgTable, serial, varchar, integer, boolean, timestamp } from 'drizzle-orm/pg-core';

// Define a "users" table
export const users = pgTable('users', {
  // Columns:
  id: serial('id').primaryKey(),              // Auto-incrementing ID
  name: varchar('name', { length: 100 }).notNull(),  // String, max 100 chars, required
  email: varchar('email', { length: 255 }).notNull().unique(),  // Unique email
  age: integer('age'),                        // Optional number
  isActive: boolean('is_active').default(true),  // Boolean with default
  createdAt: timestamp('created_at').defaultNow(),  // Auto timestamp
});

// Define a "products" table
export const products = pgTable('products', {
  id: serial('id').primaryKey(),
  name: varchar('name', { length: 200 }).notNull(),
  price: integer('price').notNull(),  // Price in cents (e.g., $25.00 = 2500)
  inStock: boolean('in_stock').default(true),
  createdAt: timestamp('created_at').defaultNow(),
});
```

**Key Column Types:**
- `serial('id')` - Auto-incrementing integer (perfect for IDs)
- `varchar('name', { length: X })` - Text with max length
- `integer('age')` - Whole numbers
- `boolean('is_active')` - true/false
- `timestamp('created_at')` - Date and time
- `.notNull()` - This field is required
- `.unique()` - No duplicates allowed
- `.default(value)` - Default value if not provided
>**Extra Important Notes**

- The keys (`id`, `name`, `email`, etc.) are **your TypeScript field names**.
    
- The strings (`'id'`, `'name'`, `'email'`) inside the functions are **actual SQL column names**.
    

This gives you the best of both worlds:  **clean TypeScript code and precise SQL structure**.

### 2.3 Creating Relationships (Foreign Keys)

Connect tables together:

```typescript
import { pgTable, serial, varchar, integer, timestamp } from 'drizzle-orm/pg-core';

// Users table
export const users = pgTable('users', {
  id: serial('id').primaryKey(),
  name: varchar('name', { length: 100 }).notNull(),
  email: varchar('email', { length: 255 }).notNull(),
});

// Posts table with a relationship to users
export const posts = pgTable('posts', {
  id: serial('id').primaryKey(),
  title: varchar('title', { length: 200 }).notNull(),
  content: varchar('content', { length: 5000 }),
  authorId: integer('author_id').references(() => users.id),  // ← Foreign key
  createdAt: timestamp('created_at').defaultNow(),
});

// This creates a relationship:
// posts.authorId → users.id
```

**What `.references()` does:** It creates a foreign key constraint. A post's `authorId` must be a valid `id` from the `users` table.

---

### 2.4 TypeScript Types from Schema (The Magic Part)

Drizzle **automatically generates TypeScript types** from your schema:

```typescript
import { users, posts } from './schema';

// Drizzle gives you these types for FREE:
type User = typeof users.$inferSelect;     // What you GET from DB
type NewUser = typeof users.$inferInsert;  // What you SEND to DB

type Post = typeof posts.$inferSelect;
type NewPost = typeof posts.$inferInsert;

// Now you can use them:
const user: User = {
  id: 1,
  name: "Ahmed",
  email: "ahmed@example.com",
  createdAt: new Date(),
};

const newUser: NewUser = {
  name: "Sara",
  email: "sara@example.com",
  // id and createdAt are auto-generated, so not required here
};
```

**Key Insight:** You define the schema once, and TypeScript types are auto-generated. No manual typing needed!

---

### 2.5 Migrations - Applying Schema to Neon

When you change your schema, you need to update your database.

**Step 1: Generate Migration**
```bash
npx drizzle-kit generate:pg
```
This creates a SQL file in `drizzle/migrations/` like:
```sql
CREATE TABLE "users" (
  "id" SERIAL PRIMARY KEY,
  "name" VARCHAR(100) NOT NULL,
  "email" VARCHAR(255) NOT NULL UNIQUE,
  ...
);
```

**Step 2: Apply Migration to Neon**
```bash
npx drizzle-kit push:pg
```
This runs the SQL on your Neon database.

**Workflow:**
```
1. Edit schema.ts
2. Run: npx drizzle-kit generate:pg  (creates SQL)
3. Run: npx drizzle-kit push:pg      (applies to Neon)
```

---

### 2.6 Querying - The Type-Safe Way

Now the fun part - reading and writing data with full autocomplete.

**Setup Connection:**
```typescript
// db.ts
import { drizzle } from 'drizzle-orm/neon-http';
import { neon } from '@neondatabase/serverless';
import * as schema from './schema';

const sql = neon(process.env.DATABASE_URL!);
export const db = drizzle(sql, { schema });
```

**Basic Queries:**

```typescript
import { db } from './db';
import { users, posts } from './schema';
import { eq, and, or } from 'drizzle-orm';

// ===== SELECT (Read) =====

// Get all users
const allUsers = await db.select().from(users);
// Type: User[] ← Autocomplete knows this!

// Get one user by ID
const user = await db
  .select()
  .from(users)
  .where(eq(users.id, 1));
// Type: User[]

// Get users with multiple conditions
const activeUsers = await db
  .select()
  .from(users)
  .where(and(
    eq(users.isActive, true),
    eq(users.age, 25)
  ));

// ===== INSERT (Create) =====

// Insert one user
const newUser = await db
  .insert(users)
  .values({
    name: "Ahmed",
    email: "ahmed@example.com",
    age: 25,
  })
  .returning();  // Returns the created user
// Type: User[]

// Insert multiple users
await db.insert(users).values([
  { name: "Ali", email: "ali@example.com" },
  { name: "Sara", email: "sara@example.com" },
]);

// ===== UPDATE (Modify) =====

// Update user by ID
await db
  .update(users)
  .set({ age: 26, isActive: false })
  .where(eq(users.id, 1));

// ===== DELETE (Remove) =====

// Delete user by ID
await db
  .delete(users)
  .where(eq(users.id, 1));

// Delete inactive users
await db
  .delete(users)
  .where(eq(users.isActive, false));
```

**Key Functions:**
- `eq(column, value)` - Equal to
- `and(condition1, condition2)` - Both must be true
- `or(condition1, condition2)` - Either can be true
- `.returning()` - Get back the created/updated row

---

### 2.7 Joins - Connecting Related Data

```typescript
// Get posts with their authors
const postsWithAuthors = await db
  .select({
    postId: posts.id,
    postTitle: posts.title,
    authorName: users.name,
    authorEmail: users.email,
  })
  .from(posts)
  .leftJoin(users, eq(posts.authorId, users.id));

// Result structure:
// [
//   {
//     postId: 1,
//     postTitle: "My First Post",
//     authorName: "Ahmed",
//     authorEmail: "ahmed@example.com"
//   },
//   ...
// ]
```

---

## Part 3: Practical Example - Full CRUD App

Let's build a simple todo app schema and queries:---
```ts
// ===== schema.ts =====
import { pgTable, serial, varchar, boolean, timestamp, text } from 'drizzle-orm/pg-core';

// Define the todos table
export const todos = pgTable('todos', {
  id: serial('id').primaryKey(),
  title: varchar('title', { length: 200 }).notNull(),
  description: text('description'),
  isCompleted: boolean('is_completed').default(false).notNull(),
  createdAt: timestamp('created_at').defaultNow().notNull(),
  updatedAt: timestamp('updated_at').defaultNow().notNull(),
});

// Auto-generated TypeScript types
export type Todo = typeof todos.$inferSelect;      // Getting data
export type NewTodo = typeof todos.$inferInsert;   // Creating data

// ===== db.ts =====
import { drizzle } from 'drizzle-orm/neon-http';
import { neon } from '@neondatabase/serverless';
import * as schema from './schema';

// Initialize connection to Neon
const sql = neon(process.env.DATABASE_URL!);
export const db = drizzle(sql, { schema });

// ===== queries.ts =====
import { db } from './db';
import { todos, Todo, NewTodo } from './schema';
import { eq, desc } from 'drizzle-orm';

// ===== CREATE =====
export async function createTodo(data: { title: string; description?: string }) {
  const newTodo = await db
    .insert(todos)
    .values({
      title: data.title,
      description: data.description,
    })
    .returning();
  
  return newTodo[0]; // Return the created todo
}

// ===== READ =====

// Get all todos
export async function getAllTodos(): Promise<Todo[]> {
  return await db
    .select()
    .from(todos)
    .orderBy(desc(todos.createdAt));
}

// Get one todo by ID
export async function getTodoById(id: number): Promise<Todo | undefined> {
  const result = await db
    .select()
    .from(todos)
    .where(eq(todos.id, id));
  
  return result[0];
}

// Get only completed todos
export async function getCompletedTodos(): Promise<Todo[]> {
  return await db
    .select()
    .from(todos)
    .where(eq(todos.isCompleted, true))
    .orderBy(desc(todos.createdAt));
}

// Get only pending todos
export async function getPendingTodos(): Promise<Todo[]> {
  return await db
    .select()
    .from(todos)
    .where(eq(todos.isCompleted, false))
    .orderBy(desc(todos.createdAt));
}

// ===== UPDATE =====

// Toggle todo completion status
export async function toggleTodoComplete(id: number) {
  // First get the current todo
  const todo = await getTodoById(id);
  if (!todo) return null;
  
  // Toggle its completion status
  const updated = await db
    .update(todos)
    .set({ 
      isCompleted: !todo.isCompleted,
      updatedAt: new Date(),
    })
    .where(eq(todos.id, id))
    .returning();
  
  return updated[0];
}

// Update todo title and description
export async function updateTodo(
  id: number, 
  data: { title?: string; description?: string }
) {
  const updated = await db
    .update(todos)
    .set({
      ...data,
      updatedAt: new Date(),
    })
    .where(eq(todos.id, id))
    .returning();
  
  return updated[0];
}

// ===== DELETE =====

// Delete a todo by ID
export async function deleteTodo(id: number) {
  const deleted = await db
    .delete(todos)
    .where(eq(todos.id, id))
    .returning();
  
  return deleted[0];
}

// Delete all completed todos
export async function deleteCompletedTodos() {
  return await db
    .delete(todos)
    .where(eq(todos.isCompleted, true))
    .returning();
}

// ===== EXAMPLE USAGE =====

async function exampleUsage() {
  // Create todos
  const todo1 = await createTodo({
    title: "Learn Drizzle ORM",
    description: "Master type-safe database queries"
  });
  
  const todo2 = await createTodo({
    title: "Build a project with Neon",
  });
  
  console.log("Created:", todo1);
  
  // Get all todos
  const allTodos = await getAllTodos();
  console.log("All todos:", allTodos);
  // Your editor knows allTodos is Todo[] - autocomplete works!
  
  // Mark first todo as complete
  const completed = await toggleTodoComplete(todo1.id);
  console.log("Completed:", completed);
  
  // Get only pending todos
  const pending = await getPendingTodos();
  console.log("Pending todos:", pending);
  
  // Update a todo
  const updated = await updateTodo(todo2.id, {
    title: "Build an amazing project with Neon",
    description: "Use Drizzle ORM for type safety"
  });
  console.log("Updated:", updated);
  
  // Delete a todo
  await deleteTodo(todo1.id);
  console.log("Deleted todo 1");
  
  // Get final list
  const final = await getAllTodos();
  console.log("Final todos:", final);
}

// Run the example
// exampleUsage();
```
## Part 4: Key Takeaways & Next Steps

### What You've Learned

**TypeScript Essentials:**
✅ Basic types (string, number, boolean)  
✅ Interfaces and types for defining data shapes  
✅ How generics appear (you recognize them, don't write them)  
✅ Letting autocomplete guide you  

**Drizzle Fundamentals:**
✅ Schema definition (tables, columns, relationships)  
✅ Type inference ($inferSelect, $inferInsert)  
✅ Migrations (generate and push)  
✅ CRUD operations (select, insert, update, delete)  
✅ Query conditions (eq, and, or)  
✅ Joins for related data  

---

### Practical Next Steps

**1. Set Up Your First Project**
```bash
# Install dependencies
npm install drizzle-orm @neondatabase/serverless
npm install -D drizzle-kit

# Create schema.ts and define your first table
# Run migration
npx drizzle-kit generate:pg
npx drizzle-kit push:pg

# Start querying!
```

**2. Practice Pattern**
```
1. Define schema in TypeScript
2. Generate migration
3. Write queries
4. Let autocomplete guide you
5. TypeScript catches errors before runtime
```

**3. Common Debugging Tips**
- If autocomplete doesn't work: Restart your editor
- If types are wrong: Regenerate with `npx drizzle-kit generate:pg`
- If query fails: Check your Neon connection string
- Always use `.returning()` to see what was created/updated

---

### Your Superpowers Now

1. **Type Safety:** Catch bugs before they happen
2. **Autocomplete:** Your editor knows everything about your database
3. **Refactoring:** Rename a column in schema.ts, TypeScript shows all places to update
4. **Documentation:** Types ARE documentation - no need for separate docs

---

### Quick Reference Card

```typescript
// Define table
export const users = pgTable('users', {
  id: serial('id').primaryKey(),
  name: varchar('name', { length: 100 }).notNull(),
});

// Get types
type User = typeof users.$inferSelect;
type NewUser = typeof users.$inferInsert;

// Query patterns
await db.select().from(users);                    // Get all
await db.select().from(users).where(eq(users.id, 1));  // Get one
await db.insert(users).values({ name: "Ahmed" }); // Create
await db.update(users).set({ name: "Ali" }).where(eq(users.id, 1));  // Update
await db.delete(users).where(eq(users.id, 1));    // Delete
```

---

**You're ready to build!** Start with a simple table, run some queries, and watch how TypeScript + Drizzle makes database work feel magical. The autocomplete will guide you through the rest.

What would you like to build first? I can help you design the schema for any project idea.