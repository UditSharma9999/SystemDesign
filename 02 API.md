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

---

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

- **Response caching**: Cache entire responses server-side keyed by query hash + variables.

## Security: Query Depth Attacks

In REST, you control every response.

This query bounces between `Event → Venue → Event → Venue` and could generate millions of database queries and return gigabytes of data. It's essentially a denial-of-service attack through your own API.

### Solutions

- **Query depth limiting**: Reject queries deeper than N levels (typically 5-10)

- **Query complexity analysis**: Assign a "cost" to each field; reject queries exceeding a total cost budget

- **Rate limiting**: Limit requests per client, but also limit total query cost per time window
- **Timeout** Kill query execution that exceeds a time limit

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

**3. Network Transport**: The request is sent to the server over the network (often via TCP or HTTP/2 in modern systems like gRPC).

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

| Aspect            | REST                                 | RPC                                            |
| ----------------- | ------------------------------------ | ---------------------------------------------- |
| Mental model      | "Interact with resources"            | "Call remote functions"                        |
| URL design        | Nouns: `/events/123`                 | Verbs: `getEvent()`                            |
| Data format       | Usually JSON (text)                  | Varies: JSON, XML, binary (Protobuf)           |
| Coupling          | Loose — client discovers resources   | Tighter — client must know function signatures |
| Best for          | Public APIs, CRUD operations         | Internal APIs, complex actions                 |
| Human readability | High — you can test with curl        | Lower — binary formats need tooling            |
| Browser support   | Native                               | Requires special tooling                       |
| Caching           | Built-in HTTP caching (GET requests) | Not built-in — must implement yourself         |
| Error handling    | HTTP status codes (404, 500, etc.)   | Framework-specific error codes                 |
| Discoverability   | URLs are self-describing             | Need documentation or `.proto` files           |

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

## HTTP/2: Multiplexing and Header Compression

Traditional REST APIs typically run over HTTP/1.1, gRPC uses HTTP/2, which provides:

| Feature                 | HTTP/1.1                           | HTTP/2                          |
| ----------------------- | ---------------------------------- | ------------------------------- |
| Requests per connection | One at a time                      | Many in parallel (multiplexing) |
| Header format           | Text, repeated every request       | Binary, compressed (HPACK)      |
| Connection overhead     | New connection per request (often) | Single long-lived connection    |
| Server push             | Not supported                      | Supported                       |

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
4. **High-throughput data pipelines**: Binary serialization saves bandwidth and CPU at scale
5. **Mobile clients on constrained networks**: Smaller payloads = faster loads, less data usage

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

# Chapter 10 Pagination

## Offset-based pagination

It is like browsing pages in a book or shopping website. Instead of loading all data at once, the API gives a small chunk at a time. For example, offset=20&limit=10 means “skip the first 20 items and show the next 10.” This helps apps load faster and avoids overwhelming the server.

- Simple to implement
- Random page access: Users can jump to page 5, page 50, page 500
- Easy to calculate total pages

#### The Data Shift Problem (This Is What Breaks It)

You're browsing a list of events sorted by newest first. You load page 1 (items 1-10). While you're reading page 1, someone publishes a new event. That new event becomes item 1 in the database.

Now you request page 2 (offset=10&limit=10). What happens?

Every existing record shifted down by one position. The item that was at position 10 (last on your page 1) is now at position 11 (first on page 2). **You see it again**.

#### Performance at Scale

Offset pagination also becomes slow with huge databases. If you ask for OFFSET 1000000, the database must first scan through and skip one million rows before returning the next results. It doesn’t directly jump to row 1,000,000. So as the offset grows, queries take longer and use more resources, making performance worse on large datasets.

## Cursor-Based Pagination: Stable Under Chaos

Cursor-based pagination works like a bookmark. Instead of saying “skip 100 records,” the client says, “start after this last record I already saw.” The server sends a cursor (a hidden token) with each response, and the client uses it to fetch the next set of data. This is faster because the database can jump directly to the correct position using indexes instead of scanning thousands or millions of rows. It also avoids duplicates and missing records when new data is added while users are browsing.

```SQL
-- Let's say the cursor decodes to {"id": 123}:
SELECT * FROM events
WHERE id < 123
ORDER BY id DESC
LIMIT 10;
```

**The response includes the next cursor:**

```JSON
{
  "data": [ ... ],
  "pagination": {
    "next_cursor": "eyJpZCI6MTEzfQ==",
    "has_more": true
  }
}
```

#### Why It's Stable Under Writes (data shift scenario)

Cursor pagination stays reliable even when new data is added. If you already saw items A–E, the cursor remembers “start after E.” Even if a new item Z is inserted at the top, your next request still correctly returns F–J. Nothing gets duplicated or skipped because the cursor tracks a specific record, not a changing row position.

**Can Cursors Use Non-ID Fields?** :Yes. Cursors can reference any field, as long as it produces a unique, stable ordering. Common choices:

| Cursor Field                   | Works? | Notes                                                       |
| ------------------------------ | ------ | ----------------------------------------------------------- |
| Auto-incrementing ID           | Yes    | Natural order, unique, perfect                              |
| Created timestamp              | Yes    | Add a tiebreaker (ID) for records with identical timestamps |
| Composite key (timestamp + ID) | Yes    | Most robust approach for sorted feeds                       |
| Name (alphabetical)            | Risky  | Not unique — multiple records named "John" break it         |
| Random field                   | No     | No stable ordering                                          |

### Cursor Edge Cases

#### What if a new record is inserted before the cursor?

If you're scrolling through a feed and someone publishes a new post above where you are, your pagination continues forward without ever fetching it. This is usually fine for feeds — the client can "pull to refresh" to see new content at the top.

#### What if the cursor record is deleted?

This depends on your implementation. Two common approaches:

1. **Use the next valid record**: If the cursor pointed to record 123 and it's been deleted, find the nearest record that still exists and start from there. This is the most robust approach.
2. **Return an error**: Tell the client the cursor is invalid and they should start over. Simpler, but worse UX.

### Keyset Pagination: Cursor's Under-the-Hood Sibling

