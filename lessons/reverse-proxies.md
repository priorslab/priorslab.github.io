---
layout: page
title: "A Million Visitors, One IP Address: Reverse Proxies Explained"
permalink: /lessons/reverse-proxies/
---

*A Priors Lab companion lesson. Watch the video: [youtu.be/SOnqWmqaq8A](https://youtu.be/SOnqWmqaq8A)*

**The point in one line:** a reverse proxy is one front door for all your servers, and it has to hand the servers what it hides, in headers you only trust if your own proxies wrote them.

## The core idea

At the bottom, the web is requests in and responses out. A request is a short text message (a method, a path and headers), and the server knows who is calling because every connection carries the caller's network address.

When traffic grows, you add servers. Giving each of four servers its own public address breaks in three ways:

- If server three crashes, the clients that picked it just get errors, because nothing in between can steer them away.
- Every server needs its own HTTPS certificate. That's four to renew, and missing one means visitors see a security warning.
- Compression, caching and rate limits get set up four times, and the copies drift apart.

The fix is one machine in front that owns the only public address and forwards each request to a server on a private network. That machine is a **reverse proxy**: a server that accepts requests on behalf of other servers and passes them along. The HTTP standard calls this kind of intermediary a gateway. Well-known ones are nginx, HAProxy, Envoy and Caddy. When server three fails its health checks, the proxy stops sending it traffic, and clients never see the error. Spreading requests like this is load balancing, one of the jobs a reverse proxy can do.

**Forward or reverse?** A forward proxy sits next to the clients, so websites see the proxy instead of the employees. A reverse proxy sits next to the servers, so clients never see the machines behind it. It's the same machinery with opposite loyalty.

## What one front door buys you

- **TLS termination.** The proxy holds the certificate and decrypts HTTPS once, at the edge. Behind it, traffic can travel unencrypted on the private network, or be encrypted again if that network isn't trusted.
- **Routing.** Paths starting with `/api` go to the API servers, and images go to a file server. Reading the Host header lets one address serve many sites.
- **One place for edge features.** Compression, caching and rate limits live in one configuration instead of four.

## Worked example from the video (illustrative)

**Buffering slow clients.** The video's numbers are illustrative. A phone on a weak signal downloads a 1 MB page at 50 KB/s, which takes 20 seconds. Without a proxy, the app's worker is tied up for all 20 seconds, doing nothing but waiting on the phone. With a proxy, the app hands the whole response over the fast internal network in a few milliseconds, and the proxy drips it out to the phone while the worker moves on. nginx buffers responses by default, and it holds slow uploads the same way until the whole request has arrived.

## What the server sees

The server's connection now comes from the proxy, so every visitor looks like `10.0.0.2`. That is the log from the opening. The proxy writes the lost facts into headers:

- `X-Forwarded-For`: the client's address.
- `X-Forwarded-Proto`: the request arrived over HTTPS, even though the server only saw plain HTTP.
- `Via`: each proxy the message passed through (defined in the HTTP standard).
- `Host`: usually kept as sent, so the server knows which site was asked for.

The X- headers are a convention, not a standard. RFC 7239 defines the standard `Forwarded` header, with fields like `for` and `proto`.

## The trap: spoofed headers

Headers are just text, and clients can write them too. A client can send its own `X-Forwarded-For` with an address it made up. A proxy that simply appends adds the address it actually saw, so the server gets two addresses: the first is whatever the client typed, and the last is real. Trust the first one, and anyone can choose their own address and dodge an IP ban or a per-address rate limit.

The rule is to read the list from the right and skip only the addresses of proxies you run. The first address you don't control is the client. Alternatively, let the edge proxy overwrite the header entirely. Either way, make sure servers only accept connections from the proxy, or anyone can skip it and send any header they like.

## Trade-offs

The proxy is now a single point of failure, so run at least two: a spare can take over the shared address with a protocol like VRRP, or DNS can list several proxies. Every request also pays one more hop, and debugging has one more layer.

## Common mistakes

- Trusting the leftmost `X-Forwarded-For` address.
- Letting clients reach the servers directly, around the proxy.
- Running a single proxy with no spare.
- Assuming a forward proxy and a reverse proxy do the same job.

## Check yourself

1. **Why does every visitor in the server log show `10.0.0.2`?**
   The server's connection comes from the reverse proxy, so it sees the proxy's address unless the proxy passes the client's address in a header.
2. **How does a proxy protect app workers from a phone on a slow connection?**
   It buffers. The app hands the full response over the fast internal network in milliseconds, and the proxy drips it out to the phone.
3. **`X-Forwarded-For` holds two addresses. Which one is the client?**
   Read from the right, skip the addresses of proxies you run, and take the first address you don't control. Never trust the leftmost one by default.

## Watch the video

[A million visitors, one IP address: reverse proxies explained](https://youtu.be/SOnqWmqaq8A). Goes public Fri Nov 6.

*The video's narration is an AI-generated (synthetic) voice. This lesson is written from the video's script.*
