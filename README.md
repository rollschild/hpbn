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

-