Keyset pagination is the database technique: using a `WHERE` clause to filter by the last-seen key instead of using `OFFSET`. Cursor pagination is the API pattern built on top of it: encoding that key into an opaque token and passing it between client and server.

**Avoid paginating inside another paginated list**. For example, if posts are paginated and each post’s comments are also paginated, the client must track pagination state for every single post, which becomes complex and buggy when data changes.

A better approach is:

- Show only a few comments inline (like the top 3) with a “view all comments” option, or
- Paginate only one level at a time — fetch paginated posts first, then load comments separately when needed.

## When to Use Which

### Use offset when:

- Data doesn't change frequently (archival data, reports)
- Users need to jump to specific pages (admin dashboards, search results)
- Dataset is small enough that high-offset performance isn't a concern
- You need a "total results" count

### Use cursor when:

- Data changes frequently (social feeds, notifications, chat messages)
- Users scroll linearly (infinite scroll, "load more" buttons)
- Dataset is large and performance matters
- Consistency matters more than random page access

### Use both:

- Some APIs offer both. The `/events` endpoint might support cursor for the mobile app's infinite scroll and offset for the admin dashboard's page-numbered table. Same data, different access patterns.

---

# Chapter 11 Versioning and API Compatibility

API versioning is like updating a restaurant menu without breaking delivery apps that still use the old one. Instead of replacing everything instantly, you create a new version while keeping the old version working for some time. This lets clients switch gradually without errors or downtime. In APIs, versioning helps developers safely change endpoints, data structures, or features without breaking existing applications.

| Change                                | Breaking? | Needs New Version? |
| ------------------------------------- | --------- | ------------------ |
| Add a new field to response           | No        | No                 |
| Add a new optional query parameter    | No        | No                 |
| Add a new endpoint                    | No        | No                 |
| Remove a field from response          | Yes       | Yes                |
| Rename a field                        | Yes       | Yes                |
| Change a field's type (string to int) | Yes       | Yes                |
| Make an optional field required       | Yes       | Yes                |
| Change the URL structure              | Yes       | Yes                |

> The pattern: **adding is safe, removing/changing is dangerous**.

## The Four Versioning Strategies

### 1. URL Versioning (The Most Common)

The version lives right in the URL:

    GET /v1/events/123
    GET /v2/events/123

| Pros                                                        | Cons                                                                     |
| ----------------------------------------------------------- | ------------------------------------------------------------------------ |
| Impossible to miss                                          | URL pollution — every endpoint now has a version prefix                  |
| Easy to test                                                | Harder to sunset — removing v1 requires updating all clients             |
| Easy to route — gateways can split `/v1/` and `/v2/` easily | Philosophical objection — version shouldn’t define the resource identity |
| Simple mental model — “v1 is old, v2 is new”                |                                                                          |

### 2. Header Versioning

The version is passed in a custom HTTP header:

    GET /events/123
    Accept-Version: v2

| Pros                                                                  | Cons                                                  |
| --------------------------------------------------------------------- | ----------------------------------------------------- |
| Clean URLs — resource path stays pure (/events/123)                   | Less visible — version isn’t obvious from the URL     |
| Follows HTTP conventions — headers are the correct place for metadata | Harder to test — requires curl/Postman to set headers |
| Date-based versioning (e.g., Stripe-style) makes evolution clear      | Easy to forget — clients may omit the version header  |

### 3. Query Parameter Versioning

The version is a query parameter:

    GET /events/123?version=2

This is a middle ground — visible in the URL but not baked into the path structure. It's less common and generally considered less clean than the other approaches.

### 4. Content Negotiation

The version is part of the `Accept` header's media type:

    GET /events/123
    Accept: application/vnd.myapi.v2+json

This is the most "RESTful" approach according to purists, but it's rarely used in practice because it's cumbersome and unfamiliar to most developers.

> 💡 **Interview Tip:**  
> URL versioning is the safe default in interviews. It's the most widely understood, the easiest to explain, and no interviewer will question it. If asked "why not headers?", say: "URL versioning is more explicit, easier to test, and simpler to route at the infrastructure level.

## Forward Compatibility: Designing APIs That Don't Break

- Forward compatibility means designing an API so future changes don’t break older clients. One key rule is to use enums instead of booleans, because booleans only allow two states, but real systems often grow more states over time. For example, instead of is_active: true/false, use status: "active", which can later expand to "inactive", "suspended", or more without breaking existing clients.

- Another important practice is to keep new fields optional, so older clients don’t break when they don’t send them. The server can safely use default values.

- Finally, prefer additive changes over modifications. Instead of changing or removing existing fields or behavior, add new fields, new endpoints, or optional parameters. This way, old clients keep working while new clients can adopt improved features gradually.

## Backward Compatibility: Old Clients Still Work

Backward compatibility means your new API should not break apps that were built using the old version. You can safely add new fields, endpoints, or optional parameters, and also expand allowed values without affecting existing clients.

But things like removing fields, renaming them, changing data types, or making optional fields required will break old clients and should be avoided or handled carefully.

When breaking changes are unavoidable, you use a `deprecation strategy`: warn users in advance, support both old and new versions together, set a sunset date, and track usage before removing the old version.

A good structure like semantic versioning helps manage this `(major (v1 to v2) = breaking changes, minor (v1.1) = new features, patch (v1.1.1) = fixes)`.

## HATEOAS (Hypermedia as the Engine of Application State)

HATEOAS is a way of designing APIs where the server tells the client what it can do next by including links inside the response. Instead of the client memorizing endpoints like “POST /events/123/cancel,” it simply follows the links provided by the API.

For example, an event response might include links for viewing itself, canceling it, or checking tickets. If an action isn’t allowed (like canceling a finished event), the API just doesn’t include that link, so the client automatically knows it’s not possible.

Most modern APIs skip HATEOAS because it adds extra complexity and isn’t very useful when the frontend already knows the API structure. But it’s becoming interesting again for AI agents, since they don’t rely on hardcoded endpoints and can “navigate” an API just by following the links provided in responses.

---

# Chapter 12 Filtering, Sorting, and Search Patterns

## Filtering: Narrowing Results

### 1. Simple Equality Filters

The most basic pattern: a query parameter matches a field value exactly.

    GET /events?city=NYC
    GET /events?category=music
    GET /events?city=NYC&category=music

