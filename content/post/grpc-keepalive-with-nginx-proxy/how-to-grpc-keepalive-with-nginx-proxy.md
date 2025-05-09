---
title: "Implementing gRPC Keepalive with Nginx Proxy"
date: 2025-05-09
description: "A deep dive into implementing a custom keepalive mechanism for gRPC server streams when working with Nginx proxy"
tags: 
    - grpc
    - go
    - nginx
    - networking
    - backend
categories:
    - backend
    - networking
---

## The Challenge

In a recent project, I faced an interesting challenge: implementing a gRPC server stream that needed to maintain a healthy connection. This was particularly important because our client was a mobile application that could experience poor signal quality, intermittent data loss, and other connectivity issues. The natural approach would be to use gRPC's built-in keepalive mechanism, but there was a catch - we were using Nginx as a proxy.

## Understanding gRPC Keepalive

Before diving into the problem, let's understand how gRPC keepalive works. The official documentation can be found in the [gRPC keepalive guide](https://grpc.io/docs/guides/keepalive/).

TCP keepalive is a well-known method of maintaining connections and detecting broken connections. When TCP keepalive is enabled, either side of the connection can send redundant packets. Once ACKed by the other side, the connection will be considered as good. If no ACK is received after repeated attempts, the connection is deemed broken.

