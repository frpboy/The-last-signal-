# Client/server network protocol

This document describes the protocol implemented by the current Python client and
Rust server. It is a snapshot of the implementation, not a specification for
planned gameplay features. The protocol is still evolving; update this document
when packet definitions or behavior change.

## Source of truth

- Python packet IDs and framing: [client_python/packet.py](../client_python/packet.py)
- Python socket handling: [client_python/client.py](../client_python/client.py)
- Python typed packets: [client_python/packets](../client_python/packets)
- Rust packet IDs, framing, and limits:
  [server_rust/src/network/packet.rs](../server_rust/src/network/packet.rs)
- Rust payload parsing:
  [server_rust/src/network/parser.rs](../server_rust/src/network/parser.rs)
- Rust packet handling:
  [server_rust/src/network/handler.rs](../server_rust/src/network/handler.rs)
- Player snapshots and broadcasts:
  [server_rust/src/world/world.rs](../server_rust/src/world/world.rs)

## Transport and frame format

The current client uses TCP. Its default endpoint is **127.0.0.1:5000**; this is a
local development endpoint, not a public game server.

Each frame is encoded as follows. All integer fields use big-endian byte order.

| Field | Size | Meaning |
|---|---:|---|
| Frame length | 4 bytes | Unsigned 32-bit length of the body that follows |
| Packet type | 2 bytes | Unsigned 16-bit packet identifier |
| Payload | Variable | Packet-specific bytes |

The frame length includes the two-byte packet type and the payload, but excludes
the four-byte length prefix. Therefore, the body length is the packet-type
length plus the payload length.

For example, a PING packet with an empty payload has a body length of two:

~~~text
00 00 00 02  00 01
| length   | |type|
~~~

The Rust receiver rejects a body shorter than two bytes, a body larger than
10 MiB, and an unknown packet type. The Python socket client also rejects frames
larger than 10 MiB. The client reads the four-byte prefix before passing the body
to its packet decoder; callers of the decoder must pass the body, not the full
frame including the prefix.

A receiver must read the complete length prefix and then exactly the declared
number of body bytes. TCP reads do not preserve application message boundaries.

## Packet identifiers

The identifier values are shared by the Python and Rust enums where both define
the packet. The spelling of a Python enum member is not always the same as the
Rust name; use the numeric ID as the wire-level identifier.

| ID | Rust name | Python name | Direction and current behavior |
|---:|---|---|---|
| 1 | Ping | PING | Client to server; the server replies with a PING carrying the bytes PONG. |
| 2 | Login | LOGIN | Client to server; length-prefixed email and password. |
| 3 | Chat | CHAT | Client to server; UTF-8 message. The server currently echoes the payload in a CHAT response. |
| 4 | Move | MOVE | Client to server; three signed 32-bit coordinates. The server updates the session position and broadcasts PLAYER_STATE. |
| 5 | Log | LOG | Client to server; JSON client-log object expected by the Rust handler. |
| 6 | SignUp | SINGUP | Client to server; same payload layout as LOGIN. The Python enum retains the spelling SINGUP. |
| 7 | LoginResponse | LoginResponse | Server to client; UTF-8 status message. |
| 8 | SignUpResponse | SignUpResponse | Server to client; UTF-8 status message. |
| 9 | BAN | BAN | Server to client; NUL-separated ban information. A BAN packet from the client is ignored. |
| 10 | DECO | DECO | Client to server; UTF-8 disconnect reason. The server ends the client session. |
| 11 | MarketBuy | MARKET_BUY | Client to server; 24-byte market buy request. Response reuses ID 11. |
| 12 | MarketSell | MARKET_SELL | Client to server; 24-byte market sell request. Response reuses ID 12. |
| 13 | MarketCancelBuy | MARKET_CANCEL_BUY | Client to server; 8-byte order ID. Response reuses ID 13. |
| 14 | MarketCancelSell | MARKET_CANCEL_SELL | Client to server; 8-byte order ID. Response reuses ID 14. |
| 15 | PlayerState | PLAYER_STATE | Server to client; player UUID and three coordinates. Used for initial world snapshots and movement broadcasts. |
| 16 | PlayerRemove | PLAYER_REMOVE | Server to client; UUID of a player that has left the world. |
| 17 | Session | SESSION | Server to client; 16-byte UUID assigned to the TCP session. |
| 18 | AccountListRequest | Not defined | Client-to-server request defined by Rust; current Python enum has no ID 18 member. |
| 19 | AccountListResponse | Not defined | Server-to-client response defined by Rust; current Python enum has no ID 19 member. |
| 20 | AccountCreate | Not defined | Client-to-server request defined by Rust; current Python enum has no ID 20 member. |
| 21 | AccountSelect | Not defined | Client-to-server request defined by Rust; current Python enum has no ID 21 member. |
| 22 | AccountResult | Not defined | Server-to-client response defined by Rust; current Python enum has no ID 22 member. |

