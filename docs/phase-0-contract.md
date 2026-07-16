# Cove — Phase 0 Contract (draft v0.1)

> This is a **contract**, not an implementation. The reference server and client
> are merely its first implementations. Keeping this document stable and
> versioned is what sets Cove apart from Discord / Stoat / Spacebar, and what
> makes an ocean of clients possible.

## 0. Cross-cutting decisions

| Topic | Decision | Why |
|-------|----------|-----|
| Identifiers | **UUIDv7** (as text in the API) | Time-sortable, standard, no worker-id coordination like Snowflake. Gives pagination by id for free. |
| Time | UTC, RFC 3339 string (`2026-07-14T19:29:00Z`) | Unambiguous and human-readable. |
| Empty lists | Always `[]`, **never `null`** | A client must not have to defend against two kinds of emptiness. (A nil slice in Go serialises to `null` — initialise with `make(...)`.) |
| Event transport | WebSocket gateway, envelope `{op, t, s, d}` | Like Discord's, but documented and versioned. |
| REST version | path prefix `/api/v1/...` | Explicit versioning of the contract. |
| Gateway version | query `?v=1` on connect | Client and server may run different versions — self-hosting demands it. |
| Permissions | 64-bit mask, roles + per-channel overrides | Discord's model, proven over years. |
| Message content | raw markdown string, rendered by the client | Simpler to store and search; Telegram's entity model is overkill to start with. |

---

## 1. Data model (Phase 0)

Fields marked `*` are required in Phase 0. The rest are groundwork for the
future, but the tables exist from the start to avoid painful migrations.

### User
```
id*            uuidv7 (pk)
username*      text, unique, [a-z0-9_], 2..32
display_name*  text, 1..64
email*         text, unique (for login/recovery)
password_hash* text (argon2id)
avatar         text (media id, nullable)
instance_owner* bool, default false   # may issue RegistrationTokens (see §6)
created_at*    timestamptz
```

### Session
```
id*         uuidv7 (pk)
user_id*    uuidv7 (fk User)
token_hash* bytea, unique        # sha256(bearer token); the token is never stored
created_at* timestamptz
expires_at* timestamptz
user_agent  text
```

### Server  (called "server" in the API; "guild" is acceptable internally)
```
id*         uuidv7 (pk)
name*       text, 1..100
owner_id*   uuidv7 (fk User)
icon        text (media id, nullable)
created_at* timestamptz
```

### Member  (a user's membership in a server)
```
server_id*  uuidv7 (fk Server)   # (server_id, user_id) = pk
user_id*    uuidv7 (fk User)
nickname    text, nullable
joined_at*  timestamptz
# a member's roles live in the MemberRole table (many-to-many)
```
> Over the API a Member carries `roles`: the ids it holds, **excluding** the
> implicit `@everyone`. Without it a client cannot colour a name or show who
> moderates — the roles would exist but be invisible.

### Role
```
id*          uuidv7 (pk)
server_id*   uuidv7 (fk Server)
name*        text, 1..100
color        int (0xRRGGBB, nullable)
permissions* bigint (bit mask, see §2)
position*    int (hierarchy; higher = above)
hoist*       bool (show as a separate group in the member list)
mentionable* bool
```
> Every server has an undeletable `@everyone` role with `position = 0`.

### Channel
```
id*         uuidv7 (pk)
server_id   uuidv7 (fk Server, nullable)   # null => DM/group chat (Phase 1)
type*       enum: category | text | voice | dm
name*       text, 1..100
topic       text, nullable
position*   int
parent_id   uuidv7 (fk Channel, nullable)   # for nesting under a category
created_at* timestamptz
```

### ChannelRecipient  (participants of a direct chat)
```
channel_id*  uuidv7 (fk Channel)   # (channel_id, user_id) = pk
user_id*     uuidv7 (fk User)
```
> A direct chat is a channel with `server_id = null` and `type = 'dm'`. Its
> participants had nowhere to live (`Member` is keyed by server), hence the
> separate table. Permissions in a DM come not from roles but from participation
> itself: `VIEW_CHANNEL | SEND_MESSAGES | ATTACH_FILES`.

### ChannelOverride  (per-channel permission overrides)
```
channel_id*   uuidv7 (fk Channel)
target_type*  enum: role | member
target_id*    uuidv7                # role_id or user_id
allow*        bigint (mask)
deny*         bigint (mask)
```
> Phase 0 need not ship UI for overrides, but the permission resolver already
> honours them.

