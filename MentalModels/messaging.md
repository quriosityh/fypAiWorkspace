---
share_link: https://share.note.sx/xoayjpy7#QuxFfCQx4ZylAmhEylkiLGe5qZNVN6u6IgIJ+DkbCEs
share_updated: 2026-01-16T16:38:00+05:00
---
## Messaging Module Architecture

### Layer Responsibilities

```mermaid
graph TB
    subgraph "Client Layer"
        FE[Frontend/Browser]
    end
    
    subgraph "API Layer"
        Routes[Routes<br/>routes.ts]
        Controller[Controller<br/>controller.ts]
        Middleware[Middleware<br/>auth.ts, errorHandler.ts]
    end
    
    subgraph "Business Logic Layer"
        Service[Service<br/>service.ts]
        Validation[Validation<br/>validations.ts]
    end
    
    subgraph "Data Layer"
        Repo[Repository<br/>repository.ts]
        DB[(PostgreSQL)]
    end
    
    subgraph "Event Layer"
        Emitter[Event Emitter<br/>messageEmitter.ts]
    end
    
    FE -->|HTTP Request| Routes
    Routes -->|Handler Array| Controller
    Controller -->|req.auth.userId| Service
    Service -->|Validate Input| Validation
    Service -->|Data Operations| Repo
    Repo -->|SQL Queries| DB
    Service -->|Emit Event| Emitter
    Emitter -->|SSE Stream| FE
    
    style Routes fill:#e1f5ff
    style Controller fill:#fff4e1
    style Service fill:#e8f5e9
    style Repo fill:#f3e5f5
    style Emitter fill:#ffe0b2
```

**Layer Responsibilities:**

| Layer | File | Responsibility | Returns |
|-------|------|---------------|---------|
| **Routes** | routes.ts | Define endpoints, attach middleware, wire handlers | N/A |
| **Controller** | controller.ts | Extract request data, call service, format response | HTTP Response |
| **Service** | service.ts | Validate input, business logic, orchestrate operations | Domain data |
| **Repository** | `repository.ts` | Database queries, data persistence | Raw DB results |
| **Validation** | validations.ts | Zod schemas for input validation | Parsed & typed data |
| **Event Emitter** | `messageEmitter.ts` | Broadcast events to SSE listeners | N/A |

---

## Flow 1: Send Message (POST /messages)

```mermaid


sequenceDiagram
    participant Client
    participant Routes
    participant Auth as requireAuth
    participant Controller
    participant Service
    participant Validation as sendMessageSchema
    participant ListingsRepo
    participant MessagesRepo
    participant DB
    participant Emitter
    participant SSE as SSE Listeners

    Client->>Routes: POST /messages<br/>{body, listing_id}
    Routes->>Auth: Check JWT token
    Auth->>Auth: Extract userId from token
    Auth-->>Routes: req.auth.userId = "uuid"
    
    Routes->>Controller: sendMessageHandler
    Controller->>Service: sendMessage(req.body, userId)
    
    Service->>Validation: Parse & validate input
    Validation-->>Service: {body, listing_id}
    
    alt New conversation (listing_id provided)
        Service->>ListingsRepo: findById(listing_id)
        ListingsRepo->>DB: SELECT * FROM listings
        DB-->>ListingsRepo: Listing data
        ListingsRepo-->>Service: {id, owner: {id}}
        
        Service->>Service: Check sender != owner
        Service->>MessagesRepo: ensureConversation(listingId, renterId, ownerId)
        MessagesRepo->>DB: INSERT INTO conversations<br/>ON CONFLICT DO NOTHING
        DB-->>MessagesRepo: Conversation created/found
        MessagesRepo-->>Service: {id, listing_id, renter_id, owner_id}
    else Existing conversation (conversation_id provided)
        Service->>MessagesRepo: findConversationById(conversation_id)
        MessagesRepo->>DB: SELECT * FROM conversations
        DB-->>MessagesRepo: Conversation data
        MessagesRepo-->>Service: {id, renter_id, owner_id}
        Service->>Service: Check sender is renter or owner
    end
    
    Service->>MessagesRepo: addMessage(conversationId, senderId, body)
    MessagesRepo->>DB: INSERT INTO messages<br/>UPDATE conversations.updated_at
    DB-->>MessagesRepo: Message row
    MessagesRepo-->>Service: {id, body, sender_id, created_at}
    
    Service->>Emitter: emit('conversation:${id}', message)
    Emitter->>SSE: Push to all connected SSE clients
    
    Service-->>Controller: {conversation, message}
    Controller-->>Client: 201 {data: message}
```

