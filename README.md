# Redis, Redis Queue & BullMQ — Revision Notes

> Based on Redis course by Hitesh Choudhary (Chai aur Code)

---

## 1. What is Redis?

Redis (**RE**mote **DI**ctionary **S**erver) is an **in-memory**, key-value **NoSQL data store**. Because data lives in RAM (not disk), reads/writes are extremely fast — commonly used for:

- Caching (reduce DB load)
- Session storage
- Rate limiting
- Real-time leaderboards
- Pub/Sub messaging
- Job/Message queues (via BullMQ, Bee-Queue, etc.)

**Key characteristics:**
- Single-threaded (mostly) → commands execute atomically
- Data structures: Strings, Hashes, Lists, Sets, Sorted Sets, Streams
- Persistence options: RDB (snapshot), AOF (append-only file)
- Default port: `6379`

### Starting Redis (CLI)
```bash
# start redis server
redis-server

# open redis CLI to run commands
redis-cli

# check connection
PING
# → PONG
```

---

## 1a. Node.js Setup — `ioredis`

This course/notes uses **two** possible Node clients: the official `redis` package and `ioredis`. `ioredis` is the one BullMQ uses internally, and it's very popular for its clean Promise-based API and cluster support.

### Install
```bash
npm install ioredis
```

### Connect
```js
import Redis from "ioredis";

// simplest connection (defaults to localhost:6379)
const redis = new Redis();

// OR with explicit config
const redis = new Redis({
  host: "127.0.0.1",
  port: 6379,
  password: "",     // if you have one set
  db: 0,
});

redis.on("connect", () => console.log("Connected to Redis"));
redis.on("error", (err) => console.log("Redis error:", err));
```

> Note: With `ioredis` you **don't** need to call `.connect()` manually like the `redis` package — it connects automatically on first command (lazy connect is also configurable).

---

## 2. Basic Key-Value Commands

Redis at its core stores everything as **key → value** pairs.

### SET — store a value
```bash
SET name "Hitesh"
# → OK
```

### GET — retrieve a value
```bash
GET name
# → "Hitesh"
```

### DEL — delete a key
```bash
DEL name
# → (integer) 1   (returns number of keys deleted)
```

### Other useful string/key commands
```bash
EXISTS name          # check if key exists → 0 or 1
EXPIRE name 60        # key auto-deletes after 60 seconds (TTL)
TTL name               # check remaining time to live
SET name "Hitesh" EX 60   # set value with expiry in one line
KEYS *                 # list all keys (avoid in production - slow)
FLUSHALL               # delete everything (careful!)
```

### Node.js example — `redis` package
```js
import { createClient } from "redis";

const client = createClient();
client.on("error", (err) => console.log("Redis Client Error", err));

await client.connect();

await client.set("name", "Hitesh");
const value = await client.get("name");
console.log(value); // "Hitesh"

await client.del("name");
```

### Node.js example — `ioredis` package
```js
import Redis from "ioredis";
const redis = new Redis();

await redis.set("name", "Hitesh");
const value = await redis.get("name");
console.log(value); // "Hitesh"

await redis.del("name");

// with expiry (EX = seconds)
await redis.set("name", "Hitesh", "EX", 60);

// check ttl
const ttl = await redis.ttl("name");
console.log(ttl); // seconds remaining
```

---

## 3. Storing Objects — Hashes

A **Hash** is like storing a JSON object / a row of a table under a single key. Instead of creating multiple string keys (`user:1:name`, `user:1:email`...), you store them as **fields inside one hash key**.

### HSET — store field-value pairs inside a hash
```bash
HSET user:1 name "Hitesh" email "hitesh@chaicode.com" age "30"
```

### HGETALL — get all fields & values of a hash
```bash
HGETALL user:1
# → 1) "name"
#   2) "Hitesh"
#   3) "email"
#   4) "hitesh@chaicode.com"
#   5) "age"
#   6) "30"
```

### Other hash commands
```bash
HGET user:1 name       # get single field → "Hitesh"
HDEL user:1 age         # delete one field from hash
HEXISTS user:1 name    # check if field exists
HKEYS user:1            # get all field names
HVALS user:1            # get all values
```

### Node.js example — `redis` package
```js
await client.hSet("user:1", {
  name: "Hitesh",
  email: "hitesh@chaicode.com",
  age: "30",
});

const user = await client.hGetAll("user:1");
console.log(user);
// { name: 'Hitesh', email: 'hitesh@chaicode.com', age: '30' }
```

### Node.js example — `ioredis` package
```js
// hset(key, field, value, field, value, ...) OR pass an object
await redis.hset("user:1", {
  name: "Hitesh",
  email: "hitesh@chaicode.com",
  age: "30",
});

const user = await redis.hgetall("user:1");
console.log(user);
// { name: 'Hitesh', email: 'hitesh@chaicode.com', age: '30' }

const name = await redis.hget("user:1", "name");
console.log(name); // "Hitesh"

await redis.hdel("user:1", "age");
```

