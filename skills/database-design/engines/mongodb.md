# MongoDB

Load when the project uses MongoDB, Atlas, or DocumentDB. One engine file per review.

**MongoDB has no schema unless you attach one, and no foreign keys at all.** Everything a relational
database would refuse, MongoDB accepts. That makes two questions decisive on every review:

1. **Is any array unbounded?** That is the failure that ends in a hard 16 MiB error in production.
2. **Does anything validate writes on the server?** If validation lives only in Mongoose or Zod, then a
   migration script, a bulk import, Compass, or a second service writes whatever it likes.

Every entry has a **Fine when** line. Read it before reporting.

## Contents

**Modeling** — unbounded arrays · the 16 MiB limit · string vs ObjectId references · `$lookup` ·
two-sided many-to-many · denormalized copies drifting · time series
**Validation** — no `$jsonSchema` · weak `$jsonSchema` · Mongoose `runValidators`
**Indexes and queries** — `unique` on an optional field · Mongoose `unique` without the index ·
`$elemMatch`
**Infrastructure** — transactions need a replica set

---

## Unbounded arrays

**Signal:** an embedded array whose length grows with usage or time — `comments: [CommentSchema]`,
`events`, `messages`, `followers`, `history`, `logs`. In query code: `$push` on that field with no
`$slice` and no matching `$pull`.

**Why it breaks:** every read of the parent loads the whole array, and WiredTiger **rewrites the entire
document on every append**, so write cost grows with array length. Index entries are generated per array
element (capped at 100,000 keys per document), so the index bloats too. Eventually inserts fail outright
at the 16 MiB BSON limit — in production, on your busiest document.

**Fix:** bound it at write time, and keep the full set in its own collection.

```js
db.posts.updateOne({ _id }, { $push: { recentComments: { $each: [c], $slice: -20 } } })

db.comments.insertOne({ postId, body, createdAt })
db.comments.createIndex({ postId: 1, createdAt: -1 })
```

**Fine when:** the array has a real-world ceiling that is small and known — order line items, a user's
three addresses, OAuth providers, a permissions list.

**The test is not "is it an array" but "can a user or a cron job make this grow forever."** Under ~200
elements always read with the parent, embedding is the correct MongoDB model and splitting it out is the
mistake.

**Confirm it:**

```js
db.coll.aggregate([{ $project: { n: { $size: { $ifNull: ["$arrayField", []] } } } },
                   { $sort: { n: -1 } }, { $limit: 5 }])
```

---

## The 16 MiB document limit

**Signal:** a document type that accumulates — a `chat` embedding `messages`, an `order` embedding
`auditTrail` — or documents nested more than a handful of levels.

**Why it breaks:** the BSON limit is 16 mebibytes per document and the nesting limit is 100 levels. Both
are **hard errors, not warnings**. A 16 MB document also destroys the working set, because MongoDB pages
the whole thing into RAM to read one field. Failure arrives as `BSONObjectTooLarge` on a write that used
to work.

**Find your real distribution before deciding:**

```js
db.coll.aggregate([{ $project: { size: { $bsonSize: "$$ROOT" } } },
                   { $sort: { size: -1 } }, { $limit: 5 }])
```

Documents routinely over 1 MB mean the model is wrong, not just tight.

**Fine when:** documents are small and bounded. 16 MB is enormous — the vast majority of well-modelled
documents are under 4 KB. **Do not raise this generically.** Raise it only when an unbounded array or a
large embedded blob is present.

Storing files in documents at all is the actual defect in that second case — use GridFS or object
storage.

---

## String vs ObjectId references

**Signal:** a reference field declared as a string — `authorId: { type: String, ref: 'User' }` — or app
code doing `{ userId: req.params.id }` where the target's `_id` is an ObjectId. Also `$lookup` where
`localField` is a string and `foreignField` is `_id`.

**Why it breaks:** BSON is strictly typed and comparison is type-sensitive. The string `"6512ab..."`
**never equals** `ObjectId("6512ab...")`. `$lookup` returns an empty array and `find()` returns zero
documents — no error, no warning. Teams discover it weeks later as "the related data just doesn't show
up sometimes."

**Fix:**

```js
authorId: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true }

const id = new ObjectId(req.params.id)   // cast once, at the edge; throws on bad input
db.posts.find({ authorId: id })
```