Each parameter maps to a WHERE clause:

```sql
SELECT * FROM events WHERE city = 'NYC' AND category = 'music';
```

### 2. Range Filters

    GET /events?date_from=2024-01-01&date_to=2024-12-31
    GET /events?price_min=20&price_max=100

| Pattern                 | Example              | Notes                                                    |
| ----------------------- | -------------------- | -------------------------------------------------------- |
| field_from / field_to   | date_from=2024-01-01 | Clear, explicit                                          |
| field_min / field_max   | price_min=20         | Good for numeric ranges                                  |
| field_gte / field_lte   | date_gte=2024-01-01  | Mirrors database operators (gte = greater than or equal) |
| field[gte] / field[lte] | date[gte]=2024-01-01 | Bracket notation, used by Stripe                         |

Pick one convention and use it everywhere. Don't use date_from on one endpoint and created_gte on another. Inconsistency is the fastest way to frustrate developers integrating with your API.

### 3. Multiple Values (IN Filters)

When a client wants records matching any of several values:

`GET /events?category=music,sports,theater`

```sql
SELECT * FROM events WHERE category IN ('music', 'sports', 'theater');
```

Some APIs use repeated parameters instead:

`GET /events?category=music&category=sports&category=theater`  
Both work. The comma-separated approach is more concise

### 4. Nested Filters (Dot Notation)

    GET /events?venue.city=NYC
    GET /orders?customer.tier=premium

```SQL
SELECT events.* FROM events
JOIN venues ON events.venue_id = venues.id
WHERE venues.city = 'NYC';
```

**Consistency Is Everything** : The most important rule is consistency. If one endpoint uses city=NYC or date_from, every other endpoint should follow the same pattern. If different endpoints behave differently, it becomes confusing and hard for developers to use the API correctly.

## Sorting: Controlling Order

### 1. Basic Sorting

    GET /events?sort=date&order=asc
    GET /events?sort=price&order=desc

### 2. Multiple Sort Fields

    GET /events?sort=-date,name

```SQL
SELECT * FROM events ORDER BY date DESC, name ASC;
```

**Only Allow Sorting on Indexed Fields** : Sorting on a non-indexed field forces a full table scan. If your `events` table has 10 million rows and someone requests `?sort=description`, the database has to read every row and sort them by description text. This can take seconds or even minutes.

## Search: Finding Items by Text

### 1. Basic Search

    GET /events?q=concert+NYC

```slq
SELECT * FROM events
WHERE to_tsvector(name || ' ' || description || ' ' || city)
      @@ to_tsquery('concert & NYC');
```

The `q` parameter is the universal convention for search queries

#### When to Use a Dedicated Search Endpoint

For simple search, a q parameter on the list endpoint works fine. But when search becomes complex — with autocomplete, facets, highlighting, relevance scoring — a dedicated endpoint is cleaner:

    GET /events/search?q=concert+NYC&autocomplete=true

This separates the "browse events" use case (list endpoint with filters) from the "find events" use case (search endpoint with text matching). Different backend optimizations, different response formats.

## Combining Everything: Filter, Sort, Search, Paginate

    GET /events?city=NYC&category=music&sort=-date&limit=20&cursor=abc123

This says: "Give me music events in NYC, sorted by newest first, 20 at a time, starting after this cursor."

Order of Operations

The server processes these in a specific order:

- **Filter** — narrow down to matching records (WHERE city='NYC' AND category='music')
- **Search** — if a q parameter is present, apply text matching
- **Sort** — order the filtered results (ORDER BY date DESC)
- **Paginate** — slice the sorted results (LIMIT 20 with cursor)

**This order matters.**

## Batch Operations: Processing Many Items at Once

Sometimes clients need to create, update, or delete many records at once. Making 100 individual API calls is slow and wasteful. Batch endpoints let clients send them all in one request.

What happens when you send 100 items in a batch and 3 of them fail? This is where most APIs get it wrong.

- **Bad approach 1**: Return 500 for the entire batch. The client has no idea what succeeded and what didn't. If they retry, the 97 successful items might get duplicated.

- **Bad approach 2**: Return 200 and silently drop the failures. The client thinks everything succeeded. Data is quietly lost.

- **Good approach**: Return a **structured partial-success** response that clearly indicates the outcome for each item:

```JSON
{
  "status": "partial_success",
  "summary": {
    "total": 100,
    "succeeded": 97,
    "failed": 3
  },
  "results": [
    { "index": 0, "status": "success", "id": 1001 },
    { "index": 1, "status": "success", "id": 1002 },
    {
      "index": 2,
      "status": "error",
      "error": {
        "code": "VALIDATION_ERROR",
        "message": "Date must be in the future",
        "field": "date",
        "retryable": false
      }
    },
    .....
  ]
}
```

**Retryable flag** — tells the client whether retrying would help.

**HTTP status code** — use `207 Multi-Status for partial success`

**Expansion / Embedding: Inline Related Resources** : sometimes the client wants more data in a single request.

    GET /events/123          → { "name": "Concert", "venue_id": 456 }
    GET /venues/456          → { "name": "Madison Square Garden", "city": "NYC" }

with With expansion, one call does it:

    GET /events/123?expand=venue

Allow expanding direct relationships but limit the depth. `?expand=venue` is fine. `?expand=venue.city.country.continent` is a database nightmare.

---

#### OData (Open Data Protocol)

A standardized query language for REST APIs used heavily in Azure, Microsoft Graph, SAP, and Dynamics 365.

OData uses a `$` prefix convention for query operators:

    GET /products?$filter=price gt 100 and category  'electronics'
                  &$select=name,price
                  &$orderby=price desc
                  &$top=20
                  &$skip=40

---

# Chapter 13 Async Operations, Caching, and Optimistic Concurrency

Some tasks in an application take a long time to finish, such as generating large reports.

If the server makes the user wait until the task is fully completed, the request might time out and the user may think something went wrong. They could even click the button again, accidentally starting the same process twice.

To avoid this, APIs use asynchronous operations, where the server accepts the request first and completes the work in the background.

## 202 Accepted + status polling