**Use case:** Storing a user profile, product details, or any structured object under one key instead of multiple flat keys.

---

## 4. Lists — Queue-like structure

Redis **Lists** are ordered collections of strings — useful for building simple queues/stacks.

### LPUSH — push to the LEFT (head) of the list
```bash
LPUSH tasks "task1"
LPUSH tasks "task2"
# list is now: task2, task1
```

### RPUSH — push to the RIGHT (tail) of the list
```bash
RPUSH tasks "task3"
# list: task2, task1, task3
```

### LPOP — remove & return from the LEFT (head)
```bash
LPOP tasks
# → "task2"
```

### RPOP — remove & return from the RIGHT (tail)
```bash
RPOP tasks
# → "task3"
```

### Other list commands
```bash
LRANGE tasks 0 -1     # view all elements (0 to -1 = full list)
LLEN tasks             # get length of list
LINDEX tasks 0        # get element at index
```

### Node.js example — `redis` package
```js
await client.lPush("tasks", "task1");
await client.lPush("tasks", "task2");
await client.rPush("tasks", "task3");

const all = await client.lRange("tasks", 0, -1);
console.log(all); // ['task2', 'task1', 'task3']

const first = await client.lPop("tasks");
const last = await client.rPop("tasks");
```

### Node.js example — `ioredis` package
```js
await redis.lpush("tasks", "task1");
await redis.lpush("tasks", "task2");
await redis.rpush("tasks", "task3");

const all = await redis.lrange("tasks", 0, -1);
console.log(all); // ['task2', 'task1', 'task3']

const first = await redis.lpop("tasks");
const last = await redis.rpop("tasks");

const length = await redis.llen("tasks");
console.log(length);
```

**Note:** `LPUSH` + `RPOP` (or `RPUSH` + `LPOP`) gives FIFO queue behavior — this is the basic idea behind a **Redis Queue**, which tools like BullMQ build upon with more features (retries, delays, concurrency, etc.)

---

## 5. Redis Pub/Sub (Publish/Subscribe)

Pub/Sub lets one client **publish** messages to a "channel", and any number of clients **subscribed** to that channel receive the message **in real-time**. Messages are **not stored** — if no one is subscribed at the time, the message is lost.

### CLI example (open 2 terminals)

**Terminal 1 — Subscriber**
```bash
SUBSCRIBE news
# waits here for messages...
```

**Terminal 2 — Publisher**
```bash
PUBLISH news "Breaking: Redis is fast!"
# → (integer) 1   (number of subscribers who received it)
```
Terminal 1 instantly receives the message.

