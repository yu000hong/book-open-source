# What is HTTP/2

HTTP/2 is the new version of HTTP (Hypertext Transfer Protocol), superseding HTTP/1.1 with several optimizations designed to improve the speed of web communications. It is fully backwards-compatible with HTTP/1.1 and retains the core HTTP semantics including methods, status codes, URIs, and header fields. HTTP/2 is based on an earlier experimental protocol called `SPDY` which was designed and tested by Google.

Here is a summary of the major changes in HTTP/2:

### FRAMES, STREAMS, AND MULTIPLEXING

Rather than transporting data in plaintext format, data is encoded as binary and encapsulated inside frames which can be multiplexed (or interleaved) along bidirectional channels known as streams – all over a single TCP connection. This allows for many parallel requests/responses to take place concurrently with no `head-of-line blocking` since the frames can be received out of order and automatically reassembled client-side. Multiplexing also eliminates the need for hacky HTTP/1.1 workarounds such as **domain sharding**, **image sprites**, and **file concatenation**.

### STREAM PRIORITIZATION AND DEPENDENCY

Priority levels can be assigned to any stream, allowing the server to allocate more resources to high-priority request/response types. In addition, streams can be considered as parents, siblings, or children of other streams, permitting fine-grained control over the order in which data should be received.

### REDUCED HEADER SIZE

`HPACK` compression is used to reduce request/response header overhead. Compressed headers are also indexed for quick re-use of previously-seen header fields and values.

### FLOW CONTROL

HTTP/2 offers a flow control mechanism to limit and regulate the resource use of any particular stream, or the connection as a whole.

### SERVER PUSH

A completely new feature in which the server can proactively send multiple responses to the client in reply to a single request. Server Push helps reduce page load time by sending all the extra resources needed to render a page (e.g. CSS / JS files) along with the initial response. These resources can then be cached and re-used across different pages on the same domain.

> HTTP/2 is supported by all major web browsers, although it should be noted that most browsers do not support unencrypted (plaintext) HTTP/2 traffic by default.