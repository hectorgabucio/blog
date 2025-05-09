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


## Playing with gRPC keepalive

Let's use our toy project to understand more about the keepalive future.
First, let's enable the keepalive mechanism in the client side

```diff
    private val channel = ManagedChannelBuilder
      .forAddress(SERVER_IP, SERVER_PORT)
        .usePlaintext()
+              // send ping frames every 10 seconds
+            .keepAliveTime(10, TimeUnit.SECONDS)
+              // server should respond to ping in 15 seconds max
+            .keepAliveTimeout(15, TimeUnit.SECONDS)
        .build()
```

Now, let's run our server with the env variable needed to debug HTTP2 traces

```bash
env GODEBUG=http2debug=2 go run main.go
```

If we connect our client and start the stream, we will see something like this

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

As you can see, the client is indeed sending ping frames every 10 seconds (by the way, this is the minimum allowed, atleast for the Java client). Even if we didn't enable the keepalive in our server, we see that it is responding with a PING frame with flag ACK. But after few pings responded, it sends a GOAWAY frame with the code ENHANCE_YOUR_CALM. This is because the server is not configured to accept ping frames, so it terminates the connection.

Let's now enable the keepalive also in the server. This should do the job

```go
s := grpc.NewServer(grpc.KeepaliveParams(keepalive.ServerParameters{
   Time:    10 * time.Second,
   Timeout: 15 * time.Second,
}))
```

Now the logs look like this:

```bash
2025/05/09 19:24:30 http2: Framer 0x14000200540: wrote PING len=8 ping="\x00\x00\x00\x00\x00\x00\x00\x00"
2025/05/09 19:24:30 http2: Framer 0x14000200540: read PING flags=ACK len=8 ping="\x00\x00\x00\x00\x00\x00\x00\x00"
2025/05/09 19:24:40 http2: Framer 0x14000200540: wrote PING len=8 ping="\x00\x00\x00\x00\x00\x00\x00\x00"
2025/05/09 19:24:40 http2: Framer 0x14000200540: read PING len=8 ping="\xa6\x9c#UD[\x83\xd0"
2025/05/09 19:24:40 http2: Framer 0x14000200540: wrote PING flags=ACK len=8 ping="\xa6\x9c#UD[\x83\xd0"
2025/05/09 19:24:40 http2: Framer 0x14000200540: read PING flags=ACK len=8 ping="\x00\x00\x00\x00\x00\x00\x00\x00"
2025/05/09 19:24:50 http2: Framer 0x14000200540: wrote PING len=8 ping="\x00\x00\x00\x00\x00\x00\x00\x00"
2025/05/09 19:24:50 http2: Framer 0x14000200540: read PING len=8 ping="\xe5\x90\xda\\\x9eё\xbf"
2025/05/09 19:24:50 http2: Framer 0x14000200540: wrote PING flags=ACK len=8 ping="\xe5\x90\xda\\\x9eё\xbf"
2025/05/09 19:24:50 http2: Framer 0x14000200540: read PING flags=ACK len=8 ping="\x00\x00\x00\x00\x00\x00\x00\x00"
2025/05/09 19:25:00 http2: Framer 0x14000200540: wrote PING len=8 ping="\x00\x00\x00\x00\x00\x00\x00\x00"
2025/05/09 19:25:00 http2: Framer 0x14000200540: read PING flags=ACK len=8 ping="\x00\x00\x00\x00\x00\x00\x00\x00"
2025/05/09 19:25:10 http2: Framer 0x14000200540: wrote PING len=8 ping="\x00\x00\x00\x00\x00\x00\x00\x00"
2025/05/09 19:25:10 http2: Framer 0x14000200540: read PING len=8 ping="X\xde\xedl\xa8\b\\\x90"
2025/05/09 19:25:10 http2: Framer 0x14000200540: wrote PING flags=ACK len=8 ping="X\xde\xedl\xa8\b\\\x90"
2025/05/09 19:25:10 http2: Framer 0x14000200540: read PING flags=ACK len=8 ping="\x00\x00\x00\x00\x00\x00\x00\x00"
```

As you can see, there is both "wrote PING", "wrote PING ACK", "read PING", and "read PING ACK". It means both sides are sending ping's to each other. That is cool!

What happens if the android app loses connection? Let's try by putting airplane mode:

```bash
2025/05/09 19:28:07 Received request: Hello Server!
2025/05/09 19:28:07 http2: Framer 0x14000200540: wrote PING len=8 ping="\x02\x04\x10\x10\t\x0e\a\a"
2025/05/09 19:28:07 http2: Framer 0x14000200540: read PING flags=ACK len=8 ping="\x02\x04\x10\x10\t\x0e\a\a"

(at this point, I enabled airplane mode in the android emulator)...

2025/05/09 19:28:17 http2: Framer 0x14000200540: wrote PING len=8 ping="\x00\x00\x00\x00\x00\x00\x00\x00"
2025/05/09 19:28:32 Context cancelled
2025/05/09 19:28:32 Stream ended

```

The server sends a PING frame at 19:28:17 to check if the connection is alive. Then, we see a gap of 15 seconds (our keepalive timeout config!!) where nothing happens (no ack, not another ping sent from the server). And finally, the server decides to terminate the connection after no response of the client.

That is really cool, right? That is what I thought too, so, after trying it locally, I decided to deploy it to a test environment. In the test (and production) environment, we use ingress nginx as our reverse proxy for the backend services, because it provides us a way of load-balancing, security, rate limiting, and many other nice features. Anyway, I tried this keepalive in the environment and, surprisingly, didn't work. I couldn't understand why, but I could see that, even if I put airplane mode on the phone, the server was sending PING frames and someone was responding with an ACK... 

TODO meme wtf

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