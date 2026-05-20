# Chapter 1 The API Landscape

## The Three Main Protocols

### REST (Representational State Transfer) — The Default Choice

**REST covers roughly 90% of API use cases. The other protocols exist for the other 10% — and knowing when to use them is what separates a senior engineer from a junior one.**

t's resource-oriented — everything is a **resource** with a URL, and you interact with resources using standard HTTP methods.

#### Why REST Is the Default

- **Everyone knows it**. Every developer, every framework, every tool supports REST.
- **HTTP semantics are built-in**. Status codes (200, 404, 500), caching headers, content negotiation — you get these for free.
- **Stateless**. Each request contains everything the server needs. No session state to manage. This makes scaling straightforward — any server can handle any request.
- **Tooling is incredible**. Swagger/OpenAPI for docs, Postman for testing, CDNs for caching — the ecosystem is massive.

#### REST's Weaknesses

- **Over-fetching** : `GET /users/42` returns everything about the user, even if you only need their name.

- **Under-fetching**. To build a user profile page, you might need `GET /users/42`, then `GET /users/42/posts`, then `GET /users/42/followers` — three round trips for one screen.

- **Rigid endpoints**. Every new data need often means a new endpoint or a new query parameter.

### GraphQL — The Client Is in Control

#### The Mental Model

With REST, the server decides what data you get back. With GraphQL, the client decides. There's a single endpoint (POST /graphql), and the client sends a query describing exactly what it wants.

#### Why GraphQL Shines

- **No over-fetching**
- **No under-fetching**
- **Diverse clients**
- **Self-documenting**
- **Strong typing**: Clients know exactly what to expect.

#### GraphQL's Weaknesses

- **Complexity on the server**: You need to build resolvers, handle nested queries, and prevent expensive operations
- **N+1 query problem**: A naive resolver for "all users and their posts" will fire one database query per user. You need tools like DataLoader to batch these.
- **Security surface area**: A malicious client could send a deeply nested query that hammers your database
- **Learning curve**: Your team needs to learn the query language, schema definition, resolvers, and client libraries.
- **Caching is harder**

### RPC (Remote Procedure Call) — Calling Functions Over the Network

The idea is simple: you call a function on a remote server as if it were a local function.

REST thinks in resources (nouns): `/users`, `/orders`, `/products`. RPC thinks in actions (verbs): `createUser()`, `calculateShipping()`, `sendEmail()`.