The common solution is called the **“202 Accepted + status polling”** pattern. When the client sends a request, the server immediately responds with `202 Accepted` instead of waiting for the task to finish. This response means, `“I received your request and I’ll process it soon.”` Along with this, the server returns a job ID and a status URL. The job ID uniquely identifies the background task, while the status URL is an endpoint the client can repeatedly check to see how the work is progressing or can use `Estimated completion time`.

After that, the client periodically sends requests to the status endpoint. If the task is still running, the server responds with information such as "processing" and maybe a progress percentage like 65%. This helps the frontend show loading indicators or progress bars to the user.

Once the task is complete, the server responds with the final result or redirects the client to the completed resource using `303 See` Other and a `Location header`. If something goes wrong, the status endpoint returns a "failed" status along with an error message explaining the problem.

### Alternatives to Polling

Polling works but wastes requests. Better options exist:

- Webhooks
- WebSockets
- SSE

## ETags and Optimistic Concurrency

ETags and optimistic concurrency are used to prevent people from accidentally overwriting each other’s changes when multiple users edit the same data at the same time. Imagine two people opening the same document. Both see the original version, make different edits, and save their changes. Without protection, the second save would overwrite the first one, causing the first person’s work to be lost. This problem is called a lost update.

### Solution : ETag

APIs use something called an `ETag (Entity Tag)`. An ETag is like a version fingerprint for a resource. Whenever the resource changes, the ETag also changes. For example, when a client requests a document, the server sends both the document data and an ETag value such as `"a1b2c3d4"`. The client stores this ETag along with the data it received.

Later, when the client wants to update the document, it sends the ETag back using the **If-Match header**. This tells the server: `“Only save my changes if the document is still the same version I originally fetched.”` The server then compares the provided ETag with the current one stored on the server.

If the ETags match, it means nobody else changed the document in the meantime, so the update is safe and the server saves it. After saving, the server generates a new ETag because the content has changed. But if the ETags do not match, it means someone else already updated the document. In that case, the server rejects the request with **412 Precondition Failed**. This prevents accidental overwriting and tells the client to fetch the latest version before trying again.

This works very well for APIs because it avoids locks, improves performance, keeps the server stateless, and still protects data from accidental overwrites.

There are also two kinds of ETags.

1. **Strong ETag** changes whenever even a tiny part of the content changes, making it ideal when exact matching is required.
2. weak ETag only checks whether the content is semantically similar, even if formatting differs.

Most APIs prefer strong ETags because they are simpler and safer.

## HTTP Caching

HTTP caching is a way to make apps and websites faster by avoiding unnecessary data downloads. Imagine a mobile app that shows event details. Every time the user opens the event page, the app sends a request to the server like GET /events/123. If the event information has not changed, sending the same data again wastes internet bandwidth and server resources.

The server controls caching using the **Cache-Control header**.  
For example, **max-age=600** means the client can use the cached response for `600 seconds (10 minutes)` without contacting the server again. During that time, the app can instantly show the stored data instead of downloading it again.

**private** : directive means only the user’s device can cache the response.

**public** : means shared systems like CDNs and proxy servers can cache it too.

**no-store** : Never cache this response at all

**no-cache** : You can cache it, but must revalidate with the server before using it.

> `No-cache`: ("It means the client may store the response, but before using it again, it must first check with the server to confirm the data is still up to date.")

To check whether cached data has changed, HTTP uses **ETags** such as `"abc123"`. when the cache expires, the client sends the ETag back using the **If-None-Match header**. This asks the server, `“Has this data changed since version abc123?”`

If the data has not changed, the server replies with **304 Not Modified**. This response contains no actual data because the client already has it cached locally. The client simply reuses the stored copy, saving bandwidth and improving speed. If the data has changed, the server sends a normal **200 OK response** with updated data and a new ETag. The client then replaces the old cached version with the new one.

### Caching Strategy by Resource Type

| Resource Type                 | Cache Strategy             | Why                                       |
| ----------------------------- | -------------------------- | ----------------------------------------- |
| Static assets (images, CSS)   | `public, max-age=31536000` | Rarely change, safe to cache aggressively |
| User-specific data            | `private, max-age=300`     | Only for this user, moderate freshness    |
| Real-time data (stock prices) | `no-store`                 | Stale data is dangerous                   |
| Event listings                | `public, max-age=60`       | Changes sometimes, CDN-cacheable          |
| Auth tokens                   | `no-store`                 | Never cache sensitive credentials         |

> 💡 Interview Tip  
> Mentioning Cache-Control headers and ETags shows you think about performance at the HTTP layer, not just application logic. This is especially relevant for CDN-heavy architectures.

## Content Negotiation

Content negotiation is a mechanism in HTTP that allows the client and server to agree on the format of the data being exchanged. Different clients may prefer different formats such as JSON, XML, HTML, or plain text. Instead of the server always sending the same format, the client tells the server what formats it can understand, and the server chooses the best matching one.

This is done using the `Accept` header. For example, a client may send:

    GET /events/123
    Accept: application/json, application/xml;q=0.9

The **q** value (0-1) indicates preference. JSON is preferred (q=1.0 by default), XML is acceptable (q=0.9).

The server then checks which formats it can produce and chooses the best match. If the server supports JSON, it responds like this:

    HTTP/1.1 200 OK
    Content-Type: application/json

    { "id": 123, "name": "Concert" }

Sometimes negotiation can fail. If the client asks for formats the server cannot provide, the server returns **406 Not Acceptable**. This means, “I understood your request, but I cannot generate any of the formats you requested.”

Another related error is **415 Unsupported Media Type**. This happens when the client sends data to the server in a format the server does not understand.

Most modern APIs only support JSON, making content negotiation less relevant than it once was.

## JSON Patch vs JSON Merge Patch

When we talked about PATCH in previous chapter's, we said it does "partial updates." But how does the client describe those partial updates? There are two standard formats.

### JSON Merge Patch (RFC 7396)

The simpler approach. Send a JSON object with only the fields you want to change:

    PATCH /users/123
    Content-Type: application/merge-patch+json
    {
      "email": "new@example.com",
      "phone": null
    }

This sets email to the new value and deletes phone (null = remove). Fields not included are unchanged.

