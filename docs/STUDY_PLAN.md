# Open Brain: Zero-to-Hero Study Plan

**Audience:** Developers with C/C++/Java experience who are new to web and full-stack development.
**Goal:** Understand every layer of the Open Brain system well enough to build, extend, and maintain it independently.
**Approach:** Theory first, then hands-on exercises using the actual repo files, then verification checkpoints.

---

## How to Use This Plan

Work through the phases in order. Each phase builds on the previous. Do not skip phases — the concepts stack. Every exercise points at a real file in this repository. Read that file before answering the verification questions.

Estimated total time is given per phase. There is no deadline. Thoroughness is the only measure.

---

## PHASE 1: JavaScript and TypeScript for Systems Developers

**Estimated time:** 2–4 weeks
**Prerequisites:** None — this is the starting point
**Key repo files:**
- `extensions/household-knowledge/index.ts`
- `extensions/household-knowledge/package.json`
- `extensions/household-knowledge/tsconfig.json`

---

### 1.1 How JavaScript Differs from C and Java

JavaScript was designed for browsers, not systems. That origin shapes everything about it. Here is what a C/Java developer will find surprising:

**Dynamic typing.** There is no type system in plain JavaScript. A variable that holds an integer can later hold a string. The compiler does not catch this — it fails at runtime. This is the single biggest source of bugs for developers coming from typed languages.

```javascript
// This is valid JavaScript and will not error until runtime:
let x = 42;
x = "hello";     // Now x is a string
x = { a: 1 };   // Now x is an object
```

**Everything is an object (roughly).** Functions are first-class values. You can store a function in a variable, pass it as an argument, or return it from another function. In C you do this with function pointers; in Java you do it with interfaces. In JavaScript, a function is just a value.

```javascript
// Functions as values — compare to passing a function pointer in C
const add = (a, b) => a + b;
const applyOperation = (fn, x, y) => fn(x, y);
console.log(applyOperation(add, 3, 4));  // 7
```

**No main() entry point (in the browser).** Scripts execute top-to-bottom when loaded. In Node.js (server-side), there is still no mandatory `main()` — but the idiom shown in every extension file is to define a `main()` function and call it at the bottom:

```typescript
async function main() { ... }
main().catch(error => { process.exit(1); });
```

**Prototypal inheritance, not classical.** JavaScript does not have classes in the traditional sense. The `class` keyword added in ES6 is syntactic sugar over prototype chains. For Open Brain purposes, you mostly use TypeScript interfaces and plain objects — you will not need to understand prototype chains deeply to read this codebase.

**The event loop.** JavaScript is single-threaded. There is no thread pool for parallelism. Instead, I/O operations (network calls, file reads) are handed to the operating system and a callback is registered. When the OS finishes, the callback is placed on the event queue and executed when the current call stack is empty. This is why async/await exists.

**`const` and `let`, not `var`.** Treat `const` like a C `const` — it cannot be reassigned (though objects it points to can be mutated). Treat `let` like a regular variable with block scope. Never use `var` in modern code — it has function scope, not block scope, which causes subtle bugs.

```javascript
const x = 5;
// x = 6;  // Error — cannot reassign const

const obj = { a: 1 };
obj.a = 2;  // Fine — the binding is const but the object is mutable

let counter = 0;
counter++;  // Fine — let can be reassigned
```

**Template literals.** String interpolation using backticks. Equivalent to `String.format()` in Java or `sprintf` in C, but cleaner:

```javascript
const name = "world";
const greeting = `Hello, ${name}!`;  // "Hello, world!"
```

**Destructuring.** Unpack values from arrays or objects into named variables in one line:

```javascript
// Object destructuring — extract named properties
const { user_id, name, category } = args;
// Equivalent to:
// const user_id = args.user_id;
// const name = args.name;
// const category = args.category;

// Array destructuring
const [first, second] = [1, 2];
```

You will see destructuring constantly in the extension files. In `extensions/household-knowledge/index.ts` line 218:
```typescript
const { user_id, name, category, location, details, notes } = args;
```

**Spread operator.** Expands an iterable into individual elements. Used for shallow-copying objects and merging them:

```javascript
const base = { a: 1, b: 2 };
const extended = { ...base, c: 3 };  // { a: 1, b: 2, c: 3 }
```

In `integrations/slack-capture/README.md`, the ingest-thought Edge Function does this:
```typescript
metadata: { ...metadata, source: "slack", slack_ts: messageTs }
```
This merges the extracted metadata object with two additional fields.

---

### 1.2 Promises and async/await

This is the most important concept to understand for this codebase. Every database call, every API call, every tool handler is asynchronous.

**The problem async solves.** When you call a database, the program should not freeze waiting for a response — it should continue processing other work. In C, you would handle this with threads or select()/epoll(). In JavaScript, the mechanism is the Promise.

**A Promise** represents a value that does not exist yet. It is either pending, fulfilled (resolved with a value), or rejected (failed with an error). Think of it like a Java `Future<T>` or C++ `std::future<T>`.

```javascript
// Old style (callback hell)
database.query("SELECT * FROM thoughts", function(error, results) {
    if (error) { handleError(error); return; }
    process(results);
});

// Promise style
database.query("SELECT * FROM thoughts")
    .then(results => process(results))
    .catch(error => handleError(error));

// async/await style — reads like synchronous code
async function fetchThoughts() {
    try {
        const results = await database.query("SELECT * FROM thoughts");
        return process(results);
    } catch (error) {
        handleError(error);
    }
}
```

**`async` marks a function as returning a Promise.** Any function marked `async` automatically wraps its return value in a Promise. Inside an `async` function, you can use `await` to pause execution until a Promise resolves.

**`await` suspends the current function.** When JavaScript hits an `await`, it suspends that function and returns control to the event loop. When the awaited Promise resolves, the function resumes from where it left off. The key insight: other code can run while one function is awaited.

**Parallel execution with `Promise.all`.** To run multiple async operations at the same time (not sequentially), use `Promise.all`. In the Slack capture Edge Function:

```typescript
const [embedding, metadata] = await Promise.all([
    getEmbedding(messageText),
    extractMetadata(messageText),
]);
```

This fires both API calls simultaneously and waits for both to complete — roughly halving the latency compared to sequential awaits.

Compare this to Java: it is like submitting two tasks to an `ExecutorService` and calling `Future.get()` on both.

---

### 1.3 TypeScript: Types Added Back

TypeScript is a superset of JavaScript — every valid JavaScript file is valid TypeScript. TypeScript adds:

- Type annotations (`:string`, `:number`, `:boolean`)
- Interfaces and type aliases
- Generics
- Compile-time type checking (the errors the TypeScript compiler catches before runtime)

The TypeScript compiler (`tsc`) compiles `.ts` files to `.js` files. The JavaScript runtime (Node.js) never sees TypeScript — it only runs the compiled JavaScript output.

**Interfaces vs Classes.** In Java, interfaces define contracts that classes implement. TypeScript interfaces work similarly but are structural (duck-typed): if an object has all the properties of an interface, it satisfies the interface regardless of how it was created.

In `extensions/household-knowledge/index.ts` lines 42–52:
```typescript
interface HouseholdItem {
    id: string;
    user_id: string;
    name: string;
    category: string | null;    // union type: string OR null
    location: string | null;
    details: Record<string, any>; // equivalent to Map<String, Object> in Java
    notes: string | null;
    created_at: string;
    updated_at: string;
}
```

`string | null` is a union type — the value can be one of several types. This is the TypeScript way to express nullable fields (compare to Java's `Optional<String>` or C's use of pointer-vs-value).

`Record<string, any>` is a generic type meaning "an object with string keys and values of any type." It maps to the JSONB `details` column in the database.

**The `any` type.** Opting out of type checking for a value. Using `any` is a code smell in production code but is sometimes unavoidable when dealing with truly dynamic data (like JSONB column contents or external API responses).

**`process.exit(1)`.** Terminates the Node.js process with exit code 1 (indicating failure). You will see this in every extension's environment validation block — if required environment variables are missing, the server refuses to start.

---

### 1.4 Arrow Functions

Arrow functions are shorthand function syntax. The `this` binding behaves differently from regular functions (but you will not need to worry about `this` in this codebase).

```typescript
// Traditional function
function add(a: number, b: number): number {
    return a + b;
}

// Arrow function — equivalent
const add = (a: number, b: number): number => a + b;

// Arrow function with block body (for multi-line)
const add = (a: number, b: number): number => {
    const result = a + b;
    return result;
};
```

In the extension files, arrow functions appear in array methods and callback positions:

```typescript
// In ob1-review.yml's JavaScript step (actions/github-script):
const botComment = comments.find(c =>
    c.user.type === 'Bot' && c.body.includes('OB1 Automated Review')
);
```

---

### 1.5 Exercise: Annotate index.ts

Open `extensions/household-knowledge/index.ts`. Read it line by line. For each line or block, write (in a comment or a separate doc) what it does. Answer:

1. Lines 21–27: What happens if environment variables are not set? Why call `process.exit(1)` instead of throwing an exception?
2. Lines 42–66: What is the difference between the `HouseholdItem` and `HouseholdVendor` interfaces? Which fields can be null?
3. Lines 69–214: The `TOOLS` array contains objects of type `Tool`. What is the structure of each tool object? How does `inputSchema` relate to JSON Schema?
4. Lines 217–241: Trace `handleAddHouseholdItem`. What is the return type? What happens if `error` is truthy?
5. Lines 376–401: The `switch` statement dispatches tool calls. What happens when an unknown tool name is passed?
6. Lines 404–413: Why is `main()` defined as `async` when it appears to have no `await`? (Hint: `server.connect()`)

---

### 1.6 Verification Checkpoints

You have mastered this phase when you can:

- [ ] Explain the difference between `const`, `let`, and `var` without looking it up
- [ ] Trace an `async/await` function and explain what happens at each `await`
- [ ] Write a TypeScript interface for a hypothetical database table
- [ ] Explain what `Promise.all` does and when you would use it over sequential awaits
- [ ] Read any line in `extensions/household-knowledge/index.ts` and explain what it does

**Self-assessment questions:**
1. What is the difference between `const obj = { a: 1 }` and `Object.freeze({ a: 1 })`?
2. If an `async` function throws an exception, what happens to the returned Promise?
3. In TypeScript, what is the difference between `type` and `interface`?
4. What does `Record<string, any>` mean? What would you use in Java for the same concept?

