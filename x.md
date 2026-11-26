# MongoDB Cheat Sheet — Units 2 & 3

## Index

- [1. Database and collection commands](#1-database-and-collection-commands)
- [2. CRUD operations](#2-crud-operations)
- [3. Query operators](#3-query-operators)
- [4. Projection, sorting, pagination and cursors](#4-projection-sorting-pagination-and-cursors)
- [5. Arrays and nested documents](#5-arrays-and-nested-documents)
- [6. Schema design and data modeling](#6-schema-design-and-data-modeling)
- [7. Aggregation framework](#7-aggregation-framework)
- [8. Text search](#8-text-search)
- [9. Indexing, optimization and profiling](#9-indexing-optimization-and-profiling)
- [10. Replication and sharding](#10-replication-and-sharding)
- [11. Transactions](#11-transactions)
- [12. Backup, restore and import/export](#12-backup-restore-and-importexport)
- [13. Security and monitoring](#13-security-and-monitoring)
- [14. Application integration (Node.js + Mongoose)](#14-application-integration-nodejs--mongoose)

---

Easy–medium level | `mongosh` syntax unless marked **Terminal** | Examples use an `ecommerce` database.

## 1. Database and collection commands

```javascript
show dbs                         // List databases
use ecommerce                    // Switch database
show collections                 // List collections
db.createCollection("products")  // Create collection
db.products.countDocuments({})    // Count documents
db.stats()                       // Database statistics
```

Sample document (BSON supports types such as Date and ObjectId):

```javascript
{
  _id: 1, name: "Laptop", categoryId: 101, price: 50000,
  stock: 10, tags: ["gaming", "electronics"],
  specifications: { ram: "16GB" },
  createdAt: ISODate("2026-11-20")
}
```

Examples are illustrative; assume referenced collections and fields exist.

## 2. CRUD operations

```javascript
// Insert
db.products.insertOne({ name: "Mouse", price: 500, stock: 20 })
db.products.insertMany([
  { name: "Keyboard", price: 1000 },
  { name: "Monitor", price: 12000 }
])

// Read
db.products.find()
db.products.findOne({ name: "Laptop" })
db.products.find({ price: 500 })

// Update
db.products.updateOne({ name: "Laptop" }, { $set: { price: 48000 } })
db.products.updateMany({ price: { $lt: 1000 } }, { $inc: { price: 100 } })
db.products.updateOne({ _id: 1 }, { $unset: { oldPrice: "" } })
db.products.updateOne({ _id: 1 }, { $inc: { stock: -1 } })

// Replace entire document (except immutable _id)
db.products.replaceOne({ _id: 1 }, { name: "Laptop", price: 50000 })

// Upsert: update if found, otherwise insert
db.products.updateOne({ name: "Tablet" }, { $set: { price: 20000 } }, { upsert: true })

// Delete
db.products.deleteOne({ name: "Mouse" })
db.products.deleteMany({ stock: 0 })
```

`deleteMany({})` removes all documents; `db.products.drop()` removes the collection. Use only intentionally.

## 3. Query operators

| Operator | Example filter | Meaning |
|---|---|---|
| `$eq` | `{ price: { $eq: 500 } }` | Equal |
| `$ne` | `{ price: { $ne: 500 } }` | Not equal |
| `$gt`, `$gte` | `{ price: { $gte: 1000 } }` | Greater than / inclusive |
| `$lt`, `$lte` | `{ price: { $lt: 5000 } }` | Less than / inclusive |
| `$in` | `{ categoryId: { $in: [101, 102] } }` | Any listed value |
| `$nin` | `{ categoryId: { $nin: [101, 102] } }` | None of listed values |
| `$exists` | `{ stock: { $exists: true } }` | Field exists |
| `$type` | `{ price: { $type: "number" } }` | BSON type |
| `$regex` | `{ name: { $regex: "^L", $options: "i" } }` | Starts with L, ignoring case |

```javascript
// Range and implicit AND
db.products.find({ price: { $gte: 500, $lte: 5000 }, stock: { $gt: 0 } })
// Explicit AND, OR, NOR, NOT
db.products.find({ $and: [{ price: { $gt: 500 } }, { stock: { $gt: 0 } }] })
db.products.find({ $or: [{ price: { $lt: 1000 } }, { categoryId: 101 }] })
db.products.find({ $nor: [{ stock: 0 }, { price: { $gt: 50000 } }] })
db.products.find({ price: { $not: { $gt: 5000 } } })
// Compare two fields in the same document
db.products.find({ $expr: { $gt: ["$sellingPrice", "$costPrice"] } })
// Date filter
db.products.find({ createdAt: { $gte: ISODate("2026-11-01") } })
```

## 4. Projection, sorting, pagination and cursors

```javascript
db.products.find({}, { name: 1, price: 1, _id: 0 }) // Include fields
db.products.find({}, { specifications: 0 })         // Exclude field
db.products.find().sort({ price: -1 }).limit(3)     // Top 3 by price
db.products.find().sort({ _id: 1 }).skip(5).limit(5) // Page 2, size 5

db.products.distinct("categoryId")
db.products.countDocuments({ stock: { $gt: 0 } })

const cursor = db.products.find({ price: { $gt: 1000 } })
while (cursor.hasNext()) { printjson(cursor.next()) }
db.products.find().forEach(p => print(p.name))
db.products.find().limit(5).toArray()
```

`1` = ascending/include, `-1` = descending, `0` = exclude. Projection cannot normally mix inclusion and exclusion except `_id`. A cursor retrieves results progressively; avoid `toArray()` for huge datasets.

## 5. Arrays and nested documents

```javascript
db.products.find({ "specifications.ram": "16GB" })
db.products.find({ tags: "gaming" })
db.products.find({ tags: { $all: ["gaming", "electronics"] } })
db.products.find({ tags: { $in: ["gaming", "sports"] } })
db.products.find({ tags: { $size: 2 } })

// Same array element must satisfy both conditions
db.products.find({ reviews: { $elemMatch: { rating: { $gte: 4 }, verified: true } } })

// Update arrays and nested fields
db.products.updateOne({ _id: 1 }, { $push: { tags: "sale" } })
db.products.updateOne({ _id: 1 }, { $addToSet: { tags: "sale" } })
db.products.updateOne({ _id: 1 }, { $pull: { tags: "sale" } })
db.products.updateOne({ _id: 1 }, { $pop: { tags: 1 } }) // Remove last
db.products.updateOne({ _id: 1 }, { $set: { "specifications.ram": "32GB" } })
```

`$push` can add duplicates; `$addToSet` avoids adding an existing value.

## 6. Schema design and data modeling

```javascript
// Embedding: related data inside one document
{ _id: 1, name: "Rahul", address: { city: "Delhi", pin: 110001 } }

// Referencing: related data in separate collections
// products
{ _id: 1, name: "Laptop", categoryId: 101 }
// categories
{ _id: 101, name: "Electronics" }
```

| Pattern | Common use |
|---|---|
| Embedding | Small, bounded data read together (address, specifications) |
| Referencing | Large or independently updated data (orders, comments) |
| Computed | Store frequently calculated totals (likesCount) |
| Subset | Embed only recent reviews; keep all reviews separately |
| Bucket | Group time-based measurements into bounded batches |

## 7. Aggregation framework

Pipeline syntax: `db.products.aggregate([{ stage1 }, { stage2 }])`.

| Stage | Purpose |
|---|---|
| `$match` | Filter documents |
| `$group` | Group and calculate |
| `$project` | Select or compute fields |
| `$addFields` | Add calculated fields |
| `$sort`, `$limit`, `$skip` | Sort and paginate |
| `$unwind` | Split array elements into documents |
| `$lookup` | Join collections; returns an array |
| `$count` | Count pipeline documents |
| `$facet` | Run multiple sub-pipelines on the same input |
| `$out` | Save results to a collection (replaces target) |

### A. Filter, group and sort

Find average price and product count in each category for in-stock products.

```javascript
db.products.aggregate([
  { $match: { stock: { $gt: 0 } } },
  { $group: {
    _id: "$categoryId",
    totalProducts: { $sum: 1 },
    averagePrice: { $avg: "$price" }
  } },
  { $sort: { averagePrice: -1 } }
])
```

### B. Overall statistics

```javascript
db.products.aggregate([
  { $group: {
    _id: null,
    totalStock: { $sum: "$stock" },
    cheapest: { $min: "$price" },
    costliest: { $max: "$price" }
  } }
])
```

`_id: null` groups all documents together. Other accumulators: `$push` (all values), `$addToSet` (unique values), `$first`, `$last` (use sorting for meaningful order).

### C. Calculated fields and projection

```javascript
db.products.aggregate([
  { $addFields: { discountedPrice: { $multiply: ["$price", 0.9] } } },
  { $project: { _id: 0, name: 1, discountedPrice: 1 } }
])
```

### D. Unwind — count products per tag

```javascript
db.products.aggregate([
  { $unwind: "$tags" },
  { $group: { _id: "$tags", count: { $sum: 1 } } },
  { $sort: { count: -1 } }
])
```

### E. Lookup — retrieve products with category names

```javascript
db.products.aggregate([
  { $lookup: {
    from: "categories", localField: "categoryId",
    foreignField: "_id", as: "category"
  } },
  { $unwind: "$category" },
  { $project: { name: 1, categoryName: "$category.name" } }
])
```

Use `preserveNullAndEmptyArrays: true` with `$unwind` (object syntax with `path`) if unmatched products must remain.

### F. Count reviews — include products with zero reviews

```javascript
db.products.aggregate([
  { $lookup: {
    from: "reviews", localField: "_id",
    foreignField: "productId", as: "reviews"
  } },
  { $project: { name: 1, totalReviews: { $size: "$reviews" } } }
])
```

If zero-review products are not needed, simply group `db.reviews` by `$productId` using `$sum: 1`.

### G. Count, pagination and facet

```javascript
db.products.aggregate([{ $match: { stock: { $gt: 0 } } }, { $count: "total" }])
db.products.aggregate([{ $sort: { _id: 1 } }, { $skip: 5 }, { $limit: 5 }])

db.products.aggregate([{ $facet: {
  statistics: [{ $count: "totalProducts" }],
  topProducts: [{ $sort: { price: -1 } }, { $limit: 3 }]
} }])
```

**`$` rule:** `$group` is an operator; `"$price"` refers to a field value in an expression; `price` is an ordinary field key; `"price"` is a literal string in an expression.

## 8. Text search

```javascript
db.products.createIndex({ name: "text", description: "text" })
db.products.find({ $text: { $search: "gaming laptop" } }) // Either term
db.products.find({ $text: { $search: '"gaming laptop"' } }) // Phrase
db.products.find({ $text: { $search: "laptop -gaming" } }) // Exclude term

db.products.find(
  { $text: { $search: "laptop" } },
  { name: 1, score: { $meta: "textScore" } }
).sort({ score: { $meta: "textScore" } })
```

`$text` requires a text index. A collection can have one text index covering multiple fields.

## 9. Indexing, optimization and profiling

```javascript
db.products.createIndex({ price: 1 })                 // Single field
db.products.createIndex({ categoryId: 1, price: 1 })  // Compound
db.products.createIndex({ tags: 1 })                  // Multikey (array)
db.users.createIndex({ email: 1 }, { unique: true })   // Unique
db.logs.createIndex({ createdAt: 1 }, { expireAfterSeconds: 86400 }) // TTL
db.products.createIndex({ categoryId: "hashed" })      // Hashed

db.products.getIndexes()
db.products.dropIndex("price_1")
```

### Analyze a query before and after creating an index

```javascript
db.products.find({ price: { $gt: 1000 } }).explain("executionStats")
db.products.createIndex({ price: 1 })
db.products.find({ price: { $gt: 1000 } }).explain("executionStats")
```

| Explain field | Meaning |
|---|---|
| `COLLSCAN` / `IXSCAN` | Collection scan / index scan |
| `totalDocsExamined` | Documents checked |
| `totalKeysExamined` | Index keys checked |
| `nReturned` | Documents returned |
| `executionTimeMillis` | Execution time in milliseconds |

Use a sufficiently large dataset and selective query for a meaningful comparison; an index is not guaranteed to be selected.

```javascript
// Database profiler (self-managed mongod; requires privileges)
db.setProfilingLevel(1, { slowms: 100 }) // Slow operations
db.system.profile.find().sort({ ts: -1 }).limit(5)
db.getProfilingStatus()
db.setProfilingLevel(0)                 // Disable
```

Levels: 0 = off, 1 = slow/filtered, 2 = all (higher overhead). Optimize by indexing frequent filters, placing `$match` early, avoiding unnecessary joins and limiting results. Indexes consume space and increase write cost.

## 10. Replication and sharding

| Replication | Sharding |
|---|---|
| Copies data across nodes | Distributes data across shards |
| High availability and failover | Horizontal scaling |
| Primary and secondary nodes | mongos, config servers, shards |

```javascript
// On a configured self-managed replica set
rs.initiate()                 // Initialize
rs.status()                   // Check members and state
rs.conf()                     // View configuration
rs.add("host2:27017")         // Add a configured member

// On an already configured sharded cluster, connected to mongos
sh.status()
sh.shardCollection("ecommerce.products", { categoryId: "hashed" })
```

Sharding an existing populated collection requires a suitable supporting index. Choose a shard key based on distribution and common queries; categoryId is illustrative, not necessarily a good production choice. Replication copies oplog operations; an eligible secondary may become primary after an election. Replication is not a backup.

## 11. Transactions

Example: reduce product stock and create an order atomically. Requires a replica set or sharded cluster.

```javascript
const session = db.getMongo().startSession()
const shop = session.getDatabase("ecommerce")
session.startTransaction()
try {
  const result = shop.products.updateOne(
    { _id: 1, stock: { $gte: 1 } }, { $inc: { stock: -1 } }
  )
  if (result.matchedCount === 0) throw new Error("Out of stock")
  shop.orders.insertOne({ productId: 1, quantity: 1 })
  session.commitTransaction()
} catch (error) {
  session.abortTransaction()
  print(error.message)
} finally {
  session.endSession()
}
```

ACID = Atomicity, Consistency, Isolation, Durability. Single-document writes are atomic by default. This is a learning example; production transactions also need retry/error handling.

## 12. Backup, restore and import/export

**Terminal commands** (MongoDB Database Tools):

```bash
mongodump --db ecommerce --out ./backup
mongorestore --nsInclude="ecommerce.*" ./backup
mongoexport --db ecommerce --collection products --out products.json
mongoimport --db ecommerce --collection products --file products.json
```

`mongodump` creates BSON backups; `mongoexport` is for JSON/CSV interchange, not full backups. For production, use a backup method that ensures consistency during active writes.

## 13. Security and monitoring

```javascript
// mongosh: run as an authorized administrator
use ecommerce
db.createUser({
  user: "appUser", pwd: passwordPrompt(),
  roles: [{ role: "readWrite", db: "ecommerce" }]
})
db.getUsers()
db.serverStatus()
db.currentOp()               // Current operations (privileges required)
```

**Terminal:**

```bash
mongosh --username appUser --authenticationDatabase ecommerce
mongostat                     # Real-time operation statistics
mongotop                      # Collection read/write time
```

Authentication = verify identity; authorization = verify permissions. Common roles: `read`, `readWrite`, `dbAdmin`, `userAdmin`. Enable access control, use TLS, restrict network access and use least privilege. Compass and Atlas provide graphical monitoring/analysis tools.

## 14. Application integration (Node.js + Mongoose)

```javascript
import mongoose from "mongoose"
await mongoose.connect(process.env.MONGO_URI)

const Product = mongoose.model("Product", new mongoose.Schema({
  name: String, price: Number
}))
const products = await Product.find({ price: { $gt: 1000 } })
```

Reuse the database connection and its connection pool. Keep credentials in environment variables. For deployment, consider indexes, backups, security, monitoring and availability.

---
**Revision priority:** CRUD → query operators → arrays → aggregation (`$match`, `$group`, `$lookup`, `$unwind`, `$project`) → indexes and `explain()` → transactions → administration commands.