The Python enum currently defines IDs 1–17 only. The Rust enum defines IDs 1–22.
Consequently, the account packet types 18–22 are not currently represented by
the Python packet enum/typed packet layer. Do not infer that the current Python
client can send or decode those packets simply because the Rust server defines
them.

IDs 11–14 exist in both enums, but the current Python packet directory does not
contain corresponding typed market-packet classes. Their payload layouts below
are based on the Rust handler and should be kept synchronized with any future
Python implementation.

## Payload formats

### LOGIN and SignUp

Both requests use the same layout:

| Field | Encoding |
|---|---|
| Email length | Unsigned 16-bit big-endian byte length |
| Email | UTF-8 bytes |
| Password length | Unsigned 16-bit big-endian byte length |
| Password | UTF-8 bytes |

Lengths count encoded bytes, not Unicode characters. The Rust parser validates
that both fields are present, complete, and valid UTF-8. The current parser and
tests accept trailing bytes after the password field; clients should send only
the defined fields.

The server answers LOGIN with ID 7 and SignUp with ID 8. The response payload
is currently plain UTF-8 status text rather than a structured JSON result. The
text can report success or a condition such as invalid credentials, a blocked
account, or a server error. Clients must not treat a human-readable status
string as a stable machine-readable error code.

### CHAT

The payload is the message encoded as UTF-8, with no length field inside the
payload. The server logs the decoded message and currently responds with the
same packet ID and unchanged payload.

### MOVE

The payload must be exactly 12 bytes:

| Offset | Size | Field |
|---:|---:|---|
| 0 | 4 bytes | X: signed 32-bit big-endian integer |
| 4 | 4 bytes | Y: signed 32-bit big-endian integer |
| 8 | 4 bytes | Z: signed 32-bit big-endian integer |

The server currently ignores a MOVE request whose payload is not 12 bytes.
For a valid request, it associates the coordinates with the current session UUID
and broadcasts a PLAYER_STATE packet.

### LOG

The Rust handler parses the payload as a JSON object representing a client log:

~~~json
{
  "level": "INFO",
  "module": "network",
  "file": "client.py",
  "line": 42,
  "message": "Connected"
}
~~~

The fields are level, module, file, line, and message. Supported level names are
TRACE, DEBUG, INFO, WARNING, and ERROR. The example values are illustrative.
The payload must be JSON accepted by the Rust ClientLog type; malformed JSON is
logged and ignored.

### PING

A client request normally has an empty payload. The server responds with packet
ID 1 and payload bytes spelling PONG.

### BAN

The payload is UTF-8 text split into three NUL-separated fields:

~~~text
<ban-type>\0<reason>\0<deban-date>
~~~

| Field | Meaning |
|---|---|
| Ban type 1 | Temporary ban |
| Ban type 2 | Permanent ban |
| Reason | Human-readable reason |
| Deban date | Date/time text for a temporary ban; empty when absent |

The current Rust encoder always emits the three fields. The Python decoder treats
an empty date as absent. Because this format uses NUL separators, the fields
must not contain an unescaped NUL character.

### DECO

The client sends a UTF-8 reason as the payload. The current server handler treats
DECO as a disconnect request and ends the session; it does not require a
structured response payload.

### MARKET_BUY and MARKET_SELL

Both request payloads are exactly 24 bytes, represented by three consecutive
big-endian integer fields:

| Offset | Size | Field |
|---:|---:|---|
| 0 | 8 bytes | Object ID: signed 64-bit integer |
| 8 | 8 bytes | Quantity: unsigned 64-bit integer |
| 16 | 8 bytes | Unit price: signed 64-bit integer |

For MARKET_BUY, the price is the maximum unit price. For MARKET_SELL, it is the
unit price offered by the seller.

