# IRC Support Desk Server

Real-time backend for a support-desk / team-chat client. Clients connect over WebSockets; persistence and live updates are powered by [RethinkDB](https://rethinkdb.com/) change feeds.

## Architecture

```
Client (WebSocket)  <-->  Go server (:4000)  <-->  RethinkDB (rtsupport)
```

- **Transport:** JSON messages over a single WebSocket connection per client.
- **Routing:** Incoming messages are dispatched by a `name` field to registered handlers (`router.go`).
- **Persistence:** Channels, users, and messages are stored in RethinkDB tables.
- **Realtime:** Subscriptions use RethinkDB `Changes()` feeds; the server pushes `add`, `edit`, and `remove` events back to the client.

On connect, each client is assigned an anonymous user row in the `user` table. The client can later call `user edit` to set a display name; that name is used as `author` when posting messages.

## Prerequisites

- [Go](https://go.dev/) 1.18+
- [RethinkDB](https://rethinkdb.com/docs/install/) running locally on the default port (`28015`)

## Database setup

Create the database and tables RethinkDB expects (names match `main.go` and `handlers.go`):

```js
r.dbCreate('rtsupport').run()
r.db('rtsupport').tableCreate('user').run()
r.db('rtsupport').tableCreate('channel').run()
r.db('rtsupport').tableCreate('message').run()

// Required for message subscribe ordering (newest first)
r.db('rtsupport').table('message').indexCreate('createdAt').run()
```

| Table     | Fields (Go / Rethink)                                      |
|-----------|------------------------------------------------------------|
| `user`    | `name`                                                     |
| `channel` | `name`                                                     |
| `message` | `channelId`, `body`, `author`, `createdAt`                 |

Connection settings are hard-coded in `main.go`:

- Address: `localhost:28015`
- Database: `rtsupport`

## Running the server

From the project root:

```bash
go mod init irc-support-desk-server   # once, if go.mod is missing
go get github.com/gorilla/websocket \
       github.com/mitchellh/mapstructure \
       gopkg.in/rethinkdb/rethinkdb-go.v6
go run .
```

The server listens on **http://localhost:4000**. WebSocket upgrades are served at `/` (any path hits the same handler).

## WebSocket protocol

### Client → server

Send JSON objects with this shape:

```json
{
  "name": "<handler>",
  "data": { }
}
```

| `name`                 | `data` | Description |
|------------------------|--------|-------------|
| `channel add`          | `{ "name": "general" }` | Insert a channel |
| `channel subscribe`    | — | Stream all channel changes (initial snapshot + updates) |
| `channel unsubscribe`  | — | Stop channel subscription |
| `user edit`            | `{ "name": "alice" }` | Update this connection’s user |
| `user subscribe`       | — | Stream all user changes |
| `user unsubscribe`     | — | Stop user subscription |
| `message add`          | `{ "channelId": "…", "body": "hello" }` | Post a message (`author` and `createdAt` set server-side) |
| `message subscribe`    | `{ "channelId": "…" }` | Stream messages for one channel |
| `message unsubscribe`  | — | Stop message subscription |

Unknown handler names are ignored. Decode or database errors are returned as:

```json
{ "name": "error", "data": "<message>" }
```

### Server → client

Subscription handlers emit change-feed events:

| Event pattern   | When |
|-----------------|------|
| `<entity> add`    | Row inserted |
| `<entity> edit`   | Row updated |
| `<entity> remove` | Row deleted |

`<entity>` is `user`, `channel`, or `message`. Payload `data` is the document (or former document on remove).

Example:

```json
{ "name": "message add", "data": { "id": "…", "channelId": "…", "body": "hello", "author": "alice", "createdAt": "…" } }
```

## Project layout

| File          | Role |
|---------------|------|
| `main.go`     | RethinkDB connection, handler registration, HTTP server |
| `router.go`   | WebSocket upgrade and handler lookup |
| `client.go`   | Per-connection read/write loops, user bootstrap, subscription stop channels |
| `handlers.go` | CRUD + change-feed logic for users, channels, and messages |

## Development notes

- **CORS / origin:** `CheckOrigin` allows all origins (`router.go`), which is convenient for local dev; tighten this before production.
- **Concurrency:** Writes and long-running change feeds run in goroutines; each client tracks stop channels so `unsubscribe` cancels the right feed.
- **Anonymous users:** Every new WebSocket connection inserts a `user` row with `name: "anonymous"` until `user edit` runs.

## License

No license file is included in this repository; add one if you plan to distribute or open-source the project.