**Fine when:** the referenced key genuinely is a string — a Stripe customer id, an Auth0 `sub`, a slug
used as `_id`, a tenant code. **The defect is a type mismatch between the two sides**, not the use of a
string. Also fine in Prisma's MongoDB connector, where `@db.ObjectId` on a String field maps correctly.

**Confirm what already exists:**

```js
db.posts.aggregate([{ $group: { _id: { $type: "$authorId" }, n: { $sum: 1 } } }])
```

---

## `$lookup`

**Signal:** `$lookup` with a `foreignField` that has no index declared in any model or migration. Or
three or more `$lookup` stages chained in one pipeline.

**Why it breaks:** an equality `$lookup` without an index on `foreignField` runs a **full collection scan
per input document**. At 10k input docs against a 1M-doc collection that is 10 billion comparisons.
MongoDB's own docs say excessive `$lookup` means you should be embedding instead.

**Fix:**

```js
db.comments.createIndex({ postId: 1 })     // must match foreignField exactly
```

In the correlated `let`/`pipeline` form, **indexes are not used for field-to-field `$expr` comparisons**,
and multikey, partial, and sparse indexes are never used there at all.

**Fine when:** one `$lookup` on an indexed foreign key, in a reporting or admin query. Do not tell the
user to denormalise a query that runs twice a day.

**The smell is `$lookup` in the request path of a high-traffic endpoint**, or a pipeline reassembling one
logical object from four collections — a relational schema wearing a MongoDB costume.

---

## Two-sided many-to-many

**Signal:** arrays on both sides — `User { groupIds: [ObjectId] }` **and** `Group { memberIds: [ObjectId]
}`. Or a compound index over two array fields.

**Why it breaks:** two arrays must be kept in sync by application code with no transaction, so they
diverge on any partial failure. And you **cannot build a compound index across two array fields** —
MongoDB rejects it with "cannot index parallel arrays", which surfaces as a mysterious failure much later
when someone adds that index.

**Fix:**

```js
// one side owns the list — the bounded side
Group { _id, name, memberIds: [ObjectId] }

// unbounded on both sides -> a join collection
memberships { userId, groupId, role, joinedAt }
db.memberships.createIndex({ userId: 1, groupId: 1 }, { unique: true })
db.memberships.createIndex({ groupId: 1 })
```

**Fine when:** one side holds a bounded, small array — a document's `tagIds`, a user's few `roleIds`.
**The problem is specifically both sides holding arrays, or one unbounded side.**

A join collection costs a `$lookup` on every read, so it is the wrong answer for a small, stable
relationship.

---

## Denormalized copies drifting

**Signal:** the same fields duplicated across collections — `order.customerName` beside a `customers`
collection, `comment.userName` — with **no code that updates the copies**. Grep shows one write site (the
insert) and no update path.

**Why it breaks:** this is the extended-reference pattern implemented halfway. The copies are correct at
creation and drift forever after: a user renames themselves and their name is now wrong on ten thousand
historical documents, with no way to tell which. Support tickets read as "the site shows my old name in
some places."

**Fix — pick one and be explicit:**

```js
// own the propagation
await db.users.updateOne({ _id }, { $set: { name } })
await db.comments.updateMany({ userId: _id }, { $set: { userName: name } })

// or accept it as a snapshot and say so in the field name
order.customerNameAtPurchase
```

Only copy fields that rarely change — name, sku, slug.

**Fine when:** the snapshot is intentional. An invoice **must** keep the price, address, and tax rate as
they were at purchase; updating them retroactively would be the bug.

**The distinction is whether the copy should track the source or freeze it.** When it is genuinely a
snapshot, recommend a clearer field name rather than a sync job.

---

## Time series data

**Signal:** append-only measurements — sensor readings, price ticks, page views — stored as one document
per reading, or pushed into a per-entity array. No `timeseries:` option in `createCollection`.

**Why it breaks:** one document per reading multiplies `_id`-index overhead and per-document storage
across millions of rows. The per-entity array hits the unbounded-array failure above.

**Fix:**

```js
db.createCollection("readings", {
  timeseries: { timeField: "ts", metaField: "sensor", granularity: "minutes" }
})
```

MongoDB 5.0+ applies the bucket pattern internally with columnar compression, typically cutting storage
several-fold. Pre-5.0, hand-roll it: one document per `{sensorId, hour}` holding that hour's readings.

**Fine when:** the data is mutable or under a few million rows — a plain collection with an index on
`{sensorId: 1, ts: -1}` is adequate and simpler.