**Function Signatures:**

```typescript
// Controller
sendMessageHandler: RequestHandler[]
  → asyncHandler(async (req: AuthenticatedRequest, res: Response))
  → calls sendMessage(req.body, req.auth!.userId)

// Service
sendMessage(payload: unknown, senderId: string)
  → validates with sendMessageSchema
  → returns {conversation, message}

// Repository
ensureConversation(listingId: string, renterId: string, ownerId: string)
  → INSERT INTO conversations with ON CONFLICT DO NOTHING
  → returns conversation object

addMessage(conversationId: string, senderId: string, body: string)
  → INSERT INTO messages
  → UPDATE conversations.updated_at
  → returns message object
```

---

## Flow 2: List Conversations (GET /conversations)

```mermaid
sequenceDiagram
    participant Client
    participant Routes
    participant Auth as requireAuth
    participant Controller
    participant Service
    participant MessagesRepo
    participant DB

    Client->>Routes: GET /conversations
    Routes->>Auth: Verify JWT
    Auth-->>Routes: userId
    
    Routes->>Controller: listConversationsHandler
    Controller->>Service: listConversations(userId)
    
    Service->>MessagesRepo: listConversationsForUser(userId)
    MessagesRepo->>DB: SELECT * FROM conversations<br/>WHERE renter_id = ? OR owner_id = ?
    DB-->>MessagesRepo: Array of conversations
    
    loop For each conversation
        MessagesRepo->>DB: SELECT * FROM messages<br/>WHERE conversation_id = ?<br/>ORDER BY created_at DESC LIMIT 1
        DB-->>MessagesRepo: Last message
        
        MessagesRepo->>DB: SELECT count(*) FROM messages<br/>WHERE conversation_id = ?<br/>AND sender_id != userId<br/>AND read_at IS NULL
        DB-->>MessagesRepo: Unread count
    end
    
    MessagesRepo-->>Service: [{...conv, last_message, unread_count}]
    Service-->>Controller: Conversation list
    Controller-->>Client: 200 {data: conversations}
```

**Function Signatures:**

```typescript
// Controller
listConversationsHandler: RequestHandler[]
  → asyncHandler(async (req: AuthenticatedRequest, res: Response))
  → calls listConversations(req.auth!.userId)

// Service
listConversations(userId: string)
  → returns conversation array with metadata

// Repository
listConversationsForUser(userId: string)
  → SELECT conversations where user is renter or owner
  → For each: fetch last_message and unread_count
  → returns enriched conversation objects
```

---

## Flow 3: List Messages (GET /conversations/:id/messages)

```mermaid
sequenceDiagram
    participant Client
    participant Routes
    participant Auth as requireAuth
    participant Controller
    participant Service
    participant Validation as listMessagesSchema
    participant MessagesRepo
    participant DB

    Client->>Routes: GET /conversations/:id/messages?limit=20&offset=0
    Routes->>Auth: Verify JWT
    Auth-->>Routes: userId
    
    Routes->>Controller: listMessagesHandler
    Controller->>Service: listMessages(conversationId, query, userId)
    
    Service->>MessagesRepo: findConversationById(conversationId)
    MessagesRepo->>DB: SELECT * FROM conversations WHERE id = ?
    DB-->>MessagesRepo: Conversation
    MessagesRepo-->>Service: {id, renter_id, owner_id}
    
    Service->>Service: Check userId is renter or owner
    
    Service->>Validation: Parse query params
    Validation-->>Service: {limit: 20, offset: 0}
    
    Service->>MessagesRepo: getMessages(conversationId, limit, offset)
    MessagesRepo->>DB: SELECT * FROM messages<br/>WHERE conversation_id = ?<br/>ORDER BY created_at DESC<br/>LIMIT ? OFFSET ?
    DB-->>MessagesRepo: Messages array
    MessagesRepo-->>Service: Messages
    
    Service->>MessagesRepo: markConversationRead(conversationId, userId)
    MessagesRepo->>DB: UPDATE messages SET read_at = NOW()<br/>WHERE conversation_id = ?<br/>AND sender_id != userId<br/>AND read_at IS NULL
    DB-->>MessagesRepo: Updated count
    
    Service-->>Controller: Messages array
    Controller-->>Client: 200 {data: messages}
```

**Function Signatures:**

