
# Redis with Node.js and MongoDB

[![Redis](https://img.shields.io/badge/Redis-red?logo=redis&logoColor=white)](https://redis.io/)  
[![Node.js](https://img.shields.io/badge/Node.js-green?logo=node.js&logoColor=white)](https://nodejs.org/)  
[![MongoDB](https://img.shields.io/badge/MongoDB-brightgreen?logo=mongodb&logoColor=white)](https://www.mongodb.com/)  

## Overview
Redis is an **in-memory data structure store** used as a cache, database, and message broker.  
When combined with **Node.js** and **MongoDB**, it can dramatically improve performance by caching frequently accessed queries.

**Why Redis?**
- Extremely fast (in-memory).
- Supports caching, queues, pub/sub, and rate limiting.
- Reduces load on MongoDB by serving cached responses.

---

## Installation

1. Install Redis (locally or with Docker):  
   - [Install Redis Docs](https://redis.io/docs/getting-started/installation/)
     
   - Docker:  
     ```bash
     docker run --name redis -p 6379:6379 -d redis
     ```

2. Install Node.js dependencies:
   ```bash
   npm init -y
   npm install express mongoose redis
   ```


---

## Basic Setup

### 1. Connect to MongoDB

```js
import mongoose from "mongoose";

export default async function connect(){
try{
let con=await mongoose.connect(process.env.MONGO_URI);
console.log(`Mongo fired up..`)
}
catch(err){
console.error(`Error connecting to mongo:${err}`)}}

```

### 2. Connect to Redis

```js
import { createClient } from "redis";

export default async function redis(){
try{
const red = createClient({ url: "redis://127.0.0.1:6379" });
await red.connect();
console.log(`Redis fired up`);}
red.on("error",(err)=>{console.log(`Error while redising:${err}`)})
catch(err){
console.error("Redis Error:", err);}}
```

---

## Cache-Aside Pattern

### User by ID (Read Flow)

```js
app.get("/user/:id", async (req, res) => {
  const userId = req.params.id;

  // Step 1: Check Redis
  const cached = await redis.get(`user:${userId}`);
  if (cached) {
    return res.json({ source: "redis", data: JSON.parse(cached) });
  }

  // Step 2: Query MongoDB
  const user = await User.findById(userId);
  if (!user) return res.status(404).json({ error: "User not found" });

  // Step 3: Save in Redis (TTL = 60s)
  await redis.setEx(`user:${userId}`, 60, JSON.stringify(user));

  res.json({ source: "mongodb", data: user });
});
```

### User Update (Write Flow)

```js
app.put("/user/:id", async (req, res) => {
  const userId = req.params.id;

  const updated = await User.findByIdAndUpdate(userId, req.body, { new: true });
  if (!updated) return res.status(404).json({ error: "User not found" });

  // Refresh cache
  await redis.setEx(`user:${userId}`, 60, JSON.stringify(updated));

  res.json(updated);
});
```

---

## Common Redis Commands

| Command    | Description                  | Example                                      |
| ---------- | ---------------------------- | -------------------------------------------- |
| `set`      | Set a key                    | `await redis.set("key", "value")`            |
| `get`      | Get a key                    | `await redis.get("key")`                     |
| `setEx`    | Set with expiry              | `await redis.setEx("key", 60, "value")`      |
| `del`      | Delete a key                 | `await redis.del("key")`                     |
| `expire`   | Set TTL (seconds)            | `await redis.expire("key", 120)`             |
| `hSet`     | Store object/hash            | `await redis.hSet("user:1", {name:"Nishu"})` |
| `hGetAll`  | Get all fields of a hash     | `await redis.hGetAll("user:1")`              |
| `sAdd`     | Add values to a set (unique) | `await redis.sAdd("tags", "redis", "db")`    |
| `sMembers` | Get members of a set         | `await redis.sMembers("tags")`               |

---

## Key Points

* Use **cache-aside pattern**:

  * Read: `check cache → query DB if miss → set cache`.
  * Update: `update DB → refresh/delete cache`.
* Redis TTL ensures data does not become stale.
* Always design cache **invalidation** when updating DB.

---

## Resources

* [Redis Documentation](https://redis.io/docs/)
* [Node.js Redis Client (npm)](https://www.npmjs.com/package/redis)
* [Mongoose Documentation](https://mongoosejs.com/docs/)
* [License](LICENSE)