**Real restrictions to state if you recommend it:** update and delete match expressions can only target
the `metaField`; you cannot add a `metaField` after creation; and **you cannot convert an existing
regular collection in place** — it is a migration.

---

## No `$jsonSchema` validator

**Signal:** no `validator` or `$jsonSchema` anywhere — no `createCollection` with a validator, no
`collMod` migration. Validation exists only as Mongoose schemas or Zod types.

**Why it breaks:** MongoDB applies **no schema constraints unless you explicitly attach a validator.** Any
write bypassing the ORM — a migration script, a bulk import, Compass, a second service, an AI agent with
a shell — writes malformed documents the app then crashes on at read time.

**Fix:**

```js
db.createCollection("orders", { validator: { $jsonSchema: {
  bsonType: "object",
  required: ["userId", "total", "status"],
  properties: {
    userId: { bsonType: "objectId" },
    total:  { bsonType: "decimal" },
    status: { enum: ["pending", "paid", "shipped"] }
}}}})
```

Existing collection: `db.runCommand({ collMod: "orders", validator: { ... } })`.

**Fine when:** a single-service app where every write goes through one well-typed ORM layer. Genuinely
lower risk, and adding validators to a live collection full of legacy documents can break writes.

**Introduce it as `validationAction: "warn"` first**, read the logs, then tighten. Not an emergency for a
solo prototype — it becomes one the moment a second writer exists.

---

## Weak `$jsonSchema`

**Signal:** a validator listing `properties` and `required` but never setting
`"additionalProperties": false`. Or `required: ["email"]` with `bsonType: ["string", "null"]`. Or
`validationLevel: "moderate"` left in place permanently.

**Why it breaks:** **`additionalProperties` defaults to true**, so the validator accepts arbitrary extra
fields — a typo'd `emial` passes silently while the real field is missing. `required` only asserts the
key *exists*, so `{ email: null }` passes. And under `moderate`, updates to documents that were already
invalid are not validated at all, so bad rows stay bad forever.

**Fix:**

```js
$jsonSchema: {
  bsonType: "object",
  required: ["email"],
  additionalProperties: false,        // must be explicit
  properties: { email: { bsonType: "string" } }   // omit "null" to reject null
}
// then: validationLevel: "strict", validationAction: "error"
```

**Fine when:** the documents are meant to be extensible — user metadata, event payloads.
`additionalProperties: false` will also reject writes during a rolling deploy where new code adds a field
the old validator does not know. Scope it per sub-object, or keep an explicit
`metadata: { bsonType: "object" }` escape hatch.

`$jsonSchema` implements JSON Schema draft 4 — **`default`, `format`, `$ref`, and `definitions` are not
supported and are silently ignored.** Flag any of those as dead weight.

---

## Mongoose `runValidators`

**Signal:** `updateOne`, `findOneAndUpdate`, or `findByIdAndUpdate` called without
`{ runValidators: true }`.

**Why it breaks:** **Mongoose update validators are off by default** — only `save()` validates. So
`findByIdAndUpdate(id, { email: 12345 })` writes an integer into a field the schema declares as a
required string, and every later read that expects a string breaks. This is the most common way an
otherwise-typed Mongoose app accumulates garbage.

**Fix:**

```js
Model.findOneAndUpdate(filter, update, { new: true, runValidators: true })
```

Even with the flag on: `required` only fires on an explicit `$unset`, defaults and setters do **not** run
on updates, and `$inc` bypasses update validators entirely.

**Fine when:** the path only `$inc`s a counter.

**Turning this on globally can break existing update paths** that legitimately do partial updates on
documents which no longer satisfy the current schema — a field made `required` after the data was
written. Enable it, run the tests, fix the fallout, but say it is not free on a legacy dataset.

---

## `unique` on an optional field

**Signal:** `db.users.createIndex({ referralCode: 1 }, { unique: true })` where `referralCode` has no
`required: true` and no default.

**Why it breaks:** a unique single-field index treats a missing field as `null`. The first document
without the field indexes as null and succeeds; **the second fails** with
`E11000 duplicate key error ... referralCode: null`. It looks like a bizarre bug because the field is not
even being set, and only appears once two users skip it.

**Fix:**

```js
db.users.createIndex(
  { referralCode: 1 },
  { unique: true, partialFilterExpression: { referralCode: { $exists: true } } }
)
```

Prefer partial over sparse — sparse still permits one explicit null, and partial states the intent.