**Limitation**: You can't set a field to null without deleting it, because null means "remove." If your data model uses null values meaningfully, merge patch won't work.

### JSON Patch (RFC 6902)

The more powerful approach. Send an array of operations:

    PATCH /users/123
    Content-Type: application/json-patch+json

    [
      { "op": "replace", "path": "/email", "value": "new@example.com" },
      { "op": "remove", "path": "/phone" },
      { "op": "add", "path": "/addresses/1", "value": { "city": "NYC" } },
      { "op": "test", "path": "/version", "value": 5 }
    ]

Operations: `add`, `remove`, `replace`, `move`, `copy`, `test` (it checks a precondition before applying changes, similar to optimistic concurrency).

| Format      | Simplicity   | Power   | Null Handling       | Use When                              |
| ----------- | ------------ | ------- | ------------------- | ------------------------------------- |
| Merge Patch | Very simple  | Limited | `null = delete`     | Simple field updates                  |
| JSON Patch  | More complex | Full    | Explicit operations | Complex mutations, arrays, conditions |

---

---

# Chapter 14 API Keys, JWTs, and OAuth 2.0

## Mechanism 1: Session-Based Authentication

### How It Works

1. **Client sends credentials**: POST /login with email and password.
2. **Server verifies**: Checks credentials against the database.
3. **Server creates a session**: Stores a session object (user ID, creation time, expiration) in memory or a session store (like Redis).
4. **Server sets a cookie**: Returns a Set-Cookie header with a session ID — a random string like sess_abc123xyz.
5. **Client sends cookie on every request**: The browser automatically includes it in every subsequent request.
6. **Server looks up the session**: On each request, the server takes the session ID from the cookie, looks it up in the session store, and retrieves the user's identity.

```text
Client                          Server                    Session Store
  |                               |                           |
  |-- POST /login (email, pass) ->|                           |
  |                               |-- verify credentials ---->|
  |                               |<-- user found ------------|
  |                               |-- store session --------->|
  |<-- Set-Cookie: sess_abc123 ---|                           |
  |                               |                           |
  |-- GET /events                 |                           |
  |   Cookie: sess_abc123 ------->|                           |
  |                               |-- lookup sess_abc123 ---->|
  |                               |<-- user_id: 42 -----------|
  |<-- 200 OK (events data) ------|                           |
```

| Pros                                                                | Cons                                                                                 |
| ------------------------------------------------------------------- | ------------------------------------------------------------------------------------ |
| Simple to implement                                                 | Server must store state (session store)                                              |
| Server has full control — revoke a session instantly by deleting it | Doesn't scale horizontally without sticky sessions or a shared session store (Redis) |
| Cookie handling is built into browsers — no client-side code needed | Cookies don't work well for mobile apps or third-party API access                    |
| Battle-tested and well-understood                                   | Cross-origin requests (CORS) with cookies get tricky                                 |

APIs consumed by mobile apps, third-party developers, or distributed microservices, sessions start to creak. That's where tokens come in.

## Mechanism 2: API Keys

An API key is a long, randomly generated string that identifies a client application.

    sk_live_4xxxxxxxxxxxxxxxxxxx
    pk_test_Txxxxxxxxxxxxxxxxxxx

The `sk_` and `pk_` prefixes are a convention popularized by Stripe: `sk` = secret key (keep on your server), `pk` = publishable key (safe for the frontend).

### How They Work

1. **Developer signs up** for your API and generates a key.
2. **Key is stored** in the database, associated with the developer's account.
3. **Client sends the key** in the Authorization header.
4. **Server looks up the key** in the database and identifies the caller.

| Pros                                                   | Cons                                                                     |
| ------------------------------------------------------ | ------------------------------------------------------------------------ |
| Dead simple to implement                               | No user context — the key identifies an application, not a user          |
| Easy to revoke — just delete the key from the database | No built-in expiration — keys live forever unless you add rotation logic |
| Great for server-to-server communication               | If leaked, attacker has full access until you manually revoke it         |
| Easy to rate-limit per key                             | Users shouldn't manage cryptographic strings — bad UX                    |

API keys are bad for user-facing apps. You don't want your mobile app users copy-pasting 40-character strings. You don't want those keys sitting in a JavaScript bundle where anyone can view-source them.

## Mechanism 3: JWT (JSON Web Tokens)

JWT decodes to three parts::

1.  **Header** — metadata about the token:

        {
          "alg": "RS256",
          "typ": "JWT"
        }

2.  **Payload** — the actual user data (called "claims"):

        {
          "user_id": 42,
          "email": "alice@example.com",
          "role": "admin",
          "exp": 1717024000
        }

3.  **Signature** — cryptographic proof that the token hasn't been tampered with.

### How JWT Authentication Works

1. **Client sends credentials:** POST /login with email and password.
2. **Server verifies**: Checks credentials against the database.
3. **Server creates a JWT**: Encodes the user's info into the payload, signs it with a secret key, and returns the token.
4. **Client stores the token**: Typically in memory or `localStorage`.
5. **Client sends the token on every request**: In the `Authorization: Bearer <token> header`.
6. **Server verifies the signature**: No database lookup needed. If the signature is valid, the payload is trustworthy.

```text
Client                          Server
  |                               |
  |-- POST /login (email, pass) ->|
  |                               |-- verify credentials against DB
  |                               |-- create JWT (sign with secret key)
  |<-- { "token": "eyJhbG..." } --|
  |                               |
  |-- GET /events                 |
  |   Authorization: Bearer eyJ.. |
  |                               |-- verify JWT signature (NO DB lookup)
  |                               |-- extract user_id: 42, role: admin
  |<-- 200 OK (events data) ------|
```

#### The Killer Feature: Stateless Verification

With sessions, the server must look up every session in a database or Redis store on every request. That's a bottleneck.

With JWTs, the server just `verifies the signature`. It's a CPU operation, not a database operation. This is a massive advantage in distributed systems — any service with the verification key can independently validate the token without calling a central auth service.

### JWT Signing: Symmetric vs. Asymmetric

1. **Symmetric signing (HS256)**: One shared secret key is used to both sign and verify the token. Like a shared password — anyone who can verify the token can also create fake tokens.

   **Problem**: Every service that needs to verify tokens must have the secret key. If any one of them is compromised, the attacker can forge tokens.

