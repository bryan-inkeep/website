---
title: Connection pooling with the Neon serverless driver
subtitle: Understand how the @neondatabase/serverless driver manages connections and interacts with connection pooling
enableTableOfContents: true
---

In serverless environments, connection management is critical for performance and reliability. This page answers common questions about how the `@neondatabase/serverless` driver handles connections and how it interacts with Neon's connection pooling.

## Does the @neondatabase/serverless driver pool connections?

No, the driver does not maintain a client-side connection pool.

The driver operates in two modes:

- **HTTP mode** (one-shot queries): The driver sends queries over HTTP without establishing a traditional PostgreSQL connection on the client side. Each query is stateless.
- **WebSocket mode** (transactions): The driver opens one WebSocket connection per session. These connections are not pooled on the client side.

This design is optimized for serverless environments where connection state is not maintained between invocations.

## How does the driver handle one-shot queries vs transactions?

The driver automatically chooses the appropriate protocol based on your usage:

**One-shot queries (HTTP)**

When you execute individual queries without a transaction, the driver uses HTTP:

```javascript
import { neon } from "@neondatabase/serverless";

const sql = neon(process.env.DATABASE_URL);
const result = await sql`SELECT * FROM users WHERE id = ${userId}`;
```

This approach is stateless and requires no PostgreSQL connection on the client side.

**Transactions (WebSocket)**

When you need transaction support, the driver uses WebSocket:

```javascript
import { Pool } from "@neondatabase/serverless";

const pool = new Pool({ connectionString: process.env.DATABASE_URL });
const client = await pool.connect();

try {
  await client.query("BEGIN");
  await client.query("INSERT INTO users (name) VALUES ($1)", ["Alex Lopez"]);
  await client.query("COMMIT");
} catch (error) {
  await client.query("ROLLBACK");
  throw error;
} finally {
  client.release();
}
```

Each session maintains one WebSocket connection. The connections are not reused across different sessions or pooled on the client side.

## If I'm using a pooled connection endpoint, do I still need client-side connection pooling?

No. If you're pointing the `@neondatabase/serverless` driver at a Neon pooled connection endpoint, you don't need client-side connection pooling.

The driver's architecture (HTTP for one-shot queries, one WebSocket per transaction session) means there's nothing to pool on the client side. Connection pooling happens at the Neon infrastructure layer when you use a pooled endpoint.

## Does using the driver with a pooler endpoint create "double pooling"?

No. Using the driver with a pooled connection endpoint does not create "stacked" or "double" pooling.

When you configure the driver to use a pooled endpoint:

- **For HTTP mode**: The driver sends stateless HTTP requests. There is no connection to pool.
- **For WebSocket mode**: The pooler becomes the PostgreSQL target for the driver's WebSocket connection. The pooler manages a pool of connections to the actual database, but the driver itself still opens one WebSocket per session.

The connection pooling layer (at the Neon infrastructure) and the driver's connection protocol (HTTP or WebSocket) work together without creating redundant pooling layers.

## Related resources

- [Neon serverless driver documentation](https://neon.com/docs/connect/serverless-driver) — Full reference for the `@neondatabase/serverless` driver
- [Connection pooling](https://neon.com/docs/connect/connection-pooling) — Learn about Neon's connection pooling features