```typescript
// Controller
listMessagesHandler: RequestHandler[]
  → extracts conversationId from req.params.id
  → calls listMessages(id, req.query, req.auth!.userId)

// Service
listMessages(conversationId: string, query: unknown, userId: string)
  → validates access to conversation
  → parses pagination params with listMessagesSchema
  → marks messages as read
  → returns message array

// Repository
getMessages(conversationId: string, limit: number, offset: number)
  → SELECT with pagination, newest first
  → returns message objects

markConversationRead(conversationId: string, userId: string)
  → UPDATE messages SET read_at for unread messages not sent by userId
```

---

## Flow 4: Delete Message (DELETE /messages/:id)

```mermaid
sequenceDiagram
    participant Client
    participant Routes
    participant Auth as requireAuth
    participant Controller
    participant Service
    participant MessagesRepo
    participant DB
    participant Emitter
    participant SSE as SSE Listeners

    Client->>Routes: DELETE /messages/:id
    Routes->>Auth: Verify JWT
    Auth-->>Routes: userId
    
    Routes->>Controller: deleteMessageHandler
    Controller->>Service: deleteMessage(messageId, userId)
    
    Service->>MessagesRepo: softDeleteMessage(messageId, userId)
    MessagesRepo->>DB: UPDATE messages<br/>SET deleted_at = NOW()<br/>WHERE id = ? AND sender_id = ?
    DB-->>MessagesRepo: Updated message or null
    MessagesRepo-->>Service: message | null
    
    alt Message found and owned
        Service->>Emitter: emit('conversation:${conversationId}',<br/>{...msg, deleted: true})
        Emitter->>SSE: Notify SSE listeners
        Service-->>Controller: message
        Controller-->>Client: 200 {data: message}
    else Not found or not owned
        Service-->>Controller: throw AppError 404
        Controller-->>Client: 404 MESSAGE_NOT_FOUND
    end
```

**Function Signatures:**

```typescript
// Controller
deleteMessageHandler: RequestHandler[]
  → extracts messageId from req.params.id
  → calls deleteMessage(id, req.auth!.userId)

// Service
deleteMessage(messageId: string, userId: string)
  → soft deletes message if owned by userId
  → emits deletion event
  → returns deleted message or throws

// Repository
softDeleteMessage(messageId: string, userId: string)
  → UPDATE messages SET deleted_at
  → WHERE id = ? AND sender_id = ? (ensures ownership)
  → returns updated row or null
```

---

## Flow 5: SSE Stream (GET /conversations/:id/stream) [[emitter and emit]]

```mermaid
sequenceDiagram
    participant Client
    participant Routes
    participant Auth as requireAuth
    participant Controller
    participant MessagesRepo
    participant DB
    participant Emitter
    participant Connection as SSE Connection

    Client->>Routes: GET /conversations/:id/stream
    Routes->>Auth: Verify JWT
    Auth-->>Routes: userId
    
    Routes->>Controller: streamConversationHandler
    Controller->>MessagesRepo: findConversationById(conversationId)
    MessagesRepo->>DB: SELECT * FROM conversations
    DB-->>MessagesRepo: Conversation
    MessagesRepo-->>Controller: {id, renter_id, owner_id}
    
    Controller->>Controller: Check userId is renter or owner
    
    Controller->>Connection: Set SSE headers
    Note over Controller,Connection: Content-Type: text/event-stream<br/>Cache-Control: no-cache<br/>Connection: keep-alive
    
    Controller->>Connection: Start heartbeat (every 25s)
    Note over Connection: Sends ':heartbeat\n\n'
    
    Controller->>Emitter: Register listener for<br/>'conversation:${id}'
    Note over Controller,Emitter: listener = (payload) => {<br/>  res.write(`data: ${JSON.stringify(payload)}\n\n`)<br/>}
    
    Emitter->>Connection: Connection kept alive
    Connection-->>Client: SSE stream established
    
    Note over Client,Emitter: Connection stays open...<br/>waiting for events
    
    alt New message arrives
        Emitter->>Controller: emit('conversation:${id}', message)
        Controller->>Connection: res.write('data: {...}\n\n')
        Connection-->>Client: Message pushed to browser
    end
    
    alt Client disconnects
        Client->>Connection: Close connection
        Connection->>Controller: req.on('close')
        Controller->>Controller: clearInterval(heartbeat)
        Controller->>Emitter: removeListener('conversation:${id}')
    end
```

**SSE Implementation Details:**