Unlike TCP keepalive, gRPC uses HTTP/2 which provides a mandatory PING frame (specified in [RFC 7540](https://httpwg.org/specs/rfc7540.html#PING)) that can be used to:
- Estimate round-trip time
- Measure bandwidth-delay product
- Test the connection health

The interval and retry mechanisms in TCP keepalive don't quite apply to HTTP/2 PING frames because the transport is reliable. Instead, they're replaced with a timeout value (equivalent to interval * retry) in gRPC's PING-based keepalive implementation.

## Hands-on with gRPC Keepalive

Let's explore how gRPC keepalive works in practice using a simple example. First, we'll enable keepalive on the client side:

```kotlin
private val channel = ManagedChannelBuilder
    .forAddress(SERVER_IP, SERVER_PORT)
    .usePlaintext()
    // Send ping frames every 10 seconds
    .keepAliveTime(10, TimeUnit.SECONDS)
    // Server should respond to ping in 15 seconds max
    .keepAliveTimeout(15, TimeUnit.SECONDS)
    .build()
```

To see the HTTP/2 frames in action, we can run our server with debug logging enabled:

```bash
env GODEBUG=http2debug=2 go run main.go
```

When we connect our client and start the stream, we'll see the following in the logs:

```bash
2025/05/09 19:06:53 http2: Framer 0x1400018a540: read PING len=8 ping="\xa1\x97GVkb\xb4|"
2025/05/09 19:06:53 http2: Framer 0x1400018a540: wrote PING flags=ACK len=8 ping="\xa1\x97GVkb\xb4|"
2025/05/09 19:07:03 http2: Framer 0x1400018a540: read PING len=8 ping="\xd9L\xad>&\b\xb8\x86"
2025/05/09 19:07:03 http2: Framer 0x1400018a540: wrote PING flags=ACK len=8 ping="\xd9L\xad>&\b\xb8\x86"
2025/05/09 19:07:13 http2: Framer 0x1400018a540: read PING len=8 ping="\xd84|3G6|\x1a"
2025/05/09 19:07:13 http2: Framer 0x1400018a540: wrote PING flags=ACK len=8 ping="\xd84|3G6|\x1a"
2025/05/09 19:07:23 http2: Framer 0x1400018a540: read PING len=8 ping="\xd4\xdeڬ\x858\x89\x82"
2025/05/09 19:07:23 http2: Framer 0x1400018a540: wrote PING flags=ACK len=8 ping="\xd4\xdeڬ\x858\x89\x82"
2025/05/09 19:07:23 http2: Framer 0x1400018a540: wrote GOAWAY len=22 LastStreamID=3 ErrCode=ENHANCE_YOUR_CALM Debug="too_many_pings"
2025/05/09 19:07:24 Context cancelled
2025/05/09 19:07:24 Stream ended
```

As we can see, the client is sending ping frames every 10 seconds (note: this is the minimum allowed for the Java client). Even though we haven't enabled keepalive on our server, it's responding with PING frames with the ACK flag. However, after a few pings, it sends a GOAWAY frame with the code `ENHANCE_YOUR_CALM`. This happens because the server isn't configured to accept ping frames, so it terminates the connection.

Let's enable keepalive on the server side as well:

```go
s := grpc.NewServer(grpc.KeepaliveParams(keepalive.ServerParameters{
    Time:    10 * time.Second,
    Timeout: 15 * time.Second,
}))
```

Now the logs show bidirectional ping communication:

```bash
2025/05/09 19:24:30 http2: Framer 0x14000200540: wrote PING len=8 ping="\x00\x00\x00\x00\x00\x00\x00\x00"
2025/05/09 19:24:30 http2: Framer 0x14000200540: read PING flags=ACK len=8 ping="\x00\x00\x00\x00\x00\x00\x00\x00"
2025/05/09 19:24:40 http2: Framer 0x14000200540: wrote PING len=8 ping="\x00\x00\x00\x00\x00\x00\x00\x00"
2025/05/09 19:24:40 http2: Framer 0x14000200540: read PING len=8 ping="\xa6\x9c#UD[\x83\xd0"
2025/05/09 19:24:40 http2: Framer 0x14000200540: wrote PING flags=ACK len=8 ping="\xa6\x9c#UD[\x83\xd0"
2025/05/09 19:24:40 http2: Framer 0x14000200540: read PING flags=ACK len=8 ping="\x00\x00\x00\x00\x00\x00\x00\x00"
```

We can see both "wrote PING", "wrote PING ACK", "read PING", and "read PING ACK" messages, indicating that both sides are actively sending pings to each other. This is exactly what we want!

### Testing Connection Loss

Let's see what happens when the Android app loses connection. We can simulate this by enabling airplane mode:

```bash
2025/05/09 19:28:07 Received request: Hello Server!
2025/05/09 19:28:07 http2: Framer 0x14000200540: wrote PING len=8 ping="\x02\x04\x10\x10\t\x0e\a\a"
2025/05/09 19:28:07 http2: Framer 0x14000200540: read PING flags=ACK len=8 ping="\x02\x04\x10\x10\t\x0e\a\a"

(at this point, I enabled airplane mode in the android emulator)...

2025/05/09 19:28:17 http2: Framer 0x14000200540: wrote PING len=8 ping="\x00\x00\x00\x00\x00\x00\x00\x00"
2025/05/09 19:28:32 Context cancelled
2025/05/09 19:28:32 Stream ended
```

The server sends a PING frame at 19:28:17 to check the connection. After 15 seconds (our configured keepalive timeout) with no response, the server terminates the connection. This is exactly the behavior we want!

### The Nginx Surprise

After successfully testing locally, I deployed the solution to our test environment. We use Ingress Nginx as our reverse proxy for backend services, which provides load balancing, security, rate limiting, and other features. However, when testing the keepalive mechanism in this environment, something unexpected happened.

Even with airplane mode enabled on the phone, **the server was still receiving ACK responses to its PING frames. This was confusing because we knew the client was disconnected, but the server thought the connection was still alive.**

<p align="center">
<img alt="WTF meme" src="https://media2.giphy.com/media/v1.Y2lkPTc5MGI3NjExZHY4OHlsZTR4Z3BpeWltaThtenN1NHYzb2kxb2hocDZ3aG5vMm43cSZlcD12MV9pbnRlcm5hbF9naWZfYnlfaWQmY3Q9Zw/b8RQzkElbBsXqEPF2X/giphy.gif"
</p>

## The Problem

The standard gRPC keepalive mechanism relies on HTTP/2 ping frames to maintain connection health. However, when using Nginx as a proxy, these ping frames don't work as expected. Nginx acknowledges (ACKs) these ping frames instead of forwarding them, effectively breaking the keepalive mechanism. This creates a situation where:

1. The client sends HTTP/2 ping frames
2. Nginx receives and acknowledges these pings
3. The actual gRPC server never sees these pings
4. The connection health check fails

Since the gRPC keepalive can be enabled on the server side as well, it creates an opposite situation:
1. The client connection is lost
2. The server sends keepalive pings to the (supposedly) client
3. Nginx acknowledges these pings
4. The server thinks the connection is still alive and keeps it open

## The Solution

To work around this limitation, I implemented a custom "home-made" ping mechanism using regular gRPC messages. This approach:

1. Uses standard gRPC messages instead of HTTP/2 ping frames
2. Can be properly proxied by Nginx
3. Maintains the same functionality as the built-in keepalive mechanism

## Implementation Details

I've created a demonstration repository that shows how to implement this custom keepalive mechanism: [nginx-hates-grpc-keepalive](https://github.com/hectorgabucio/nginx-hates-grpc-keepalive).

The custom ping mechanism works as follows:

1. **Client-side behavior:**
   - Sends frequent ping messages to the server
   - Expects a ping reply from the server
   - Terminates the connection if no reply is received within a reasonable timeframe

2. **Server-side behavior:**
   - Monitors incoming ping messages from the client
   - Terminates the connection if no pings are received within the expected interval

This bidirectional ping mechanism ensures both sides can detect connection issues and take appropriate action.

## Best Practices

When implementing custom keepalive mechanisms for gRPC with Nginx:

1. Always test the connection health monitoring
2. Consider the impact on your message throughput
3. Monitor the additional overhead of custom ping messages
4. Document the workaround for other team members
5. **Security Considerations:**
   - Protect the stream with authentication tokens to prevent unauthorized access
   - Implement rate limiting on the backend to protect against potential DoS attacks from misbehaving clients
   - Consider the impact of ping frequency on your server resources

## Conclusion

While not ideal, this workaround provides a reliable solution for maintaining healthy gRPC connections through Nginx proxies. It's a good example of how sometimes we need to think outside the box when working with complex infrastructure setups, especially when dealing with mobile clients that may have unreliable connections.

For a complete implementation example, check out the [nginx-hates-grpc-keepalive](https://github.com/hectorgabucio/nginx-hates-grpc-keepalive) repository.