**Recommended external resources:**
- [TypeScript Handbook](https://www.typescriptlang.org/docs/handbook/intro.html) — start with "The Basics" and "Everyday Types"
- [JavaScript.info — Promises](https://javascript.info/promise-basics) — the clearest written explanation of the event loop and Promises
- [MDN — async/await](https://developer.mozilla.org/en-US/docs/Learn/JavaScript/Asynchronous/Promises) — practical examples

---

## PHASE 2: Node.js and the Package Ecosystem

**Estimated time:** 3–5 days
**Prerequisites:** Phase 1
**Key repo files:**
- `extensions/household-knowledge/package.json`
- `extensions/household-knowledge/tsconfig.json`
- `extensions/meal-planning/package.json`

---

### 2.1 What Node.js Is

Node.js is the V8 JavaScript engine (the one inside Google Chrome) extracted from the browser and packaged as a standalone runtime. It lets you run JavaScript on a server, on your laptop, or anywhere — without a browser.

The analogy for Java developers: Node.js is to JavaScript what the JVM is to Java. The analogy for C developers: Node.js is like compiling your program with a very specific C runtime that happens to be event-loop-based.

Node.js includes a standard library for file system access, networking, cryptography, and process management. You access it via `import` (or `require` in older code).

**Why do MCP extension servers use Node.js?** Because the `@modelcontextprotocol/sdk` package is distributed as a Node.js package, and the extensions in this repo are local stdio-based MCP servers that must be installed and run on the user's machine. Deno (used by Supabase Edge Functions) is an alternative JavaScript runtime, covered in Phase 4.

---

### 2.2 npm and package.json

npm (Node Package Manager) is the package manager for Node.js. The mental model:

| Concept | Java | C | JavaScript/Node |
|---------|------|---|-----------------|
| Package manager | Maven / Gradle | vcpkg / conan | npm / yarn |
| Dependency manifest | pom.xml | CMakeLists.txt | package.json |
| Dependency directory | .m2 repository | installed headers | node_modules/ |
| Lock file | (Maven resolves) | — | package-lock.json |

Open `extensions/household-knowledge/package.json`:

```json
{
  "name": "household-knowledge-mcp",
  "version": "1.0.0",
  "type": "module",
  "main": "build/index.js",
  "scripts": {
    "build": "tsc",
    "watch": "tsc --watch",
    "prepare": "npm run build"
  },
  "dependencies": {
    "@modelcontextprotocol/sdk": "^1.0.4",
    "@supabase/supabase-js": "^2.48.1"
  },
  "devDependencies": {
    "@types/node": "^22.10.5",
    "typescript": "^5.7.3"
  },
  "engines": {
    "node": ">=18.0.0"
  }
}
```

Line by line:

- `"type": "module"` — Use ES module syntax (`import`/`export`) rather than CommonJS (`require`). ES modules are the modern standard. The `Node16` module resolution in `tsconfig.json` corresponds to this.
- `"main": "build/index.js"` — The entry point after compilation. TypeScript compiles `index.ts` to `build/index.js`.
- `"scripts"` — Shortcuts run via `npm run <script-name>`. `npm run build` runs `tsc` (the TypeScript compiler). `prepare` runs automatically before `npm publish`.
- `"dependencies"` — Packages needed at runtime. The `^` prefix means "compatible with this version" — any version with the same major number.
- `"devDependencies"` — Packages only needed during development (building, type-checking). Not included in production installs. `@types/node` provides TypeScript type definitions for Node.js built-in modules. `typescript` is the compiler itself.
- `"engines"` — Documents the minimum required Node.js version. The SDK requires Node 18+ because it uses modern features.

**`node_modules/`** is created when you run `npm install`. It is gitignored (never committed). It can be 100MB+ and contains the full source of every dependency and their dependencies recursively. Delete it and regenerate it with `npm install` at any time.

---

### 2.3 tsconfig.json Explained

Open `extensions/household-knowledge/tsconfig.json`:

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "Node16",
    "moduleResolution": "Node16",
    "outDir": "./build",
    "rootDir": "./",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true
  }
}
```

- `"target": "ES2022"` — What JavaScript version to compile to. ES2022 is supported by Node.js 18+. The compiler down-levels newer syntax to this target.
- `"module": "Node16"` — How to handle `import`/`export` statements. Node16 mode uses ES modules with `.js` extensions in imports.
- `"outDir": "./build"` — Where compiled `.js` files go.
- `"rootDir": "./"` — Where TypeScript source files are.
- `"strict": true` — Enables all strict type-checking options. This is what makes TypeScript actually useful — without it, many bugs slip through.
- `"esModuleInterop": true` — Allows `import x from 'module'` syntax for CommonJS modules that don't have default exports. Required for compatibility with many npm packages.
- `"skipLibCheck": true` — Skip type-checking of `.d.ts` declaration files from npm packages. Speeds up compilation and avoids errors from poorly-typed dependencies.

---

### 2.4 The Build Process

For any extension server, the workflow is:

```bash
cd extensions/household-knowledge
npm install          # Download dependencies into node_modules/
npm run build        # Run tsc: compile index.ts → build/index.js
node build/index.js  # Run the compiled server
```

The compiled output in `build/index.js` is what actually runs. If you change `index.ts`, you must run `npm run build` again before the changes take effect. Use `npm run watch` during development — it recompiles automatically when files change.

---

### 2.5 Imports and Module Resolution

```typescript
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
```

This imports the `Server` class from the MCP SDK package. The path `@modelcontextprotocol/sdk/server/index.js` is a subpath export — the package exposes specific entrypoints rather than one giant module.

Note the `.js` extension in the import path even though the source file is `.ts`. This is a Node16 module resolution requirement — TypeScript in Node16 mode requires you to write the final `.js` extension in import paths, even when importing `.ts` files. It is counterintuitive but correct.

---

### 2.6 Exercise: Dependency Audit

1. Open `extensions/household-knowledge/package.json`. For each dependency, answer: What does this package do? Why does the extension need it?
2. Open `extensions/meal-planning/package.json`. It has the same dependencies as household-knowledge. Why? What is different about `extensions/meal-planning/shared-server.ts` vs `index.ts`?
3. The `@types/node` package is in `devDependencies`. What happens if you put it in `dependencies` instead? What happens if you omit it entirely?
4. What is the difference between `npm install` and `npm ci`? When would you use each?

---

### 2.7 Verification Checkpoints

- [ ] Explain what `package.json` is and what each major section does
- [ ] Explain what `node_modules/` contains and why it is gitignored
- [ ] Explain the difference between `dependencies` and `devDependencies`
- [ ] Given a TypeScript project, execute the full build cycle from source to running binary
- [ ] Explain what `tsc` does and where its output goes

**Recommended external resources:**
- [Node.js official docs — Getting Started](https://nodejs.org/en/docs/guides/getting-started-guide)
- [npm documentation](https://docs.npmjs.com/)
- [TypeScript — tsconfig reference](https://www.typescriptlang.org/tsconfig)

---

## PHASE 3: How the Web Works

**Estimated time:** 1 week
**Prerequisites:** Phase 1
**Key repo files:**
- `docs/01-getting-started.md` (Step 6 and Step 7)
- `integrations/slack-capture/README.md`
- `.github/workflows/ob1-review.yml` (the `actions/github-script` step at the bottom)

---

### 3.1 HTTP: The Protocol

HTTP (HyperText Transfer Protocol) is a request-response protocol over TCP. The mental model for C developers: it is socket programming with a standardized message format. For Java developers: it is what Servlet containers implement.

**Every HTTP interaction has:**

1. A **request** from a client: contains a method, a URL, headers, and optionally a body
2. A **response** from a server: contains a status code, headers, and a body

**HTTP Methods (verbs):**

| Method | C analogy | Purpose |
|--------|-----------|---------|
| GET | Read-only function call | Retrieve data, no side effects |
| POST | Write function call | Send data to create or trigger |
| PUT | Overwrite function call | Replace a resource entirely |
| PATCH | Partial update | Modify specific fields |
| DELETE | Free/remove | Delete a resource |

**Status Codes:**

| Range | Meaning | Examples |
|-------|---------|---------|
| 2xx | Success | 200 OK, 201 Created, 204 No Content |
| 3xx | Redirect | 301 Moved Permanently |
| 4xx | Client error | 400 Bad Request, 401 Unauthorized, 404 Not Found |
| 5xx | Server error | 500 Internal Server Error, 503 Service Unavailable |

**Headers** are key-value pairs of metadata attached to requests and responses. Common ones:

- `Content-Type: application/json` — the body is JSON
- `Authorization: Bearer eyJ...` — authentication token
- `x-brain-key: your-access-key` — the custom header Open Brain uses for MCP authentication

**The body** is arbitrary bytes. For REST APIs, it is almost always JSON. In Supabase API calls, you send JSON to the server and receive JSON back.

---

### 3.2 REST APIs

REST (Representational State Transfer) is an architectural style for web APIs. It maps HTTP verbs to CRUD operations on "resources" (conceptually: records in a database).

The Supabase auto-generated REST API follows this pattern:

```
GET    /rest/v1/thoughts            → SELECT * FROM thoughts
POST   /rest/v1/thoughts            → INSERT INTO thoughts
PATCH  /rest/v1/thoughts?id=eq.123  → UPDATE thoughts WHERE id = '123'
DELETE /rest/v1/thoughts?id=eq.123  → DELETE FROM thoughts WHERE id = '123'
```

The ChatGPT importer (`recipes/chatgpt-conversation-import/import-chatgpt.py`) uses this REST API directly instead of the Supabase client library:

```python
resp = http_post_with_retry(
    f"{SUPABASE_URL}/rest/v1/thoughts",
    headers={
        "apikey": SUPABASE_SERVICE_ROLE_KEY,
        "Authorization": f"Bearer {SUPABASE_SERVICE_ROLE_KEY}",
        "Prefer": "return=minimal",
    },
    body={
        "content": content,
        "embedding": embedding,
        "metadata": metadata_dict,
    },
)
```

This is making an HTTP POST to Supabase's auto-generated REST endpoint, passing authentication in the headers, and inserting a row.

---

### 3.3 JSON

JSON (JavaScript Object Notation) is a text format for structured data. It maps directly to JavaScript objects. The mental model for C developers: it is like a self-describing struct, serialized as text. For Java developers: it is what Jackson or Gson serialize/deserialize.

```json
{
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "content": "Sarah mentioned consulting",
    "metadata": {
        "type": "person_note",
        "people": ["Sarah"],
        "topics": ["career", "consulting"]
    },
    "created_at": "2026-03-13T10:00:00Z"
}
```

Rules: strings use double quotes, numbers have no quotes, booleans are `true`/`false`, null is `null`, arrays use `[]`, objects use `{}`.

In JavaScript, `JSON.stringify(obj)` serializes an object to a JSON string. `JSON.parse(str)` deserializes a JSON string to an object. The extension handlers return `JSON.stringify(result, null, 2)` — the `null, 2` arguments produce pretty-printed output with 2-space indentation.

---

### 3.4 Authentication in HTTP

Open Brain uses two authentication mechanisms:

**Service role key (Supabase):** A long JWT token that grants full database access. Passed in the `Authorization: Bearer <key>` header and/or as `apikey: <key>`. This key bypasses Row Level Security — it is an admin key. Extension servers use this key because they run on the user's own machine and need full access.

**MCP Access Key:** A random hex string generated during setup. The core MCP Edge Function checks this key on every request. It is passed either as a URL query parameter (`?key=...`) or as the `x-brain-key` header. See `docs/01-getting-started.md` Step 5.

**Why two layers?** The Supabase service role key is a secret known only to your server. The MCP access key is what clients present to prove they are allowed to talk to your server. Defense in depth.

---

### 3.5 Webhooks

A webhook is an HTTP callback — instead of your code calling an external service to ask for data, the external service calls your code when something happens. It is the opposite of polling.

The Slack integration (`integrations/slack-capture/README.md`) works via webhooks:

1. You deploy a Supabase Edge Function at a public URL
2. You register that URL with Slack as the "Event Subscriptions" endpoint
3. When a user posts a message, Slack makes an HTTP POST to your Edge Function with the message payload
4. Your Edge Function processes it and returns 200 OK

Slack requires a URL verification step (the `url_verification` challenge in the Edge Function code) to confirm you control the endpoint before it sends real events.

---

### 3.6 Exercise: Trace an HTTP Request

Read `integrations/slack-capture/README.md` carefully, then trace the full lifecycle of a message posted to Slack:

1. What triggers the HTTP request from Slack to your Edge Function?
2. What HTTP method does Slack use? Where in the Edge Function code is the method checked?
3. What is in the request body? What TypeScript type does it have?
4. How does the Edge Function authenticate itself to Supabase? To Slack?
5. What HTTP call does the Edge Function make back to Slack for the confirmation reply?
6. What status code should the Edge Function return on success? On failure? Why does the code return 200 even for filtered messages (bot messages, wrong channel)?

---

### 3.7 Verification Checkpoints

- [ ] Explain GET vs POST vs PUT vs PATCH vs DELETE without looking it up
- [ ] Explain what a webhook is and how it differs from polling
- [ ] Trace the authentication headers in any Supabase API call in the codebase
- [ ] Explain what a JWT is at a conceptual level (a signed, self-describing token)
- [ ] Explain what happens at the network level when `npm install` downloads a package

**Recommended external resources:**
- [HTTP on MDN](https://developer.mozilla.org/en-US/docs/Web/HTTP/Overview)
- [REST API Tutorial](https://restapitutorial.com/)
- [JWT.io — Introduction to JWTs](https://jwt.io/introduction)

---

## PHASE 4: PostgreSQL and the Database Layer

**Estimated time:** 2–3 weeks
**Prerequisites:** Phases 1–3
**Key repo files:**
- `extensions/household-knowledge/schema.sql`
- `extensions/home-maintenance/schema.sql`
- `extensions/family-calendar/schema.sql`
- `extensions/professional-crm/schema.sql`
- `extensions/meal-planning/schema.sql`
- `extensions/job-hunt/schema.sql`
- `docs/01-getting-started.md` (Step 2 — the three SQL blocks)
- `primitives/rls/README.md`

---

### 4.1 PostgreSQL Fundamentals

PostgreSQL is a full-featured relational database. If you know SQL (MySQL, SQLite, Oracle), most concepts transfer. If not, here is the foundation:

**Tables** are like structs in a flat file. Each row is a record. Each column has a fixed type. Unlike C structs, you query by value rather than by position.

**Data types used in this repo:**

| PostgreSQL Type | Java/C Equivalent | Notes |
|-----------------|-------------------|-------|
| `UUID` | `java.util.UUID` / 16-byte struct | Globally unique identifier. 128-bit random. Stored as hex string. |
| `TEXT` | `String` / `char*` | Variable-length string, no artificial limit. Better than `VARCHAR(n)` for most use cases. |
| `INTEGER` | `int` / `int32_t` | 32-bit signed integer |
| `DECIMAL(p,s)` | `BigDecimal` | Fixed-point number with precision p and scale s |
| `BOOLEAN` | `bool` | true/false |
| `DATE` | `java.time.LocalDate` | Calendar date without time |
| `TIME` | `java.time.LocalTime` | Time of day without date |
| `TIMESTAMPTZ` | `java.time.ZonedDateTime` | Timestamp with timezone. Always use this, never `TIMESTAMP`. |
| `TEXT[]` | `String[]` | PostgreSQL native array |
| `JSONB` | `Map<String, Object>` | Binary JSON — indexed, queryable |

**`UUID PRIMARY KEY DEFAULT gen_random_uuid()`** is used everywhere in this repo. This generates a random UUID for each new row. Compare to Java's `UUID.randomUUID()`. The advantage over auto-increment integers: UUIDs can be generated client-side without a round-trip to the database, and they do not reveal the table's row count.

**`TIMESTAMPTZ DEFAULT now() NOT NULL`** is the standard timestamp pattern. Always timezone-aware. `now()` returns the current timestamp at transaction start. The `NOT NULL` constraint is explicit even though `DEFAULT` provides a value — this ensures inserted rows cannot explicitly set the column to null.

**`ON DELETE CASCADE`** on foreign keys means: when the referenced row is deleted, automatically delete all rows that reference it. In `extensions/household-knowledge/schema.sql`:

```sql
user_id UUID REFERENCES auth.users(id) ON DELETE CASCADE NOT NULL
```

If the user account is deleted, all their household items are automatically deleted. This is equivalent to a destructor chain in C++ or the cascading behavior you implement manually in Java.

**`CHECK` constraints** enforce data validity at the database level. From `extensions/household-knowledge/schema.sql`:

```sql
rating INTEGER CHECK (rating >= 1 AND rating <= 5)
```

From `extensions/job-hunt/schema.sql`:

```sql
status TEXT DEFAULT 'applied' CHECK (status IN (
    'draft', 'applied', 'screening', 'interviewing',
    'offer', 'accepted', 'rejected', 'withdrawn'
))
```

This is equivalent to an enum in C/Java but enforced at the storage layer regardless of which application inserts the data.

---

### 4.2 Indexes

Indexes let the database find rows without scanning every row. They trade write speed and storage for read speed.

**B-tree index (default):** Used for equality and range comparisons. In `extensions/household-knowledge/schema.sql`:

```sql
CREATE INDEX IF NOT EXISTS idx_household_items_user_category
    ON household_items(user_id, category);
```

This is a composite index on two columns. Queries that filter by `user_id` alone, or by `user_id AND category`, can use this index. Queries that filter only by `category` cannot use it efficiently (the leading column must be present).

**GIN index:** Used for JSONB and array containment queries. From `docs/01-getting-started.md`:

```sql
CREATE INDEX ON thoughts USING GIN (metadata);
```

This enables fast `metadata @> '{"type": "observation"}'` queries (does the metadata column contain this JSON?).

**HNSW index:** Used for vector similarity search. Covered in Phase 5.

**Partial index:** An index with a WHERE clause. From `extensions/professional-crm/schema.sql`:

```sql
CREATE INDEX IF NOT EXISTS idx_professional_contacts_follow_up
    ON professional_contacts(user_id, follow_up_date)
    WHERE follow_up_date IS NOT NULL;
```

This index only includes rows where `follow_up_date` is set. Smaller, faster for the specific query pattern of "show me contacts I need to follow up on."

---

### 4.3 JSONB: Structured Flexibility

JSONB is PostgreSQL's binary JSON type. It is not just storing a text blob — PostgreSQL parses the JSON and stores it in a binary format that supports indexing and querying.

**Why JSONB exists:** The alternative to JSONB is adding a new column for every possible attribute. If you want to store paint colors, you add `brand`, `color_name`, `color_code` columns. If you then want to store appliances, you add `model_number`, `serial_number`, `warranty_expiry` columns. Most rows leave most columns null. JSONB avoids this by allowing each row to have different keys.

The `details` column in `household_items` is JSONB:

```sql
details JSONB DEFAULT '{}'
```

A paint color row might have: `{"brand": "Sherwin Williams", "color": "Sea Salt", "code": "SW 6204"}`
An appliance row might have: `{"brand": "Bosch", "model": "SHPM65Z55N", "serial": "FD12345678"}`

Both live in the same column with no schema change required.

**The `@>` containment operator:** "Does this JSONB value contain this JSON?" Used in the core `match_thoughts()` function:

```sql
WHERE t.metadata @> filter
```

If `filter = '{"type": "observation"}'`, this returns only rows where the metadata includes `"type": "observation"` as a key-value pair. With a GIN index, this is fast.

**JSONB arrays in meal planning.** The `ingredients` and `instructions` columns in `recipes` are JSONB arrays:

```sql
ingredients JSONB NOT NULL DEFAULT '[]'  -- array of {name, quantity, unit}
instructions JSONB NOT NULL DEFAULT '[]' -- array of step strings
```

The shopping list's `items` column is a JSONB array of objects:

```sql
items JSONB NOT NULL DEFAULT '[]'  -- array of {name, quantity, unit, purchased: bool, recipe_id}
```

The `mark_item_purchased` tool in `extensions/meal-planning/shared-server.ts` shows how to mutate a JSONB array: fetch the full array into application code, modify it, write it back. PostgreSQL has JSONB manipulation operators (`jsonb_set`, `||`), but the application-side approach is simpler to read.

**`TEXT[]` (PostgreSQL arrays):** Used for tags in `professional_contacts` and `recipes`:

```sql
tags TEXT[] DEFAULT '{}'
```

Queried with `contains()` in the Supabase client:

```typescript
if (args.tag) {
    query = query.contains("tags", [args.tag]);
}
```

This becomes a `tags @> ARRAY['tag']` SQL query.

---

### 4.4 Triggers and PLpgSQL Functions

Triggers are database-level event handlers — they run automatically when rows are inserted, updated, or deleted. This is the database equivalent of an event listener or a destructor.

**The `update_updated_at_column` pattern.** Every extension that has a mutable table uses the same trigger:

From `extensions/household-knowledge/schema.sql`:

```sql
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = now();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER update_household_items_updated_at
    BEFORE UPDATE ON household_items
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at_column();
```

Reading this:
- `CREATE OR REPLACE FUNCTION` — defines a stored procedure. `OR REPLACE` means "update it if it already exists."
- `RETURNS TRIGGER` — this function is called by a trigger, not directly
- `$$` is a dollar-quote delimiter — everything between `$$` and `$$` is the function body (avoids escaping single quotes)
- `BEGIN ... END` — PLpgSQL block
- `NEW` — the new row being inserted/updated. You can read and modify `NEW` in BEFORE triggers.
- `RETURN NEW` — return the (possibly modified) new row. Required for BEFORE row triggers.
- `BEFORE UPDATE ON household_items` — this trigger fires before each UPDATE operation
- `FOR EACH ROW` — fires once per row (as opposed to `FOR EACH STATEMENT` which fires once per query)

**The cascading trigger in home-maintenance.** `extensions/home-maintenance/schema.sql` has a more complex trigger that updates the parent `maintenance_tasks` row when a `maintenance_logs` row is inserted:

```sql
CREATE OR REPLACE FUNCTION update_task_after_maintenance_log()
RETURNS TRIGGER AS $$
DECLARE
    task_frequency INTEGER;
BEGIN
    SELECT frequency_days INTO task_frequency
    FROM maintenance_tasks
    WHERE id = NEW.task_id;

    UPDATE maintenance_tasks
    SET
        last_completed = NEW.completed_at,
        next_due = CASE
            WHEN task_frequency IS NOT NULL
            THEN NEW.completed_at + (task_frequency || ' days')::INTERVAL
            ELSE NULL
        END,
        updated_at = now()
    WHERE id = NEW.task_id;

    RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

This is the same pattern as an `AFTER INSERT` callback in Java or a trigger in SQLite. When you log that you changed your HVAC filter, the trigger automatically computes when the next change is due and updates the task record. No application code needed.

The `DECLARE` block declares local variables — `task_frequency INTEGER` is like `int task_frequency;` in C. `SELECT ... INTO` assigns a query result to the variable.

`(task_frequency || ' days')::INTERVAL` — the `||` operator concatenates strings, producing `'90 days'`. The `::INTERVAL` casts that string to a PostgreSQL interval type. Adding an interval to a timestamp gives a new timestamp.

**The contact tracking trigger in professional-crm.** `extensions/professional-crm/schema.sql` has a trigger that updates `last_contacted` on the parent contact whenever a new interaction is logged:

```sql
CREATE OR REPLACE FUNCTION update_last_contacted()
RETURNS TRIGGER AS $$
BEGIN
    UPDATE professional_contacts
    SET last_contacted = NEW.occurred_at
    WHERE id = NEW.contact_id;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER update_contact_last_contacted
    AFTER INSERT ON contact_interactions
    FOR EACH ROW
    EXECUTE FUNCTION update_last_contacted();
```

Note: this is `AFTER INSERT`, not `BEFORE UPDATE`. It fires after an interaction row is created.

---

### 4.5 Row Level Security (RLS)

RLS is PostgreSQL's per-row access control. It is enforced at the database engine level — not the application level. Even if your application has a bug that queries the wrong user's data, RLS prevents that data from being returned.

Read `primitives/rls/README.md` completely before continuing.

**The core concept.** When RLS is enabled on a table and a query runs, PostgreSQL evaluates the RLS policies for the current user against each row. Rows that do not satisfy any policy are invisible — not returned by SELECT, not affected by UPDATE or DELETE.

**`ALTER TABLE x ENABLE ROW LEVEL SECURITY`** activates RLS. Once enabled, all queries are subject to policies. If there are no policies, no rows are returned to anyone except superusers and the service role.

**`USING` clause** controls which rows a user can read or delete. It is a boolean expression evaluated per row:

```sql
CREATE POLICY household_items_user_policy ON household_items
    FOR ALL
    USING (auth.uid() = user_id)
    WITH CHECK (auth.uid() = user_id);
```

- `FOR ALL` — applies to SELECT, INSERT, UPDATE, DELETE
- `USING (auth.uid() = user_id)` — a row is visible only when the authenticated user's ID matches the row's `user_id`. `auth.uid()` is a Supabase function that returns the UUID of the currently authenticated user from their JWT.
- `WITH CHECK (auth.uid() = user_id)` — applies to INSERT and UPDATE. The new row being written must also satisfy this condition. This prevents a user from inserting a row with someone else's `user_id`.

**The service role bypasses RLS.** When extension servers use `SUPABASE_SERVICE_ROLE_KEY`, they connect as the Postgres service role, which has superuser-like permissions and is exempt from all RLS policies. This is why extension servers do their own filtering: `.eq("user_id", user_id)` in the query. The RLS policies provide a second layer of protection when users connect with their own JWTs (e.g., through a dashboard or REST API directly).

**Household-scoped RLS in meal-planning.** The `meal-planning` extension introduces a more complex pattern: `auth.jwt() ->> 'role' = 'household_member'`. This reads a custom claim from the user's JWT token. If the claim is present, the user is a household member with read access:

```sql
CREATE POLICY "Household members can view recipes"
    ON recipes
    FOR SELECT
    USING (
        auth.jwt() ->> 'role' = 'household_member'
        OR auth.uid() = user_id
    );
```

**Three RLS patterns in this repo:**
1. `auth.uid() = user_id` — personal data (all extensions)
2. `auth.jwt() ->> 'role' = 'household_member'` — shared household data (meal-planning)
3. `auth.role() = 'service_role'` — server-only access (core thoughts table from `docs/01-getting-started.md`)

---

### 4.6 The Core thoughts Table

From `docs/01-getting-started.md` Step 2:

```sql
CREATE TABLE thoughts (
    id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
    content TEXT NOT NULL,
    embedding VECTOR(1536),
    metadata JSONB DEFAULT '{}'::jsonb,
    created_at TIMESTAMPTZ DEFAULT now(),
    updated_at TIMESTAMPTZ DEFAULT now()
);
```

This is the one table that must never be altered destructively (no dropping or renaming existing columns). It is the foundation everything else connects to. Extensions create their own separate tables — they do not add columns to `thoughts`.

The `embedding VECTOR(1536)` column stores a 1536-dimensional vector. The `VECTOR` type is provided by the `pgvector` extension. This is covered in depth in Phase 5.

**The `match_thoughts()` function.** This is the semantic search function:

```sql
CREATE OR REPLACE FUNCTION match_thoughts(
    query_embedding VECTOR(1536),
    match_threshold FLOAT DEFAULT 0.7,
    match_count INT DEFAULT 10,
    filter JSONB DEFAULT '{}'::jsonb
)
RETURNS TABLE (
    id UUID,
    content TEXT,
    metadata JSONB,
    similarity FLOAT,
    created_at TIMESTAMPTZ
)
LANGUAGE plpgsql
AS $$
BEGIN
    RETURN QUERY
    SELECT
        t.id,
        t.content,
        t.metadata,
        1 - (t.embedding <=> query_embedding) AS similarity,
        t.created_at
    FROM thoughts t
    WHERE 1 - (t.embedding <=> query_embedding) > match_threshold
        AND (filter = '{}'::jsonb OR t.metadata @> filter)
    ORDER BY t.embedding <=> query_embedding
    LIMIT match_count;
END;
$$;
```

This function is a stored procedure that returns a table. `RETURN QUERY` followed by a SELECT statement is PLpgSQL syntax for returning multiple rows from a table-returning function. The `<=>` operator is the cosine distance operator from pgvector — covered in Phase 5.

---

### 4.7 Exercise: Schema Archaeology

For each schema file, draw (on paper or in a doc) the entity-relationship diagram — boxes for tables, arrows for foreign keys, labels for key columns.

1. `extensions/household-knowledge/schema.sql` — 2 tables, simple
2. `extensions/home-maintenance/schema.sql` — 2 tables, one trigger with side effects
3. `extensions/family-calendar/schema.sql` — 3 tables, note which foreign keys are nullable
4. `extensions/professional-crm/schema.sql` — 3 tables, 2 triggers
5. `extensions/meal-planning/schema.sql` — 3 tables, JSONB arrays, TEXT[] tags
6. `extensions/job-hunt/schema.sql` — 5 tables, cascade chain, cross-extension reference

For Extension 6 (`job-hunt`), find the `professional_crm_contact_id` column in `job_contacts`. The comment says: "FK to Extension 5's professional_contacts table (not enforced by DB, managed by application)." Answer: Why is this not a real PostgreSQL foreign key? What are the tradeoffs of application-enforced vs DB-enforced constraints?

---

### 4.8 Verification Checkpoints

- [ ] Explain UUID vs auto-increment integer primary keys, including the tradeoffs
- [ ] Explain what `ON DELETE CASCADE` does with a concrete example
- [ ] Write a SQL trigger from memory that updates an `updated_at` column
- [ ] Explain the difference between `USING` and `WITH CHECK` in RLS policies
- [ ] Explain why `auth.uid()` returns different values for different queries
- [ ] Explain when a GIN index is appropriate vs a B-tree index
- [ ] Read `match_thoughts()` and explain what `RETURN QUERY` does

**Self-assessment questions:**
1. A table has RLS enabled but no policies. Who can read from it?
2. Why does the `household-knowledge` extension filter by `user_id` in application code even though RLS is set up?
3. What is the difference between `JSONB` and `JSON` in PostgreSQL?
4. If you add a column to the `thoughts` table, does that violate the guard rails? What if you rename a column?

**Recommended external resources:**
- [PostgreSQL Documentation — Data Types](https://www.postgresql.org/docs/current/datatype.html)
- [Supabase — Row Level Security](https://supabase.com/docs/guides/auth/row-level-security)
- [PostgreSQL — Trigger Functions](https://www.postgresql.org/docs/current/plpgsql-trigger.html)
- [Use The Index, Luke](https://use-the-index-luke.com/) — free book on SQL indexing

---

## PHASE 5: Vector Embeddings and pgvector

**Estimated time:** 1 week
**Prerequisites:** Phase 4
**Key repo files:**
- `docs/01-getting-started.md` (the `match_thoughts()` function and "How It Works Under the Hood")
- `integrations/slack-capture/README.md` (the `getEmbedding()` function)

---

### 5.1 What Embeddings Are

An embedding is a numerical representation of semantic meaning. The idea: given a piece of text, a trained neural network produces a fixed-length array of floating-point numbers such that texts with similar meanings produce arrays that are close together in the high-dimensional space.

Think of it as a function: `embed("the cat sat on the mat") → [0.02, -0.14, 0.89, ..., 0.04]` — a vector of 1536 numbers for `text-embedding-3-small`.

The critical property: similarity in meaning produces similarity in vector space. "Sarah is thinking about leaving her job" and "career change" will produce vectors that are close together, even though they share no words.

This is fundamentally different from keyword search (LIKE, ILIKE, full-text search). Keyword search finds texts that contain the same words. Semantic search finds texts that mean the same thing.

**Why 1536 dimensions?** OpenAI's `text-embedding-3-small` produces 1536-dimensional vectors. The dimensionality is a property of the model — more dimensions generally means more nuance but higher storage cost. A vector of 1536 floats at 4 bytes each is 6,144 bytes per row.

---

### 5.2 Cosine Similarity

Given two embedding vectors, how similar are they? The standard measure is cosine similarity: the cosine of the angle between them in the 1536-dimensional space.

- Cosine similarity of 1.0: identical vectors (same meaning)
- Cosine similarity of 0.0: orthogonal vectors (unrelated meaning)
- Cosine similarity of -1.0: opposite vectors (antonyms)

The formula: `similarity = (A · B) / (|A| × |B|)` where `·` is dot product and `|A|` is the magnitude (L2 norm) of vector A.

**pgvector's `<=>` operator** computes cosine distance (1 - cosine similarity), not similarity. So:

- Distance of 0.0: identical (similarity = 1.0)
- Distance of 1.0: orthogonal (similarity = 0.0)

In `match_thoughts()`:

```sql
1 - (t.embedding <=> query_embedding) AS similarity
```

Converts distance back to similarity by subtracting from 1.

```sql
WHERE 1 - (t.embedding <=> query_embedding) > match_threshold
```

Only returns rows where similarity exceeds the threshold (default 0.7). Rows with similarity below 0.7 are not returned, even if they are the closest matches available.

```sql
ORDER BY t.embedding <=> query_embedding
```

Orders by distance ascending (most similar first).

---

### 5.3 The HNSW Index

A naive implementation of vector similarity search scans every row and computes the cosine distance — O(n) per query. With millions of rows, this becomes too slow.

HNSW (Hierarchical Navigable Small World) is an approximate nearest neighbor algorithm. It builds a multi-layered graph where each node connects to its approximate nearest neighbors. Searching the graph is much faster than exhaustive scan — typical complexity is O(log n).

The cost: it is approximate. With default parameters, it may miss the true nearest neighbor a small percentage of the time. For the Open Brain use case (personal memory search), this is acceptable — a slightly imperfect semantic match is returned quickly rather than a perfect match returned slowly.

From `docs/01-getting-started.md`:

```sql
CREATE INDEX ON thoughts
    USING HNSW (embedding vector_cosine_ops);
```

- `USING HNSW` — use the HNSW algorithm
- `embedding` — the column to index
- `vector_cosine_ops` — use cosine distance as the similarity metric (matches the `<=>` operator)

Alternatives: `vector_l2_ops` for Euclidean distance (`<->` operator), `vector_ip_ops` for inner product (`<#>` operator). For text embeddings, cosine similarity is standard.

---

### 5.4 The Embedding API Call

From `integrations/slack-capture/README.md`, the `getEmbedding()` function:

```typescript
async function getEmbedding(text: string): Promise<number[]> {
    const r = await fetch(`${OPENROUTER_BASE}/embeddings`, {
        method: "POST",
        headers: {
            "Authorization": `Bearer ${OPENROUTER_API_KEY}`,
            "Content-Type": "application/json"
        },
        body: JSON.stringify({
            model: "openai/text-embedding-3-small",
            input: text
        }),
    });
    const d = await r.json();
    return d.data[0].embedding;
}
```

This makes an HTTP POST to OpenRouter's embeddings endpoint with the text. The response is a JSON object; `d.data[0].embedding` is the array of 1536 numbers. OpenRouter routes this call to OpenAI's embedding model.

The same call in Python (from `recipes/chatgpt-conversation-import/import-chatgpt.py`):

```python
resp = http_post_with_retry(
    f"{OPENROUTER_BASE}/embeddings",
    headers={"Authorization": f"Bearer {OPENROUTER_API_KEY}", ...},
    body={"model": "openai/text-embedding-3-small", "input": truncated},
)
data = resp.json()
return data["data"][0]["embedding"]
```

Same structure, different language.

---

### 5.5 The Metadata Extraction Call

From the same Slack integration:

```typescript
async function extractMetadata(text: string): Promise<Record<string, unknown>> {
    const r = await fetch(`${OPENROUTER_BASE}/chat/completions`, {
        method: "POST",
        headers: { ... },
        body: JSON.stringify({
            model: "openai/gpt-4o-mini",
            response_format: { type: "json_object" },
            messages: [
                { role: "system", content: `Extract metadata...` },
                { role: "user", content: text },
            ],
        }),
    });
    const d = await r.json();
    try { return JSON.parse(d.choices[0].message.content); }
    catch { return { topics: ["uncategorized"], type: "observation" }; }
}
```

This calls an LLM (GPT-4o-mini via OpenRouter) to classify the thought. `response_format: { type: "json_object" }` instructs the model to always respond with valid JSON. The system prompt defines the schema of the expected JSON output. The `try/catch` handles cases where the model returns malformed JSON.

**Parallel execution.** In the Edge Function, both calls fire simultaneously:

```typescript
const [embedding, metadata] = await Promise.all([
    getEmbedding(messageText),
    extractMetadata(messageText),
]);
```

The embedding call and the metadata extraction call are independent. Running them in parallel (roughly) halves the total latency.

---

### 5.6 Exercise: Semantic Search Intuition

1. Read the "How It Works Under the Hood" section at the bottom of `docs/01-getting-started.md`. In your own words: why does searching for "career changes" retrieve a thought about "Sarah wanting to start consulting"?

2. The `match_threshold` parameter defaults to 0.7. What happens to search results if you lower the threshold to 0.3? What happens if you raise it to 0.95?

3. Why does the metadata (type, topics, people) matter if the embedding already captures semantic meaning? Think about a use case where you want BOTH semantic search AND a metadata filter.

4. Look at the `getEmbedding` function. What would you need to change to use a different embedding model that produces 3072-dimensional vectors? (Hint: there is more than one place to change.)

---

### 5.7 Verification Checkpoints

- [ ] Explain in plain English why two semantically similar sentences produce similar embeddings
- [ ] Explain what cosine similarity measures and what a value of 0.7 means
- [ ] Explain why HNSW returns approximate results, not exact results
- [ ] Trace a capture operation from raw text to the final database row
- [ ] Trace a search operation from a natural language query to ranked results

**Recommended external resources:**
- [Supabase — Vector Embeddings](https://supabase.com/docs/guides/ai/vector-embeddings)
- [pgvector GitHub README](https://github.com/pgvector/pgvector) — especially the index types section
- [What are embeddings? — OpenAI docs](https://platform.openai.com/docs/guides/embeddings)

---

## PHASE 6: The MCP Protocol

**Estimated time:** 1–2 weeks
**Prerequisites:** Phases 1–3
**Key repo files:**
- `extensions/household-knowledge/index.ts` (complete read-through)
- `extensions/family-calendar/index.ts`
- `extensions/meal-planning/shared-server.ts`
- `docs/01-getting-started.md` (Step 6 — the Edge Function deployment)

---

### 6.1 What Problem MCP Solves

AI assistants (Claude, ChatGPT, Cursor) are isolated. They know what they were trained on and what is in the current conversation context. They cannot read your notes, your database, your files, or your calendar — unless something explicitly gives them that access.

Before MCP, each AI provider invented their own function-calling format. Claude had one format, OpenAI had another, Gemini had a third. Every integration had to be built separately for each AI.

MCP (Model Context Protocol) is an open protocol (designed by Anthropic) that standardizes how AI assistants call external tools. The mental model:

- **Client:** the AI assistant (Claude Desktop, ChatGPT)
- **Server:** a program you run that exposes tools
- **Protocol:** a standard JSON-based message format

The AI client says "here are the available servers." When the AI wants to use a tool, it sends a structured request. The server responds with structured data. The AI incorporates the response into its answer.

This is analogous to RPC (Remote Procedure Call) — specifically, it is how the AI's "function calling" capability connects to external systems.

---

### 6.2 MCP Transports

**stdio (Standard Input/Output):** The AI client starts the MCP server as a child process. The client sends JSON over the server's stdin; the server responds on stdout. Simple, no network needed, local-only. All the extension servers use this transport.

**HTTP (Streamable HTTP):** The MCP server is a web server. The AI client makes HTTP requests to it. Works for remote servers. The core Open Brain MCP server (deployed as a Supabase Edge Function) uses this transport.

Why do the extensions use stdio? Because they are installed locally (via `npm install`), run on the user's machine, and talk to Supabase directly with the service role key. The key never leaves the user's machine.

Why does the core server use HTTP? Because it is deployed to Supabase's cloud infrastructure and accessed remotely by AI clients.

---

### 6.3 MCP Server Architecture

Every MCP server in this repo has the same structure. Read `extensions/household-knowledge/index.ts` as the canonical example:

**Step 1: Environment validation (lines 21–27)**

```typescript
const SUPABASE_URL = process.env.SUPABASE_URL;
const SUPABASE_SERVICE_ROLE_KEY = process.env.SUPABASE_SERVICE_ROLE_KEY;

if (!SUPABASE_URL || !SUPABASE_SERVICE_ROLE_KEY) {
    console.error("Error: SUPABASE_URL and SUPABASE_SERVICE_ROLE_KEY must be set");
    process.exit(1);
}
```

Fail fast at startup if configuration is missing. Never start in a broken state. This is the same principle as checking constructor arguments in Java or validating config at program start in C.

**Step 2: Client initialization (lines 30–39)**

```typescript
const supabase: SupabaseClient = createClient(
    SUPABASE_URL,
    SUPABASE_SERVICE_ROLE_KEY,
    { auth: { autoRefreshToken: false, persistSession: false } }
);
```

Create the Supabase client with the service role key. `autoRefreshToken: false` and `persistSession: false` are appropriate for server-side use — the MCP server does not maintain a user session.

**Step 3: Type definitions (lines 42–66)**

TypeScript interfaces that mirror the database table schemas. These are used for type safety in the handler functions.

**Step 4: Tool definitions (lines 69–214)**

The `TOOLS` array is a `Tool[]` — an array of tool descriptor objects. Each tool has:
- `name`: machine-readable identifier (snake_case)
- `description`: human-readable explanation shown to the AI
- `inputSchema`: a JSON Schema object describing the expected parameters

The `inputSchema` follows the [JSON Schema](https://json-schema.org/) standard. The AI uses the `description` to decide when to call the tool, and the `inputSchema` to know what arguments to provide. Good descriptions are essential — the AI reads them.

**Step 5: Handler functions (lines 217–356)**

One `async` function per tool. Each:
1. Destructures arguments from `args`
2. Makes one or more Supabase queries
3. Returns a JSON string (not an object — the MCP protocol expects text)

**Step 6: Server creation (lines 359–401)**

```typescript
const server = new Server(
    { name: "household-knowledge", version: "1.0.0" },
    { capabilities: { tools: {} } }
);

server.setRequestHandler(ListToolsRequestSchema, async () => ({
    tools: TOOLS,
}));

server.setRequestHandler(CallToolRequestSchema, async (request) => {
    const { name, arguments: args } = request.params;
    try {
        switch (name) {
            case "add_household_item":
                return { content: [{ type: "text", text: await handleAddHouseholdItem(args) }] };
            // ...
            default:
                throw new Error(`Unknown tool: ${name}`);
        }
    } catch (error) {
        return {
            content: [{ type: "text", text: JSON.stringify({ success: false, error: errorMessage }) }],
            isError: true,
        };
    }
});
```

Two request handlers:
- `ListToolsRequestSchema`: the AI calls this to discover what tools are available. Returns the `TOOLS` array.
- `CallToolRequestSchema`: the AI calls this to execute a tool. Dispatches to the appropriate handler function.

The response format `{ content: [{ type: "text", text: "..." }] }` is the MCP protocol's way of returning text content. The `isError: true` flag tells the AI that the tool call failed.

**Step 7: Start (lines 404–413)**

```typescript
async function main() {
    const transport = new StdioServerTransport();
    await server.connect(transport);
    console.error("Household Knowledge Base MCP Server running on stdio");
}
main().catch(error => {
    console.error("Fatal error in main():", error);
    process.exit(1);
});
```

`StdioServerTransport` connects the server to stdin/stdout. `console.error` (not `console.log`) is used for log messages because stdout is reserved for MCP protocol messages.

---

### 6.4 The Shared Server Pattern

`extensions/meal-planning/shared-server.ts` introduces a second MCP server with restricted access:

- Uses `SUPABASE_HOUSEHOLD_KEY` instead of `SUPABASE_SERVICE_ROLE_KEY`
- Exposes only read-focused tools
- Cannot create or delete recipes or meal plans
- Can update shopping lists (mark items purchased)

This implements the principle of least privilege at the MCP layer. The shared server is given to a household member; the main server is kept private. See `primitives/shared-mcp/README.md` for the full explanation of this pattern.

---

### 6.5 Differences: Extension Servers vs Edge Function Server

| Aspect | Extension Servers | Core Edge Function |
|--------|------------------|---------------------|
| Runtime | Node.js | Deno |
| Transport | stdio | HTTP |
| Deployment | User's machine | Supabase cloud |
| Entry point | `main()` → `StdioServerTransport` | `Deno.serve(async req => ...)` |
| Module imports | npm packages | URL imports or `npm:` prefix |
| Secrets | Environment variables | Supabase secrets |
| Auth | Service role key | MCP access key header |

The Edge Function (the core server) does not use the MCP SDK's `Server` class with a transport — it uses Hono (a lightweight web framework for Deno) with an MCP adapter. This is a different pattern from the extension servers, appropriate for HTTP-based deployment.

---

### 6.6 Exercise: Trace a Tool Call

Follow the complete lifecycle of calling `search_household_items` from Claude Desktop:

1. Where does Claude Desktop read the list of available tools? Which request handler returns them?
2. When Claude calls `search_household_items`, what JSON does it send to the MCP server? (Sketch the structure based on `CallToolRequestSchema`)
3. Which handler function is invoked? Trace the Supabase query it builds.
4. If `query` is "blue paint" and no `category` is specified, what SQL does the Supabase client generate?
5. What does the handler return? What does the MCP server wrap it in before sending to Claude?
6. Compare the error handling in `household-knowledge/index.ts` vs `family-calendar/index.ts`. What is different? Which approach do you prefer and why?

---

### 6.7 Verification Checkpoints

- [ ] Explain MCP in one paragraph to someone who has never heard of it
- [ ] Explain the difference between stdio and HTTP transports
- [ ] Given a new tool to implement, write the tool definition object from memory
- [ ] Trace a `CallToolRequest` from the AI to the database and back
- [ ] Explain the purpose of `isError: true` in the MCP response

**Recommended external resources:**
- [MCP Official Documentation](https://modelcontextprotocol.io/docs)
- [MCP TypeScript SDK on GitHub](https://github.com/modelcontextprotocol/typescript-sdk)

---

## PHASE 7: Supabase Platform

**Estimated time:** 1 week
**Prerequisites:** Phases 4–6
**Key repo files:**
- `docs/01-getting-started.md` (complete)
- `integrations/slack-capture/README.md` (the Edge Function code)

---

### 7.1 What Supabase Is

Supabase is an open-source Firebase alternative built on PostgreSQL. It wraps PostgreSQL with:

- **Auto-generated REST API:** Any table you create immediately gets CRUD endpoints
- **Authentication:** User management (signup, login, JWT issuance) built in
- **Storage:** File storage (not used in Open Brain)
- **Edge Functions:** Serverless functions running on Deno
- **Real-time:** WebSocket subscriptions to table changes (not used in Open Brain)
- **Dashboard:** GUI for the SQL editor, table browser, function logs, and more

For Open Brain, you use: PostgreSQL (obviously), the auto-generated REST API (for the Python importer), and Edge Functions (for the core MCP server and Slack integration). Authentication is leveraged for RLS (`auth.uid()`, `auth.users` table).

---

### 7.2 Supabase Client Library

The `@supabase/supabase-js` library provides a typed client for interacting with Supabase from JavaScript/TypeScript. You use this in every extension server.

**Client creation:**

```typescript
const supabase = createClient(SUPABASE_URL, SUPABASE_SERVICE_ROLE_KEY, {
    auth: { autoRefreshToken: false, persistSession: false }
});
```

**Query builder pattern:** Supabase uses a fluent builder API. Methods chain and the query does not execute until you await the result.

```typescript
// SELECT * FROM household_items WHERE user_id = ? ORDER BY created_at DESC
const { data, error } = await supabase
    .from("household_items")
    .select("*")
    .eq("user_id", user_id)
    .order("created_at", { ascending: false });
```

**Filter methods:**

| Method | SQL equivalent | Example |
|--------|---------------|---------|
| `.eq("col", val)` | `WHERE col = val` | `.eq("user_id", id)` |
| `.ilike("col", pattern)` | `WHERE col ILIKE '%pattern%'` | `.ilike("name", "%paint%")` |
| `.gte("col", val)` | `WHERE col >= val` | `.gte("date_value", today)` |
| `.lte("col", val)` | `WHERE col <= val` | `.lte("date_value", future)` |
| `.or("expr")` | `WHERE (expr)` | `.or("name.ilike.%foo%,notes.ilike.%foo%")` |
| `.contains("col", arr)` | `WHERE col @> arr` | `.contains("tags", ["ai"])` |
| `.is("col", null)` | `WHERE col IS NULL` | `.is("end_date", null)` |

**Mutations:**

```typescript
// INSERT
const { data, error } = await supabase
    .from("household_items")
    .insert({ user_id, name, category })
    .select()      // return the inserted row
    .single();     // expect exactly one row

// UPDATE
const { data, error } = await supabase
    .from("shopping_lists")
    .update({ items: updatedItems })
    .eq("id", shopping_list_id)
    .select()
    .single();
```

**Error handling:** Every Supabase query returns `{ data, error }`. If `error` is non-null, the query failed. Always check it:

```typescript
if (error) {
    throw new Error(`Failed to add household item: ${error.message}`);
}
```

**The `.single()` method:** Expects exactly one row. Returns an error if zero or more than one row is returned. Use this when querying by primary key or unique constraint.

---

### 7.3 Edge Functions

Supabase Edge Functions are serverless functions running on Deno. Think of them as AWS Lambda functions, but built into Supabase and billed by execution time.

**Deno vs Node.js:**

| Aspect | Node.js | Deno |
|--------|---------|------|
| Package management | npm (`package.json`) | URL imports or `npm:` prefix |
| TypeScript | Requires compilation step | Native TypeScript support |
| Standard library | Node built-ins | Deno standard library |
| Security | Open by default | Deny by default (explicit permissions) |
| HTTP server | Requires framework (Express) | `Deno.serve()` built in |

The Slack integration Edge Function starts with:

```typescript
import { createClient } from "https://esm.sh/@supabase/supabase-js@2";
```

This imports from a CDN URL instead of npm. Deno downloads and caches it automatically — no `npm install` needed.

Environment variables in Deno: `Deno.env.get("VARIABLE_NAME")` instead of `process.env.VARIABLE_NAME`.

HTTP server in Deno:

```typescript
Deno.serve(async (req: Request): Promise<Response> => {
    // handle request
    return new Response("ok", { status: 200 });
});
```

**Supabase secrets** are environment variables set via CLI and available inside Edge Functions:

```bash
supabase secrets set OPENROUTER_API_KEY=sk-or-...
supabase secrets set SLACK_BOT_TOKEN=xoxb-...
```

`SUPABASE_URL` and `SUPABASE_SERVICE_ROLE_KEY` are automatically injected — you do not set them manually.

**Cold starts.** The first invocation of an Edge Function after a period of inactivity takes a few extra seconds to start the Deno runtime. Subsequent calls within the warm period are faster. This explains the troubleshooting note in `docs/01-getting-started.md`: "First search on a cold function takes a few seconds."

**Deployment:**

```bash
supabase functions deploy function-name --no-verify-jwt
```

`--no-verify-jwt` disables Supabase's built-in JWT verification. The Open Brain Edge Function implements its own authentication (the access key check), so it does not need Supabase to verify JWTs.

---

### 7.4 The Supabase CLI

Key commands used in this repo:

```bash
supabase login                        # Authenticate with Supabase
supabase link --project-ref REF       # Link local CLI to your Supabase project
supabase functions new name           # Scaffold a new Edge Function
supabase functions deploy name        # Deploy a function to the cloud
supabase secrets set KEY=value        # Set an environment variable for Edge Functions
supabase secrets list                 # List all set secrets (values hidden)
```

The CLI works by communicating with Supabase's management API. `supabase link` stores your project reference locally so subsequent commands know which project to target.

---

### 7.5 Exercise: Deploy and Inspect

If you have a Supabase project set up (from following `docs/01-getting-started.md`):

1. View your Edge Function logs: Supabase dashboard → Edge Functions → open-brain-mcp → Logs. What do you see after making a tool call?
2. Run `supabase secrets list`. What secrets are set? Can you see their values?
3. Make a direct REST API call to your thoughts table using `curl` or a REST client (Postman, Insomnia, or even a browser extension). Use the auto-generated REST endpoint at `https://YOUR_REF.supabase.co/rest/v1/thoughts` with the service role key in the `Authorization` header and `apikey` header. What does it return?

---

### 7.6 Verification Checkpoints

- [ ] Explain the difference between Node.js and Deno
- [ ] Write a Supabase query to find all household items where `category` contains "paint" for a specific `user_id`
- [ ] Explain what `supabase secrets set` does and where secrets are stored
- [ ] Explain what a cold start is and why it happens
- [ ] Explain the difference between the service role key and the anon key

**Recommended external resources:**
- [Supabase — JavaScript Client Library](https://supabase.com/docs/reference/javascript/introduction)
- [Supabase — Edge Functions](https://supabase.com/docs/guides/functions)
- [Deno documentation](https://deno.land/manual)

---

## PHASE 8: The Extension System

**Estimated time:** 2 weeks
**Prerequisites:** Phases 4–7
**Key repo files:** All 6 extension directories

---

### 8.1 Anatomy of an Extension

Every extension is a self-contained directory with five components:

| File | Purpose |
|------|---------|
| `schema.sql` | Database tables, indexes, RLS policies, triggers |
| `index.ts` | MCP server — tool definitions and handler logic |
| `package.json` | Node.js package manifest — dependencies and build scripts |
| `tsconfig.json` | TypeScript compiler configuration |
| `metadata.json` | Structured metadata for the repository index |
| `README.md` | Human documentation — prerequisites, setup steps, expected outcome |

An extension is deployed by:
1. Running the `schema.sql` in Supabase's SQL Editor
2. Running `npm install && npm run build` locally
3. Configuring the AI client to run the compiled server

---

### 8.2 The Learning Path: Progressive Complexity

The six extensions are designed as a learning sequence. Each introduces concepts not in the previous ones.

**Extension 1 — Household Knowledge Base** (`extensions/household-knowledge/`)
- Concepts introduced: basic MCP server structure, CRUD operations, ILIKE text search, JSONB storage, composite indexes
- Schema: 2 tables, simple user-scoped RLS
- Complexity: beginner

**Extension 2 — Home Maintenance Tracker** (`extensions/home-maintenance/`)
- Concepts introduced: one-to-many relationships (tasks → logs), date arithmetic in PostgreSQL, cascading triggers (log insert → parent task update), DECIMAL for cost, partial indexes
- Schema: 2 tables, one-to-many FK, two triggers
- Complexity: beginner

**Extension 3 — Family Calendar** (`extensions/family-calendar/`)
- Concepts introduced: multi-entity scheduling (family members → activities → important dates), nullable foreign keys, date range queries, joining related tables in SELECT (`family_members:family_member_id (name, relationship)`)
- Schema: 3 tables, nullable FKs
- Complexity: intermediate

The Supabase client's join syntax:

```typescript
.select(`
    *,
    family_members:family_member_id (name, relationship)
`)
```

This performs a JOIN: `SELECT activities.*, family_members.name, family_members.relationship FROM activities LEFT JOIN family_members ON activities.family_member_id = family_members.id`. The result embeds the related family member data in each activity object.

**Extension 4 — Meal Planning** (`extensions/meal-planning/`)
- Concepts introduced: household-scoped RLS (shared access), JSONB arrays as complex column types (ingredients, instructions, shopping items), TEXT[] arrays for tags, the shared MCP server pattern
- Schema: 3 tables, household-scoped RLS, JSONB arrays
- Complexity: intermediate

**Extension 5 — Professional CRM** (`extensions/professional-crm/`)
- Concepts introduced: contact tracking and interaction logging, second trigger pattern (`update_last_contacted`), cross-extension data (thoughts ↔ contacts), opportunities pipeline
- Schema: 3 tables, cascade of triggers
- Complexity: intermediate

**Extension 6 — Job Hunt Pipeline** (`extensions/job-hunt/`)
- Concepts introduced: 5-table cascade schema (company → posting → application → interview → contact), enum-like CHECK constraints, soft cross-extension FK (`professional_crm_contact_id`), partial indexes with WHERE clause
- Schema: 5 tables, full cascade chain
- Complexity: advanced

---

### 8.3 Cross-Extension Integration

Extensions are designed to be independent but can reference each other.

**Soft foreign key in job-hunt.** In `extensions/job-hunt/schema.sql`, the `job_contacts` table has:

```sql
professional_crm_contact_id UUID,
-- FK to Extension 5's professional_contacts table (not enforced by DB, managed by application)
```

This is intentionally not a PostgreSQL foreign key. Why?
- Extensions are optional — not everyone installs Extension 5
- PostgreSQL would enforce the FK at insert time, causing errors if Extension 5's table does not exist
- The application is responsible for keeping this reference valid

This is a common pattern when systems grow organically — "soft references" that carry no DB-level enforcement but document the intended relationship.

**Meal planning ↔ family calendar.** The meal planning README notes that it can check the family calendar before suggesting meals. This is not implemented at the schema level — it is implemented at the MCP server level: the AI can call tools from both extensions in the same conversation and the AI itself synthesizes the information.

**Core thoughts as the universal store.** The `thoughts` table is never directly referenced by extension tables (no FK). But Extension 5 notes "cross-extension integration with core Open Brain thoughts." The intended pattern is: when you log a conversation with a professional contact, you also capture a thought via the MCP capture tool. Both are independent rows that the AI can retrieve together.

---

### 8.4 The metadata.json Schema

Every extension (and every recipe, schema, dashboard, integration, primitive) requires a `metadata.json`. This is validated by the automated review workflow.

Required fields (from `.github/metadata.schema.json`):
- `name`, `description`, `category`, `version` — basic identification
- `author.name` — attribution
- `requires.open_brain: true` — confirms the contribution requires Open Brain
- `tags` — at least one tag
- `difficulty` — "beginner", "intermediate", or "advanced"
- `estimated_time` — how long setup takes

Extension-specific fields:
- `requires_primitives` — array of primitive slugs this extension depends on
- `learning_order` — integer 1–6 for the learning path position

---

### 8.5 Exercise: Extension Comparison Matrix

Create a comparison table (in a doc or spreadsheet) for all 6 extensions:

| Extension | Tables | Triggers | JSONB usage | RLS pattern | New concepts |
|-----------|--------|----------|-------------|-------------|--------------|

Fill in each cell after reading each extension's `schema.sql` and `index.ts`.

Then answer:
1. Which extension has the most complex trigger logic? Explain what it does.
2. Which extension would be hardest to remove from the system without breaking other extensions?
3. Why does Extension 4 (meal planning) have two separate server files (`index.ts` and `shared-server.ts`)?
4. Extension 6's `job_contacts` table has no `updated_at` column. Is that an oversight? When would you add one?

---

### 8.6 Verification Checkpoints

- [ ] Explain what the 5 files in each extension do without looking
- [ ] Read a schema.sql and draw its ER diagram from memory
- [ ] Explain why the job-hunt ↔ professional-crm reference is a "soft" FK
- [ ] Explain the household-scoped RLS pattern and when you would use it
- [ ] Given a new extension idea, sketch the schema and tool list

---

## PHASE 9: CI/CD and the GitHub Actions Workflow

**Estimated time:** 4–6 days
**Prerequisites:** Phase 3 (HTTP concepts), familiarity with git
**Key repo files:**
- `.github/workflows/ob1-review.yml`
- `CONTRIBUTING.md`
- `.github/metadata.schema.json`
- `.github/PULL_REQUEST_TEMPLATE.md`

---

### 9.1 What GitHub Actions Is

GitHub Actions is a CI/CD platform built into GitHub. A "workflow" is a YAML file in `.github/workflows/` that describes automated tasks triggered by GitHub events (pull request opened, code pushed, etc.).

Think of it as: a build server that runs scripts automatically when things happen in your repo. The mental model for C developers: it is like a Makefile that runs on a cloud server triggered by git events. For Java developers: it is like Jenkins, but hosted by GitHub.

**Trigger:** When a PR is opened against `main`, the workflow runs.

**Job:** A unit of work running on a virtual machine. The `ob1-review.yml` workflow has one job: `review`.

**Steps:** Sequential tasks within a job. Steps share a filesystem. The ob1-review workflow has 4 steps: checkout code, get changed files, run review checks, post a comment.

---

### 9.2 The ob1-review.yml Workflow

Open `.github/workflows/ob1-review.yml` and read it in full. Here is an explanation of each step:

**Step 1: Checkout PR**

```yaml
- name: Checkout PR
  uses: actions/checkout@v4
  with:
    fetch-depth: 0
```

`uses: actions/checkout@v4` is a pre-built action that clones the repository. `fetch-depth: 0` fetches the full git history (not just the latest commit), which is needed to compare branches.

**Step 2: Get changed files**

```yaml
- name: Get changed files
  id: changed
  run: |
    changed=$(git diff --name-only origin/main...HEAD)
    echo "files<<EOF" >> $GITHUB_OUTPUT
    echo "$changed" >> $GITHUB_OUTPUT
    echo "EOF" >> $GITHUB_OUTPUT
```

`id: changed` gives this step an identifier so later steps can reference its outputs.

`run: |` executes a bash script. The `|` is YAML block scalar syntax — everything below it (indented) is the script.

`$GITHUB_OUTPUT` is a special file. Writing to it in the format `name<<EOF ... EOF` exposes values as step outputs that subsequent steps can access via `${{ steps.changed.outputs.name }}`.

**Step 3: Run review checks**

This is the core of the workflow — a 400-line bash script that implements 13 checks. Each check:
1. Sets a boolean flag (`rule1_pass=true`)
2. Scans the changed files
3. Appends to `results` string with `pass_check` or `fail_check`
4. Increments `pass_count` or `fail_count`

At the end, it writes the full results and a pass/fail boolean to `$GITHUB_OUTPUT`.

**Check highlights:**

Rule 3 (Metadata validation) uses `jq` — a command-line JSON processor:
```bash
if ! jq empty "$dir/metadata.json" 2>/dev/null; then
    # not valid JSON
fi
val=$(jq -r ".$field // empty" "$dir/metadata.json")
```

`jq -r ".$field // empty"` extracts a field from JSON. The `// empty` returns an empty string if the field does not exist.

Rule 4 (No credentials) uses `grep` with regex patterns:
```bash
matches=$(grep -nE '(sk-[a-zA-Z0-9]{20,}|AKIA[0-9A-Z]{16}|ghp_[a-zA-Z0-9]{36}|xoxb-[0-9]|...)' "$f")
```

This pattern matches common API key formats: OpenAI keys (`sk-`), AWS access keys (`AKIA`), GitHub tokens (`ghp_`), Slack bot tokens (`xoxb-`).

Rule 5 (SQL safety) checks for dangerous SQL:
```bash
dangerous=$(grep -niE '(DROP\s+TABLE|DROP\s+DATABASE|TRUNCATE)' "$f")
```

Rule 10 (Primitive dependencies) checks that declared `requires_primitives` exist as actual directories:
```bash
if [ ! -d "primitives/$prim" ]; then
    # fail
fi
```

**Step 4: Post review comment**

```yaml
- name: Post review comment
  uses: actions/github-script@v7
  env:
    REVIEW_COMMENT: ${{ steps.review.outputs.comment }}
  with:
    script: |
      // JavaScript running in Node.js
      const body = process.env.REVIEW_COMMENT;
      const { data: comments } = await github.rest.issues.listComments({ ... });
      const botComment = comments.find(c => c.user.type === 'Bot' && ...);
      if (botComment) {
          await github.rest.issues.updateComment({ ... body });
      } else {
          await github.rest.issues.createComment({ ... body });
      }
```

`actions/github-script` runs JavaScript with a pre-configured `github` object that provides access to the GitHub REST API. This step:
1. Finds any existing bot comment on the PR (to avoid spamming)
2. Updates it if it exists, creates a new one if not
3. The comment contains the full review results

**Step 5: Fail if checks failed**

```yaml
- name: Fail if checks failed
  if: steps.review.outputs.failed == 'true'
  run: exit 1
```

If any check failed, the workflow exits with code 1, which marks the GitHub Actions check as "failed" and blocks the PR from merging (if branch protection requires the check to pass).

---

### 9.3 Contribution Standards

From `CONTRIBUTING.md`:

**Branch convention:** `contrib/<github-username>/<short-description>`

**PR title format:** `[category] Short description` — e.g. `[recipes] Email history import via Gmail API`

**The 13 automated checks:**
1. Folder structure — files in allowed directories
2. Required files — README.md and metadata.json exist
3. Metadata valid — JSON parses and has all required fields
4. No credentials — no API keys in code
5. SQL safety — no DROP TABLE, no unguarded DELETE
6. Category artifacts — recipes have code or detailed instructions; schemas have SQL; dashboards have frontend code; integrations have code; primitives have 200+ word READMEs; extensions have both SQL and code
7. PR format — title follows `[category]` format
8. No binary blobs — no files over 1MB, no archives
9. README completeness — has Prerequisites, numbered steps, Expected Outcome
10. Primitive dependencies — declared dependencies exist and are linked
11. LLM clarity review — planned for v2, currently auto-passes
12. Scope check — PR only modifies files in its own contribution folder
13. Internal links — relative links in READMEs resolve to real files

---

### 9.4 Exercise: Simulate the Review

Pick an extension and mentally run every check against it:

1. Are all its files in the correct directory? Which rule would catch a misplaced file?
2. Does it have `README.md` and `metadata.json`? Open the metadata and validate every required field against the schema in `.github/metadata.schema.json`.
3. Does the `README.md` contain the words "prerequisite", numbered steps, and "expected outcome"? (Use Ctrl+F to check — that is literally what Rule 9 does.)
4. Are there any SQL files? If so, scan for DROP TABLE, TRUNCATE, or DELETE without WHERE.
5. Does the metadata declare any `requires_primitives`? If so, do those directories exist in `primitives/`?

Then open `CONTRIBUTING.md` and locate a detail about extension READMEs that the automated checks do not catch. What human judgment is required that a script cannot perform?

---

### 9.5 Verification Checkpoints

- [ ] Explain what `$GITHUB_OUTPUT` is and how values written to it are accessed
- [ ] Explain what `actions/github-script` does
- [ ] List the 13 automated checks from memory (approximate)
- [ ] Explain what `if: steps.review.outputs.failed == 'true'` does in YAML
- [ ] Given a PR with a missing `README.md`, identify which check fails and what the error message looks like

**Recommended external resources:**
- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [jq Manual](https://jqlang.github.io/jq/manual/)
- [YAML Specification](https://yaml.org/spec/1.2.2/) — particularly block scalars

---

## PHASE 10: Security Model

**Estimated time:** 4–5 days
**Prerequisites:** Phases 4–7
**Key repo files:**
- `primitives/rls/README.md`
- `primitives/shared-mcp/README.md`
- `SECURITY.md`
- `docs/01-getting-started.md` (Step 3 and Step 5)

---

### 10.1 The Key Hierarchy

Open Brain uses three levels of credentials:

**Supabase Service Role Key:** Superuser-level database access. Bypasses all RLS policies. Used by extension MCP servers (which run on the user's own machine). Never exposed to clients. Treat like a root password.

**Supabase Anon Key:** Limited public access key. Subject to RLS policies. Would be used by frontend dashboards or mobile apps where the key might be visible in client code. Not directly used in the current extension architecture.

**MCP Access Key:** A random hex string you generate. Used to authenticate requests to your core MCP Edge Function. This is what you give to AI clients — it goes in URLs or headers. If compromised, generate a new one with `openssl rand -hex 32` and update the Supabase secret.

The access key pattern prevents your Edge Function from being called by anyone who discovers the Supabase project ref in the URL.

---

### 10.2 Defense in Depth

Open Brain uses multiple overlapping layers of security:

**Layer 1: MCP Access Key** — the first gate. Requests without the correct key are rejected at the Edge Function level before any database query runs.

**Layer 2: Service Role Key confidentiality** — extension servers use the service role key, which is stored in environment variables on the user's machine. It is never in code or config files committed to version control.

**Layer 3: RLS policies** — even if someone obtained the Supabase URL and used the anon key (not the service role key), RLS policies ensure they can only see rows belonging to their authenticated user ID.

**Layer 4: No credentials in code** — the CI/CD review (Rule 4) scans every PR for credential patterns. This prevents accidental secrets in contributed code.

**Layer 5: SQL safety rules** — Rule 5 prevents destructive SQL in contributed schemas.

---

### 10.3 The Shared Server Security Model

Read `primitives/shared-mcp/README.md` in full. The security model relies on:

1. **Separate credentials:** `SUPABASE_HOUSEHOLD_KEY` is a different key from `SUPABASE_SERVICE_ROLE_KEY`. It has limited table-level GRANT permissions.
2. **Limited tool surface:** The shared server only exposes read tools (plus the shopping list update). The full write/delete tools from the main server are absent.
3. **No RLS bypass:** If the household key is not a service role key, RLS policies apply and further restrict access.

The test script in `primitives/shared-mcp/README.md` is the right approach: explicitly test the security boundaries. "Does this client see what it should? Does it NOT see what it should not?" Test both the positive and negative cases.

---

### 10.4 What NOT to Do

The CLAUDE.md and CONTRIBUTING.md document the guard rails. These are non-negotiable:

- Never put credentials in any committed file (even `.env` files if committed)
- Never use `DROP TABLE`, `DROP DATABASE`, `TRUNCATE`, or unguarded `DELETE FROM` in SQL contributions
- Never modify existing columns of the `thoughts` table (adding new columns is fine)
- Never commit binary files over 1MB

These rules exist because this is a community repository. A malicious or careless contribution could wipe users' data or steal credentials if these guards were not in place.

---

### 10.5 Verification Checkpoints

- [ ] Explain when you would use the service role key vs the anon key
- [ ] Explain what happens if the MCP Access Key is compromised and how to rotate it
- [ ] Explain why the CI/CD workflow scans for credentials in code
- [ ] Explain the shared server security model at the database level (GRANT, RLS) and at the application level (limited tool surface)

---

## PHASE 11: Integrations and External Systems

**Estimated time:** 1 week
**Prerequisites:** Phases 3–7
**Key repo files:**
- `integrations/slack-capture/README.md`
- `integrations/discord-capture/README.md`
- `recipes/chatgpt-conversation-import/import-chatgpt.py`

---

### 11.1 The Capture Architecture

Open Brain has two write paths:

**MCP capture tool:** The AI client calls `capture_thought` via MCP. The Edge Function generates an embedding, extracts metadata, stores the row. This is synchronous from the user's perspective — they ask the AI to remember something, the AI calls the tool, the AI confirms.

**Webhook capture (Slack, Discord):** An external service sends an HTTP POST to an Edge Function when an event occurs. The function processes it and stores the row. Asynchronous from the user's perspective — they type in Slack, the reply comes in the thread a few seconds later.

Both paths end at the same `thoughts` table with the same schema. The only difference is the `source` field in the metadata: `"slack"`, `"discord"`, or `"mcp"`.

---

### 11.2 The Slack Capture Integration

Read `integrations/slack-capture/README.md` completely. The architecture:

1. **Slack App:** Created in the Slack API dashboard. Gets scopes `channels:history`, `groups:history`, `chat:write`. Produces a Bot OAuth Token.
2. **Edge Function (`ingest-thought`):** Receives webhook events from Slack. Deployed to Supabase.
3. **Slack Event Subscriptions:** Configured with the Edge Function URL. Sends `message.channels` and `message.groups` events when messages are posted.

**The URL verification handshake.** When you configure the Edge Function URL in Slack's Event Subscriptions, Slack immediately POSTs a `url_verification` challenge:

```json
{ "type": "url_verification", "challenge": "random_string_here" }
```

Your Edge Function must respond with the challenge value. The Edge Function handles this:

```typescript
if (body.type === "url_verification") {
    return new Response(JSON.stringify({ challenge: body.challenge }), {
        headers: { "Content-Type": "application/json" },
    });
}
```

This one-time handshake proves to Slack that you control the endpoint.

**The filtering logic.** Not every message should be stored. The Edge Function filters out:
- Messages from bots (`event.bot_id`)
- Messages that are edits or deletions (`event.subtype`)
- Messages in channels other than the capture channel (`event.channel !== SLACK_CAPTURE_CHANNEL`)
- Empty messages

**The Slack retry problem.** If the Edge Function does not respond within 3 seconds, Slack retries delivery. Since generating an embedding and extracting metadata takes 4–5 seconds, duplicates can occur. This is noted as a known limitation in the troubleshooting section.

---

### 11.3 The ChatGPT Import Recipe

`recipes/chatgpt-conversation-import/import-chatgpt.py` is a standalone Python script that:

1. Reads a ChatGPT export ZIP file
2. Filters trivial conversations (too short, creative tasks, already imported)
3. Summarizes each conversation into 1–3 distilled thoughts via GPT-4o-mini
4. Generates embeddings for each thought
5. Inserts rows into the `thoughts` table

Notable patterns:

**The sync log.** A local `chatgpt-sync-log.json` file tracks which conversations have been imported. This prevents re-importing if you run the script multiple times:

```python
if conv_id in sync_log["ingested_ids"]:
    return "already_imported"
```

**The conversation hash.** Instead of using ChatGPT's internal conversation ID (which may change), a stable hash is computed from the title and creation time:

```python
def conversation_hash(conv):
    raw = f"{title}|{create_time}"
    return hashlib.sha256(raw.encode()).hexdigest()[:16]
```

**The summarization prompt.** The script does not import raw conversation text — it asks GPT-4o-mini to distill the valuable parts:

```
CAPTURE these (1-3 thoughts max):
- Decisions made and the reasoning behind them
- People mentioned with context
- Project plans, strategies, or architectural choices
...

SKIP these entirely (return empty):
- One-off creative tasks
- Generic Q&A or factual lookups
```

This curation is intentional — raw conversation logs are noisy. Distilled thoughts are more valuable in a personal knowledge base.

---

### 11.4 Verification Checkpoints

- [ ] Explain the Slack URL verification handshake
- [ ] Explain why `message.channels` and `message.groups` must both be subscribed
- [ ] Trace the full lifecycle of a Slack message from post to database row
- [ ] Explain why the ChatGPT importer uses a sync log
- [ ] Explain the difference between the MCP capture path and the Slack webhook path

---

## PHASE 12: Capstone — Build Your Own Extension

**Estimated time:** 2–4 weeks
**Prerequisites:** All previous phases
**Goal:** Design and implement a new extension from scratch, following all repo standards

---

### 12.1 Design Phase

Choose a domain that benefits from AI-assisted access (examples: book tracking, workout logging, travel journaling, project management, habit tracking).

Answer these questions before writing a line of code:

1. What is the human problem this extension solves? Write it in one sentence.
2. What tables do you need? What are the columns, types, and relationships?
3. What are the MCP tools? Name them, describe them in one sentence each, and list their parameters.
4. Does this need household sharing? If yes, what tables and operations does a household member need?
5. Does this reference the core `thoughts` table? How?
6. Does this reference another extension? If so, is the FK hard (PostgreSQL-enforced) or soft (application-managed)?
7. What triggers do you need? Draw the trigger dependency graph.
8. What RLS pattern do you need? User-scoped, household-scoped, or mixed?

---

### 12.2 Implementation Order

Follow this order to build your extension:

1. Write `schema.sql` — tables, indexes, RLS, triggers
2. Test it in Supabase SQL Editor (run it, verify tables exist, verify RLS)
3. Write `metadata.json` — validate against `.github/metadata.schema.json`
4. Write `README.md` — prerequisites, numbered steps, expected outcome, troubleshooting
5. Write `package.json` and `tsconfig.json` (copy and modify from an existing extension)
6. Write TypeScript interfaces for each table
7. Write tool definitions (the `TOOLS` array)
8. Write handler functions (one per tool)
9. Write the server setup and `main()` function
10. Build and test locally

---

### 12.3 Testing Checklist

Before submitting a PR:

- [ ] Run the schema SQL in a test Supabase project — no errors
- [ ] Run `npm install && npm run build` — no TypeScript errors
- [ ] Configure the MCP server in Claude Desktop and test each tool manually
- [ ] Verify that user isolation works: create data as user A, confirm user B cannot see it
- [ ] If household-scoped: verify a household member can see shared data but cannot see private data
- [ ] Verify error cases: what happens if required fields are missing? If a UUID is invalid?
- [ ] Check all 13 automated review rules manually (simulate the CI checks)

---

### 12.4 Verification Checkpoints

- [ ] Built and deployed a complete extension that passes all 13 automated review checks
- [ ] Tested all MCP tools via an actual AI client
- [ ] Documented a real troubleshooting scenario encountered during development
- [ ] Can explain every design decision made (schema choices, RLS pattern, tool granularity)

---

## PHASE 13: Advanced Topics

**Estimated time:** Ongoing
**Prerequisites:** All previous phases
**These topics build depth, not breadth — study them as needed**

---

### 13.1 Vector Search Optimization

**The `match_threshold` parameter.** The default is 0.7. This means only results with cosine similarity above 70% are returned. Too high: you miss relevant results. Too low: you get noise. The right value depends on your content and use patterns. Start at 0.7 and adjust based on experience.

**The HNSW `m` and `ef_construction` parameters.** Advanced tuning for the HNSW index:

```sql
CREATE INDEX ON thoughts
    USING HNSW (embedding vector_cosine_ops)
    WITH (m = 16, ef_construction = 64);
```

- `m`: number of connections per node in the graph. Higher = better recall but more memory
- `ef_construction`: size of the dynamic candidate list during construction. Higher = better index quality but slower build time

For personal use (thousands to low millions of rows), the defaults are fine.

**Chunking long content.** Each row in `thoughts` is one thought (typically 1–3 sentences). Longer content should be chunked before embedding — each chunk becomes a separate row. Chunking strategy affects search quality: chunks that are too small lose context, chunks that are too large dilute the semantic signal.

**Hybrid search.** Combine vector similarity with metadata filtering for precision:

```sql
SELECT * FROM match_thoughts(
    query_embedding := $1,
    match_threshold := 0.6,
    match_count := 20,
    filter := '{"type": "task"}'::jsonb
);
```

This returns semantically relevant results that are also tagged as tasks. The vector search narrows by meaning; the metadata filter narrows by category.

---

### 13.2 PostgreSQL Row Limits and Performance

A single-user Open Brain at 20 thoughts/day generates ~7,300 rows per year. At 100 thoughts/day: ~36,500/year. PostgreSQL handles millions of rows comfortably with proper indexing.

The HNSW index maintains performance at large scale. The GIN index on `metadata` enables fast JSONB containment queries. The `created_at DESC` index enables fast date-range browsing.

Supabase's free tier has a 500MB database limit. At 6KB per row (content + 1536-float embedding + metadata), that is ~80,000 rows before hitting the limit. The Pro tier (25GB) allows millions of rows.

---

### 13.3 Building a Dashboard

A simple web dashboard for browsing thoughts would use:

**Supabase JavaScript client in the browser:**

```javascript
const supabase = createClient(SUPABASE_URL, SUPABASE_ANON_KEY);

// After user logs in:
const { data } = await supabase
    .from("thoughts")
    .select("*")
    .order("created_at", { ascending: false })
    .limit(20);
```

Note: browser dashboards use the **anon key**, not the service role key. RLS then controls what the authenticated user can see.

**Authentication flow:** Supabase handles user authentication (email/password, OAuth). After login, Supabase issues a JWT that is automatically included in subsequent queries. RLS policies use `auth.uid()` from this JWT.

**Real-time updates:** Supabase supports WebSocket subscriptions:

```javascript
supabase.channel("thoughts")
    .on("postgres_changes", { event: "INSERT", schema: "public", table: "thoughts" },
        payload => { addThoughtToUI(payload.new); })
    .subscribe();
```

When a new thought is inserted (from any source — MCP, Slack, script), the dashboard updates automatically without polling.

---

### 13.4 Verification Checkpoints for Advanced Topics

- [ ] Explain what `m` and `ef_construction` control in HNSW
- [ ] Design a chunking strategy for a document longer than 500 words
- [ ] Explain why browser dashboards use the anon key instead of the service role key
- [ ] Write a hybrid search query that filters by both semantic similarity and metadata

---

## Quick Reference: Files by Topic

| Topic | Primary Files |
|-------|--------------|
| TypeScript / MCP server structure | `extensions/household-knowledge/index.ts` |
| Simplest schema | `extensions/household-knowledge/schema.sql` |
| Triggers and cascading updates | `extensions/home-maintenance/schema.sql` |
| Multi-table schema with nullable FKs | `extensions/family-calendar/schema.sql` |
| JSONB arrays, TEXT[], household RLS | `extensions/meal-planning/schema.sql` |
| Shared MCP server | `extensions/meal-planning/shared-server.ts` |
| Second trigger pattern | `extensions/professional-crm/schema.sql` |
| 5-table cascade schema, soft FKs | `extensions/job-hunt/schema.sql` |
| Core thoughts table and match_thoughts() | `docs/01-getting-started.md` |
| RLS patterns (all three) | `primitives/rls/README.md` |
| Shared access pattern | `primitives/shared-mcp/README.md` |
| Webhook integration | `integrations/slack-capture/README.md` |
| Python ingestion script | `recipes/chatgpt-conversation-import/import-chatgpt.py` |
| CI/CD workflow | `.github/workflows/ob1-review.yml` |
| metadata.json schema | `.github/metadata.schema.json` |
| Contribution standards | `CONTRIBUTING.md` |

---

## Self-Assessment: Am I Ready to Build and Maintain Open Brain?

Answer these questions without looking at the files. If you cannot answer them confidently, return to the relevant phase.

**Database:**
1. A user deletes their account from Supabase Auth. What happens to their household items? Why?
2. Why does the thoughts table use `TIMESTAMPTZ` instead of `TIMESTAMP`?
3. The `match_thoughts()` function has a `filter` parameter defaulting to `'{}'::jsonb`. What does that default do in the WHERE clause?

**MCP:**
4. An AI client calls a tool that does not exist. What does the MCP server return?
5. Why do extension servers print log messages to `console.error` instead of `console.log`?

**Security:**
6. A contributor accidentally includes their OpenRouter API key in a PR. Which automated check catches it? What pattern matches?
7. Your service role key is compromised. What steps do you take to recover?

**Extension system:**
8. You want to build an extension that shares read access to data with another user. Which primitive should you read first?
9. A new extension wants to reference data from Extension 5's `professional_contacts` table. Should it use a PostgreSQL foreign key or a soft reference? Why?

**CI/CD:**
10. A PR titled "Add new recipe" is submitted. Which automated check fails immediately?

---

*This study plan is designed to be read alongside the actual code. Every claim can be verified by reading the referenced file. When the plan and the code disagree, trust the code.*