- gRPC (Google's modern RPC framework) uses Protocol Buffers for binary serialization:

```protobuf
// user_service.proto — the contract
service UserService {
  rpc GetUser (GetUserRequest) returns (UserResponse);
  rpc CreateUser (CreateUserRequest) returns (UserResponse);
}

message GetUserRequest {
  int32 user_id = 1;
}

message UserResponse {
  int32 id = 1;
  string name = 2;
  string email = 3;
}
```

The `.proto` file generates client and server code in any language. Your Go service and your Python service can talk to each other with type-safe, auto-generated code. No hand-writing HTTP requests. No parsing JSON.

#### Why gRPC Is Popular for Microservices

- **Binary serialization with Protocol Buffers**. This is the big one. Instead of sending human-readable JSON text (`{"name": "Alice"}`), Protocol Buffers encode data into a compact binary format. Smaller payloads, faster serialization/deserialization.

- **HTTP/2 under the hood**. gRPC uses HTTP/2, which gives you multiplexing (multiple requests over one connection), header compression, and bidirectional streaming.

- **Streaming**. gRPC natively supports server streaming, client streaming, and bidirectional streaming

- **Strong contracts**. The .proto file is the single source of truth. Both sides know exactly what to send and expect.

#### gRPC Is Faster Than HTTP -> NO

gRPC is not "faster than HTTP." gRPC uses HTTP/2. It literally runs on top of HTTP. The performance advantage comes from two things:

1. **Binary serialization (Protocol Buffers)** is much more compact and faster to parse than JSON text. A message that's 100 bytes in JSON might be 30 bytes in Protocol Buffers.
2. **HTTP/2** features like multiplexing and header compression reduce overhead compared to HTTP/1.1.

#### RPC's Weaknesses

- **Not browser-friendly**. Browsers speak HTTP natively. gRPC requires a proxy layer to work in browsers. This is why gRPC is almost always used for `backend-to-backend communication`.

- **Tighter coupling**. Both client and server need the .proto file. Changing the API means regenerating code on both sides.
- **Less discoverable**. You can't just curl a gRPC endpoint and see what happens. REST APIs are inherently explorable.
- **Overkill** for simple APIs. If you have a CRUD API with 5 endpoints, REST is simpler and more appropriate.

---

# Chapter 2 The Decision Framework — Picking the Right Protocol

![Text9](/assets/9.png)

## When GraphQL Actually Makes Sense

### 1. Under-Fetching Causes Waterfall Requests

To render one screen, your app makes 4-5 sequential REST calls:

- `GET /users/42`
- `GET /users/42/posts`
- `GET /posts/101/comments`
- `GET /users/42/followers?limit=5`

### 2. Over-Fetching Is Measurable

### 3. Diverse Clients with Different Needs

- `/users/42?fields=name,avatar (mobile)`
- `/users/42?include=posts,followers,activity (web)`
- `/users/42/summary (TV)`

### 4. Rapid Frontend Iteration

Your frontend team ships new features weekly and constantly needs new data combinations.

## When RPC/gRPC Makes Sense

### 1. You Control Both Sides

gRPC requires a shared `.proto` file between client and server. This tight coupling is fine when both services are in your organization.

### 2. Performance Is Measurable

### 3. You Need Streaming

gRPC natively supports four communication patterns:

| Pattern                 | Description                      | Example                                        |
| ----------------------- | -------------------------------- | ---------------------------------------------- |
| Unary                   | One request, one response        | `GetUser(id) → User`                           |
| Server streaming        | One request, stream of responses | `WatchStockPrice(ticker) → stream of prices`   |
| Client streaming        | Stream of requests, one response | `UploadChunks(stream of bytes) → UploadResult` |
| Bidirectional streaming | Stream in both directions        | `Chat(stream of messages) ↔ stream o           |

## The Mixed-Protocol Pattern

In the real world, production systems almost always use multiple protocols.

```text
┌─────────────┐     REST/GraphQL      ┌──────────────┐
│ Browser /   │ ───────────────────→ │ API Gateway  │
│ Mobile App  │ ←── WebSockets ────  │              │
└─────────────┘                       └──────┬───────┘
                                             │
                                        gRPC │ (internal)
                                             │
                         ┌───────────────────┼───────────────────┐
                         │                   │                   │
                   ┌─────▼─────┐       ┌─────▼─────┐       ┌─────▼─────┐
                   │ User      │       │ Order     │       │ Notif.    │
                   │ Service   │       │ Service   │       │ Service   │
                   └───────────┘       └───────────┘       └───────────┘
```

- **REST or GraphQL** for the public-facing API (clients, partners, third-party developers)
- **gRPC** for internal service-to-service communication (speed, type safety, streaming)
- **WebSockets** for real-time features (notifications, live updates)

## Common Interview Signals and What They Mean

| Interviewer says...                                                       | They're really asking...                        | Strong answer includes...                                                                         |
| ------------------------------------------------------------------------- | ----------------------------------------------- | ------------------------------------------------------------------------------------------------- |
| "How would the client and server communicate?"                            | "Do you know the basics of API design?"         | State REST as default, define key endpoints with HTTP methods                                     |
| "The mobile app is slow — too much data."                                 | "Do you know about over-fetching?"              | Mention field filtering (`?fields=`), or suggest GraphQL if the problem is systemic               |
| "We have 50 microservices calling each other."                            | "Do you know about gRPC?"                       | Suggest gRPC for internal communication, explain why (binary serialization, code gen, streaming)  |
| "How do users see updates in real time?"                                  | "Do you know about push protocols?"             | Distinguish between `WebSockets (bidirectional)` and `SSE (server-push only)`, pick the right one |
| "What if we need to support a mobile app AND a web app AND a public API?" | "Can you architect for diverse clients?"        | GraphQL or REST with BFF (Backend for Frontend) pattern. Explain the trade-off.                   |
| "How would you version this API?"                                         | "Do you think about backward compatibility?"    | URL versioning (`/v2/users`) vs header versioning, deprecation strategy                           |
| "Walk me through an API call end-to-end."                                 | "Do you understand the full request lifecycle?" | Client → DNS → TLS → HTTP request → load balancer → API gateway → service → database → response   |

---

# Chapter 3 Resource Modeling

**URLs should represent nouns (things), HTTP methods should represent verbs (actions).**

![text10](/assets/10.png)

## The Golden Rule: Plural Nouns, Never Verbs

| Bad (verb-based)           | Good (noun-based)   |
| -------------------------- | ------------------- |
| `POST /createEvent`        | `POST /events`      |
| `GET /getEventById?id=123` | `GET /events/123`   |
| `POST /deleteUser`         | `DELETE /users/456` |
| `PUT /updateTicket`        | `PUT /tickets/789`  |
| `GET /fetchAllBookings`    | `GET /bookings`     |

Notice the pattern? Every good endpoint is a plural noun. The HTTP method tells you the action — you don't need to repeat it in the URL.

Why plural? Because the endpoint represents a collection. `/events` is the collection of all events. `/events/123` is one specific event inside that collection. It reads naturally: "Give me all events" `(GET /events)` or "Give me event 123" `(GET /events/123)`.

## Worked Example: Modeling a Ticketmaster-Like API

Let's say you're designing APIs for a platform like Ticketmaster. What are the things in this domain?

- **Events** — concerts, sports games, theater shows
- **Venues** — stadiums, arenas, theaters
- **Tickets** — individual seats available for purchase
- **Bookings** — a user's confirmed purchase of tickets
- **Users** — the people using the platform

### Nested Resources: When Relationships Are Required

A booking doesn't exist in a vacuum. Every booking is for a specific event. This is a required parent-child relationship — a booking without an event doesn't make sense.

For required relationships like this, you nest the child under the parent:

    POST   /events/123/bookings       # Create a booking for event 123
    GET    /events/123/bookings       # List all bookings for event 123
    GET    /events/123/bookings/456   # Get a specific booking for event 123

**Why is this better than a flat URL?** Because the path itself tells you the relationship. When you read POST /events/123/bookings, you immediately know:

### The Nesting Debate: POST /bookings/:eventId vs POST /events/:eventId/bookings

This is a common discussion that comes up in both design reviews and interviews. Which is more RESTful?

    # Option A: Flat with event ID as path param
    POST /bookings/123

    # Option B: Nested under the parent resource
    POST /events/123/bookings

**Option B is more RESTful. Here's why:**

In Option A, /bookings/123 looks like you're accessing booking with ID 123 — that's the standard REST convention for "get a specific resource by its ID." It's ambiguous. Is 123 the booking ID or the event ID?

In Option B, the URL reads like a sentence: "For event 123, create a booking." The hierarchy is crystal clear. The path segment after a collection name always means "the resource with this ID in that collection."

### Flat Resources + Query Parameters: When Relationships Are Optional

Not every relationship needs nesting. Sometimes you want to filter or search across resources without requiring a parent.

    GET /tickets?section=VIP                  # All VIP tickets across all events
    GET /tickets?price_max=100                # All tickets under $100
    GET /tickets?event_id=123                 # All tickets for event 123

### The Decision Rule: Nest or Flatten?

| Use nesting when...                          | Use query params when...                 |
| -------------------------------------------- | ---------------------------------------- |
| The child can't exist without the parent     | The relationship is optional             |
| The parent is required to identify the child | You're filtering or searching            |
| The hierarchy is always the same             | Users might filter by different criteria |
| There's only one level of nesting            | You'd need 3+ levels of nesting          |

**One important caveat: don't go deeper than two levels of nesting.** `GET /events/123/bookings/456` is fine. `GET /venues/1/events/123/bookings/456/tickets/789` is a nightmare.

---

# Chapter 4 HTTP Methods and Idempotency

![text11](/assets/11.png)

- ### GET
  - **Cacheable**

  - **No request body**

- ### POST
  - **Not safe** — it changes server state (creates something).
  - **Not idempotent** — calling it twice creates two bookings. This is the duplicate booking problem.
  - **Not cacheable** — every POST should be treated as unique.
  - **Has a request body** — the data for the new resource goes in the body.

### The Duplicate Problem

What happens if your network drops after the server processes a POST but before you get the response? You don't know if it worked. So you retry. Now you have two bookings.

This is exactly why POST is not idempotent, and why it matters. We'll talk about idempotency keys later in this lesson — they're the standard solution to this problem.

- ### PUT
  - **Not safe** — it modifies server state.
  - **Idempotent** — sending the same PUT request ten times produces the same result as sending it once. The resource ends up in the same state regardless.
  - **Has a request body** — the complete representation of the resource.

- ### PATCH — Partial Update
  - **Not safe** — it modifies server state.
  - **NOT idempotent** — this one surprises people. We'll dig into why below.
  - **Has a request body** — only the fields being updated.

### PUT vs PATCH: When to Use Which

| Scenario                                              | Use PUT | Use PATCH |
| ----------------------------------------------------- | ------- | --------- |
| You have the complete resource and want to replace it | Yes     | No        |
| You want to change one field out of twenty            | No      | Yes       |
| The client should provide all required fields         | Yes     | No        |
| The client only knows what changed                    | No      | Yes       |
| Idempotency is critical for your use case             | Yes     | Maybe     |

- ### DELETE — Remove a Resource
  - **Not safe** — it changes server state.
  - **Idempotent** — deleting something that's already gone doesn't change the server state. The first DELETE returns 204 (success, no content). Subsequent DELETEs might return 404 (not found), but the server state is the same — the resource is gone either way.
  - **Usually no request body** — though some APIs do accept one.

**A request method is considered idempotent if the intended effect on the server of multiple identical requests with that method is the same as the effect for a single such request.**

## Why PATCH Is Not Idempotent

This catches people off guard because many PATCH operations seem idempotent. Consider:

    PATCH /users/123
    { "email": "new@example.com" }

Call this ten times — the email is new@example.com every time. Seems idempotent, right?

Now consider this PATCH:

    PATCH /posts/456
    { "op": "append", "path": "/tags", "value": "featured" }

Call this ten times — you get "featured" appended ten times. Definitely not idempotent.

The HTTP spec (RFC 5789) correctly classifies PATCH as not idempotent because the spec for the method itself cannot guarantee idempotency. Some PATCH operations are idempotent in practice ("set email to X"), but others are not ("append to list"). Since the method as a category can't make the guarantee, PATCH is classified as non-idempotent.

- POST and PATCH are on the "not guaranteed" side. You can implement them idempotently (and sometimes should), but the spec doesn't promise it.

## Idempotency Keys: Making POST Safe to Retry

So POST isn't idempotent, and network failures happen. How do you prevent duplicate bookings?

The solution is an idempotency key — a unique token the client sends with the request. The server uses it to detect duplicates.

Here's what happens:

- **First call**: Server sees `Idempotency-Key: abc-123-unique-token` for the first time. It processes the request, creates the booking, and stores the response alongside the key.
- **Retry** (same key): Server recognizes the key, skips processing, and returns the stored response from the first call.

The client is responsible for generating a unique key (usually a UUID) for each logical operation. If you want to retry the same booking, you send the same key. If you want to make a different booking, you send a new key.

---

# Chapter 5 Request and Response Design

## Three Places to Put Data (And When to Use Each)

When you send an API request, data can go in exactly three places. Think of it like mailing a package:

- **Path parameters** are the address on the envelope — they tell the system where you're going. Which resource? Which specific item?
- **Query parameters** are special instructions written on the label — "fragile," "signature required," "leave at door." They modify how the request is handled.
- **Request body** is the contents inside the package — the actual stuff you're sending.

![text12](/assets/12.png)

**Rules for path parameters:**

- Always required — the URL doesn't make sense without them
- Identify a specific resource or a parent in a hierarchy
- Usually IDs (numeric or UUID)

**Rules for query parameters:**

- Always optional — removing them gives a broader or default result
- Used for filtering, sorting, pagination, and feature flags
- Never for identifying a specific, required resource

**Rules for the request body:**

- Used with POST, PUT, and PATCH (never GET or DELETE in practice)
- Contains structured data (usually JSON)
- Represents the "what" — the actual content being sent

**| Bearer token | Authorization header | Authentication — who is making the request |**

This is a real design question that comes up in practice. Should notify=true be a query parameter or in the request body?

Both can work, but here's the guideline:
| Put it in the query string when... | Put it in the body when... |
|---|---|
| It controls behavior (how the server processes the request) | It's part of the resource data (what gets stored) |
| It's a flag or toggle | It's a complex object or nested data |
| It doesn't get persisted | It gets saved to the database |
| Examples: `?notify=true`, `?dry_run=true`, `?async=true` | Examples: `{"email": "..."}`, `{"settings": {...}}` |

notify=true controls server behavior (send an email) but isn't part of the booking itself. That makes it a good fit for a query parameter.

## Security: Where User Identity Should Come From

This is a critical security point that many junior developers get wrong.

Never pass the user's identity in the request body. Don't do this:

Why? Because any client can put any user_id in the body. A malicious user could create bookings under someone else's account by simply changing the number.

    POST /events/123/bookings
    {
    "user_id": 789,        # WRONG — never trust the client
    "ticket_ids": [101, 102]
    }

The user's identity should always come from the JWT token in the Authorization header. The server extracts it server-side:

    POST /events/123/bookings
    Authorization: Bearer eyJhbGciOiJSUzI1NiIs...

    {
    "ticket_ids": [101, 102]
    }

## Some Status Codes

- **401 Unauthorized** — "I don't know who you are." The request has no valid auth token, or the token is expired. Fix: log in again.
- **403 Forbidden** — "I know who you are, and you're not allowed." The token is valid, but this user doesn't have permission. Fix: get the right permissions (or accept you can't access this).

- **400 Bad Request** — The request is structurally broken. Malformed JSON, missing required fields, wrong Content-Type. The server can't even parse what you sent.
- **422 Unprocessable Entity** — The request is structurally fine (valid JSON, all fields present) but semantically wrong. You're trying to book an event in the past, or you're requesting more tickets than exist.

## A Good Error Response Structure

```JSON
{
  "error": {
    "code": "TICKETS_UNAVAILABLE",
    "message": "The requested tickets are no longer available.",
    "details": [
      {
        "field": "ticket_ids",
        "issue": "Ticket 789 was sold to another customer at 2025-03-15T14:22:00Z"
      }
    ]
  }
}
```

- **code** — A machine-readable error code the client can switch on (not the HTTP status code — this is your application-level code)
- **message** — A human-readable description the developer can understand
- **details** — Optional array with field-level specifics

## Batch API Error Responses

What happens when a single request contains multiple operations and some succeed while others fail? This is common in batch APIs.

### 1. All-or-Nothing (Simpler)

Fail the entire batch if any item fails. Return a 4xx with details about what went wrong.

```JSON
HTTP/1.1 409 Conflict
{
  "error": {
    "code": "PARTIAL_AVAILABILITY",
    "message": "Not all tickets are available.",
    "details": [
      {"ticket_id": 789, "status": "available"},
      {"ticket_id": 790, "status": "available"},
      {"ticket_id": 791, "status": "sold_out"}
    ]
  }
}
```

**Pros**: Simple, predictable. The client knows: either everything worked, or nothing did. **Cons**: One bad item blocks the whole batch.

### Option 2: Partial Success (More Flexible)

Process what you can and report per-item results. Use 207 Multi-Status or 200 with a structured body:

```JSON
HTTP/1.1 207 Multi-Status
{
  "results": [
    {"ticket_id": 789, "status": "success", "booking_id": 1001},
    {"ticket_id": 790, "status": "success", "booking_id": 1002},
    {"ticket_id": 791, "status": "failed", "error": {
      "code": "TICKET_SOLD_OUT",
      "message": "Ticket 791 is no longer available.",
      "retryable": false
    }}
  ],
  "summary": {
    "total": 3,
    "succeeded": 2,
    "failed": 1
  }
}
```

**Pros**: More efficient — successful items don't need to be retried. **Cons**: More complex for the client to handle.

**Content-Type** — Tells the server what format the request body is in. Almost always application/json for modern APIs.

**Accept** — Tells the server what format the client wants the response in. Usually application/json.

If the server can't produce the requested format, it should return **406 Not Acceptable**. If the server can't parse the request body format, it should return **415 Unsupported Media Type**.

---

# Chapter 6 Why GraphQL Exists — The Problem with REST

## TThe BFF Pattern

The Backend for Frontend (BFF) pattern puts a dedicated API layer in front of your microservices, one per client type

Each BFF aggregates data from multiple backend services and returns exactly what its client needs. This is actually a legitimate pattern that companies like Netflix use. But it means you're maintaining multiple backend services just to reshape data. That's a lot of code that doesn't add business logic

## GraphQL: One Endpoint

GraphQL uses a single endpoint `(typically POST /graphql)`. The client sends a query in the request body, and the server returns exactly that shape of data — nothing more, nothing less.

### With GraphQL, it's one request:

```graphQL
query {
  event(id: "501") {
    name
    date
    venue {
      name
      city
    }
    availableTickets {
      section
      price
    }
  }
}
```

```JSON
{
  "data": {
    "event": {
      "name": "Taylor Swift Eras Tour",
      "date": "2026-06-15",
      "venue": {
        "name": "SoFi Stadium",
        "city": "Los Angeles"
      },
      "availableTickets": [
        { "section": "Floor", "price": 450 },
        { "section": "Lower Bowl", "price": 250 }
      ]
    }
  }
}
```

| Aspect         | REST                                       | GraphQL                                               |
| -------------- | ------------------------------------------ | ----------------------------------------------------- |
| Endpoints      | Multiple (`/users`, `/posts`, `/comments`) | Single (`/graphql`)                                   |
| Data shape     | Server decides what to return              | Client decides what to receive                        |
| Over-fetching  | Common — endpoints return fixed shapes     | Eliminated — you ask for exactly what you need        |
| Under-fetching | Common — requires multiple requests        | Eliminated — nested queries in one request            |
| Versioning     | `/api/v1/`, `/api/v2/`                     | No versioning needed — add fields, deprecate old ones |
| Caching        | Simple — HTTP caching works naturally      | Complex — single POST endpoint breaks HTTP caching    |
| Learning curve | Low — most developers know REST            | Higher — schema, resolvers, query language            |

## Schema Design, Queries, and Mutations

**Schema** : Think of a GraphQL schema like the menu at a restaurant. The menu tells you exactly what's available, what each dish contains, and what substitutions are allowed.

A GraphQL schema is **executable documentation**. It's not a separate artifact that can drift — it is the API. If the schema says a field exists, it exists. If a field is marked non-nullable, the server guarantees it will always return a value. The schema is checked at build time, at request time, and by every tool in the ecosystem.

### Defining Types: The Ticketmaster Schema

```graphql
type Event {
  id: ID!
  name: String!
  date: String!
  description: String
  venue: Venue!
  availableTickets: [Ticket!]!
  totalTickets: Int!
  isSoldOut: Boolean!
}

type Venue {
  id: ID!
  name: String!
  city: String!
  state: String!
  capacity: Int!
  events: [Event!]!
}

type Ticket {
  id: ID!
  event: Event!
  section: String!
  row: String
  seat: Int
  price: Float!
  status: TicketStatus!
}

enum TicketStatus {
  AVAILABLE
  RESERVED
  SOLD
}
```

#### The ! Operator: Non-Nullable (every field is nullable by default)

### (IMP) Reading the Type Annotations

| Annotation   | Meaning                           | Can the list be null? | Can items be null? |
| ------------ | --------------------------------- | --------------------- | ------------------ |
| `[Ticket]`   | Nullable list of nullable tickets | Yes                   | Yes                |
| `[Ticket!]`  | Nullable list of non-null tickets | Yes                   | No                 |
| `[Ticket]!`  | Non-null list of nullable tickets | No                    | Yes                |
| `[Ticket!]!` | Non-null list of non-null tickets | No                    | No                 |

**Enums** define a fixed set of allowed values. In our schema, TicketStatus can only be `AVAILABLE`, `RESERVED`, or `SOLD`. If a client tries to set it to `"PENDING"`, the schema rejects the request before any code runs. This is way safer than passing around raw strings.

**Relationships** : Notice how `Event` has a `venue: Venue!` field, and `Venue` has an `events: [Event!]!` field. These are relationships — they tell GraphQL that types are connected. The client can traverse these relationships to any depth in a single query.

### Queries: Reading Data

```graphql
type Query {
  event(id: ID!): Event
  events(city: String, limit: Int = 10): [Event!]!
  venue(id: ID!): Venue
  searchEvents(query: String!, first: Int = 20): [Event!]!
}
```

- **Arguments**: event(id: ID!) takes a required id argument. The client must provide it.
- **Default values**: limit: Int = 10 means if the client doesn't specify a limit, it defaults to 10.
- **Nullable return**: event(id: ID!): Event (no ! on the return type) means the event might not exist — it can return null.
- **Non-nullable list return**: events(...): [Event!]! means this always returns a list, even if it's empty.

Here's how a client would use these queries:

```graphql
# Get a single event with its venue and tickets
query GetEventDetails {
  # $eventId is variables
  event(id: $eventId) {
    name
    date
    description
    venue {
      name
      city
      capacity
    }
    availableTickets {
      section
      row
      price
      status
    }
    isSoldOut
  }
}

# Search for events in a city
query FindConcerts {
  searchEvents(query: "concert", first: 5) {
    name
    date
    venue {
      name
      city
    }
  }
}
```

The beauty here is that `GetEventDetails` asks for `description` and `capacity` while `FindConcerts` skips them entirely. Same schema, different queries, different response shapes. The mobile app can ask for less, the web app can ask for more.


### Mutations: Writing Data

If queries are `GET` requests, mutations are `POST/PUT/DELETE`. They modify data on the server:


```GRAPHQL
type Mutation {
  createBooking(input: BookingInput!): BookingResult!
  cancelBooking(bookingId: ID!): CancelResult!
  updateEvent(id: ID!, input: UpdateEventInput!): Event!
}

input BookingInput {
  eventId: ID!
  ticketIds: [ID!]!
  paymentMethodId: String!
}

input UpdateEventInput {
  name: String
  date: String
  description: String
}

type BookingResult {
  success: Boolean!
  booking: Booking
  error: String
}

type CancelResult {
  success: Boolean!
  refundAmount: Float
  error: String
}
```
#### Calling a Mutation
```graphql
mutation BookTickets($input: BookingInput!) {
  createBooking(input: $input) {
    success
    booking {
      id
      confirmationCode
      tickets {
        section
        row
        seat
      }
      totalPrice
    }
    error
  }
}
```

```json
{
  "variables": {
    "input": {
      "eventId": "501",
      "ticketIds": ["tkt_1001", "tkt_1002"],
      "paymentMethodId": "pm_stripe_abc123"
    }
  }
}
```

**Subscriptions** are GraphQL's mechanism for real-time data. They use WebSockets under the hood

**???......**

----

# Chapter 7 The N+1 Problem and Other GraphQL Pitfalls


In GraphQL, every field has a **resolver** — a function that fetches the data for that field. The server processes the query top-down:
1. **Step 1**: The `events` resolver runs. It executes 1 database query: `SELECT * FROM events WHERE city = 'Los Angeles' LIMIT 100`. You get back 100 events.

2. **Step 2**: For each of those 100 events, GraphQL calls the `venue` resolver. Each resolver sees its parent event's `venueId` and fetches the venue: `SELECT * FROM venues WHERE id = ?`.

That's 100 separate database queries for venues. **Plus the 1 query for events**.

**Total: 101 database queries for what looks like one simple GraphQL query.**

The N+1 problem is NOT unique to GraphQL. REST can have it too. If you have a REST endpoint that returns a list of events with venueId fields, and your server-side code loops through each event to fetch its venue, you have the exact same problem.

The difference is that GraphQL makes N+1 much easier to accidentally create. Here's why:
1. In REST, the server fully controls the response structure through predefined endpoints, so data-fetching logic is optimized during API design. Since clients can only access the fixed data exposed by those endpoints, they usually cannot accidentally create inefficient database query patterns like the N+1 problem.
2. In GraphQL, the client controls the query shape and can request deeply nested or custom combinations of fields. While this provides flexibility, it also means each resolver may fetch data independently, making it easier for new client queries to unintentionally trigger N+1 performance issues unless the server uses optimizations like batching or caching.

### The Solution: DataLoader
The standard fix for GraphQL's N+1 problem is the DataLoader pattern, originally created by Facebook (of course).

The idea is simple: instead of fetching one venue at a time, collect all the venue IDs needed in a single execution tick, then batch them into one query.

DataLoader does two things:

1. **Batching**: Collects all .load() calls within a single tick of the event loop and fires the batch function once with all collected IDs.
2. **Caching**: If venueId: 10 is requested twice in the same request, DataLoader returns the cached result from the first fetch. This is per-request caching, not a long-lived cache.

## Caching Solutions
- **Persisted queries** : Hash each query at build time, send the hash instead of the full query. Server maps hash → query. Can use GET requests with the hash as a URL parameter.

- **Apollo Client cache** : Client-side normalized cache. Stores entities by `__typename + id` and deduplicates across queries.

- **Response caching**:	Cache entire responses server-side keyed by query hash + variables.

## Security: Query Depth Attacks
In REST, you control every response.

This query bounces between `Event → Venue → Event → Venue` and could generate millions of database queries and return gigabytes of data. It's essentially a denial-of-service attack through your own API.


### Solutions
- **Query depth limiting**:	Reject queries deeper than N levels (typically 5-10)

- **Query complexity analysis**:	Assign a "cost" to each field; reject queries exceeding a total cost budget

- **Rate limiting**:	Limit requests per client, but also limit total query cost per time window
- **Timeout**	Kill query execution that exceeds a time limit

## Field-Level Authorization
In GraphQL, a single query can request both public and sensitive data together, so permissions must be checked for each field individually. This gives more flexible and fine-grained security, but also makes authorization more complex because every sensitive field needs its own access rule.

## Performance Monitoring: Harder Than REST
Monitoring REST APIs is easier because each endpoint is separate, so tools can directly track response times, errors, and performance for each route. In GraphQL, all requests usually go through a single /graphql endpoint, so different queries are mixed together and harder to analyze. Also, GraphQL often returns HTTP 200 even when part of the query fails, with errors stored inside the JSON response body, which means normal monitoring tools may miss those errors unless they specifically understand GraphQL.

## Schema Evolution: Easy to Add, Hard to Remove
Adding new fields to a GraphQL schema is trivial and backward-compatible. Existing queries don't request the new field, so they're unaffected.

Removing fields is much harder. You can't just delete a field — any client still querying it will break. Instead, you deprecate it.

---


# Chapter 8 How RPC Works
RPC is like calling someone on the phone. You dial a number, tell them exactly what you need — "Hey, get me the details for event 123" — and they do it and tell you the result. You don't care where their filing cabinet is. You just called a function and got an answer.

When you write `getEvent('123')` in your code, it feels like a local function call. But a lot is happening behind the scenes to make that illusion work. Here's the step-by-step flow:

```text
Client Code                          Server Code
     |                                    |
     | 1. Call getEvent('123')            |
     |        |                           |
     |  [Client Stub]                     |
     |   2. Serialize args                |
     |   3. Send over network  -------->  |
     |                              [Server Stub]
     |                               4. Deserialize args
     |                               5. Call real getEvent('123')
     |                               6. Get result
     |                               7. Serialize response
     |                          <--------  8. Send over network
     |  [Client Stub]                     |
     |   9. Deserialize response          |
     |  10. Return result                 |
     |                                    |

```
**1. Client Stub (Proxy)** :This is auto-generated code on the client side. When you call `getEvent('123')`, you're actually calling this stub, not the real function. It acts like a fake local version of the function and handles all network work behind the scenes.

**2. Serialization** : The stub converts the input (`'123'`) into a transferable format like JSON, Protocol Buffers, or XML so it can be sent over the network.

**3. Network Transport**:  The request is sent to the server over the network (often via TCP or HTTP/2 in modern systems like gRPC).

**4–5. Server Stub (Skeleton)**  
   The server receives the request, converts it back into usable data (deserialization), and calls the actual `getEvent()` function, which may fetch data from a database.

**6–8. Response Handling**  
   The server gets the result, serializes it again, and sends it back over the network.

**9–10. Client Receives Result**  
   The client stub deserializes the response and returns it to your code, making it look like a normal local function call.

### Key Idea
The main advantage of RPC is that all the network complexity (serialization, transport, server execution, response handling) is hidden from the developer. You just write: 
`result = getEvent('123')`


You don't need RPC — your functions are all in the same process. But when you break that monolith into 50 or 500 microservices, those services need to talk to each other. A lot. With tight performance requirements.

REST works great for public-facing APIs where human readability matters. But for internal service-to-service communication where:

- Both sides are machines, not humans
- You control both the client and the server
- You need maximum performance
- You have dozens of languages across services
- You need strict type safety across team boundaries

## REST vs. RPC: The Mental Model Comparison

Here's a comprehensive comparison to cement the difference:

| Aspect | REST | RPC |
|---|---|---|
| Mental model | "Interact with resources" | "Call remote functions" |
| URL design | Nouns: `/events/123` | Verbs: `getEvent()` |
| Data format | Usually JSON (text) | Varies: JSON, XML, binary (Protobuf) |
| Coupling | Loose — client discovers resources | Tighter — client must know function signatures |
| Best for | Public APIs, CRUD operations | Internal APIs, complex actions |
| Human readability | High — you can test with curl | Lower — binary formats need tooling |
| Browser support | Native | Requires special tooling |
| Caching | Built-in HTTP caching (GET requests) | Not built-in — must implement yourself |
| Error handling | HTTP status codes (404, 500, etc.) | Framework-specific error codes |
| Discoverability | URLs are self-describing | Need documentation or `.proto` files |

<br/>
<br/>

Think of gRPC as the modern answer to the question: "How do I make RPC that's fast, type-safe, and works across any programming language?"

The answer has three parts:

1. **Protocol Buffers** — A binary serialization format that's the "language" services speak
2. **HTTP/2** — The transport protocol that carries the data
3. **Code Generation** — Tooling that auto-generates client and server stubs from a single definition file

**Protocol Buffers (Protobuf)** is a way to define both the structure of data and the available API actions in a single `.proto` file, which acts as a strict contract between client and server. It specifies what functions (RPCs) exist, what inputs they take, and what outputs they return. Both sides then generate code from this file, ensuring they always agree on the exact format. Unlike JSON, Protobuf uses numeric field IDs instead of field names when sending data, making communication much smaller and faster.

**Type Safety: Catch Bugs at Compile Time**

With JSON APIs, there’s no strict contract between client and server, so if the server changes something like a field name (date → event_date), the client won’t immediately know and may fail silently at runtime. This often leads to bugs that only show up in production. With Protocol Buffers in systems like gRPC, both client and server generate code from the same .proto contract, so any mismatch (like renamed or removed fields) breaks compilation. This means many integration bugs are caught early during development instead of showing up later in production.

##  HTTP/2: Multiplexing and Header Compression
Traditional REST APIs typically run over HTTP/1.1, gRPC uses HTTP/2, which provides:

| Feature | HTTP/1.1 | HTTP/2 |
|---|---|---|
| Requests per connection | One at a time | Many in parallel (multiplexing) |
| Header format | Text, repeated every request | Binary, compressed (HPACK) |
| Connection overhead | New connection per request (often) | Single long-lived connection |
| Server push | Not supported | Supported |

> 💡 **Interview Tip:**  
> If you say "gRPC is faster because it doesn't use HTTP," a knowledgeable interviewer will immediately flag this. The correct framing is: "gRPC leverages HTTP/2 for multiplexing and header compression, combined with Protocol Buffers for compact binary serialization. **The performance gain comes from both the transport efficiency and the serialization format.**"

**Protocol Buffers isn't the only binary serialization format. Here's how it compares to alternatives**

## (Imp) Protobuf Field Numbers and Backward Compatibility

Those field numbers in `.proto` files `(= 1, = 2, etc.)` are critical for `backward compatibility`.

```protobuf
message Event {
  string event_id = 1;
  string name = 2;
  string venue = 3;
  int64 date_unix = 4;
  // Added in v2 — old clients won't know about this field
  string category = 5;
  // Added in v3
  bool is_sold_out = 6;
}
```

Protobuf field numbers act like permanent IDs for each field, not the field names. Because of this, services stay compatible even when the data model changes.

If you add a new field (like category = 5), old clients `(code)` just ignore it, and new clients can still work even if older servers don’t send it. If you remove a field, old data is still safely ignored as long as you don’t reuse its number. If you rename a field, nothing breaks because names are only used in code, not in the actual data sent over the network. The only dangerous change is changing or reusing field numbers, because that would make old and new systems interpret data incorrectly. This system lets different services evolve independently without breaking each other.


```protobuf
// This is SAFE — renaming doesn't change the wire format
message Event {
  string event_id = 1;
  string event_name = 2;  // was "name" — field number 2 is unchanged
}

// This is DANGEROUS — never reuse or change field numbers
message Event {
  string event_id = 1;
  string name = 3;  // WRONG — changed from 2 to 3, breaks all existing clients
}
```

**The rule of thumb**: if you control both sides of the communication and you need performance or type safety, use gRPC. If your API needs to be consumed by the general public, use REST with JSON.

---

# Chapter 9 Streaming Patterns and When to Use gRPC

## The Four gRPC Communication Patterns
REST has one communication pattern: request in, response out. gRPC has four, and understanding when to use each one is what separates a surface-level answer from a strong one in interviews.

![text13](/assets/13.png)

### Pattern 1: Unary RPC (The Familiar One)
**When to use**: Any standard request-response operation

### Pattern 2: Server Streaming (The Firehose)
The client sends a single request, and the server responds with a stream of messages.

When to use:

- Real-time price feeds (stocks, crypto, sports scores)
- Live progress updates (file processing, ML model training)
- Event logs (streaming log entries as they occur)
- Search results delivered incrementally (return results as they're found, not all at once)


**Why not just poll with REST?** With REST, you'd call GET /stocks/AAPL/price every second. That's 60 HTTP requests per minute, each with full headers, connection setup, and JSON parsing. With server streaming, you open one connection and the server pushes small binary updates as prices change. **Drastically less overhead**.

### Pattern 3: Client Streaming (The Upload)
The client sends a stream of messages, and the server responds with a single message after it's received everything (or enough).

When to use:

- File uploads in chunks
- Sending batches of sensor/IoT data
- Aggregating data from the client before processing (e.g., collecting GPS points for a route, then calculating the total distance)
- Log shipping (client streams log entries, server acknowledges when batch is stored)

### Pattern 4: Bidirectional Streaming (The Conversation)
Both the client and server send streams of messages simultaneously. Neither side has to wait for the other to finish. This is full-duplex communication over a single connection.

When to use:

- Real-time chat applications
- Collaborative editing (Google Docs-style)
- Multiplayer game state synchronization
- Interactive voice/video processing (send audio frames, receive transcription in real time)


## Deadlines and Timeouts: gRPC's Built-In Safety Net
One of gRPC's underrated features is deadline propagation. In a microservices architecture, a single user request might trigger a chain of internal calls:

`User → API Gateway → Order Service → Payment Service → Fraud Detection`

With REST, if the user's request has a 5-second timeout, each service in the chain has no idea about that deadline. The Order Service might wait 4 seconds for Payment, then Payment waits 4 seconds for Fraud Detection — and the user's request times out at the gateway while services are still working.

gRPC solves this by propagating deadlines through the call chain:

- Each service in the chain knows exactly how much time it has left. 
- This is particularly important for avoiding `cascading failures`.

## Load Balancing with gRPC

Load balancing is harder in gRPC because it uses long-lived HTTP/2 connections instead of creating a new connection for every request like REST. In REST, each request can easily be sent to a different server, so load balancers can distribute traffic evenly. But in gRPC, one connection can carry many requests, so if a client connects to one server, all its requests may keep going to that same server, causing uneven load.

**Solutions**:
- L7 (application-level) load balancer
- Client-side load balancing
- service meshes which manage traffic distribution automatically.


## gRPC-Web: Bringing gRPC to the Browser
- Browsers can’t directly use gRPC because gRPC needs low-level HTTP/2 features (like trailers and streaming) that browser APIs such as fetch() don’t fully support. 
- To fix this, gRPC-Web acts as a bridge: the browser sends a simplified HTTP request, then a proxy like Envoy converts it into real gRPC for the backend service. 
- This allows web apps to still use Protobuf-based APIs, but with some trade-offs like extra latency (because of the proxy) and limited streaming support. 
- That’s why `REST is still more common for public browser-facing` APIs, while gRPC is mostly used for backend-to-backend communication.

- Client streaming and bidirectional streaming are not fully supported in all implementations 

## Use gRPC When:
1. **Internal service-to-service calls**
2. **Polyglot environments (Go + Java + Python services)**
3. **Streaming requirements**
4. **High-throughput data pipelines**:	Binary serialization saves bandwidth and CPU at scale
5. **Mobile clients on constrained networks**:	Smaller payloads = faster loads, less data usage

## Do NOT Use gRPC When:
1. **Public-facing APIs for third-party developers**
2. **Simple CRUD applications**
3. **Browser-first applications without a proxy**
4. **Quick prototypes or MVPs**: JSON is simpler to debug, test, and iterate on
5. **APIs that need human readability**


## The Common Production Pattern: REST + gRPC


```text
                    Internet
                       |
                  [API Gateway]
                   /        \
            REST (JSON)    REST (JSON)
              /                \
    [Mobile App]          [Web Browser]
         |                      |
         +------→ [API Gateway] ←------+
                       |
                  gRPC (Protobuf)
                 /     |      \
        [User       [Event    [Payment
         Service]    Service]  Service]
              \       |        /
          gRPC (Protobuf) internally
                  |
           [Notification Service]
```
**Public-facing layer**: REST with JSON. Browsers and mobile apps send HTTP requests with JSON bodies. Easy to debug, easy to document (OpenAPI/Swagger), easy for third-party developers.

**Internal layer**: gRPC with Protocol Buffers. Services communicate with binary messages over HTTP/2. Fast, type-safe, and streamable. Teams define contracts in .proto files and generate code in whatever language they prefer.

---

