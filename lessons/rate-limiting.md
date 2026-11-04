---
layout: page
title: 'Rate Limiting Explained: Why "5 per Minute" Let 10 Through'
permalink: /lessons/rate-limiting/
---

*A Priors Lab companion lesson. Watch the video: [youtu.be/qRIEZOW4m-Y](https://youtu.be/qRIEZOW4m-Y)*

**The point in one line:** a rate limiter is three choices (an algorithm, a key, and where the state lives), and the algorithm decides which bursts slip through.

## The core idea

A server has finite capacity: CPU, memory, database connections, or a paid upstream API. Most of the time, traffic from many clients stays under it. Then one client misbehaves (a buggy loop, a scraper, someone trying stolen passwords) and eats the shared capacity, so everyone slows down. That is the noisy neighbor problem.

A rate limiter caps how many requests a client may make per unit of time. For each request it makes one decision: allow or reject. The video walks through five ways to make that decision:

- **Fixed window counter.** Chop time into one-minute windows and keep one counter per client per window. Cheapest option, but a client can send a full limit at the end of one window and another full limit at the start of the next. Across a boundary it can let through double the limit.
- **Sliding window log.** Store a timestamp for every request, drop the ones older than one window, and count the rest. Exact, with no boundary to game, but memory-hungry: a limit of a thousand per hour across a million clients means up to a billion entries.
- **Sliding window counter.** Keep two counters (previous minute and current minute) and blend them by how far into the current minute you are. Nearly exact with two integers, though it assumes the previous minute's requests were spread evenly.
- **Token bucket.** A bucket holds up to B tokens and refills at r tokens per second; each request spends one. Bursts can reach B, but the long-run rate never exceeds r. You only store two numbers: the token count and the last update time.
- **Leaky bucket.** Requests pour into a fixed-size queue that drains to the server at a constant rate. Output is perfectly smooth, but queued requests wait.

Every limiter also needs a **key**, meaning whose counter it is: a user ID, an API key, a client IP, or a per-endpoint budget. Real systems layer them, for example per IP, per user and a global cap.

## Worked example from the video (illustrative)

These are illustrative numbers, computed by simulation for the video's example. Take a token bucket with capacity ten that refills two tokens per second.

- **A burst of thirty requests at time zero.** The bucket holds ten tokens, so 10 are allowed and 20 are rejected.
- **A steady three per second for twenty seconds** (sixty requests). The bucket drains by a net one token per second, empties after about ten seconds, and then allows only two per second. The result is 49 allowed and 11 rejected.

Now the boundary attack, with a limit of a hundred per minute. A fixed window lets 200 through in two seconds that straddle the boundary. A token bucket with capacity a hundred and the same average rate allows 103: the full bucket plus about three refilled tokens.

That is the opening puzzle solved. The limiter that let ten through in ten seconds was a fixed window doing exactly what it was built to do.

## Many servers, one limit

If each of your servers keeps its own counters, a user spread across three servers gets three times the limit. The usual fix is a shared store such as Redis. But "read, check, then write" is a race: two servers both read four, both decide four is under five, both write five, and six requests get through. Make the update atomic instead. Redis's `INCR` increments and returns the new value in one step, and a small Lua script can do a token bucket's refill-and-spend atomically.

A shared store adds a network round trip and a new dependency. If it goes down, you have to decide ahead of time whether to fail open (allow everything) or fail closed (reject everything). That is a deliberate trade between availability and protection.

## Saying no, and hearing no

Over HTTP, the standard rejection is status **429 Too Many Requests** (RFC 6585), and the **Retry-After** header says how long to wait. On the client side, never retry a 429 in a tight loop. Honor Retry-After. Otherwise, double your wait after each failure and add random jitter. That is exponential backoff.

## Common mistakes

- Using a fixed window and assuming it can never allow more than the limit.
- Doing read-check-write against a shared counter without an atomic operation.
- Relying on IP limits alone, when one office can sit behind one address and an attacker can rotate addresses.
- Not deciding fail-open versus fail-closed until the store is already down.
- Retrying immediately after a 429.

## Check yourself

1. **Why can a fixed window of five per minute allow ten requests in about ten seconds?**
   Five arrive at the end of one window and five at the start of the next. Each window only sees five.
2. **A token bucket has capacity B and refill rate r. Which knob controls bursts, and which controls the long-run rate?**
   B caps the burst size, and r caps the sustained rate.
3. **Two servers share a Redis counter and both allow a request that should have been rejected. What went wrong, and what fixes it?**
   They did read, check, then write, so both saw the same old value. An atomic increment such as `INCR` (or a Lua script) fixes it.

## Watch the video

[Rate Limiting Explained: Why "5 per Minute" Let 10 Through](https://youtu.be/qRIEZOW4m-Y)

*The video's narration is an AI-generated (synthetic) voice. This lesson is written from the video's script.*