2. **Asymmetric signing (RS256)**: A private key signs the token, and a public key verifies it. The public key cannot be used to create tokens.

   **Advantage**: Only the auth service has the private key. Every other service gets the public key, which is safe to distribute widely. Even if an API service is compromised, the attacker can't forge tokens.

### JWT Gotchas

1. **JWTs can't be revoked easily**. Unlike sessions, you can't just delete a JWT. It's valid until it expires.
2. **Payload is NOT encrypted**. Anyone can decode a JWT and read the payload. Never put sensitive data in a JWT. The signature `prevents tampering, not reading`.
3. Token size. JWTs are larger than session IDs because they carry data. This adds up when sent with every request.

## OAuth 2.0 — The Industry Standard

OAuth 2.0 is not a replacement for API keys or JWTs. It's a framework that combines them.

### The Common Misconception

Many developers think the choice is:

- Option A: Use API keys
- Option B: Use JWTs
- Option C: Use OAuth 2.0

But that's wrong. OAuth 2.0 is Option A + Option B working together with a defined flow.

### How OAuth 2.0 Works (Client Credentials Flow)

This is the flow used for **server-to-server** and **developer API access** — the most relevant flow for system design interviews.

1. **Developer registers** an application and receives a Client ID (like a username) and a Client Secret (like a password). These are essentially API keys.
2. **Application sends credentials** to the token endpoint: POST /oauth/token with the Client ID and Client Secret.
3. **Auth server verifies** the credentials and returns a Bearer Token — which is typically a JWT.
4. **Application uses the Bearer Token** for all API calls: Authorization: Bearer <jwt>.
5. **Token expires (typically 1 hour)**. Application requests a new token with its credentials.

```text
Developer App                    Auth Server                 API Server
  |                                 |                           |
  |-- POST /oauth/token             |                           |
  |   client_id: abc123             |                           |
  |   client_secret: xyz789         |                           |
  |   grant_type: client_credentials|                           |
  |                                 |                           |
  |<-- { "access_token": "eyJ..",   |                           |
  |      "token_type": "Bearer",    |                           |
  |      "expires_in": 3600 }       |                           |
  |                                 |                           |
  |-- GET /api/events ----------------------------------------->|
  |   Authorization: Bearer eyJ..   |                           |
  |                                 |                   verify JWT
  |<-- 200 OK (events data) ----------------------------------------|
```

### Why OAuth 2.0 Exists

- **API keys alone have no expiration**. If leaked, they're valid forever.
- **OAuth tokens expire**. Even if intercepted, they're only valid for a short time (usually 1 hour).
- **OAuth provides scoped authorization**. A token can grant read:events but not write:events. API keys are typically all-or-nothing.
- **OAuth separates authentication from API access**. The auth server handles credentials; the API server just verifies tokens.

## When to Use What in Interviews

Here's the practical guide:

- **Designing a web app with a single backend?** Session-based auth is fine. Mention it and move on.
- **Designing a system with multiple microservices?** JWTs. "The auth service issues JWTs, and any downstream service can verify them independently using the public key."
- **Designing a public API for developers?** OAuth 2.0. "Developers register for Client ID and Secret, exchange them for a short-lived Bearer Token, and use that token for API calls."
- **The interviewer didn't specifically ask about auth?** Say "I'll use JWT-based authentication" and move on. Most interviewers don't deep-dive here — they want to see that you've thought about it.

---

# Chapter 15 Authorization, RBAC, and Multi-Tenancy

## Role-Based Access Control

Role-Based Access Control. Instead of giving permissions directly to each user, the system creates roles such as customer, manager, or admin, and each role has a set of allowed actions.

For example, in a ticket booking platform, a customer can browse events and buy tickets. A venue manager can do everything a customer can do, plus create or edit events for their venue. An admin has full access, including managing users and approving venues. When a request comes to the API, the backend first checks if the user is authenticated, then checks if their role allows the requested action.

Sometimes roles alone are not enough. The system may also need ownership checks. For instance, a customer should only be able to see their own bookings, not another customer’s bookings. An admin can view all bookings, while a venue manager can only see bookings related to their own venue. So authorization is often a mix of checking the user’s role and checking whether the resource belongs to them.

## CRITICAL: User Identity Comes from the JWT, NEVER from the Request Body

A very important rule in API security is that the server must never trust identity information sent in the request body. The user’s identity should always come from the JWT or authentication token because the token is verified and secure. Anything inside the request body can be changed by the client and cannot be trusted.

For example, imagine a booking cancellation API where the request body contains a user_id and a booking_id. If the server trusts the user_id from the body, an attacker could simply change that number and pretend to be another user. The server might then incorrectly allow them to cancel someone else’s booking. This happens because request body data is just plain input from the client and can easily be manipulated.

The **correct approach** is for the server to extract the user’s identity from the verified JWT token after authentication. Since the JWT is cryptographically signed, users cannot modify it without invalidating the signature. The server then compares the authenticated user’s ID from the token with the booking’s owner in the database. If they match, the action is allowed; otherwise, access is denied.

## Multi-Tenancy: The "Immediately Disqualify" Mistake

In a multi-tenant system, many different companies use the same application and database infrastructure at the same time. But each company’s data must stay completely private from others.

A common mistake developers make is putting the tenant or company ID directly in the API URL, such as `/tenants/123/events`. This looks simple, but it creates a serious security risk because a user can manually change the number in the URL to another tenant’s ID and try to access someone else’s data.

Imagine a user from Company A changes the request from /tenants/123/events to /tenants/456/events. If the backend accidentally trusts the URL or forgets to properly validate permissions on even one endpoint, the user could see Company B’s private information. These kinds of mistakes have caused real-world data breaches, lawsuits, and compliance violations. The problem is that developers are depending on every API endpoint to correctly check tenant access every single time, and in large systems, eventually someone forgets.

**Recommended approach** is to store the tenant ID inside the JWT (JSON Web Token) when the user logs in. A JWT is cryptographically signed, which means users cannot modify its contents without invalidating the token. When a request comes in, the server reads the tenant ID directly from the verified token instead of trusting anything from the URL or request body. Then every database query is automatically filtered using that tenant ID. This ensures users can only access data belonging to their own organization, no matter what URL they try to send.