### Message
```
id*         uuidv7 (pk)            # sorting = message order
channel_id* uuidv7 (fk Channel)
author_id*  uuidv7 (fk User)
content*    text (markdown), 0..4000
type*       enum: default | system
reply_to    uuidv7 (fk Message, nullable)
created_at* timestamptz
edited_at   timestamptz, nullable
```

### Attachment
```
id*          uuidv7 (pk)
message_id*  uuidv7 (fk Message)      # an attachment always belongs to a message
media_id*    text (key in the storage layer)
filename*    text
content_type text
size         bigint
```
> Storage sits behind an interface (`internal/storage`): the default is `local`
> (disk, `COVE_STORAGE_PATH`), and an S3-compatible backend is added as another
> implementation of the same interface. In JSON an attachment carries a relative
> `url` (`/api/v1/attachments/<id>`) — the client prefixes its own instance's
> address.

### Invite  (joining a specific SERVER)
```
code*        text (pk, short, e.g. 8 base62 chars)
server_id*   uuidv7 (fk Server)
created_by*  uuidv7 (fk User)
max_uses     int, nullable (null = unlimited)
uses*        int, default 0
expires_at   timestamptz, nullable
created_at*  timestamptz
```

### RegistrationToken  (access to the INSTANCE — the Matrix model)
```
code*        text (pk, short)
created_by*  uuidv7 (fk User)          # the instance admin
max_uses     int, nullable
uses*        int, default 0
expires_at   timestamptz, nullable
created_at*  timestamptz
```
> Deliberately a separate entity from `Invite`. A `RegistrationToken` unlocks
> creating an account **on the instance**; an `Invite` adds an already existing
> user **to a server**. Registration is "open" but requires a valid token issued
> by the instance owner.

---

## 2. Permission model

A 64-bit mask. The starting set (the number is the bit shift):

| Bit | Permission | Meaning |
|-----|------------|---------|
| 0 | `VIEW_CHANNEL` | see the channel and read its history |
| 1 | `SEND_MESSAGES` | post messages |
| 2 | `MANAGE_MESSAGES` | delete/pin other people's messages |
| 3 | `MANAGE_CHANNELS` | create/edit/delete channels |
| 4 | `MANAGE_ROLES` | manage roles (none above your own) |
| 5 | `KICK_MEMBERS` | kick |
| 6 | `BAN_MEMBERS` | ban |
| 7 | `MANAGE_SERVER` | server name/icon/settings |
| 8 | `CREATE_INVITE` | create invites |
| 9 | `MENTION_EVERYONE` | ping @everyone |
| 10 | `ATTACH_FILES` | attach files |
| 11 | `CONNECT` | join a voice channel |
| 12 | `SPEAK` | talk and share the screen |
| 63 | `ADMINISTRATOR` | bypasses every check |

**Order of resolving a member's permissions in a channel:**
1. Server owner OR the `ADMINISTRATOR` bit → everything allowed, stop.
2. `base = permissions` of the `@everyone` role.
3. `base |= permissions` of each of the member's roles (OR).
4. Apply the channel's `@everyone` override: `base = (base & ~deny) | allow`.
5. Merge the overrides of the member's roles: `base = (base & ~Σdeny) | Σallow`.
6. Apply the channel's member override (highest priority).

---

## 2a. Role hierarchy — what makes MANAGE_ROLES safe

Without the rules below, `MANAGE_ROLES` is not a moderator power but a synonym
for "owns the server": its holder mints a role with `ADMINISTRATOR`, wears it,
and the rest of the permission model is decoration. Both rules are enforced
server-side; a client must not be the thing that stops this.

**Rank** is the position of a member's highest role. The owner outranks every
role that can exist. `@everyone` sits at 0, so a member with no other role
ranks 0 and can act on nothing.

1. **Rank rule.** You may only create, edit, delete or hand out a role
   **strictly below** your rank. Equal loses — holding a role must not let you
   edit it, or a moderator would promote their own role and climb.
2. **Grant rule.** You may only put permissions on a role that **you already
   hold**. Otherwise a moderator who cannot ban mints a role that can.
   `ADMINISTRATOR` bypasses this, since its holder already has everything.

Editing checks the role **both as it stands and as it would become**: otherwise
you could move a harmless role up and only then add permissions to it, or edit
a role you outrank into one you do not.

Acting on a *member* (taking a role away) additionally requires outranking that
member — peers must not strip each other, and nobody touches the owner.