### Node.js example — `redis` package
Pub/Sub requires **two separate client connections** — one for publishing, one for subscribing (a subscribed client can't run normal commands).

```js
import { createClient } from "redis";

const subscriber = createClient();
const publisher = createClient();

await subscriber.connect();
await publisher.connect();

// Subscribe to a channel
await subscriber.subscribe("news", (message) => {
  console.log("Received:", message);
});

// Publish a message
await publisher.publish("news", "Breaking: Redis is fast!");
```

### Node.js example — `ioredis` package
Same rule applies: **create a separate connection for the subscriber** — once a client calls `.subscribe()`, it can only be used for Pub/Sub commands.

```js
import Redis from "ioredis";

const subscriber = new Redis();
const publisher = new Redis();

// Subscribe to a channel
subscriber.subscribe("news", (err, count) => {
  if (err) console.error("Failed to subscribe:", err);
  else console.log(`Subscribed to ${count} channel(s)`);
});

// Listen for incoming messages
subscriber.on("message", (channel, message) => {
  console.log(`Received on ${channel}:`, message);
});

// Publish a message
await publisher.publish("news", "Breaking: Redis is fast!");

// Pattern-based subscribe (wildcards)
subscriber.psubscribe("news.*", (err, count) => {
  console.log(`Pattern subscribed to ${count} channel(s)`);
});
subscriber.on("pmessage", (pattern, channel, message) => {
  console.log(`[${pattern}] ${channel}:`, message);
});
```

**Use cases:** real-time chat, live notifications, broadcasting events between microservices.

**Limitation:** No message persistence/replay, no acknowledgment, no queue/retry logic → for that, use **Redis Streams** or **BullMQ**.

---

## 6. Redis Queue (Concept)

A "queue" built on Redis lists (LPUSH/RPOP) is simple but lacks production features like:
- Job retries on failure
- Delayed/scheduled jobs
- Concurrency control
- Job priority
- Progress tracking
- Dead-letter/failed job handling

This is exactly what **BullMQ** provides — a robust queue system built on top of Redis.

---

## 7. BullMQ

**BullMQ** is a Node.js library for creating robust **job/message queues** backed by Redis. It has 3 main building blocks:

| Concept | Purpose |
|---|---|
| **Queue** | Where jobs are added (producer side) |
| **Worker** | Processes jobs from the queue (consumer side) |
| **Job** | A single unit of work with data & options |

### Install
```bash
npm install bullmq ioredis
```

### Setting up a Redis connection
BullMQ uses **`ioredis`** internally, so you can either pass a plain config object, or pass an actual `ioredis` instance directly (useful if you want to reuse the same connection elsewhere, like in Pub/Sub or caching).

```js
import { Queue, Worker } from "bullmq";
import Redis from "ioredis";

// Option 1: plain config object (BullMQ creates the ioredis client internally)
const connection = {
  host: "localhost",
  port: 6379,
};

// Option 2: pass your own ioredis instance
const connection2 = new Redis({
  host: "localhost",
  port: 6379,
  maxRetriesPerRequest: null, // required by BullMQ when passing your own instance
});
```

### Creating a Queue (Producer)
```js
const emailQueue = new Queue("emailQueue", { connection });

// Adding a job to the queue
async function addJob() {
  await emailQueue.add(
    "sendWelcomeEmail",          // job name
    { to: "user@example.com", subject: "Welcome!" }, // job data
    {
      attempts: 3,                 // retry 3 times on failure
      backoff: { type: "exponential", delay: 1000 },
      delay: 5000,                  // run after 5 seconds
    }
  );
  console.log("Job added to queue");
}

addJob();
```

### Creating a Worker (Consumer)
The **Worker** listens to the queue and actually processes jobs.

```js
const worker = new Worker(
  "emailQueue",
  async (job) => {
    console.log(`Processing job: ${job.name}`);
    console.log("Job data:", job.data);

    // simulate sending an email
    // throw new Error("failed") // <- would trigger a retry

    return { status: "sent" };
  },
  { connection }
);

// Event listeners
worker.on("completed", (job) => {
  console.log(`Job ${job.id} completed`);
});

worker.on("failed", (job, err) => {
  console.log(`Job ${job.id} failed: ${err.message}`);
});
```

### Other useful BullMQ features

```js
// Get job counts
const counts = await emailQueue.getJobCounts();
console.log(counts); // { waiting, active, completed, failed, delayed }

// Pause / resume a queue
await emailQueue.pause();
await emailQueue.resume();

// Remove a job
await emailQueue.removeJobScheduler("jobId");

// Concurrency - process multiple jobs at once in a worker
const worker2 = new Worker(
  "emailQueue",
  async (job) => { /* ... */ },
  { connection, concurrency: 5 } // process 5 jobs in parallel
);
```

### QueueEvents (listen to events from another process)
```js
import { QueueEvents } from "bullmq";

const queueEvents = new QueueEvents("emailQueue", { connection });

queueEvents.on("completed", ({ jobId }) => {
  console.log(`${jobId} has completed`);
});

queueEvents.on("failed", ({ jobId, failedReason }) => {
  console.log(`${jobId} failed: ${failedReason}`);
});
```

---

## 8. Quick Command Cheat-Sheet

| Category | Command | Description |
|---|---|---|
| Basic | `SET key value` | Set a string value |
| Basic | `GET key` | Get a string value |
| Basic | `DEL key` | Delete a key |
| Basic | `EXPIRE key sec` | Set TTL |
| Hash | `HSET key field value` | Set hash field |
| Hash | `HGETALL key` | Get all hash fields/values |
| Hash | `HGET key field` | Get one field |
| Hash | `HDEL key field` | Delete a field |
| List | `LPUSH key value` | Push to left/head |
| List | `RPUSH key value` | Push to right/tail |
| List | `LPOP key` | Pop from left |
| List | `RPOP key` | Pop from right |
| List | `LRANGE key 0 -1` | View full list |
| Pub/Sub | `SUBSCRIBE channel` | Listen to channel |
| Pub/Sub | `PUBLISH channel msg` | Send message |

---

## 9. Mental Model Summary

```
Redis (in-memory store)
│
├── Simple KV ops → GET/SET/DEL  → caching, sessions
├── Hashes → HSET/HGETALL         → storing objects (like a row)
├── Lists → LPUSH/RPOP            → basic queue/stack structures
├── Pub/Sub → SUBSCRIBE/PUBLISH  → real-time messaging (no persistence)
└── BullMQ (built on Redis)
      ├── Queue  → add jobs
      ├── Worker → process jobs
      └── Job    → unit of work (with retries, delay, priority)
```

---

## 10. Things to revise/practice next
- [ ] Redis Sets (`SADD`, `SMEMBERS`, `SINTER`)
- [ ] Redis Sorted Sets (`ZADD`, `ZRANGE`) — for leaderboards
- [ ] Redis Streams — persistent Pub/Sub alternative
- [ ] BullMQ job priorities & rate limiting
- [ ] BullMQ Flows (parent-child jobs)
- [ ] Redis persistence: RDB vs AOF
- [ ] Redis + Node.js caching layer in front of a real DB (e.g., MongoDB/Postgres)