A successful request returns the same packet ID with the created order ID as an
8-byte signed 64-bit big-endian integer. On errors, the current handler may
return UTF-8 error text in the same packet type instead. A selected account is
required.

### MARKET_CANCEL_BUY and MARKET_CANCEL_SELL

The request payload is exactly one signed 64-bit big-endian order ID (8 bytes).
A successful response reuses the request packet type and returns the order ID as
an 8-byte big-endian integer. Error responses may contain UTF-8 text instead.
A selected account is required.

### PLAYER_STATE

The payload is exactly 28 bytes:

| Offset | Size | Field |
|---:|---:|---|
| 0 | 16 bytes | Player/session UUID |
| 16 | 4 bytes | X: signed 32-bit big-endian integer |
| 20 | 4 bytes | Y: signed 32-bit big-endian integer |
| 24 | 4 bytes | Z: signed 32-bit big-endian integer |

The UUID uses its 16-byte binary representation. Coordinates are signed
32-bit integers in big-endian order. The server sends PLAYER_STATE packets
when synchronizing the world and when broadcasting position changes.

### PLAYER_REMOVE

The payload is exactly 16 bytes: the UUID of the player/session being removed,
using the UUID's binary representation.

### SESSION

The server sends SESSION when a TCP client connects. Its payload is exactly
16 bytes: the binary UUID assigned to that connection. This identifier is a
network session identifier; it must not be confused with the persistent account
ID used by database-backed account and market operations.

### Account packets (Rust-defined; not yet mirrored by the Python enum)

The following formats are implemented in the Rust packet handler. The Python
enum and typed packet layer must be extended before these IDs can be used
reliably by the current Python client.

| ID and name | Request payload | Response |
|---|---|---|
| 18 AccountListRequest | Empty payload; requires an authenticated user | ID 19: JSON array of accounts on success; current error path may return UTF-8 text |
| 20 AccountCreate | UTF-8 account name; requires an authenticated user | ID 22: JSON result object |
| 21 AccountSelect | 8-byte signed 64-bit big-endian account ID; must be positive and owned by the authenticated user | ID 22: JSON result object |

ID 22 AccountResult uses a JSON object with an ok boolean and a message string.
Depending on the operation and result, it can also include account or account_id.
For example, the shape of a successful account selection is similar to:

~~~json
{
  "ok": true,
  "message": "Compte sélectionné",
  "account_id": 1
}
~~~

The example is illustrative; current response messages are not a stable
localization-independent error-code system. AccountResult is a server response,
not a valid request. The server rejects response packet types received from a
client rather than treating them as new requests.

## Connection and world synchronization

The current server flow is:

1. A TCP connection is accepted and a session UUID is generated.
2. The server sends SESSION (ID 17) with the session UUID.
3. The server sends PLAYER_STATE (ID 15) packets for the current world snapshot.
4. The new player's position is registered and a PLAYER_STATE broadcast is sent.
5. Valid MOVE packets update the position associated with the session UUID and
   trigger another PLAYER_STATE broadcast.
6. On session termination, the server removes the player from the world and
   notifies the other clients using PLAYER_REMOVE (ID 16).

The world currently stores active positions in memory. This section describes
the current prototype behavior and does not imply that position persistence
across reconnects is implemented.

## Validation and implementation caveats

- Keep numeric IDs synchronized between Python and Rust. A change to an ID or
  payload format can break both directions of communication.
- Respect exact payload sizes for binary fields. For example, MOVE is 12 bytes,
  PLAYER_STATE is 28 bytes, PLAYER_REMOVE and SESSION are 16 bytes, and market
  requests have the sizes described above.
- The Rust packet receiver rejects body lengths below two bytes, lengths above
  10 MiB, and unknown packet IDs.
- Current account IDs 18–22 are declared in Rust but missing from the Python
  enum. Market IDs 11–14 are declared in both enums but have no typed Python
  packet modules in the current packet directory.
- LOGIN and SignUp responses, market error responses, and account-list errors
  use text payloads in some cases. Do not assume that every response payload
  has a single machine-readable structure.
- The current project documents its cryptographic implementation as
  experimental and not production-ready. This protocol document does not claim
  that raw TCP communication is protected by a secure transport. Do not deploy
  the prototype on an untrusted network on the basis of this document.

When changing the protocol, update the Python and Rust implementations, relevant
tests, and this document together.