**Fine when:** the field is genuinely required and always written, or a default guarantees a value.

The multikey case surprises in the opposite direction: a unique index on an **array** field permits
duplicate values inside one document's array while enforcing uniqueness of each value across documents.

**Confirm it:** `db.users.countDocuments({ referralCode: { $exists: false } })` above 1 means the index
cannot be built as-is.

---

## Mongoose `unique: true` without the index

**Signal:** `unique: true` combined with `autoIndex: false`, or index creation gated on
`NODE_ENV === 'production'`.

**Why it breaks:** `unique: true` is **not a validator** — it is a request to build a unique index. If
`autoIndex` is disabled in production (the standard recommendation, since index builds load the
database), that index may never exist there, so duplicate emails insert cleanly. **The uniqueness
guarantee exists only in dev.**

**Fix:**

```js
await db.collection('users').createIndex({ email: 1 }, { unique: true })   // explicit migration
db.users.getIndexes()                                                       // verify against the cluster
```

Even with the index, check-then-insert races — catch the `E11000` error instead of pre-checking.

**Fine when:** the repo has explicit index migrations or calls `Model.syncIndexes()` on deploy. Then
`autoIndex` being off is correct practice, not a bug. **Check for a migrations directory before
flagging.**

---

## `$elemMatch`

**Signal:** two conditions on the same array of subdocuments via dot notation:
`db.orders.find({ "items.sku": "A1", "items.qty": { $gte: 5 } })`.

**Why it breaks:** without `$elemMatch`, the conditions match against the array **as a whole** —
*different* elements may satisfy each one. That query matches an order containing
`[{sku:"A1",qty:1},{sku:"Z9",qty:50}]`, which nobody intends. It silently returns too many results and is
rarely caught by tests using single-element arrays.

**Fix:**

```js
db.orders.find({ items: { $elemMatch: { sku: "A1", qty: { $gte: 5 } } } })

db.orders.find({ "items.sku": "A1" })    // a single condition needs no $elemMatch
```

**Fine when:** the parent field is a single embedded **object**, not an array —
`{"address.city":"X","address.zip":"Y"}` has no cross-element problem and `$elemMatch` would be wrong.
Also fine when the two conditions genuinely are independent.

**Read the schema to confirm the field is an array before flagging.**

---

## Transactions need a replica set

**Signal:** `session.withTransaction(...)` in application code, against a connection string with no
`replicaSet=` and a `docker-compose.yml` running `mongod` with no `--replSet`.

**Why it breaks:** standalone `mongod` does not support multi-document transactions at all. The code fails
at runtime with "Transaction numbers are only allowed on a replica set member or mongos" — typically only
in local dev or CI, so it reads as an environment flake.

**Fix:** a single-node replica set is enough.

```bash
mongod --replSet rs0 --dbpath /data
mongosh --eval 'rs.initiate()'
# mongodb://localhost:27017/app?replicaSet=rs0&directConnection=true
```

**Fine when:** the transaction spans genuinely independent aggregates — a payment plus a ledger entry.
**Do not advise removing transactions.**

The design point matters more than the config fix: MongoDB's docs state transaction availability "should
not be a replacement for effective schema design." A transaction spanning three collections on **every**
write usually means data that changes together was split apart. Check whether the split was arbitrary.

---

## Sources

[design antipatterns](https://www.mongodb.com/docs/manual/data-modeling/design-antipatterns/),
[embedding vs references](https://www.mongodb.com/docs/manual/data-modeling/concepts/embedding-vs-references/),
[limits](https://www.mongodb.com/docs/manual/reference/limits/),
[schema validation](https://www.mongodb.com/docs/manual/core/schema-validation/specify-validation-level/),
[$jsonSchema](https://www.mongodb.com/docs/manual/reference/operator/query/jsonSchema/),
[$lookup](https://www.mongodb.com/docs/manual/reference/operator/aggregation/lookup/),
[multikey indexes](https://www.mongodb.com/docs/manual/core/indexes/index-types/index-multikey/),
[unique indexes](https://www.mongodb.com/docs/manual/core/index-unique/),
[query array of documents](https://www.mongodb.com/docs/manual/tutorial/query-array-of-documents/),
[time series](https://www.mongodb.com/docs/manual/core/timeseries-collections/),
[transactions](https://www.mongodb.com/docs/manual/core/transactions/),
[Mongoose validation](https://mongoosejs.com/docs/validation.html).
