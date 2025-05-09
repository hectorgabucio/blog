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
<img alt="WTF meme" src="https://media2.giphy.com/media/v1.Y2lkPTc5MGI3NjExZHY4OHlsZTR4Z3BpeWltaThtenN1NHYzb2kxb2hocDZ3aG5vMm43cSZlcD12MV9pbnRlcm5hbF9naWZfYnlfaWQmY3Q9Zw/b8RQzkElbBsXqEPF2X/giphy.gif" />
</p>

### Reproducing the Issue

To better understand the problem, I used my toy project to reproduce the issue. I added an nginx proxy between the Android client and the server by running a Docker container with nginx, configured to forward traffic to my Go service.

Let's start by running our Docker Compose setup:

```bash
docker-compose up --build
```

#### First Weird Behavior: Missing Client Pings

We immediately notice something strange - the server isn't receiving the client's ping frames:

```bash
grpc-server-1  | 2025/05/09 17:51:17 Received request: Hello Server!
grpc-server-1  | 2025/05/09 17:51:27 http2: Framer 0x40000e4540: wrote PING len=8 ping="\x00\x00\x00\x00\x00\x00\x00\x00"
grpc-server-1  | 2025/05/09 17:51:27 http2: Framer 0x40000e4540: read PING flags=ACK len=8 ping="\x00\x00\x00\x00\x00\x00\x00\x00"
grpc-server-1  | 2025/05/09 17:51:37 http2: Framer 0x40000e4540: wrote PING len=8 ping="\x00\x00\x00\x00\x00\x00\x00\x00"
grpc-server-1  | 2025/05/09 17:51:37 http2: Framer 0x40000e4540: read PING flags=ACK len=8 ping="\x00\x00\x00\x00\x00\x00\x00\x00"
```

Notice how we only see PINGs that the server is writing, and someone is sending ACKs. But who?

#### The Mystery Deepens

Let's try enabling airplane mode on the phone. Surely we shouldn't see any PING ACKs now, right?

```bash
grpc-server-1  | 2025/05/09 17:53:50 Received request: Hello Server!

(I enabled airplane mode here)

grpc-server-1  | 2025/05/09 17:54:00 http2: Framer 0x40000e4540: wrote PING len=8 ping="\x00\x00\x00\x00\x00\x00\x00\x00"
grpc-server-1  | 2025/05/09 17:54:00 http2: Framer 0x40000e4540: read PING flags=ACK len=8 ping="\x00\x00\x00\x00\x00\x00\x00\x00"
grpc-server-1  | 2025/05/09 17:54:10 http2: Framer 0x40000e4540: wrote PING len=8 ping="\x00\x00\x00\x00\x00\x00\x00\x00"
grpc-server-1  | 2025/05/09 17:54:10 http2: Framer 0x40000e4540: read PING flags=ACK len=8 ping="\x00\x00\x00\x00\x00\x00\x00\x00"
```

<p align="center">
<img alt="Mystery meme" src="https://media1.tenor.com/m/0UBIdauP8CoAAAAC/iker-jimenez-misterioso.gif" />
</p>

Who is responding to the pings when the client has airplane mode enabled?

### The Investigation

After some research, I found a [GitHub issue](https://github.com/kubernetes/ingress-nginx/issues/4402) where someone reported the same problem:

> "Currently my grpc server is doing keep alive ping each 10 seconds and ngnix proxy is doing ack of the ping but ngnix itself is not pinging client."

However, there wasn't a clear solution in that issue. Then I discovered an even older issue in the nginx issue tracker, [closed as "WONT FIX" from 5 years ago](https://trac.nginx.org/nginx/ticket/1887).

The issue description was particularly relevant:

> "gRPC has multiple options for keepalive, which are especially relevant for streaming messages:
> 
> https://github.com/grpc/grpc/blob/master/doc/keepalive.md
> 
> With these you are able to track disappearing clients and narrow connection latency. gRPC uses HTTP/2 ping messages for keepalives. When proxying through nginx, nginx does not passthrough ping messages to the client.
> 
> When using nginx with gRPC you currently have to implement a form of keepalive yourself. Send and receive timeouts from nginx are not helpful, as we use gRPC with long-lived streaming connections.
> 
> Possible solutions:
> 
> 1. Add an option to passthrough HTTP/2 ping messages to the gRPC backend: `grpc_ping_passthrough yes/no`.
> 2. Add ability to send keepalive pings from nginx. In this case nginx would need additional options and nginx would have to terminate the connection if there is no response to the ping messages, similar to what gRPC does."

### The NGINX Team's Response

The NGINX team's response was clear. They explained that using HTTP/2 pings to check client connectivity doesn't work well with NGINX as a proxy because:

1. A single connection to the backend may serve multiple clients (especially with HTTP/2 multiplexing)
2. There's no persistent connection between the client and the backend unless there's an active request
3. Multiple simultaneous client requests mean multiple client-server connections

Their conclusion was straightforward: HTTP/2 pings only help verify direct (peer-to-peer) connections, not end-to-end connectivity through a proxy.

### The TCP Keepalive Alternative

As a possible solution, they mentioned using TCP keepalive. However, I rejected this idea because TCP keepalive is notoriously difficult to maintain, with many edge cases to handle and debug. For a great example of these challenges, you can read this [insightful post from Cloudflare](https://blog.cloudflare.com/when-tcp-sockets-refuse-to-die/).

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

## The Work-Around

To be completely honest, first I thought about getting rid of nginx. But I can't just remove the reverse proxy, maybe some other proxy behaves like I want, forwarding PING frames?

But nope, I tried envoy proxy and behaves the same way: it just ACK's the PING frames.

To work around this limitation, I implemented a custom "home-made" ping mechanism using regular gRPC messages. This approach:

1. Uses standard gRPC messages instead of HTTP/2 ping frames
2. Can be properly proxied by Nginx
3. Maintains (almost) the same functionality as the built-in keepalive mechanism

## Implementation Details

I've created a demonstration repository that shows how to implement this custom keepalive mechanism: [nginx-hates-grpc-keepalive](https://github.com/hectorgabucio/nginx-hates-grpc-keepalive).

This work-around introduced a breaking change in my original stream: I can't just rely on a unidirectional server-stream. I actually need a bidirectional stream, where both client and server can send and receive ping requests.

This is the proto definition of the contract:

```proto
service StreamService {
    // Bidirectional streaming RPC
    rpc Stream (stream StreamMessage) returns (stream StreamMessage) {}
}

message StreamMessage {
    oneof content {
        StreamRequest request = 1;
        StreamResponse response = 2;
        Ping ping = 3;
        Pong pong = 4;
    }
}

message StreamRequest {
    string query = 1;
}

message StreamResponse {
    string message = 1;
    int32 index = 2;
}

message Ping {
    int64 timestamp = 1;
}

message Pong {
    int64 original_timestamp = 1;
    int64 server_timestamp = 2;
}
```

Please note the `Ping` and `Pong` messages: those are part of our home-made keepalive system.

The custom ping mechanism works as follows:

1. **Client-side behavior:**
   - Sends frequent ping messages to the server (StreamMessage with content Ping)
   - Expects a ping reply from the server (StreamMessage with content Pong)
   - Terminates the connection if no reply is received within a reasonable timeframe

2. **Server-side behavior:**
   - Monitors incoming ping messages from the client (StreamMessage with content Ping) and replies (StreamMessage with content Pong)
   - Terminates the connection if no pings are received within the expected interval

This bidirectional ping mechanism ensures both sides can detect connection issues and take appropriate action.

It is really simple to mantain and it works great, nginx will forward those messages since are just gRPC messages. 

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

