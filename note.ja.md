# HTTP/2 (draft-ietf-httpbis-http2-17) sorah's reading note

https://tools.ietf.org/html/draft-ietf-httpbis-http2-17

## 2.

### 2.2. Conventions and Terminology

Quoting,

```
client:  The endpoint that initiates an HTTP/2 connection.  Clients
send HTTP requests and receive HTTP responses.

connection:  A transport-layer connection between two endpoints.

connection error:  An error that affects the entire HTTP/2
connection.

endpoint:  Either the client or server of the connection.

frame:  The smallest unit of communication within an HTTP/2
connection, consisting of a header and a variable-length sequence
of octets structured according to the frame type.

peer:  An endpoint.  When discussing a particular endpoint, "peer"
refers to the endpoint that is remote to the primary subject of
discussion.

receiver:  An endpoint that is receiving frames.

sender:  An endpoint that is transmitting frames.

server:  The endpoint that accepts an HTTP/2 connection.  Servers
receive HTTP requests and serve HTTP responses.

stream:  A bi-directional flow of frames within the HTTP/2
connection.

stream error:  An error on the individual HTTP/2 stream.

Finally, the terms "gateway", "intermediary", "proxy", and "tunnel"
are defined in Section 2.3 of [RFC7230].  Intermediaries act as both
client and server at different times.```
```

## 3. Starting HTTP/2

https://tools.ietf.org/html/draft-ietf-httpbis-http2-17#section-3

### 3.1. HTTP/2 Version Identification

- `h2`: HTTP/2 over TLS
- `h2c`: HTTP/2 (cleartext)
  - Used in HTTP/1.1 `Upgrade` header

      ### 3.2. Starting HTTP/2 for "http" URIs

      ```
      > GET / HTTP/1.1
      > Host: ...
      > Connection: Upgrade, HTTP2-Settings
      > Upgrade: h2c
      > HTTP2-Settings: ...
      >
      >
      < HTTP/1.1 101 Switching Protocols
      < Connection: Upgrade
      < Upgrade: h2c
      <
      < [ HTTP/2 ]
      ```

- [HTTP2-Settings](https://tools.ietf.org/html/draft-ietf-httpbis-http2-17#section-3.2.1): Required.
  - 3.2.1.
  - Server MUST NOT send this header
  - `token68`: https://tools.ietf.org/html/rfc7235#section-2.1
- Servers MUST ignore `h2` identifier (non plaintext, it's http/2 over TLS)
- TODO:

    > Requests that contain an payload body MUST be sent in their entirety
    > before the client can send HTTP/2 frames.  This means that a large
    > request can block the use of the connection until it is completely
    > sent.

   - `payload body`: https://tools.ietf.org/html/rfc7230#section-3.3

- Initial HTTP/2 frames from server to client, MUST include a response to the initial request.
  - The first frame from server MUST be a `SETTINGS` frame as the _server connection preface_ .
- Client MUST send _connection preface_ including a `SETTINGS` frame to server after receiving `101 Switching Protocols`
- HTTP/1.1 request sent prior `Upgrade`, is a `stream` with ID `1`.
  - _priority_ is default
  - _half closed_
  - After upgrading, the stream is used for the response

      ### 3.3. Starting HTTP/2 for "https" URIs

- TLS ALPN with `h2` protocol identifier
- after TLS negotiation, both server and client MUST send a _connection preface_

    ### 3.4. Starting HTTP/2 with Prior Knowledge

- mentioning [ALT-SVC; HTTP Alternative Services](https://tools.ietf.org/html/draft-ietf-httpbis-alt-svc-06)

    TODO

    ### 3.5. HTTP/2 connection preface

- client's _connection preface_ starts with `"PRI * HTTP/2.0\r\n\r\nSM\r\n\r\n"`
  - MUST be followed by `SETTINGS` frame (which may be empty)
  - Client sends this immediately upon successful upgrade or as the first data of TLS connection
- server's _connection preface_ consists of a potentially empty `SETTINGS` frame
  - the frame MUST be sent as the first frame sent from server
- Peer MUST _acknowledge_ `SETTINGS` frames in a _connection preface_ after sending its? connection preface
- Clients are permitted to send additional frames to server without waiting responses from server, for minimize latency
- Clients and servers MUST treat invalid connection preface as PROTOCOL_ERROR

    ## 4. HTTP Frames

    ### Format

    9 bytes (72 bits)

- 24 bit (3 bytes): Length (unsigned int)
  - without frame header
  - 16384+ (2^14) MUST NOT be sent, unless receiver has set `SETTINGS_MAX_FRAME_SIZE`
- 8 bit (1 byte): Type
  - MUST ignore unknown types
- 8 bit (1 byte): Flags (booleans)
  - frame type specific
- 1 bit: Reserved
  - MUST remain `0x0`
- 31 bit (3 bytes + 7 bit): Stream Identifier (unsigned int)
  - value `0x0`: reserved for a whole connection

      ### 4.2. Frame Size

- `SETTINGS_MAX_FRAME_SIZE` setting
  - 2^14 (16384) .. 2^24-1 (16777215) bytes
- All impl MUST be capable 2^14 bytes + 9 bytes (header) at minimum
- _Frame size_ doesn't include header size (usually)
- Note: Certain frame types (such as PING), has additional limits.
- FRAME_SIZE_ERROR for any violations (small, bigger, ...)
  - it's _connection error_ when the frame can alter the state of connection

      ### 4.3. Header Compression and Decompression

- _header lists_ are collection of _header fields_
  - serialized into a _header block_ using HTTP Header Copression (HPACK)
  - serlilized block can be divided into 1+ byte sequences (called _header block fragments_ )
- Cookie header field is treated specially
  - [COOKIE]: Join fields with `; ` for prior protocols (HTTP/1.1)
- A complete _header block_
    1. a single `HEADERS` or `PUSH_PROMISE` frame, with `END_HEADERS` flag
    2. a `HEADERS` or `PUSH_PROMISE` frame with `END_HEADERS` flag cleared 
    and one or more `CONTINUATION` frames, where the last `CONTINUATION` frame
    has the `END_HEADERS` flag set (TODO)
- HPACK is stateful, context is shared entire connection
  - error MUST be treated as _connection error_ `COMPRESSION_ERROR`
- _header blocks_ MUST be transmitted as a contiguous sequence of frames
  - last frame has `END_HEADERS` flag set
- _header block fragments_ can only be sent as the payload of:
  - `HEADERS`
  - `PUSH_PROMISE`
  - `CONTINUATION`

      ## 5. Streams and Multiplexing

- _stream_, independent, bi-directional sequence of frames, exchanges between client and server
  - single HTTP/2 _connection_ can contain multiple concurrently streams
  - can be established and used unilaterally or shared by either client or server
  - can be closed by either endpoint
  - frames order is significant (e.g. `HEADERS`, `DATA`)

      ### 5.1. Frame states

      Quoting:

      ```
      +--------+
      send PP |        | recv PP
      ,--------|  idle  |--------.
      /         |        |         \
      v          +--------+          v
      +----------+          |           +----------+
      |          |          | send H /  |          |
      ,------| reserved |          | recv H    | reserved |------.
      |      | (local)  |          |           | (remote) |      |
      |      +----------+          v           +----------+      |
      |          |             +--------+             |          |
      |          |     recv ES |        | send ES     |          |
      |   send H |     ,-------|  open  |-------.     | recv H   |
      |          |    /        |        |        \    |          |
      |          v   v         +--------+         v   v          |
      |      +----------+          |           +----------+      |
      |      |   half   |          |           |   half   |      |
      |      |  closed  |          | send R /  |  closed  |      |
      |      | (remote) |          | recv R    | (local)  |      |
      |      +----------+          |           +----------+      |
      |           |                |                 |           |
      |           | send ES /      |       recv ES / |           |
      |           | send R /       v        send R / |           |
      |           | recv R     +--------+   recv R   |           |
      | send R /  `----------->|        |<-----------'  send R / |
      | recv R                 | closed |               recv R   |
      `----------------------->|        |<----------------------'
      +--------+

      send:   endpoint sends this frame
      recv:   endpoint receives this frame

      H:  HEADERS frame (with implied CONTINUATIONs)
      PP: PUSH_PROMISE frame (with implied CONTINUATIONs)
      ES: END_STREAM flag
      R:  RST_STREAM frame
      ```

- `PRIORITY` frame can be sent in any stream state
- open state
  - observe _stream level flow control limits_

      #### 5.1.1. Stream Identifiers

- 31-bit unsigned integer
- initiated by a client MUST use odd numbers
- initiated by a server MUST use even numbers
- `0` is used for connection control
- requests upgraded from HTTP/1.1 are responded with id `1`
- newly established stream's identifier MUST be numerically greater than all streams of their
  - violations are _connection error_ `PROTOCOL_ERROR`
  - streams with lower identifier values might used from endpoint, are implicitly closed when any stream with greater identifier value opened
    - For example, stream 5 will go `closed` state from `idle` state when stream 7 gets opened
- stream identifier can't be reused; long-lived connection may exhaust the all available stream identifier. Clients can establish a new connection for new streams. Also server can send `GOAWAY` for such case.

    #### 5.1.2. Stream Concurrency

- `SETTINGS_MAX_CONCURRENT_STREAMS`
  - effect to specific endpoint; client sets limit for servers, and servers sets limit for clients.
  - Both `open` and `half closed` streams are counted, `reserved` streams aren't.
- Endpoints MUST NOT exceed the limit
  - or _stream error_ `PROTOCOL_ERROR` or `REFUSED_STREAM`
  - (error code choice changes behavior around automatic retry -- ref. Section 8.1.4.)
- When endpoint wishes to reduce the value of limit, can close streams or allow streams to complete (?) TODO

    ### 5.2. Flow Control

- `WINDOW_UPDATE` frame

    #### 5.2.1. Frow Control Principles

- HTTP/2 flow control aims to allow a veriety of algorithms, without requiring protocol specification change
- flow control is
    1. specific to a connection, single hop
    2. based on `WINDOW_UPDATE` frames, advertised from receivers
    - credit-based scheme
        3. directional
    - receiver MANY choose any window size for each stream or for entire connection
    - sender MUST respect limits
        4. initial value is 65535 bytes (new streams, and the overall connection)
        5. frame type determines application of flow control; in this document, only `DATA` frames are subject to flow control.
        6. Can't be disabled
        7. HTTP/2 only defines the format and semantics of `WINDOW_UPDATE` frame

        #### 5.2.2. Appropiate Use of Flow Control

- flow control is defined to protect endpoints under resource constraints
  - A proxy: share memory between many connections
  - slow upstream/downstream
- addresses case where receiver can't process data on _one_ stream; even they still wants to continue processing other stream, in the same connection.
- if don't require this capability, can advertise window at the maximum size (2^31-1).
  - disables effectively, but senders still subject to the flow control window.
- TODO: RFC 7323
  - for full awareness for bandwidth-delay products, flow control can be difficult.
- receiver MUST read TCP receive buffer in a timely fashion, or results deadlock when critical frames (e.g. `WINDOW_UPDATE`) aren't read and acted.

    ### 5.3. Stream priority

- client can assign a priority for new stream
  - use prioritization information in `HEADERS` frame (which opening stream)
  - for other time, use `PRIORITY` frame
- purpose
  - allow an endpoint to express how it would prefer peer allocate resource when managing streams
  - select streams for transmitting frames, when under limited capacity to send
- prioritization by marking streams as dependent on the other streams
  - relative weight
- expressing priority is a suggestion

    #### 5.3.1. Stream Dependencies

- no dependency = priority value `0`, which means the root of tree
- A stream depends on another stream is a _dependent stream_
  - _parent stream_
- dependent streams aren't ordered (allowed to be any order)
- _exclusive flag_ allows insertion of a new level of dependencies
  - `A -> B,C` to `A -> D -> B,C` (Figure 4)
- A _dependent stream_ SHOULD only be allocated resources, if all of the streams are either closed or it's not possible to make progress
- A stream can't depend on itself. For violations, an endpoint MUST treat s a stream error `PROTOCOL_ERROR`

    #### 5.3.2. Dependency Weighting

- _dependent streams_ are allocated weight in 1..256 (integer)
- Streams with the same parent SHOULD be allocated resources proportionally based on their weight

    #### 5.3.3. Reprioritization

- `PRIORITY` frame
- _dependent stream_ moves with their parent, when parent gets reprioritized

    #### 5.3.4. Prioritization State Management

- when a stream is removed from the dependency tree
  - its dependencies can be dependent on the parent of the closed stream (? TODO)
- Streams that are removed from the tree cause some information to be lost
- an endpoint SHOULD retain priorization state for a period, after become closed 
  - to avoid problems around correct resource allocation?
- The retention of priority information for streams are not counted toward `SETTINGS_MAX_CONCURRENT_STREAMS`
  - Therefore retention MAY be limited
- streams in `idle` state can be assigned priority or become a parent of other streams
  - allows grouping
- Under high load, prioritization state can be discarded
  - Extreme cases, endpoint could discard state for active or reserved streams
  - if a limit is applied, endpoints SHOULD maintain state for their `SETTINGS_MAX_CONCURRENT_STREAMS`
  - implementation SHOULD attempt to retain states for all active streams in the tree
- When received a `PRIORITY` frame for closed stream SHOULD alter the tree if it still retains enough state.

    TODO

    #### 5.3.5. Default Priorities

- all streams depends on `0` basically
- but _Pushed streams_ depends on their associated streasm.
- For both case, weight are initially 16.

    ### 5.4. Error Handling

- _connection error:_ makes the entire connection unusable
- _stream error:_ for individual streasm

    #### 5.4.1. Connection Error Handling

- An endpoint encountered a _connection error_ SHOULD first send a `GOAWAY` frame
  - with the stream identifier of the last stream that successfully received from its peer.
  - `GOAWAY` frame includes an error code
  - An endpoint MUST close TCP connection after `GOAWAY` frame has sent.
- An endpoint MAY choose to treat a _stream error_ as a _connection error._
  - Endpoints SHOULD send a `GOAWAY` when ending a connection

      #### 5.4.2. Stream Error Handling

- An endpoint sends `RST_STREAM` frame containing the stream identifier and error code
- `RST_STREAM` frame is the last frame that endpoint can send on a stream
- The peer MUST be prepared to receive any frames by the remote peer.
  - These can be ignored except if they modify connection state (header compression or flow control)
- an endpoint SHOULD NOT send more than one `RST_STREAM`
  - but MAY send `RST_STREAM` if it receives frames on a closed stream after more than a RTT time
- An endpoint MUST NOT responds a `RST_STREAM` for a `RST_STREAM` to avoid loops.

    #### 5.4.3. Connection Termination

    TODO

    ### 5.5. Extending HTTP/2

- frame types, settings, error codes
- Implementation MUST ignore/discard unknown/unsupported values (where supports extension)
- Extension frames in the middle of a _header block_ aren't permitted
  - MUST be treated as a _connection error_ `PROTOCOL_ERROR`
- Extensions that could change the existing semantics MUST be negotiated before use
  - e.g. changeing layout of `HEADERS` frame can't be used until the peer says it's acceptable
- This document doesn't define such negotiation, but `SETTINGS` could be used for such purpose.
  - if `SETTINGS` is used for extension negotioation, the initial value MUST be state like disabled.

      ## 6. Frame Definitions

      ### 6.1. DATA

- `DATA` frames (type: `0x0`)
- arbitrary variable-length sequences of bytes on stream
- for instance, to carry HTTP request or response payloads
- can have padding
  - Security feature to abscure the size of messages
- fields:
  - 8 bit: Pad length
    - conditional; present only when `PADDED` flag is set
  - Data
  - Padding
- Flags
  - `END_STREAM` (`0x1`): Bit 0
    - indicate a frame is the last from the endpoint will send on the associated stream
    - turns the stream to `half closed` or `closed` state
  - `PADDED` (`0x8`): Bit 3
    - indicates Pad Length and padding are present
- `DATA` frames MUST be associated with a stream
  - the recipient MUST respond with a _connection error_ `PROTOCOL_ERROR` for violations
- Subject to flow control
- Can only be sent when a stream is `open` or `half closed (remote)` states
  - violations, the recipient MUST respond with a _stream error_ `STREAM_CLOSED`
- total number of padding bytes is determined by the value of Pad Length
  - If pedding length is  frame payload or greater, the recipient MUST treat as a _connection error_ `PROTOCOL_ERROR`
  - A frame can be increased in size by 1 byte by including Pad Length with `0`

      ### 6.2. HEADERS

- `HEADER` frame (type: `0x1`)
- used to open stream
- additionally carries a _header block fragment_
- can be sent on a stream in the `open` or `half closed (remote)` states
- fields:
  - 8 bit: Pad Length
    - conditional; present only when `PADDED` flag is set
  - 1 bit: E (Exclusive flag)
    - conditional; present only when `PRIORITY` flag is set
    - a bit flag indicates _stream dependency_ is _exclusive_
  - 31 bit: Stream Dependency
    - conditional; present only when `PRIORITY` flag is set
    - _Stream identifier_ for the stream that depends on
  - 8 bit: Weight
    - conditional; present only when `PRIORITY` flag is set
    - unsigned int
    - increase value by 1 to obtain the value in range (1..256)
  - _Header Block Fragment_
  - Padding
- flags
  - `END_STREAM` (`0x1`): Bit 0
    - indicate a frame is the last from the endpoint will send on the associated stream
    - but `HEADERS` frame can be followed by `CONTINUATION` frames
      - the frames are part of `HEADERS` frame
  - `END_HEADERS` (`0x4`): Bit 2
    - Indicates that this frame contains an entire _header block_
    - also indicates no `CONTINUATION` frames
    - A `HEADERS` frame without this flag set MUST be followed by a `CONTINUATION` frame on same stream
      - Violations, A receiver MUST treat as a _connection error_ `PROTOCOL_ERROR`
      - Even if received another frame on a _different_ stream
  - `PADDED` (`0x8`): Bit 3
    - set indicates that the Pad Length field and padding are present
  - `PRIORITY` (`0x20`): Bit 5
    - set indicates Exclusive flag, Stream Dependency, Weight fields are present
- Payload contains a _header block fragment_
- MUST be associated with a stream
  - the recipient MUST respond with a _connection error_ `PROTOCOL_ERROR` for violations

      ### 6.3. PRIORITY

- `PRIORITY` frame (type: `0x2`)
- priority advertisement from sender for a stream
- can be sent any time (including `idle` and `closed` streams)
- fields:
  - 1 bit: E (Exclusive flag)
    - a bit flag indicates _stream dependency_ is _exclusive_
  - 31 bit: Stream Dependency
    - _Stream identifier_ for the stream that depends on
  - 8 bit: Weight
    - unsigned int
    - increase value by 1 to obtain the value in range (1..256)
- No flags
- MUST be associated with a stream
  - the recipient MUST respond with a _connection error_ `PROTOCOL_ERROR` for violations (stream `0x0`)
- This frame cn be sent for `idle` or `closed` streams
  - Allowing reprioritization of a dependent group by unused or closed _parent streams._
- Length other than 5 bytes MUST be treated as a _stream error_ `FRAME_SIZE_ERROR`

    ### 6.4. RST_STREAM

- `RST_STREAM` (type: `0x3`)
- immediate termination of a stream
- fields:
  - 32 bit: Error Code
    - 32 bit unsigned integer
- no flags
- fully terminates stream and makes stream `closed`
- receiver MUST NOT send additional frames for the stream
  - except `PRIORITY` frames
- after sending `RST_STREAM`, the sender MUST be prepared to receive and process addtional frames, where sent from receiver prior to receive `RST_STREAM`
  - e.g. header compression?
- MUST be associated with a stream
  - the recipient MUST respond with a _connection error_ `PROTOCOL_ERROR` for violations
- MUST NOT be sent for stream in `idle` state
  - the recipient MUST treat as a _connection error_ `PROTOCOL_ERROR` for violations
- Length other than 4 bytes MUST be treated as a _connection error_ `FRAME_SIZE_ERROR`

    ### 6.5. SETTINGS

- `SETTINGS` frame (type: `0x4`)
- configuration parameters
- also used to acknowledge of parameters
- parameters are not negotiated
  - sender, receiver may have different value for the same setting
- A `SETTINGS` frame MUST be sent by both endpoints at the start of a connection
  - and MAY be sent at any other time from either endpoint
- Implementation MUST support all perameters defined in this specification
- the value of a parameter is the last value seen from receiver
- parameters are acknowledged by the receiver (`ACK` flag)
- flags:
  - `ACK` (`0x1`): Bit 0
    - if this bit set, payload MUST be empty
      - violation MUST be treated as a _connection error_ `FRAME_SIZE_ERROR`
- `SETTINGS` always applies to a connection, not a stream.
- The stream identifier MUST be `0`.
  - violation, the endpoint MUST respond ith a _connection error_ `PROTOCOL_ERROR`
- Badly formed or incomplete `SETTINGS` frme MUST be treated as a _connection error_ `PROTOCOL_ERROR`

    #### 6.5.1. SETTINGS Format

- consists of zero or more parameters
- fields (each)
  - 16 bit: Identifier (unsigned)
  - 32 bit: Value (unsigned)

      #### 6.5.2 Defined SETTINGS Parameters

- `SETTINGS_HEADER_TABLE_SIZE` (`0x1`)
  - maximum size of the header compression table
  - Encoder can select any value equal or less than the value
  - initial value: 4096 bytes
- `SETTINGS_ENABLE_PUSH` (`0x2`)
  - Can be use to disable server push
  - An endpoint MUST NOT send a `PUSH_PROMISE` frame if this set to `0`
    - violation, MUST treat the receipt of a `PUSH_PROMISE` as a _connection error_ `PROTOCOL_ERROR`
  - value other than `0` or `1` MUST be treated as a _connection error_ `PROTOCOL_ERROR`
  - initial value: `1` (enabled)
- `SETTINGS_MAX_CONCURRENT_STREAMS` (`0x3`)
  - maximum number of concurrent streams, that sender will allow
  - directional
  - recommended to be no smaller than 100
  - A value `0` SHOULD NOT be treated as special
    - used to prevent the creation of new streams
    - Servers SHOULD only set `0` for short value
      - (if servers doesn't wish to accept requests, close the connection)
  - initial value: no limit
- `SETTINGS_INITIAL_WINDOW_SIZE` (`0x4`)
  - initial window size (in bytes) for flow control
  - effect all streams
  - value above 2^31-1 MUST be treated as a _connection error_ `FLOW_CONTROL_ERROR`
  - initial value: 2^16-1 (65535) bytes
- `SETTINGS_MAX_FRAME_SIZE` (`0x5`)
    - largest frame payload that sender is willing to receive
    - Range MUST be between 16384 .. 2^24-1 (16777215)
      - violations MUST be treated as a _connection error_ `PROTOCOL_ERROR`
    - initial value: 2^14 (16384) bytes
- `SETTINGS_MAX_HEADER_LIST_SIZE` (`0x6`)
    - maximum size of header list that the sender is prepared to accept
    - in bytes
    - value based on the uncompressed size of _header fields_ including name, value, and overhead of 32 bytes for _header field_
    - TODO: a lower limit than what is advertised MAY be enforced ???
    - initial value: unlimited

- An endpoint MUST ignore unknown or unsupported settings

    #### 6.5.2. Settings Synchronization

- recipient MUST apply the update parameters ASAP upon receipt
- the values MUST be processed in the order they appear
- Unsupported parameters MUST be ignored
- receipient MUST emit `SETTINGS` frame with `ACK` flag after processing
- Upon receiving a `ACK` flag, sender can rely on the new settings
- Sender MAY issue a _connection error_ `SETTINGS_TIMEOUT` after a reasonable amount of time when sender didn't receive `ACK`.

    ### 6.6. PUSH_PROMISE

- `PUSH_PROMISE` frame (type: `0x5`)
- fields:
  - 8 bit: Pad Length
    - conditional; present only when `PADDED` flag is set
  - 1 bit: R (reserved)
  - 31 bit: Promised Stream ID
    - unsigned 31 bit _stream identifier_ to reserve
  - Header Block Fragment:
    - _header block fragment_ of request header fields
  - Padding
- flags:
  - `END_HEADERS` (`0x4`): Bit 2
    - indicates this frame contains entire _header block_
    - indicates frame is not followed by any `CONTINUATION` frame
    - without this flag set MUST be followed by a `CONTINUATION` frame on same stream
      - Violations, A receiver MUST treat as a _connection error_ `PROTOCOL_ERROR`
      - Even if received another frame on a _different_ stream
  - `PADDED` (`0x8`): Bit 3
    - set indicates that the Pad Length field and padding are present
- MUST be associated with a peer-initiated stream in `open` or `half closed (remote)` state
- if the _stream identifier_ field specifies `0`, recipient MUST respond with a _connection error_ `PROTOCOL_ERROR`
- _Promised streams_ are not required to be used in the order of promise
  - Just to reserve streams for later use
- `PUSH_PROMISE` MUST NOT be sent if receiver sets `SETTINGS_ENABLE_PUSH` to `0`
  - violations MUST treat the receipt of a `PUSH_PROMISE` as a _connection error_ `PROTOCOL_ERROR`
- Recipients can choose to reject _promised streams_ by returning `RST_STREAM` for the promised stream
- `PUSH_PROMISE` modifies the connection state:
  - State maintained for header compression
  - changing _promised stream_ state to `reserved`
- Sender MUST NOT send a `PUSH_PROMISE` on a stream not in `open` or `half closed (remote)` state
  - violations, A receiver MUST treat the receipt of a `PUSH_PROMISE` as a _connection error_ `PROTOCOL_ERROR`
- Sender MUST ensure the _promised stream_ is a valid as new _stream identifier_
  - violations, A receiver MUST treat the receipt of a `PUSH_PROMISE` as a _connection error_ `PROTOCOL_ERROR`

      ### 6.7. PING

- `PING` frame (type: `0x6`)
- mechanism for measuring minimal RTT
- determining idle connection is still function
- can be sent from any endpoint
- MUST contain 8 bytes of data in the payload
- receiver responses with `ACK` flag and identical payload
- flags:
  - `ACK` (type: `0x1`): Bit 0
    - MUST set this flag in response
    - MUST NOT respond to `PING` if this set
- no association with any stream (always `0x0`)
  - violations, the recipient MUST respond with a _connection error_ `PROTOCOL_ERROR`
- length other than 8 bytes MUST be treated as a connection error `FRAME_SIZE_ERROR`

    ### 6.8. GOAWAY

- `GOAWAY` frame (type: `0x7`)
- informs the remote peer to stop creating streams
- can be sent from any endpoint
- Once sent, the sender will ignore frames sent on any new streams with identifiers higher than specified _stream identifier_
- Purpose: gracefully stop accepting new streams
- To avoid race conditions, `GOAWAY` frame contains the last peer-initiated _stream identifier_
- Endpoints SHOULD always send `GOAWAY` before closing a connection
  - client can know partially processed or not
- Endpoints MAY choose to close a connection without `GOAWAY` for misbehaving peers. (might? MAY?)
- Fields:
  - 1 bit?: R (reserved?)
  - 31 bit: Last-Stream-ID
  - 32 bit: Error Code
  - Additional Debug Data
- No flags
- applies to the connection, not a specific stream
  - violation (other than `0x0`) MUST treat as as _connection error_ `PROTOCOL_ERROR`
- All streams up to and including the Last-Stream-ID might have processed
  - (in this context, "processed" means higher layer)
- if connection terminates without `GOAWAY`, the last _stream identifier_ is effectively the highest possible _stream identifier_
- re-attempting activity in lower or equal _stream identifier_ than Last-Stream-ID, is not possible with exception of idempotent actions
- Any protocol activity that uses higher _stream identifier_ than Last-Stream-ID can be safely retried
- The sender MAY gracefully shutdown a connection, maintaining the connection in an open state until all progress streams complete (MAY? might?)
- An endpoint MAY send multiple `GOAWAY`
  - See the last Last-Stream-ID
  - Endpoint MUST NOT increase the Last-Stream-ID, because peer might have already retried
- Sender can discard frames for streams with identifier higher than Last-Stream-ID
  - frames can alter connection state can't be ignored completely 
  - `HEADERS`, `PUSH_PROMISE`, `CONTINUATION` MUST be minimally processed to ensure the state for header compression
  - similally, `DATA` frame MUST be counted for flow control window
- Endpoints MAY append data to the payload, for diagnostic purpose.
  - could contain privacy-sensitive data, so MUST have adequate safeguards when logging

      ### 6.9. WINDOW_UPDATE

- `WINDOW_UPDATE` frame (type: `0x8`)
- for flow control
- operates at 2 levels, each individual stream and entire connection
- hop-by-hop
- Frames that are exempt from flow control MUST be accepted and processed if the receiver can.
- A reciver MAY respond with a _stream error_ or _connection error_ `FLOW_CONTROL_ERROR` when unable to accept frames
- Fields:
  - 1 bit: R (reserved)
  - 31 bit: Window Size Increment
    - unsigned 31-bit integer
    - 1 .. 2^31-1 (2147483647) bytes
- no flags
- can be specific to a stream or a connection
  - for streams, _stream idenifier_ indicates the affected stream
  - for connections, `0` indicates so.
- A receiver MUST treat the recipt with an flow control window increment of `0` as a _stream error_ `PROTOCOL_ERROR`
  - if the target is connection, MUST be a _connection error_ `PROTOCOL_ERROR`
- can be sent on `half closed (local)` or `closed` streams
  - receiver may receive on `half closed (remote)` or `closed` streams
  - MUST NOT treat as an error
- A receiver that received a flow controlled frame MUST always account for its contribution against the window unless the receiver treats this as a connection error (? TODO)
- Length other than 4 bytes MUST be treated as a _connection error_ `FRAME_SIZE_ERROR`

    #### 6.9.1. The Flow Control Window

- 2 flow control windows: for stream and for connection
- Sender MUST NOT send a _flow controlled frame_ with a length that exceeds the space available in either the window (advertised from receiver)
- Frames with 0 length with `END_STREAM` (empty `DATA` frame) MAY be sent when no available space
- After sending a _flow controlled frame,_ the sender reduces the space available in the both windows by sent length
- A sender MUST NOT allow a _flow control window_ to exceed 2^31-1 bytes
  - TODO: if a sender receives (!?) ...
  - when exceeding this maximum, it MUST terminate stream or the connection.

      #### 6.9.2. Initial Flow Control Window Size

- initial window size 65535 bytes
- both endpoints can adjust by setting `SETTINGS_INITIAL_WINDOW_SIZE` for new streams
- connection _flow control window_ can be changed using `WINDOW_UPDATE` frames
- Prior to receiving `SETTINGS_INITIAL_WINDOW_SIZE` value, an endpoint can only use the default size when sending _flow controlled frames._
- the setting can alter the initial flow control window size for all streams (in `open` or `half closed (remote)` state)
  - when the setting value changes, a receiver MUST adjust the size of all stream that it maintains by the difference between the new value and the old value. (? TODO)
- changing the setting can cause the available space to become negative
  - A sender MUST track the negative _flow control window_
    - MUST NOT send _flow controlled frames_ until it receives `WINDOW_UPDATE` to update the window to positive

        #### 6.9.3. Reducing the Stream Window Size

- A receiver that wishes to use a smaller _flow control window_ can send a `SETTINGS`
  - However receivers MUST be prepared to receive data that exceeds this (???) window size
  - Sender might send data that the new limit prior to processing the new `SETTINGS`
- a receiver MAY continue to process streams that exceed the limit
  - TODO
  - the receiver MAY instead send a `RST_STREAM` with `FLOW_CONTROL_ERROR` for the affected streams.

      #### 6.10. CONTINUATION

- `CONTINUATION` frame (type: `0x9`)
- to continue a sequence of _header block fragments_
- Fields:
  - _Header Block Fragment_
- Flags:
  - `END_HEADERS` (`0x4`): Bit 2
    - Indicates that this frame contains an entire _header block_
    - without this flag set MUST be followed by a `CONTINUATION` frame on same stream
      - Violations, A receiver MUST treat as a _connection error_ `PROTOCOL_ERROR`
      - Even if received another frame on a _different_ stream
- MUST be associated with a stream
  - the recipient MUST respond with a _connection error_ `PROTOCOL_ERROR` for violations
- MUST be preceded by a `HEADERS`, `PUSH_PROMISE` or `CONTINUATION` frame without `END_HEADERS` flag set
  - violations, the recipient MUST respond with a _connection error_ `PROTOCOL_ERROR`

      ## 7. Error Codes

- 32-bit fields
- Used in `RST_STREAM`, `GOAWAY`
- Defined errors:
  - `NO_ERROR` (`0x0`)
    - no error
    - Used in `GOAWAY` for graceful shutdown 
  - `PROTOCOL_ERROR` (`0x1`)
    - unspecific protocol error
  - `INTERNAL_ERROR` (`0x2`)
    - unexpected internal error
  - `FLOW_CONTROL_ERROR` (`0x3`)
    - violation of flow control
  - `SETTINGS_TIMEOUT` (`0x4`)
    - endpoint didn't receive settings acknowledgement in a timely manner
  - `STREAM_CLOSED` (`0x5`)
    - frame on half closed stream
  - `FRAME_SIZE_ERROR` (`0x6`)
    - frame has invalid size
  - `REFUSED_STREAM` (`0x7`)
    - refuses the stream (TODO)
  - `CANCEL` (`0x8`)
    - indicate the stream is no longer needed
  - `COMPRESSION_ERROR` (`0x9`)
    - unable to maintain the _header compression context_
  - `CONNECT_ERROR` (`0xa`)
    - TODO
  - `ENHANCE_YOUR_CALM` (`0xb`)
    - exhibiting a behavior that might be generating excessive load
  - `INADEQUATE_SECURITY` (`0xc`)
    - doesn't meet minimum security requirements
  - `HTTP_1_1_REQUIRED` (`0xd`)
    - requires HTTP/1.1 instead of HTTP/2
- Unknown or unsupported error codes MUST NOT trigger any special behavior
  - MAY be treated by an implementation as being equivalent to `INTERNAL_ERROR` (? TODO)

      ## 8. HTTP Message Exchanges

- HTTP/2 is intended to be as compatibile with current use of HTTP as possible.

    ### 8.1. HTTP Request/Response Exchange

- A client sends an HTTP request on a new stream. Server sends an HTTP response on the same stream.
- HTTP message (request or response) consists of
    1. (responses only,) 0 or more `HEADERS` frames (+ `CONTINUATION` frames) containing the message headers of 1xx HTTP responses
    2. one `HEADERS` frame (+ `CONTINUATION` frame) containing the message headers
    3. zero or more `DATA` frames containing the payload body
    4. optionally one `HEADERS` frame (+ `CONTINUATION` frame) containing the trailer-part
- Other frames (from any stream) MUST NOT occur between `HEADERS` and `CONTINUATION` frames
- HTTP/2 uses DATA frames to carry message payloads.
- "chucked" transfer encoding MUST NOT be used in HTTP/2
- Trailing _header fields_ are carried in a _header block_, that also terminates the stream.
  - Such a _header block_ is a sequence starting with a `HEADERS` frame (+ `CONTINUATION` frames)
  - TODO: where the `HEADERS` frame bears an `END_STREAM` flag
- `HEADERS` (+ `CONTINUATION`) frames can only appear at the start or end of stream
  - Receiving `HEADERS` fields without the `END_STREAM` flag set after receiving a final (non-informational) status, MUST treat the corresponding request or response as _malformed_
- http request/response exchange, fully consumes a single stream:
  - Request starts with `HEADERS` frame opening stream.
  - Request ends with a frame bearing `END_STREAM`, which causes the stream to `half closed (local)` or `half closed (remote)`
  - Reponse starts with `HEADERS` frame
  - Response ends with a frame bearing `END_STREAM`, which causes the stream to `closed` state
  - HTTP response completes after the server sends or client receives a frame (+ `CONTINUATION`) with `END_STREAM` flag set
- Server can send a complete response prior to the client sending an entire request
  - if response doesn't depend on any portion of the request that not sent.
  - Server MAY request the client to abort transmission without error, by sending `RST_STREAM` with `NO_ERROR`
    - after sending complete response
  - Clients MUST NOT discard responses as a result of such `RST_STREAM`

      #### 8.1.1. Upgrading From HTTP/2

- Semantics in 101 Switchng Protocols aren't applicatable for a multiplexed protocol
- Use Alternative protocols.

    #### 8.1.2. HTTP Header Fields

- _HTTP header fields_ carry information as a series of key-value pairs.
- see the _Message Header Field Registry_ maintained at [4]...?
- header field names are strings of ASCII chars
  - case insensitive
  - MUST be converted to lowercase in HTTP/2
    - violations MUST be treated as _malformed_

        ##### 8.1.2.1. Pseudo-Header Fields

- HTTP/1.1 used the _message start-line_ (e.g. `GET /... HTTP/1.1\r\n`) to tell target URI and method of request
- HTTP/2 uses special pseudo-header fields beginning with `:` char for this purpose
  - these are not _HTTP header fields._
  - Endpoints MUST NOT generate pseudo-header fields not defined in this document.
  - these are only valid in the context in which they're defined
  - MUST NOT appear in requests.
  - MUST NOT appear in trailers.
  - Endpoints MUST treat a _HTTP message_ that contains invalid pseudo-headers, as _malformed._
  - MUST appear in the _header block_ before regular _header fields._

      ##### 8.1.2.2. Connection-Specific Header Fields

- HTTP/2 doesn't use the `Connection` header field, connection-specific metadata is conveyed by other means.
- An endpoint MUST NOT generate an HTTP/2 message containing connectio-specific header fields
  - violation MUST be treated as _malformed._
- The only exception to this is the `TE` _header field,_
  - MAY be present in an HTTP/2 request.
  - when it's present, it MUST NOT contain any value other than `trailers`.
- An intermediary transforming an HTTP/1.x message to HTTP/2 will need to remove any _header fields_ nominated by the `Connection` _header field._ 
  - Intermediaries SHOULD also remove other connection-specific header fields:
    - `Keep-Alive`
    - `Proxy-Connection`
    - `Transfer-Encoding`
    - `Upgrade`

        ##### 8.1.2.3. Request Pseudo-Header Fields

- defined:
  - `:method`
    - HTTP method
  - `:scheme`
    - not restricted to `http` or `https`
    - proxy or gateway can translate requests for non-HTTP schemes
  - `:authority`
    - includes the authority portion of the target URI
    - MUST NOT include the deprecated `userinfo` subcomponent for `http`, `https` URIs
    - MUST be ommitted when translating requests from HTTP/1.1 requests target in origin or asterisk form.
      - to ensure accurate reproducibility of HTTP/1.1 request line
    - Clients can generate HTTP/2 requests directly SHOULD use the `:authority` pseudo-header field instead of `Host` field.
  - `:path`
    - path and query parts of the targer URI
    - asterisk form requests includes value `*` for `:path`
    - MUST NOT be empty for `http` or `https` URIs
      - MUST include a value of `/` for no path
      - MUST include a value of `*` for no path, and if it's OPTIONS requests
- All HTTP/2 requests MUST include one valid value for `:method`, `:scheme` and `:path`
  - unless it's a CONNECT request
  - An HTTP request that omits mandatory pseudo-header fields is _malformed_

      ##### 8.1.2.4. Response Pseudo-Header Fields

- For HTTP/2 responses, only `:status` pseudo-header field is defined
  - carries HTTP status code
  - MUST be included in all response, or be _malformed_
- Note: http/2 doesn't define a way to carry the version/reason phrase

    ##### 8.1.2.5. Compressing the Cookie Header Field

- `Cookie` _header field_ uses `;` for delimiters
- For better compression efficiency, `Cookie` _header field_ MAY be split into separate header fields
- If there are multiple `Cookie` _header fields,_ these MUST be concatenated into a single string using delimiter `; ` before passing into non-HTTP/2 context (e.g. HTTP/1.1)
- the following two is semantically equal:

    ```
    cookie: a=1; b=2; c=3
    ```

    ```
    cookie: a=1
    cookie: b=2
    cookie: c=3
    ```

    ##### 8.1.2.6. Malformed Requests and Responses

- invalid `content-length` _header field_ is also _malformed_
- Intermediaries (that process HTTP requests/responses) MUST NOT forward a _malformed_ request or response.
- _Malformed_ requests or responses that are detected MUST be treated as a _stream error_ `PROTOCOL_ERROR`
- a server MAY send an HTTP response prior to closing/resetting stream
- a client MUST NOT accept a malformed response
- these requirements are intended to protect attacks

    #### 8.1.3. Examples

    TODO

    #### 8.1.4. Request Reliability Mechanisms in HTTP/2

- HTTP/1.1 client is unable to retry a non-idempotent request when an error occurs
- HTTP/2 provies mechanisms for providing a guarantee:
    1. `GOAWAY` indicates the Last-Stream-ID that might have been processed; higher numbers than that are guaranteed to be safe to retry.
    2. `REFUSED_STREAM` error code in `RST_STREAM` frame indicates the stream is being closed prior to any processing.
- clients MAY automatically retry even non-idempotent methods.
- A server MUST NOT indicate that a stream hasn't been processed unless it can guarantee.
  - e.g.
    - frames are passed to the app layer, then `REFUSED_STREAM` MUST NOT be used for that.
    - and `GOAWAY` MUST include a _stream identifier_ that's grater than or equal to such _stream identifier._
- `PING` frames provides a way to test connection.

### 8.2. Server Push

- HTTP/2 allows a server to push responses to a client
- Useful when the server knows the client will need to some responses to fully process the original response
  - HTML Assets?
- A client can disable pushes using `SETTINGS_ENABLE_PUSH` setting
- Promised requests,
  - MUST be cacheable
  - MUST be _safe_ ([RFC 7231: 4.2.1.](https://tools.ietf.org/html/rfc7231#section-4.2.1))
  - MUST NOT include a request body
- Clients MUST reset the promised stream with a _stream error_ `PROTOCOL_ERROR` when received such invalid promised requests.
  - e.g. not known to be safe or indicates the presence of a request body.
- pushed responses are considered successfully validated on the origin
  - while the stream identified by the promised stream ID is still open (? TODO)
- Pushed responses,
  - that aren't cacheable MUST NOT be stored by any cache.
  - MAY be made available to the application separately (? TODO)
- Server MUST include `:authority` _header field_
  - Clients MUST treat a `PUSH_PROMISE` from non-authoritative server as a _stream error_ `PROTOCOL_ERROR`
- An intermediary can receive pushes from server but can choose not to forward to clients
  - Up to Intermediaries
  - intermediaries might choose to make additional pushes to the clients
- A client can't push.
  - Servers MUST treat the receipt of a `PUSH_PROMISE` as a _connection error_ `PROTOCOL_ERROR`.

#### 8.2.1. Push Requests

- Server push is semantically equivalent to a server responding to a request
  - but in this case that request is also sent by the server as a `PUSH_PROMISE` frame.
- `PUSH_PROMISE` frame includes a _header block_, contains a complete set of request _header fields_
- `PUSH_PROMISE` frames sent by server are sent on that explicit requests' stream.
  - (means requires at least one requests from client?)
- The _promised stream identifier_ are chosen from the _stream identifiers_ available to the server
- The _header fields_ in `PUSH_PROMISE` + `CONTINUATION` MUST be valid and complete set of request _header fields_
- The server MUST include a method in `:method` _header field_ that's safe and cacheable.
  - client MUST respond with a _stream error_ `PROTOCOL_ERROR` if the method is not safe, cacheable.
- The server SHOULD send `PUSH_PROMISE`  frames prior to any frames (including `DATA`) that reference the promised responses
  - To avoid a race where clients send same requests to server
  - e.g. Send `PUSH_PROMISE` before the `DATA` contains the image links
- `PUSH_PROMISE` frames MUST NOT be sent by the client.
- Server MUST send `PUSH_PROMISE` to a stream in `open` or `half closed (remote)` state
- `PUSH_PROMISE` frames are interspersed with the frames comprising response, but can't be sent while `HEADERS` + `CONTINUATION` frames are sent.

#### 8.2.2. Push Responses

- after sending `PUSH_PROMISE` frame, the server can begin delivering the _pushed response_ on a server-initiated stream, identified by _promised stream identifier._
- clients SHOULD NOT issue any requests for the promised response (?)
- clients can send `RST_STREAM` with `CANCEL` or `REFUSED_STREAM` for _promised stream_
- clients can use `SETTINGS_MAX_CONCURRENT_STREAMS` to limit the number of concurrent responses pushed by a server.
- clients MUST validate the server is authorative for pushed response
  - or the proxy is configured for the corresponding request
- _Promised responses_ begins with a `HEADER` frame, which immediately puts the _promised stream_ into `half closed (remote)` on server and `half closed (local)` on client.
  - then ends with a frame bearing `END_STREAM` to put it into `closed` state.

### 8.3. The CONNECT Method

- HTTP/1.x CONNECT method is used to convert an HTTP connection into a tunnel, for `https` resources.
- HTTP/2 CONNECT Method is used to establish a tunnel over a single HTTP/2 stream to a remote host for similar purposes.
- HTTP header field mapping works as defined with few difference:

  1. `:method` is set to `CONNECT`
  2. `:scheme`, `:path` MUST be omitted
  3. `:authority` contains the host and port to connect to.

- or considered _malformed._
- proxy that supports CONNECT established a TCP connection to the specified server
- once connection established, proxy sends a `HEADERS` with 2xx status to the client.
- Then all subsequent `DATA` correspond to data sent on the TCP connection.
- Frame types other than `DATA` or stream management frames (`RST_STREAM`, `WINDOW_UPDATE`, `PRIORITY`) MUST NOT be sent on such stream.
  - violation MUST be treated as a _stream error_ (? with what code ?)
- TCP connection can be closed by either peer
  - `END_STREAM` flag on `DATA` is treated as like TCP FIN bit
  - Clients is expected to send `DATA` with `END_STREAM` flag set after receiving `END_STREAM` flag
    - Proxy sends the received data with the FIN bit set
  - Final TCP segment or `DATA` frame could be empty
- TCP connection error is signaled with `RST_STREAM`
  - A proxy MUST send a TCP segment with the RST bit set if proxy detects an error.

## 9. Additional HTTP Requirements/Considerations

### 9.1. Connection Management

- HTTP/2 connections are persistent
  - for best performance, it's expected clients will not close connections until it's determined that no further communication with a server is necessary
  - or server closes a connection.
- Clients SHOULD NOT open more than one HTTP/2 connection to a given host and port pair.
- A client can create additional connections as replacements
  - e.g. when exhausted _stream identifiers_
- A client MAY open multiple connections to the same IP addr and port using different SNI values or to provide different TLS client certs
  - But SHOULD avoid creating multiple connection with the same configuration.

#### 9.1.1. Connection Reuse
