# High Performance Browser Networking

## Latency & Bandwidth

- `traceroute`
- **Wavelength-Division Multiplexing (WDM)**
  - optical fibers

## TCP

- **three-way handshake**
  - makes creating new connection expensive
  - **connection reuse** is critical
- **TCP Fast Open (TFO)**
  - allows data transfer within the `SYN` packet
- **congestion collapse**

### Flow Control

- prevents the sender from overwhelming the receiver with data it may not be able to process
- Approach:
  - each side advertises its own **receive window (rwnd)** _within_ the `ACK` packets
    - size of the available buffer space to hold the incoming data
- originally 16 bits allocated for the **rwnd** size
- **TCP window scaling** - now it's up to 1Gb
  - `$ sysctl net.ipv4.tcp_window_scaling`
  - `$ sysctl -w net.ipv4.tcp_window_scaling=1`

### Slow Start

- estimates the available capacity between the client/server by exchanging data
- Steps:
  1. server initializes a new congestion window (**cwnd**) variable per TCP connection, initialized to a conservative, system-specified value (`initcwnd` on Linux)
  2. new rule introduced: max amount of data in flight (not ACKed) is `min(rwnd, cwnd)`
  3. start slow and grow the window size as the packets are ACKed
- **cwnd**
  - sender-side limit on the amount of data the sender can have _in flight_ _before_ receiving an `ACK` from the client
  - _NOT_ advertised/exchanged
  - private variable maintained by server
  - by default 4 segments
  - 10 segments (**IW10**) in latest [RFC 6928](https://datatracker.ietf.org/doc/html/rfc6928)
- For every received `ACK` packet (roundtrip), _two_ new packets can be sent
- _NOT_ ideal for _short_ and _bursty_ connections
  - request terminated _before_ max window size reached
- **Slow-Start Restart**
  - resets the congestion window of a connection after idling for a while
  - big impact on performance for long-lived TCP connections
  - HTTP keepalive connections recommended to disable SST on the server
  - `$ sysctl net.ipv4.tcp_slow_start_after_idle`
  - `$ sysctl -w net.ipv4.tcp_slow_start_after_idle=0`

### Congestion Avoidance

- the **congestion avoidance window** takes over when the amount of data in flight exceeds:
  - the receiver's flow-control window, or
  - system-configured congestion threshold (**ssthresh**) window, or
  - until a packet is lost
- Algorithms:
  - **Multiplicative Decrease and Additive Increase (AIMD)**
    - too conervative
  - **Proportional Rate Reduction (PRR)**
    - improves the speed of recovery
    - default in Linux 3.2+ kernel

### Bandwidth-Delay Product

- product of **data link's capacity** and its **end-to-end delay**
- max amount of unacknowledged data that can be in flight at any point in time
- If the connection is transmitting at a fraction of the available bandwidth,
  - likely due to a _small_ window size:
    - a saturated peer advertising low receive window
    - bad network weather and high packet loss _resetting_ the congestion window
    - explicit traffic shaping

### Head-of-Line Blocking

- makes sure packets arrive in order
- happens within TCP layer
- the application has no visibility
  - it only sees a delivery delay

> Packet loss is OK.

> No bit is faster than one that is not sent; send fewer bits

> We can't make bits travel faster; but we can move the bits closer

> TCP connection reuse is critical to improve performance

## UDP

- **Datagram**
  - vs. packet
- used by **DNS**
- **WebRTC**
- It's not IP's responsibility to guarantee delivery
- UDP datagrams have definitive boundaries
  - each datagram is carried in a single IP packet
  - each application read yields the full message
  - datagrams _CANNOT_ be fragmented
- **NAT**
  - intermediate timeouts for UDP, even for TCP sometimes
- No handshake
- No connection termination
- Has **NAT Traversal problems**

## TLS

- designed to work on top of a _reliable_ transport protocol
  - TCP
  - but over UDP is possible - DTLS
- TLS provides three essential services:
  - **encryption**
  - **authentication**
  - **data integrity**
- **public (asymmetric) key cryptography**
- **Message Authentication Code (MAC)**
  - one-way cryptographic hash function (checksum)

### TLS Handshake

- Things to do:
  - agree on the version of TLS
  - choose the ciphersuite
  - verify certificates
- `ClientHello` and `ServerHello` are in plain text
- By default the handshake requires _two_ roundtrips to complete
  - to optimize, **session resumption** and **false start**
- use **Diffie-Hellman** key exchange
  - client and server negotiate a shared secrete without explicitly communicating it in the handshake
- **Forward Secrecy**: Diffie-Hellman + ephemeral session keys
- **ALPN (Application Layer Protocol Negotiation)**
  - TLS extension
  - introduces support for application protocol negotiation into TLS handshake
  - client puts `ProtocolNameList` in `ClientHello`
  - server puts `ProtocolName` in `ServerHello`
- **SNI (Server Name Indication)**
  - part of the handshake
  - TLS + SNI == `Host` header in HTTP

### TLS Session Resumption

- resume/share the _same_ negotiated secret key between multiple connections

#### Session Identifiers - Session Caching

- server creates/sends a 32-byte session identifier as part of `ServerHello`
  - keeps it in cache
  - maintains a session cache for _every_ client
- afterwards, client could store the session ID in `ClientHello` for a _subsequent_ session
- removes a round trip
- Most modern browsers intentionally _wait_ for the first TLS connection to complete _before_ opening new connections to the _same_ server
  - subsequent TLS connections can reuse the SSL session parameters to avoid the costly handshake
- Requires _careful thinking_ for multi-server deployment

#### Session Tickets - Stateless Resumption

- removes the requirement for server to keep per-client session state
- server creates a **New Session Ticket** record, encrypted by secret key on server's side
- session ticket stored on _client only_
  - included in `SessionTicket` _extension_ within `ClientHello` of a _subsequent_ session
- all session data stored on client
- Usecase: deploying session tickets across a set of load-balanced servers
  - rotating the shared key across all servers periodically

### Chain of Trust and Certificate Authorities

- Root CA
- **Certificate Revocation List (CRL)**

#### Online Certificate Status Protocol (OCSP)

- check for status of the certificate in real-time

### TLS Record Protocol

- For id-ing different types of messages via the **Content Type** field
  - handshake
  - alert
  - data
- Max size 16Kb
- Small records incur a larger overhead due to record framing
- Large records will have to be delivered and reassembled by TCP layer _before_ they can be processed/delivered

### Optimizing for TLS

- The operational pieces for TLS deployment:
  - how/where the servers are deployed
  - size of TLS record
  - size of memory buffers
  - size of certificate
  - support for abbreviated handshakes

#### Early Termination

- up to three roundtrip to set up TCP+TLS session
- place servers closer to the user!
- nearby server establishes a pool of long-lived, secure connections to the origin servers
  - proxy all incoming requests/responses to/from origin servers
- CDN (or proxy server) can maintain a "warm connection pool" to relay data to origin servers

#### Key Points

- servers with multiple processes/workers should use a shared session cache
- In a multi-server setup, routing the same client IP or the same TLS session ID to the _same_ server is one way to provide good _session cache utilization_
- A shared cache should be used between different servers (where "sticky" load balancing is not an option)
  - secure mechanism needed to share/update secret keys to decrypt the provided session tickets

#### TLS False Start

- _One_ roundtrip handshake for new and repeat visitors
- optional protocol extension
- allows sender to send application data when the handshake is only _partially_ complete
- application data sent alongside `ClientKeyExchange` record
- _only_ affects the protocol timing of when the application data can be sent

#### TLS Record Size

- max of each record is 16Kb
- 20 - 40 bytes of overhead for MAC, padding, etc
- IP and TCP overhead (if record can fit into a single TCP packet)
  - 20-byte header for IP
  - 20-byte header for TCP
- The smaller the record, the higher the framing overhead
- _NOT_ necessarily a good idea to increase record size to max 16KB
  - if record spans multiple TCP packets, TLS _must_ wait for _all_ TCP packets to arrive,
  - _before_ decrypting the data
  - additional latency!
- Small records incur overhead; large records incur latency
- For web applications (consumed by the browser): dynamically adjust record size based on TCP connection state
  - when
    - connection is new and TCP congestion window is low, or
    - connection been idle for some time (**Slow-Start Restart**)
    - each TCP packet should carry exactly one TLS record, with max segment size (**MSS**) allocated by TCP
  - when
    - connection congestion window is large and large stream is being transferred
    - size of TCP record can be increased to span multiple TCP packets (up to 16KB) to reduce framing and CPU overhead on the client and server
- Goal - to minimize buffering at the application layer due to lost packets, reordering, and retransmissions
  - if TCP connection been idle
  - if slow-start restart is disabled on server
  - best strategy: decrease record size when sending a new burst of data
- Small record eliminates unnecessary buffering latency and improves time-to-first-{HTML byte, .., video frame}
- Larger record optimizes throughput by minimizing overhead of TLS for long-lived streams
- Typical strategy:
  - increase record size to up to 16KB after `X` KB of data transferred
  - reset record size after `Y` milliseconds of idle time

#### TLS Compression

- support for lossless compression of data transferred within record protocol
- compression algo negotiated during TLS handshake
- compression applied _prior_ to encryption of each record
- _HOWEVER_, should be _DISABLED_ on server!
- Server should be configured to Gzip all text-based assets

#### Certificate-Chain Length

- Should verify server does not forget to include al intermediate certificates during handshake
  - otherwise browser has to verify itself
  - a new DNS lookup, TCP connection, HTTP GET request
- Minimize the size of certificate chain
- Ideally the sent certificate chain should contain exactly _two_ certificates:
  - the site's certificate
  - the CA's intermediary certificate
- NO NEED for the site to include root certificate of their CA

#### OCSP Stapling

- browser needs to check the certificate is not _revoked_
- an OCSP request during the verification process for "real-time" check
- **OCSP Stapling**
  - server includes (staples) the OCSP response from the CA to its certificate chain
  - so that browser can skip the online check
  - server can _cache_ the signed OCSP response

#### HTTP Strict Transport Security (HSTS)

- a security policy mechanism that allows server to declare rules to (compliant) browsers
  - via HTTP header
  - `Strict-Transport-Security: max-age=31536000`

#### Performance Checklist

- Enable/configure session caching and stateless resumption
- Monitor session caching hit rates
- Configure forward secrecy ciphers to enable TLS False Start
- Terminate TLS session closer to the user to minimize roundtrip latencies
- Use dynamic TLS record sizing
- Ensure your certificate chain does not overflow the initial congestion window
- Remove unnecessary certificates from chain; minimize the depth
- Configure OCSP on server
- Disable TLS compression on server
- Configure SNI support on server
- Append **HSTS** header

## HTTP History

- client sends header `Connection: close` to terminate a persistent connection
- Technically, _either side_ can terminate the connection
- For HTTP/1.1, `Connection: Keep-Alive` is _not_ needed

## Web Performance

- **Plage Load Time (PLT)**
  - time to onload event in the browser
  - fired by browser once the document and all its dependent resources (JavaScript, images, etc) have finished loading

### DOM, CSSOM, and JavaScript

- HTML document parsed -> DOM
- CSS parsed -> CSSOM
- DOM + CSSOM -> RenderTree
- JavaScript can _block_ both DOM and CSSOM
- Construction of DOM and CSSOM is _interwined_
  - DOM construction cannot proceed until JS is executed
  - JS execution cannot proceed untill CSSOM is available
  - that's why styles are put at top and scripts at the bottom!

### Speed, Performance, Human Perception

- Less than 100ms: instant
- Less than 20ms to keep users engaged
- HTML parsing is performed _incrementally_
- For many requests, response times are often dominated by:
  - roundtrip latency
  - server processing time
- For _most_ web applications,
  - bandwidth is _not_ the limiting performance factor
  - the real bottleneck is the network roundtrip latency between client and server

### Performance Pillars: Computing, Rendering, Networking

- Three tasks of a web program:
  - fetching resources
  - page layout
  - JavaScript execution
- Rendering & scripting
  - single-threaded
  - interleaved model of execution
- bandwidth limited vs. latency limited
- Number of roundtrips is largely due to handshakes to start communicating between client & server:
  - DNS, TCP, HTTP
  - TCP slow start

### Synthetic & Real-User Performance Measurement

- **Navigation Timing**
- **User Timing**
- **Resource Timing**
- via `performance.timing`
- When analyzing performance data, look at the underlying distribution
  - _NOT_ the averages
  - look at histograms, medians, and quantiles

### Browser Optimization

- Two broad classes:
  - **Document-aware optimization**
    - resource priority assignments
    - lookahead parsing
  - **Speculate optimization**
    - pre-resolving DNS names
    - pre-connecting to likely hostnames
- **Resource pre-fetching and prioritization**
  - document/CSS/JS parsers
  - blocking resources required for first rendering are given high priority
- **DNS pre-resolve**
  - likely hostnames pre-resolved ahead of time to avoid DNS latency on a future HTTP request
  - triggered through:
    - navigation history
    - user action
      - hovering over a link
- **TCP pre-connect**
  - _following_ a DNS resolution,
  - browser _may_ speculatively open the TCP connection in an anticipation of an HTTP request
- **Page pre-rendering**
  - allows user to hint the likely next destination
  - pre-renders the entire page in a hidden tab
- How can we (developers) help?
  - Critical resources (CSS/JS) should be discovered _as early as possible_ in the document
  - CSS should be delivered _as early as possible_ to unblock rendering and JS execution
  - Non-critical JS should be _deferred_ to avoid blocking DOM/CSSOM construction
  - HTML document parsed _incrementally_ by the parser - document should be _periodically flushed_ for best performance
  - Hints for browser for additional optimizations:
    - `<link rel="dns-prefetch" href="//hostname_to_resolve.com">`
    - `<link rel="subresource" href="//javascript/myapp.js">`
    - `<link rel="prefetch" href="//images/big.jpeg">`
    - `<link rel="prerender" href="//example.org/next_page.html">`

## HTTP/1.X

### Compared with HTTP/1.0

- Improvements over HTTP/1.0:
  - persistent connections to allow connection reuse
  - chunked transfer encoding to allow response streaming
  - request pipelining to allow parallel request processing
  - byte serving to allow range-based resource requests
  - improved (much better-specified) caching mechanisms
- Networking optimizations:
  - reduce DNS lookups
  - Add an **Expires** header and configure **ETags**
  - Gzip assets
    - _all_ text-based assets should be compressed with gzipped
  - Avoid HTTP redirects
    - especially redirecting to a different hostname - additional DNS lookup, TCP connection latency, etc

### Keepalive Connections

- a.k.a. persistent connection
- enabled by default on HTTP/1.1
- `Connection: Keep-Alive`
- a _strict_ FIFO queuing order on the client:
  - dispatch request
  - wait for full response
  - dispatch _next_ request from the client queue

### HTTP Pipelining

- Allows for relocating the FIFO queue from the client (request queuing) to the server (response queuing)
- server _could_ process the requests in parallel
- _However_, data from multiple responses _CANNOT_ be interleaved (multiplexed) on the same connection,
  - forcing each response to be returned in full before the bytes for the next response can be transferred
- **Head-of-line** blocking
- When processing requests in parallel, server must _buffer_ pipelined responses, which may exhaust server resources
- A failed response may _terminate_ the TCP connection, forcing client to re-request all subsequent resources
- Not widely enabled

### Using multiple TCP connections

- Up to **6** connections _per host_
  - _NOTICE_: per host!
- Reasons to use:
  - as workaround for limitations of HTTP
  - as workaround for low starting congestion window size in TCP
  - as workaround for clients that cannot use TCP window scaling

### Domain Sharding

- increase browser's connection limit
- more shards, higher parallelism
- _HOWEVER_, every new hostname:
  - requires additional DNS lookup
  - consumes additional resources on both sides for each additional socket
- In practice, different shards could resolve to the same IP
  - they are CNAME DNS records
  - the browser connection limits are enforced on hostnames, _NOT_ IP
- What affects the optimal number of shards?
  - number/size/response-time of each resource
  - client latency & bandwidth

### Measuring/Controlling Protocol Overhead

- Each browser-initiated HTTP request carries at least an additional 500-800 bytes of HTTP metadata
  - worse - cookies
- HTTP/1.1 does _not_ define size limit of HTTP headers,
  - but 8KB or 16KB limit widely adopted

### Concatenation and Spriting

- Bundle multiple resources into a single network request
- **Concatenation**: multiple JS/CSS files combined into a single resource
- **Spriting**: multiple images combined
- Application-level pipelining
- _NOT_ friendly to caching!
  - a single update to any one individual file invalidates the cache!
- Increased memory usage!
  - all decoded images are stored as memory-backed RGBA bitmaps within browser
  - one byte for each of the RGBA
  - 4 bytes for each pixel
- Affecting execution
  - JS and CSS parsing & execution is held back _until_ entire file is downloaded
  - no incremental execution!
- What is the _ideal_ size for a CSS/JS file?
  - probably 30 - 50 KB (compressed)
- Some places considered worth optimizing:
  - separating and delivering first-paint critical CSS from the rest of CSS
  - separating and delivering smaller JS chunks for incremental execution

### Resource Inlining

- Embed the resource within document itself
- using the data URI theme
- `<img src="data:image/gif;base64,xxxxxxxx" alt="sample image" />`
- Rule of thumb:
  - inline resources under 1 - 2 KB
  - probably _NOT_ for frequently changed resources
- Considerations:
  - if files are small and limited to specific pages, consider inlining
  - if the small files are frequently reused across pages, consider _bundling_
  - if the small files have high update frequency, keep them separate
  - Minimize the protocol overhead by reducing the size of HTTP cookies

## HTTP/2

### Design

- **Binary Framing Layer**
- **frame** - smallest unit of communication in HTTP/2
  - contains frame header
  - may be interleaved across different streams
  - reassembled via the embedded stream identifier in the header of each frame
- **stream** - bidirectional flow of bytes within an established connection
  - carrying one or more messages
- **message** - a complete sequence of frames that map to a logical request/response
- _ALL_ communication performed over a single TCP connection that can carry any number of bidirectional streams
- Each stream has:
  - a unique identifier
  - optional priority information

#### Request and Response Multiplexing

- HTTP/2 enables full request/response multiplexing

#### Stream Prioritization

- Each stream may be assigned an integer weight between `1` and `256`
- Each stream may be given an explicit dependency on another stream
- **prioritization tree**
  - dependency: referencing the other stream's ID as parent
  - streams do not depend on each other have a implicit _root_ stream
  - _if possible_, parent stream should be allocated resources ahead of its dependencies
    - e.g. deliver response `<parent>` before response `<child>`
- streams with the same parent (siblings) should be allocated resources in proportion to their weight
- client is allowed to update preferences (dependencies & weights) at any point
- Browser request prioritization
  - can learn priority from previous visits
  - if rendering was blocked on a certain asset in a previous visit, then
    - the same asset _may be_ prioritized higher in the future

#### One Connection Per Origin

- HTTP/2 connections are persistent
- only one connection per origin

#### Downsides and Tradeoffs

- using one TCP connection per origin:
  - still head-of-line blocking at TCP level
  - when packet loss occurs, TCP congestion window size is reduced
    - reduces max throughput of the entire connection
  - effects of bandwidth-delay product may limit connection throughput if TCP window scaling is disabled
- What if multiple connections per origin?
  - less effective header compression due to distinct compression contexts
  - less effective request prioritization due to distinct TCP streams
  - less effective utilization of each TCP stream
  - higher likelihood of congestion due to more competing flows
  - increased resource overhead due to more TCP flows

### Flow Control

- flow control is _directional_
  - each receiver may choose to set any window size for each stream and the entire connection
- flow control is credit-based
  - each receiver advertises initial connection and stream flow control window (bytes)
  - window reduced whenever sender emits a `DATA` frame
  - incremented via `WINDOW_UPDATE` frame sent by receiver
- flow control _CANNOT_ be disabled
  - when HTTP/2 connection is established, `SETTINGS` frames exchanged between client and server
  - set flow control window sizes in _both_ directions
  - default value for **flow control window** is `65535` bytes
  - maintained by `WINDOW_UPDATE` frame whenever any data received
- flow control is _hop-by-hop_

#### Server Push

- server can send _multiple_ responses for a single client request
- one-to-many
- server-initiated push
- server initiates new streams (**promises**) for push resources
  - NOTE these are _NEW_ streams!
- Inlining resources is actually server push
- Pushed resources can be:
  - cached by client
  - reused across different pages
  - multiplexed alongside other resources
  - prioritized by server
  - declined by client
- Security policy:
  - pushed resources must obey **same-origin**: server must be authoritative for the provided content
- `PUSH_PROMISE`
  - all server push streams are initiated with `PUSH_PROMISE` frames
  - delivery order is critical!
    - client needs to know which resources the server intends to push
    - to solve this, server sends _all_ `PUSH_PROMISE` frames, which contain just the HTTP headers of the promised resource
      - ahead of the parent's response (i.e. `DATA` frames)
  - clients can reject using `RST_STREAM`

### Header Compression

- request/response headers compressed using **HPACK** compression format
- Allows individual values to be compressed when transferred,
  - the indexed list of previously transferred values allows for encoding duplicate values by
    - transferring index values that can be used to
    - efficiently look up and reconstruct the full header keys and values
- **static table** & **dynamic table**
  - static table
    - defined in specification
    - provides list of common HTTP header fields
  - dynamic table
    - initially empty
    - updated based on exchanged values within a particular connection
- To negotiate HTTP/2 protocol
  - TLS & ALPN is recommended
  - client & server negotiate the desired protocol as part of TLS handshake
  - without adding extra latency/roundtrips
- Establishing HTTP/2 connection over a regular, non-encrypted channel is still _possible_
  - use **HTTP `Upgrade`**
  - `Connection: Upgrade, HTTP2-Settings`
  - `Upgrade: h2c` - initial HTTP/1.1 request with HTTP/2 upgrade header
  - server could:
    - reject by returning `HTTP/1.1 200 OK`
    - accept by `HTTP1.1 101 Switching Protocols`
      - then immediately switch to HTTP/2
      - return response using the new binary framing protocol

### Binary Framing

- **frames** exchanged between client & server
- all frames have a 9-byte header
  - 24-bit length field
    - theoretical 2^24 bytes (16MB) data per frame
    - _but_ default is 2^14 bytes (16KB)
    - bigger is _NOT_ always better!
  - 8-bit type field
    - format
    - semantics
  - 8-bit flag field
  - 1-bit reserve field - always set to 0
  - 31-bit stream identifier, uniquely identifying the HTTP/2 stream

### Initiating a New Stream

- client initiates by sending a `HEADERS` frame
- payload in `DATA` frames
- flow control is applied _only_ to `DATA` frames
- non-`DATA` frames always processed with high priority!
- To eliminate stream ID collisions between client- and server- initiated streams:
  - client-initiated streams have odd IDs
  - server-initiated streams have even IDs
- payload can be split into multiple `DATA` frames
  - last framing contains `END_STREAM` indicating end of the message

## Optimizing Application Delivery

### Best Practices

- Reduce DNS lookups
- Reuse TCP connections
  - keepalive wherever possible
- Minimize number of HTTP redirects
  - redirect to a different origin can result in DNS, TCP, TLS roundtrips
- Reduce roundtrip times
- Eliminate unnecessary resources
- Cache resources on the client
- Compress assets during transfer
- Eliminate unnecessary request bytes
  - HTTP cookies
- Parallelize request/response processing
- Apply protocol-specific optimizations

### Cache Resources on Client

- You should specify _both_:
  - `Cache-Control` - cache lifetime
  - `Last-Modified` & `ETag` - validation

### Compress Transferred Data

- gzip

### Eliminate unnecessary request bytes

- **HTTP State Management Mechanism**
  - extension to HTTP
  - allows for cookies
  - saved by browser
  - auto appended onto every request to the origin within the `Cookie` header
- allowed to associate many cookies per origin
- Best practices:
  - Transfer the min amount of required data (e.g. secure session token)
  - leverage **shared session cache** on server to lookup other metadata

### Parallelize request/response processing

- without `Keep-Alive`, a new TCP connection is required for each HTTP request

### Optimizing Resource loading in Browser

- browser uses a **preload scanner**
  - _only when_ the document parser is blocked
  - _HOWEVER_, _NOT_ applicable for resources scheduled via JS
  - it cannot speculatively execute scripts

### Optimizing for HTTP/1.X

- Leverage HTTP pipelining
  - if, you control both client and server
- Domain sharding
- Bundle resources to reduce HTTP requests
- Inline small resources

### Optimizing for HTTP/2

- requests are cheap
- both requests & responses can be multiplexed efficiently
- **Domain sharding** is _anti pattern_!
- HTTP/2 has a **connection-coalescing** mechanism
  - allows client to
    - coalesce requests from different origins
    - then dispatch them over _the same_ connection when the following conditions are satisfied:
  - origins are covered by same TLS certificate
    - wildcard certificate, or
    - certificate with matching **Subject Alternative Names**
  - origins resolve to the same server IP address

### Eliminate Roundtrips with Server Push

- resource inlining is a form of application-layer server push
- with HTTP/2, no longer a reason to inline resources _just because_ they are small
- more of latency optimization
- prime candidates:
  - critical resources that block page construction
- Client can control how/where server push is used, by indicating to server:
  - the max number of pushed streams initiated by server
  - amount of data can be sent on each stream before acknowledged by client
- server push subject to same-origin restrictions
- server can learn from `Referrer` headers
  - and auto initiate server push for related resources
- A well-implemented server should give precedence to high priority streams, but
  - should also interleave lower priority streams if all higher priority streams are blocked (head-of-line blocking)

## Primer on Browser Networking

### Connection Management & Optimization

- Browser _intentionally_ separates **request management** lifecycle from **socket management**
- sockets are organized in **pools**
  - grouped by origin
  - same sockets can be automatically reused across multiple requests
- **Origin** - the combination of:
  - application protocol
  - domain name
  - port number
- **Socket pool**
  - group of sockets belonging to _the same_ origin
  - in practice, all browsers limit max pool size to 6 sockets
- With **Automatic socket pooling**, browser can:
  - service queued requests in priority order
  - reuse sockets to minimize latency and improve throughput
  - be proactive in opening sockets in anticipation of request
  - optimize when idle sockets are closed
  - optimize bandwidth allocation across all sockets
- ^^^ these are managed by the browser!!!

### Network Security and Sandboxing

- Defer management of individual sockets:
  - allows browser to sandbox and enforce a consistent set of security and policy constraints on untrusted code
- **Connection limits**
  - browser manages all open socket pools
  - browser enforces connection limits to protect both client and server
- **Request formatting & response processing**
- **TLS negotiation**
- **Same-origin policy**

### Resource and Client State Caching

- Browser provides authentication, session, and cookie management
- browser maintains _separate_ "cookie jars" for each origin,
