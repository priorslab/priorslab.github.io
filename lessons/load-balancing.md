---
layout: page
title: "Forty Arrivals, and Thirty of Them Time Out"
permalink: /lessons/load-balancing/
---

*A Priors Lab companion lesson. Watch the video: [youtu.be/65ii01AIsAE](https://youtu.be/65ii01AIsAE)*

**The point in one line:** a load balancer picks a server for each new connection, and the best rule keeps one slow machine from setting everyone's wait.

## The core idea

One server can fail in two ways. It can **run out of speed**: in the video's illustrative numbers, it finishes ten requests a second, so when forty arrive, thirty wait too long and time out. Or it can **disappear**, and then every request fails because it was the only place the work could go. A faster machine fixes the first problem but not the second. You need a second server.

**DNS round robin** is the cheap first step. Give the name two addresses and rotate which comes first (RFC 1794 describes it; it is not a protocol standard). But the answer is cached with a TTL, so a whole office can pin itself to one server, and if a server dies, DNS doesn't notice. DNS shares the name. It doesn't watch the machines.

A **load balancer** fixes that. It holds one stable address in front of the servers, and for every new connection it picks a live server and forwards the request. At **layer 4** it reads only the envelope (source, destination and port). At **layer 7** it opens the letter: it reads the HTTP path or a cookie, so images can go to file servers and checkout to app servers.

## The rules for choosing

- **Round robin**: take turns, A, B, C, then back to A. It's nginx's default. Equal counts are not equal work, though. If a 2-second report lands on B, B's line grows while A and C sit almost idle.
- **Weights**: if B is a smaller machine, give A weight two and B weight one. A weight doesn't notice that B just got stuck.
- **Least connections**: send the next request to the server with the fewest open requests (nginx `least_conn`). A long report keeps a slot busy on B, and new work flows around it.
- **Least response time**: open isn't the same as fast. If A's recent replies took 40 ms and B's took 400 ms (illustrative), send the next one to A. nginx calls this `least_time`.
- **Power of two choices**: asking every server on every request is a lot of bookkeeping. Pick two servers at random and keep the less loaded one.

## Worked example from the video (illustrative)

Eight servers and 200 requests, with illustrative numbers. With **one** random pick per request, the busiest server ends with 34 and the quietest with 20. With **two** random picks, keeping the lighter one each time, every server ends with 25. Azar, Broder, Karlin and Upfal showed that with two choices the heaviest bin grows more slowly. nginx's `random two` does this.

The keys example works the same way. Hash each key modulo the number of servers, then add a fourth server, and nine of twelve keys move. Put the servers on a ring instead, where a key walks clockwise to the next server. Adding a server then moves only the keys in its new arc: two of the twelve here, while the other ten stay put. That is **consistent hashing** (Karger and coauthors).

## Keeping it honest

- **Health checks.** If server C stops answering, round robin still sends it every third connection. The balancer should test servers instead of trusting them. nginx's default marks a server failed after one failed attempt in 10 seconds, using real requests; HAProxy can send its own probes.
- **Sticky sessions.** If a cart lives in A's memory, a cookie or IP hash can pin that browser to A. The cost is that one heavy browser can overload A, and if A dies the cart is gone.
- **Connection draining.** To take A out for a deploy, stop sending it new work, let the work already on it finish, then remove it. Cutting it right away kills checkouts mid-flight.
- **The balancer itself.** If the balancer stops, everyone fails while A and B sit healthy and idle. Run two: one holds a virtual address and the other waits, with VRRP (RFC 9568) deciding who owns the address. **Anycast** (RFC 4786) lets one address live in many cities, as long as packets from one connection don't split across nodes.

## Chase the tail

A page that calls four servers and waits for all of them is only as fast as the slowest. If three answer in 30 ms and one in 800 ms (illustrative), the page takes 800 ms. Dean and Barroso called this "the tail at scale." A smarter choice keeps that slow machine from collecting the calls.

## Common mistakes

- Treating DNS round robin as failover.
- Assuming equal request counts mean equal load.
- Using plain modulo hashing for keys that should stick, then adding a server.
- Removing a server without draining it.
- Forgetting that a sticky pin, or a single balancer, can become the only way through.

## Check yourself

1. **Why doesn't DNS round robin protect you when a server dies?**
   The answer is cached with a TTL, and DNS doesn't watch the machines, so clients keep the old address.
2. **In the video's example, what does picking two random servers instead of one change?**
   One pick leaves the busiest at 34 and the quietest at 20. Two picks, keeping the lighter, gives every server 25.
3. **You add a server to a ring of consistent hashing. Which keys move?**
   Only the keys in the new server's arc. In the example, two of twelve.

## Watch the video

[Forty arrivals, and thirty of them time out](https://youtu.be/65ii01AIsAE). Goes public Tue Nov 3.

*The video's narration is an AI-generated (synthetic) voice. This lesson is written from the video's script.*