> Historical note: this was specified from the start ("manage roles, none above
> your own") but not implemented. A member with `MANAGE_ROLES` could create an
> `ADMINISTRATOR` role at position 9999 and got a 200. It stayed unexploitable
> only because no endpoint could assign a role yet — the fix landed together
> with those endpoints.

## 2b. Permissions travel as strings

`permissions` is a **decimal string** in JSON (`"48"`, `"-9223372036854775808"`),
never a number. This is not fussiness — a JSON number cannot carry the mask:

- **Precision.** The mask is int64 and `ADMINISTRATOR` is bit 63. A JavaScript
  client parsing `ADMINISTRATOR | KICK_MEMBERS` (`-9223372036854775776`) gets
  `-9223372036854776000` back: the KICK bit is gone, silently. Round-tripping
  such a role through a client would corrupt it.
- **Bitwise.** JavaScript's `&` and `|` truncate to 32 bits, so `1 << 63`
  overflows to `-2147483648` and testing the ADMINISTRATOR bit with plain
  numbers is not merely lossy, it is impossible.

Clients must parse it into a 64-bit type (`BigInt` in JavaScript) and send it
back as a string. Discord does the same, for the same reason.

---

## 3. Gateway protocol (WebSocket, v1)

Connect: `GET /gateway?v=1&encoding=json` → upgrade to WebSocket.

**Envelope of every frame:**
```jsonc
{
  "op": 0,          // opcode (see below)
  "t":  "MESSAGE_CREATE", // event type, only for op=0 (DISPATCH)
  "s":  42,         // sequence, only for DISPATCH; the client echoes it in heartbeats
  "d":  { }         // payload
}
```

**Opcodes:**
| op | Name | Direction | Purpose |
|----|------|-----------|---------|
| 0 | DISPATCH | S→C | event delivery (`t` + `d`) |
| 1 | HELLO | S→C | first frame: `{ heartbeat_interval_ms }` |
| 2 | IDENTIFY | C→S | `{ token }` |
| 3 | READY | S→C | initial state (see below) |
| 4 | HEARTBEAT | C→S | `{ s }` (last received sequence) |
| 5 | HEARTBEAT_ACK | S→C | acknowledgement |
| 6 | RESUME | C→S | `{ token, session_id, s }` — reconnect without loss |
| 9 | INVALID_SESSION | S→C | you must identify again from scratch |

**Handshake:** connect → `HELLO` → the client sends `IDENTIFY` → the server sends
`READY` → from then on the client sends `HEARTBEAT` every
`heartbeat_interval_ms`.

**`READY.d`** (the initial snapshot):
```jsonc
{
  "session_id": "…",
  "user": { /* self */ },
  "servers": [ { "id", "name", "icon",
                 "channels": [...], "roles": [...],
                 "members_count": 7 } ],
  "dms": [ { "id", "type": "dm", "server_id": null,
             "recipients": [ /* User */ ] } ]
}
```

**DISPATCH events (Phase 0):**
```
MESSAGE_CREATE   MESSAGE_UPDATE   MESSAGE_DELETE
CHANNEL_CREATE   CHANNEL_UPDATE   CHANNEL_DELETE
ROLE_CREATE      ROLE_UPDATE      ROLE_DELETE
MEMBER_JOIN      MEMBER_UPDATE    MEMBER_LEAVE
TYPING_START     PRESENCE_UPDATE  DM_CREATE
```
> Channel events are delivered only to those holding `VIEW_CHANNEL` (for a
> direct chat, only to its participants). Subscriptions: server channels via the
> server, direct chats per channel.

The `d` of each event is the corresponding object from §1 (or a delta of it).
`TYPING_START.d` = `{ channel_id, user_id, user }` — the user is inlined in full
so the client never has to resolve the name separately.

---

## 4. REST surface (Phase 0)

Everything lives under `/api/v1`. Authentication: the `Authorization: <token>`
header.

```
# auth
POST   /auth/register            { username, display_name, email, password, registration_token }
POST   /auth/login               { login, password } -> { token, user }
       # Both are rate limited per client address: COVE_AUTH_BURST requests
       # back to back (default 10), then one more every COVE_AUTH_REFILL_SECONDS
       # (default 15). Over budget => 429 with a Retry-After header in seconds.
       # A client should honour Retry-After rather than spin.
POST   /auth/logout

# self
GET    /users/@me
PATCH  /users/@me
GET    /users/@me/channels          # direct chats
POST   /users/@me/channels          { username } | { recipient_id }
                                    # creates a chat or returns the existing one (200)

# servers
POST   /servers                  { name } -> Server
GET    /servers/:id
PATCH  /servers/:id
DELETE /servers/:id
GET    /servers/:id/members
GET    /servers/:id/channels
POST   /servers/:id/channels     { type, name, parent_id? }
GET    /servers/:id/roles
POST   /servers/:id/roles          { name, permissions, position, color?, hoist?, mentionable? }
PATCH  /servers/:id/roles/:rid
DELETE /servers/:id/roles/:rid
PUT    /servers/:id/members/:uid/roles/:rid    # give a member a role
DELETE /servers/:id/members/:uid/roles/:rid    # take it away
       # All of the above need MANAGE_ROLES *and* obey the hierarchy in §2a.
       # @everyone (role id == server id) cannot be deleted, cannot leave
       # position 0, and cannot be handed out — it is implicit.

# channels
GET    /channels/:id
PATCH  /channels/:id
DELETE /channels/:id
GET    /channels/:id/messages    ?before=<msgid>&limit=50   (cursor pagination)
POST   /channels/:id/messages    { content, reply_to? }
       # or multipart/form-data: a payload_json field (the same JSON) + files fields.
       # Files are sent together with the message in one request, so orphaned
       # attachments cannot exist by design. Requires ATTACH_FILES.

GET    /attachments/:id          # serves the file; checks VIEW_CHANNEL on the message's channel
PATCH  /channels/:id/messages/:mid
DELETE /channels/:id/messages/:mid
POST   /channels/:id/typing
POST   /channels/:id/voice/token    # -> { url, token, can_speak }
       # A LiveKit token for a voice channel. Room = channel id,
       # identity = user id. Requires CONNECT; without SPEAK it returns
       # can_speak:false (joins as a listener, canPublish=false in the token).
       # 503 if voice is not configured on the instance.

# invites
POST   /servers/:id/invites      { max_uses?, expires_at? } -> Invite
POST   /invites/:code            # accept an invite, join the server
GET    /invites/:code            # preview the server before joining

# admin (instance-level; requires instance_owner)
POST   /admin/registration-tokens   { max_uses?, expires_at? } -> RegistrationToken
GET    /admin/registration-tokens
DELETE /admin/registration-tokens/:code

# instance (public) — a client must know a given server's capabilities
GET    /instance                 -> { voice_enabled, open_registration, max_upload_mb }

# gateway discovery
GET    /gateway                  -> { url: "wss://host/gateway" }
```

---

## 4a. Rate limiting and client addresses

`/auth/register` and `/auth/login` are the only endpoints reachable without a
token, and each costs an argon2id hash — 64 MB of memory and four threads by
design. That cost is right against password cracking, but it means an
unauthenticated caller can make the server allocate 64 MB per request as fast as
it can send them; on a home server that exhausts RAM long before it exhausts the
password space. So both are limited **before** the handler runs, and a rejected
request never starts the hash.

Over budget returns **429** with `Retry-After` in seconds.

**Attributing a request to an address** is the subtle part, and both naive
readings are wrong:

- Always trusting `X-Forwarded-For` lets anyone send a random value and land in
  a fresh bucket every time. The limiter then does nothing while appearing to
  work — worse than not having one.
- Never trusting it collapses every user behind the reverse proxy into a single
  bucket keyed by the proxy, so one attacker locks out everybody.

Cove trusts the header exactly when the immediate peer is loopback or a private
address — i.e. a reverse proxy on the same host or container network, which is
what `deploy/` sets up. When the server is exposed directly, the peer is public,
the header is ignored, and `RemoteAddr` is used (it cannot be forged over TCP).
The header is read right to left, taking the first address that is not a trusted
proxy, so an entry a client prepends itself cannot win.

**Implication for anyone deploying Cove:** if you put it behind a proxy, that
proxy must set `X-Forwarded-For` (Caddy's `reverse_proxy` does automatically).
If you put it behind a proxy that does *not*, every user shares one bucket.

## 5. Settled decisions

1. **Backend** — Go + PostgreSQL + sqlc. ✓
2. **Registration** — open, but gated by a `RegistrationToken` from the instance
   owner (the Matrix model). ✓
3. **Presence** — online/offline in Phase 0, derived from the state of gateway
   connections. ✓
4. **Repositories** — three independent repos side by side: `cove-meta/`
   (`docs/`, `deploy/`), `cove-server/` and `cove-web/`. ✓
5. **Client** — React (Vite). ✓
6. **Dev environment** — a Nix flake dev-shell in `cove-server/` (go, sqlc,
   golang-migrate, psql). ✓
7. **Language** — everything inside the repositories is written in English:
   comments, configuration and these docs. ✓

## 6. The "instance owner" role

The first account to register (or the one named by `COVE_ADMIN_USERNAME`) gets
`instance_owner = true`. Only they issue `RegistrationToken`s and, later,
administer the instance. This is **not** the same as a server owner
(`Server.owner_id`): one is about the instance, the other about one server.