This approach is considered a best practice in real production systems because it provides strong tenant isolation and reduces the chances of human error.

## ABAC — Attribute-Based Access Control

RBAC is great for most systems, but sometimes roles aren't granular enough. What if the authorization rule is:

for example Imagine a document editing system where a user should only be allowed to edit a document if they created it, the document is still a draft, and it was created recently. A single role cannot fully represent all these conditions.

ABAC solves this problem by making authorization decisions based on attributes. These attributes can belong to the user (subject), the resource, or even the environment. For example, the system may check who owns the document, what status the document is in, what time it is, or from which IP address the request is coming. Instead of asking “Does this user have the editor role?”, ABAC asks “Do all the required conditions match?”

For most system design interviews, **RBAC is sufficient**. Mention ABAC if:

- The system has complex authorization rules that don't map cleanly to roles.
- You're designing something in healthcare, finance, or government where fine-grained access control is a regulatory requirement.
- The interviewer specifically asks about granular permissions.

## Field-Level Authorization

Sometimes authorization isn't about entire endpoints — it's about specific fields within a response. This is especially relevant for GraphQL APIs, where clients choose which fields to query.

Consider a user profile:

```json
{
  "id": 42,
  "name": "Alice Smith",
  "email": "alice@example.com",
  "salary": 95000,
  "ssn": "123-45-6789",
  "department": "Engineering"
}
```

Different users should see different fields.

In REST, you might handle this with different endpoints or response serializers per role. In GraphQL, you need field-level resolvers that check permissions:

```graphql
type User {
  name: String!
  email: String!
  salary: Int! @auth(requires: [SELF, MANAGER, HR_ADMIN])
  ssn: String @auth(requires: [SELF, HR_ADMIN])
}
```

## Policy Enforcement: Where to Put Authorization Logic

You have three main options for where authorization checks live:

| Approach     | How It Works                                                    | Pros                           | Cons                                                             |
| ------------ | --------------------------------------------------------------- | ------------------------------ | ---------------------------------------------------------------- |
| Per-endpoint | Each route handler checks permissions                           | Simple, explicit               | Easy to forget on new endpoints, scattered logic                 |
| Middleware   | Authorization middleware runs before handlers                   | Centralized, consistent        | Can be too coarse-grained for complex rules                      |
| API Gateway  | Gateway (Kong, AWS API Gateway) enforces authorization policies | Offloads from application code | Limited to simple rules (role checks), can't do ownership checks |

> In practice, most systems use a combination: the API gateway handles simple checks (is the token valid? is the user an admin?), and the application handles fine-grained checks (does the user own this resource?).

---

# Chapter 16 Rate Limiting, Throttling, and API Protection

Rate limiting and throttling are both techniques used to protect systems from too much traffic, but they work in different ways. A simple way to understand them is through a nightclub example.

Imagine a nightclub that can only safely hold 500 people. Rate limiting acts like a strict bouncer at the entrance. Once the limit is reached, no more people are allowed in. If someone tries to enter after the limit, they are rejected and told to come back later.In APIs, this usually means the server returns a 429 Too Many Requests error. The goal of rate limiting is fairness — making sure one user, bot, or application does not consume all the resources and affect everyone else.

Throttling works differently. Instead of rejecting people immediately, it slows things down. If the nightclub is crowded, people wait in a queue and are allowed in gradually as space becomes available.Requests may be delayed, queued, or processed more slowly instead of being denied outright.The purpose of throttling is system stability — preventing servers from crashing during sudden traffic spikes or heavy load.

Rate limiting is usually applied at a user level, such as per user account, IP address, or API key.  
Throttling is often applied at the system or service level to handle overload situations and prevent cascading failures across distributed systems.

## Rate Limiting Strategies

1. **per-user rate limiting**. Here, every authenticated user gets their own request quota, usually identified through a JWT or API key. For example, a normal user might be allowed 1,000 requests per hour, while premium customers get 10,000 requests per hour. This prevents a single user from consuming excessive resources while still rewarding higher-paying customers with larger limits.

2. **per-IP rate limiting**, mainly used for unauthenticated endpoints such as login, signup, or public search APIs. Since these requests do not yet have a user identity, the system tracks the client’s IP address instead. For example, an API may allow only 100 requests per hour from the same IP. This helps block brute-force attacks, spam, and scrapers. However, IP-based limits must be used carefully because many legitimate users can share the same IP address in offices, universities, or mobile networks.

3. **per-endpoint rate limiting** because not all endpoints are equally expensive or risky. Read-only endpoints like `GET /events` may allow thousands of requests per hour because they are cheap and cacheable. On the other hand, sensitive endpoints such as `POST /login` may allow only a few requests per minute to prevent password attacks.

4. **per-tenant rate limiting**. Here, the organization itself gets an overall quota, while users inside that organization have smaller individual quotas. For example, a tenant may get 50,000 requests per hour overall, while each user inside that tenant gets 5,000 requests per hour. This ensures that one large customer cannot overwhelm the entire system while still maintaining fairness among users within the same organization.

## Rate Limiting Algorithms

### 1. Token Bucket

Think of a bucket that holds tokens. Each request consumes one token. Tokens refill at a fixed rate. If the bucket is empty, requests are rejected.

    Bucket capacity: 10 tokens
    Refill rate: 1 token per second

    Second 0:  Bucket has 10 tokens
              → 10 requests arrive → all served → bucket: 0
    Second 1:  1 token refills → bucket: 1
              → 1 request arrives → served → bucket: 0
    Second 5:  5 tokens have refilled → bucket: 5
              → 8 requests arrive → 5 served, 3 rejected

**Pros**: Allows short bursts (up to bucket capacity), smooth refill. **Cons**: Slightly more complex to implement.

### 2. Sliding Window

Count requests in a rolling time window. If a user made 100 requests in the last 60 minutes, they've hit their hourly limit.

