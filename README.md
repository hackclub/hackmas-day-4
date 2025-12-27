# Build Your Own Christmas API 🎄

![Meme](https://hc-cdn.hel1.your-objectstorage.com/s/v3/e7b2ae69ca0c6140_image.png)

*Don't worry, we're building something WAY simpler than a machine learning CRM!*

## Prerequisites

Make sure you have [bun](https://bun.com/) installed!

You will also need:
- a GitHub account
- a ID verified Hack Club account

## What you’re learning

### Hono
A tiny web framework. You define routes like this:
- “When someone does `GET /api/stuff`, run this function.”

### SQLite
A database that's just a file (like `my.db`) on your computer. This way things can persist!

### Drizzle
A TypeScript ORM that lets you:
- define your tables in `schema.ts`
- query them with TypeScript instead of raw SQL strings

### drizzle-kit
A CLI tool that takes your schema and creates/updates the real database so you don't have to do it manually!

---

## Your Project

You’ll end up with your own project that has:

- a working Hono server
- a SQLite file on disk
- Drizzle schema + queries
- a couple API endpoints that read/write the DB

In this workshop I'll show you how to set all this up. **What you actually make is up to you!** It'd be super cool if it was Christmas themed though, you do need at least 2 API routes! 🎅

---

# Project structure

This is the layout you're aiming for, if something later breaks check if your stuff looks like this!:

```
beans-cool-api/
  .env
  drizzle.config.ts
  my.db                (created after push)
  src/
    index.ts           (Hono server + routes)
    db/
      schema.ts        (table definitions)
      index.ts         (runtime DB connection)
      queries.ts       (helper functions: list/create/update/delete)
```

---

# Step 1 - Create a Hono project

### 1.1 Create + run

```bash
bun create hono@latest my-project
```

When it asks:
- **Install project dependencies?** `Yes` (press Enter)
- **Which package manager?** `bun` (press Enter)

TLDR: yes, bun

Then run:

```bash
cd my-project
bun run dev
```

Your terminal prints a URL. Open it.

### 1.2 Add a route

In `src/index.ts`:

```ts
import { Hono } from "hono"

const app = new Hono()

app.get("/", (c) => c.text("Beans!"))

export default app
```

If you see “Beans!” when you open the URL Hono printed, your server works! Yipeeeee!

---

# Step 2 - Install Drizzle + tools

```bash
bun add drizzle-orm dotenv
bun add -D drizzle-kit @types/bun better-sqlite3
```

What this does:

- `drizzle-orm` = what you use in your code
- `drizzle-kit` = what you run in the terminal
- `dotenv` = loads `.env` file values

---

# Step 3 - Define where your database file lives

Create `.env` in the project root:

```env
DB_FILE_NAME=./my.db
```

This is the file SQLite will store data in.

Add to `.gitignore` (so you don't commit your secrets or DB!):

```
.env
my.db
```

---

# Step 4 - Define your database schema (the blueprint)

**🛑 STOP & THINK!**

![Table](https://hc-cdn.hel1.your-objectstorage.com/s/v3/26bb1fa6e4995708_cleanshot_2025-12-27_at_22.19.29.png)

This is where you decide what your app is about.
Think of schema as the database’s "types":
- table name (e.g., `presents`, `elves`, `cookies`)
- column names
- column types
- default values

### Example schema

Create `src/db/schema.ts`.
*You can copy this exactly if you want to make a "Wishlist" app, OR rename `wishes` to whatever you want!*

```ts
import { sqliteTable, integer, text } from "drizzle-orm/sqlite-core"

// "wishes" is the table name in the DB
export const wishes = sqliteTable("wishes", {
  id: integer("id").primaryKey({ autoIncrement: true }),
  item: text("item").notNull(),
  fulfilled: integer("fulfilled").notNull().default(0),
  createdAt: integer("created_at").notNull(),
})
```

**Cheat Sheet for your own ideas:**

**Primary key (your table needs this)**
```ts
id: integer("id").primaryKey({ autoIncrement: true }),
```

**Text / strings**
```ts
text("name").notNull()     // required
text("description")        // optional
```

**Numbers / booleans**
```ts
integer("score").notNull()
integer("is_nice").notNull().default(0)  // 0 = false, 1 = true
```

---

# Step 5 - Tell drizzle-kit where your schema is

Create `drizzle.config.ts` in project root:

```ts
import "dotenv/config"
import { defineConfig } from "drizzle-kit"

export default defineConfig({
  out: "./drizzle",
  schema: "./src/db/schema.ts",
  dialect: "sqlite",
  dbCredentials: {
    url: process.env.DB_FILE_NAME!,
  },
})
```

---

# Step 6 - Create/update the DB file from your schema

Run:

```bash
bunx drizzle-kit push
```

After this:
- your `my.db` file should exist
- your table should exist inside it

**Tip:** If you change your schema later (like adding a column), run `push` again!

---

# Step 7 - Connect your running server to the DB

This is the part that makes routes actually talk to the database.

Create `src/db/index.ts`:

```ts
import "dotenv/config"
import { Database } from "bun:sqlite"
import { drizzle } from "drizzle-orm/bun-sqlite"

const sqlite = new Database(process.env.DB_FILE_NAME!)
export const db = drizzle(sqlite)
```

---

# Step 8 - Put DB code in helper functions

Routes should be readable. So we make helper functions.

**If you changed your table name in Step 4, update these functions to match!**

Create `src/db/queries.ts`:

```ts
import { db } from "./index"
import { wishes } from "./schema" // <--- Import YOUR table here
import { eq, desc } from "drizzle-orm"

export function listWishes() {
  return db.select().from(wishes).orderBy(desc(wishes.id)).all()
}

export function createWish(item: string) {
  const createdAt = Math.floor(Date.now() / 1000)

  const res = db.insert(wishes).values({
    item,
    fulfilled: 0,
    createdAt,
  }).returning({ id: wishes.id }).get()

  return res
}

export function fulfillWish(id: number) {
  const res = db.update(wishes)
    .set({ fulfilled: 1 })
    .where(eq(wishes.id, id))
    .returning({ id: wishes.id })
    .get()

  return res
}

export function deleteWish(id: number) {
  const res = db.delete(wishes)
    .where(eq(wishes.id, id))
    .returning({ id: wishes.id })
    .get()
    
  return res
}
```

---

# Step 9 - Write API routes that use those helpers

Now your routes become short and understandable.

In `src/index.ts`:

```ts
import { Hono } from "hono"
import { createWish, deleteWish, fulfillWish, listWishes } from "./db/queries"

const app = new Hono()

app.get("/", (c) => c.text("Beans!"))

// GET all wishes
app.get("/api/wishes", (c) => c.json(listWishes()))

// POST a new wish
app.post("/api/wishes", async (c) => {
  const body = await c.req.json().catch(() => null)
  // "item" matches the column name in our schema
  const item = (body?.item ?? "").toString().trim() 
  
  if (!item) return c.json({ error: "item is required" }, 400)

  return c.json(createWish(item), 201)
})

// PATCH (update) a wish
app.patch("/api/wishes/:id/fulfill", (c) => {
  const id = Number(c.req.param("id"))
  if (!Number.isFinite(id)) return c.json({ error: "bad id" }, 400)

  const res = fulfillWish(id)
  if (!res) return c.json({ error: "not found" }, 404)

  return c.json({ ok: true })
})

// DELETE a wish
app.delete("/api/wishes/:id", (c) => {
  const id = Number(c.req.param("id"))
  if (!Number.isFinite(id)) return c.json({ error: "bad id" }, 400)

  const res = deleteWish(id)
  if (!res) return c.json({ error: "not found" }, 404)

  return c.json({ ok: true })
})

export default app
```

---

# Step 10 - Test it

Here is how you can test your API!

**Add a wish:**
```bash
curl -X POST http://localhost:3000/api/wishes \
  -H "content-type: application/json" \
  -d '{"item":"lego"}'
```

**List wishes:**
```bash
curl http://localhost:3000/api/wishes
```

**Fulfill a wish:**
```bash
curl -X PATCH http://localhost:3000/api/wishes/1/fulfill
```

**Delete a wish:**
```bash
curl -X DELETE http://localhost:3000/api/wishes/1
```

---

# Step 11 - Make the server deployable

Hosting providers set the port for you. You **must** read `process.env.PORT`.

At the bottom of `src/index.ts`, replace the `export default app` with:

```ts
const port = Number(process.env.PORT) || 3000

console.log(`Server is running on port ${port}`)

export default {
  port,
  fetch: app.fetch,
}
```

---

# Step 12 - Deploy to FastDeploy

Let's put this on the internet! We'll use a tool just made for this workshop, usally you would use a server like nest... but thats gone so lets use this custom tool i made, it will only work for this workshop! 

### 12.1 Install the tool
```bash
bun install -g fastdeploy-hono
```

### 12.2 Login
```bash
fastdeploy login
```

### 12.3 Deploy!
```bash
# Make sure your DB is up to date first
bunx drizzle-kit push

# Ship it!
fastdeploy
```

Once it's done, it will give you a URL.
You can now use that URL instead of `localhost:3000` in your curl commands!

**⚠️ Important Note for Mac/Linux Users:**
If your URL contains an exclamation mark `!`, you need to wrap the URL in single quotes `'` when using curl, otherwise your terminal might try to interpret it as a command history expansion.

Example:
```bash
curl 'https://fastdeploy.deployor.dev/u/ident!RLwfBZ/test12121/api/wishes'
```

---

# Step 13 - Push to GitHub

### 13.1 Init git

```bash
git init
git add .
git commit -m "initial hono + drizzle + sqlite api"
```

### 13.2 Create GitHub repo

- Go to GitHub and create a **New Repository**
- Name it whatever you want
- **Do not** add a README or .gitignore (we already have them)

### 13.3 Push

Copy the commands GitHub gives you, which look like this:

```bash
git branch -M main
git remote add origin https://github.com/YOURNAME/YOUR-REPO.git
git push -u origin main
```

---

# You're Done! 🎉

Once you are done making your own project, take the URL that FastDeploy gave you and submit it here: [https://forms.hackclub.com/haxmas-day-4](https://forms.hackclub.com/haxmas-day-4)

Have fun and Happy Hacking! 🎄
