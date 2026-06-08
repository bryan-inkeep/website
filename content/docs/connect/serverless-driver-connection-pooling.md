---
title: Serverless driver connection pooling
subtitle: How the Neon serverless driver handles connections and interacts with connection pooling
enableTableOfContents: true
---

This page answers common questions about how the `@neondatabase/serverless` driver manages connections and how it interacts with Neon's connection pooling.

## Does the serverless driver pool connections?

No, the serverless driver does not pool connections on the client side.

The driver uses two different protocols depending on the operation:

- **HTTP for one-shot queries** — When you run a single query outside of a transaction, the driver speaks HTTP. There's no Postgres connection at all on the client side. The driver sends your query over HTTP, and the result comes back in the response.

- **WebSocket for transactions** — When you start a transaction, the driver opens a WebSocket connection for the duration of that transaction session. Each session uses one WebSocket, but the driver doesn't maintain a client-side pool of these connections.

Because the driver doesn't maintain a persistent connection pool on the client, you don't need to worry about managing connection lifecycle or cleanup in most serverless environments.

## If I'm already pointing at the pooler endpoint, do those stack on top of each other?

No, they don't stack. The pooler endpoint simply becomes the Postgres endpoint that the driver connects to.

Here's how it works:

- When you use the Neon pooler endpoint (typically ending in `-pooler.us-east-2.aws.neon.tech`), you're telling the driver where to send its WebSocket connections.
- The driver opens a WebSocket to the pooler endpoint when you start a transaction.
- The pooler then manages the actual Postgres connections on the server side.

There's no "double pooling" happening. The driver's HTTP/WebSocket behavior stays the same whether you point at the pooler endpoint or the direct database endpoint. The pooler just becomes the target that the driver talks to.

**Example connection strings:**

```text
# Direct database endpoint
postgresql://alex:AbC123dEf@ep-cool-darkness-123456.us-east-2.aws.neon.tech/dbname

# Pooler endpoint
postgresql://alex:AbC123dEf@ep-cool-darkness-123456-pooler.us-east-2.aws.neon.tech/dbname
```

In both cases, the driver behavior is the same: HTTP for one-shot queries, WebSocket for transactions. The pooler endpoint just adds server-side connection pooling for the WebSocket connections.

## When should I use the pooler endpoint?

Use the pooler endpoint when:

- You're running many concurrent serverless functions that open transactions
- You want to reduce connection overhead on the Postgres server
- You're approaching connection limits for your compute

The pooler helps manage server-side connections efficiently, especially in high-concurrency environments typical of serverless deployments.

<NeedHelp />