Window: 60 minutes, rolling
Limit: 100 requests

    12:00 → User makes 50 requests (50/100 used)
    12:30 → User makes 40 requests (90/100 used)
    12:45 → User makes 15 requests... 10 served, 5 rejected (100/100)
    13:01 → The 50 requests from 12:00 PM are now older than 60 minutes, so they “slide out” of the window. This means the user now has only 55 active requests counted in the current window.

**Pros**: Precise, no boundary issues. **Cons**: Requires storing timestamps for each request (memory-intensive).

### 3. Fixed Window

Simpler than sliding window. Count requests in fixed time intervals (e.g., every hour from :00 to :59).

    Window: 12:00-12:59, Limit: 100

    12:00 → requests start counting
    12:55 → User has made 95 requests
    12:59 → User makes 5 more → hits 100 → rejected for rest of window
    13:00 → Counter resets to 0 → user can make requests again

**Pros**: Simple, low memory. **Cons**: Boundary problem — a user could send 100 requests at 12:59 and 100 more at 13:00, effectively getting 200 requests in 2 minutes.

## The 429 Response

When a client exceeds the rate limit, the API returns HTTP 429 Too Many Requests with helpful headers

| Header                  | Meaning                                  | Example      |
| ----------------------- | ---------------------------------------- | ------------ |
| `Retry-After`           | Seconds until the client can retry       | `30`         |
| `X-RateLimit-Limit`     | Maximum requests allowed in the window   | `100`        |
| `X-RateLimit-Remaining` | Requests remaining in the current window | `0`          |
| `X-RateLimit-Reset`     | Unix timestamp when the window resets    | `1717024800` |

## DDoS Protection

Rate limiting is your first line of defense against Distributed Denial of Service (DDoS) attacks, but it's not enough on its own. A real DDoS attack comes from thousands of IP addresses, so per-IP rate limiting won't stop it.

### Defense in Depth

| Layer          | Tool                       | What It Does                                                                                                            |
| -------------- | -------------------------- | ----------------------------------------------------------------------------------------------------------------------- |
| Edge / CDN     | Cloudflare, AWS CloudFront | Absorbs traffic at the network edge before it reaches your servers. Geographic distribution makes it hard to overwhelm. |
| WAF            | AWS WAF, Cloudflare WAF    | Web Application Firewall — pattern detection for malicious requests (SQL injection, XSS, known attack signatures).      |
| API Gateway    | Kong, AWS API Gateway      | Rate limiting, authentication, request validation. First application-layer defense.                                     |
| Application    | Your code                  | Per-user rate limiting, business logic validation, input sanitization.                                                  |
| Infrastructure | AWS Shield, auto-scaling   | Network-level DDoS mitigation, automatic scaling to absorb traffic spikes.                                              |

The principle is defense in depth — multiple layers, each catching what the previous layer missed. No single layer is sufficient.

## Replay Attack Prevention

A replay attack is when an attacker intercepts a valid request and sends it again. Imagine someone eavesdrops on your "transfer $100 to Alice" API call and replays it 50 times. You just lost $5,000.

### How Replay Attacks Work

    1. User sends: POST /transfer { "to": "alice", "amount": 100 }
      Authorization: Bearer eyJ...

    2. Attacker intercepts this request (via network sniffing, proxy, etc.)

    3. Attacker replays the exact same request 50 times
      → Server sees a valid JWT, valid request → processes all 50

### Prevention Strategies

**1. Short-lived JWT** token means the login token expires quickly, for example in 15 minutes. So even if someone steals it, they only have a very small time window to misuse it before it becomes invalid.

**2. Nonce** is a unique random value added to every request. The server remembers which nonces it has already seen. If the same request is sent again with the same nonce, the server knows it is a duplicate and rejects it. This prevents the same request from being replayed multiple times, even if someone captures it.

**3. idempotency key**. The server processes the request the first time and stores the result along with that key. If the same request is accidentally sent again (because of retries, network issues, or double-clicking), the server does not run the payment again. Instead, it recognizes the same key and simply returns the original response.

## CORS — Cross-Origin Resource Sharing

CORS is a browser security mechanism that prevents a website at `evil.com` from making API calls to `yourbank.com` using your cookies. It's not an API design choice — it's a browser restriction you need to configure correctly.

### How CORS Works

When a browser makes a cross-origin request (e.g., JavaScript on frontend.com calls api.backend.com), the browser first sends a preflight request (an `OPTIONS` request) asking the server "is this allowed?"

If the server's response allows the origin, method, and headers, the browser proceeds with the actual request. If not, the browser blocks it — the JavaScript code gets a CORS error.

### Key CORS Headers

| Header                           | What It Controls                         | Example                                                                        |
| -------------------------------- | ---------------------------------------- | ------------------------------------------------------------------------------ |
| Access-Control-Allow-Origin      | Which origins can make requests          | https://frontend.com (specific) or \* (any — dangerous for authenticated APIs) |
| Access-Control-Allow-Methods     | Which HTTP methods are allowed           | GET, POST, PUT, DELETE                                                         |
| Access-Control-Allow-Headers     | Which custom headers are allowed         | Authorization, Content-Type                                                    |
| Access-Control-Allow-Credentials | Whether cookies/auth headers are sent    | true (required for authenticated requests)                                     |
| Access-Control-Max-Age           | How long to cache the preflight response | 86400 (24 hours)                                                               |

### Common Mistakes

- **Using** `Access-Control-Allow-Origin: *` **with credentials**. The browser rejects this combination. If you send cookies or Authorization headers, you must specify exact origins.
- **Forgetting the preflight**. `OPTIONS` requests must return CORS headers. If your server doesn't handle OPTIONS, the browser blocks the actual request.
- **Overly permissive** origins. Allowing `*` on an authenticated API means any website can make requests with your users' credentials.

### Cross-Site Scripting (XSS)

It is when an attacker injects malicious JavaScript through user input that gets rendered in other users' browsers.

**Prevention:**

- **Sanitize on input**: Strip or encode HTML tags from user input.
- **Escape on output**: When rendering user-generated content, HTML-encode special characters (`<` → `&lt;`, `>` → `&gt;`).
- **Content-Security-Policy header**: Tell the browser to only execute scripts from trusted sources.
- **HttpOnly cookies**: Prevent JavaScript from accessing session cookies.

---