```typescript
// Controller
streamConversationHandler: RequestHandler[]
  → validates conversation access
  → sets SSE headers
  → establishes long-lived HTTP connection
  → registers event listener on messageEmitter
  → sends heartbeat every 25s to keep alive
  → cleans up on client disconnect

// Event Flow
1. res.setHeader('Content-Type', 'text/event-stream')
2. res.setHeader('Cache-Control', 'no-cache')
3. res.setHeader('Connection', 'keep-alive')
4. res.flushHeaders() // Send headers immediately

5. setInterval(() => res.write(':heartbeat\n\n'), 25000)
   // Prevents timeout, ignored by EventSource

6. messageEmitter.on(`conversation:${id}`, (payload) => {
     res.write(`data: ${JSON.stringify(payload)}\n\n`)
   })
   // Listens to events from sendMessage service

7. req.on('close', () => {
     clearInterval(heartbeat)
     messageEmitter.off(`conversation:${id}`, listener)
   })
   // Cleanup on disconnect
```

---

## Complete Data Flow Diagram

```mermaid
graph TB
    subgraph "Frontend"
        UI[UI Component]
        EventSource[EventSource API]
    end
    
    subgraph "API Routes"
        R1[POST /messages]
        R2[GET /conversations]
        R3[GET /conversations/:id/messages]
        R4[GET /conversations/:id/stream]
        R5[DELETE /messages/:id]
    end
    
    subgraph "Controllers"
        C1[sendMessageHandler]
        C2[listConversationsHandler]
        C3[listMessagesHandler]
        C4[streamConversationHandler]
        C5[deleteMessageHandler]
    end
    
    subgraph "Services"
        S1[sendMessage]
        S2[listConversations]
        S3[listMessages]
        S4[deleteMessage]
    end
    
    subgraph "Repository"
        RP[messagesRepository]
    end
    
    subgraph "Database"
        DBT1[(conversations)]
        DBT2[(messages)]
    end
    
    subgraph "Events"
        EM[messageEmitter]
    end
    
    UI -->|POST body| R1
    UI -->|GET| R2
    UI -->|GET + params| R3
    EventSource -->|Open connection| R4
    UI -->|DELETE| R5
    
    R1 --> C1 --> S1 --> RP
    R2 --> C2 --> S2 --> RP
    R3 --> C3 --> S3 --> RP
    R4 --> C4 --> RP
    R5 --> C5 --> S4 --> RP
    
    RP --> DBT1
    RP --> DBT2
    
    S1 -->|emit event| EM
    S4 -->|emit event| EM
    EM -->|push data| C4
    C4 -->|SSE stream| EventSource
    EventSource -->|onmessage| UI
    
    style R4 fill:#ffcccc
    style C4 fill:#ffcccc
    style EM fill:#ffcccc
    style EventSource fill:#ffcccc
```

---

## SSE Connection Lifecycle

```mermaid
stateDiagram-v2
    [*] --> Connecting: Client opens EventSource
    
    Connecting --> Connected: Headers sent, listener registered
    
    Connected --> Heartbeat: Every 25s
    Heartbeat --> Connected: Send heartbeat
    
    Connected --> MessageReceived: Event emitted
    MessageReceived --> Connected: Push message data
    
    Connected --> Disconnected: Client closes or network error
    
    Disconnected --> CleanedUp: Cleanup resources
    
    CleanedUp --> [*]
    
    note right of Connected
        Connection stays open
        No polling needed
        Server pushes data
    end note
    
    note right of MessageReceived
        messageEmitter.emit()
        triggered by sendMessage
        or deleteMessage
    end note
    
    note right of Heartbeat
        Writes colon-heartbeat
        to keep connection alive
    end note
```

---

## Key Concepts

**Why SSE over Polling?**
- **Efficiency**: Single persistent connection vs repeated requests
- **Latency**: <500ms vs 5-10s (polling interval)
- **Battery**: Less network activity on mobile
- **Simplicity**: Built-in browser API, no library needed

**In-Memory Emitter Limitation:**
- Works only with **single server instance**
- For multi-instance: replace with Redis Pub/Sub
- EventEmitter broadcasts to all listeners on same process

**Auto-reconnection:**
- Browser EventSource automatically reconnects on disconnect
- Heartbeat prevents proxy/firewall timeouts
- Client should re-fetch missed messages on reconnect

**Message Flow Summary:**
1. **Send**: POST → Service validates → Repo saves → Emitter broadcasts → SSE pushes
2. **List Conversations**: GET → Repo joins last_message + unread_count → Return enriched data
3. **List Messages**: GET → Repo paginated query → Mark as read → Return messages
4. **Stream**: GET → Long-lived connection → Listen to emitter → Push events as they happen
5. **Delete**: DELETE → Soft delete → Emitter broadcasts deletion → SSE notifies clients