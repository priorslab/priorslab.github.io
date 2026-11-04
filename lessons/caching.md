---
layout: page
title: "Why Does a 1% Hit Rate Drop Double Database Load?"
permalink: /lessons/caching/
---

*A Priors Lab companion lesson. Watch the video: [youtu.be/_7ZKsem-uzY](https://youtu.be/_7ZKsem-uzY)*

**The point in one line:** the database only feels the misses, so watch the miss rate, not the hit rate.

## The core idea

A cache is a small, fast store that keeps copies of answers you computed recently. You ask it first, and you go to the database only when it doesn't have the answer. When it has the answer, that's a **hit**. When it doesn't, that's a **miss**. The fraction of requests that hit is the **hit rate**.

The simplest pattern is **cache-aside**, where the application owns the logic in three steps:

1. Look up the key in the cache.
2. On a hit, return it.
3. On a miss, query the database, store the result in the cache, then return it.

## Worked example from the video (illustrative)

The video uses illustrative numbers. A product page needs a database query that takes 50 ms, and a thousand people per second open it. Without a cache, that's a thousand identical queries per second. With cache-aside, the first visitor pays the 50 ms and the next 999 get the page from memory in about a millisecond.

Now the opening number. Call the hit rate *h*. Every miss goes to the database, so:

> database load = request rate × (1 − h)

At 99%, that's ten queries per second. At 98%, it's twenty. The hit rate barely moved, but the miss rate doubled, and the database only feels the misses. Going from 99% to 99.9% cuts database load tenfold.

Latency works the same way. Average time is one millisecond plus 50 times the miss rate, so about one and a half milliseconds at 99%. Averages hide the tail, though: one request in a hundred still takes 51 ms. Your slowest 1% got no faster.

**Why a small cache works.** Popularity is skewed. Measurements of web traffic have found Zipf-like popularity, where the item ranked *k* gets traffic roughly proportional to 1/*k*. With a million products following that pattern, caching just the top 1% catches about 68% of requests (illustrative, but the shape is the point). If every product were equally popular, the same cache would hit only 1% of the time. A cache only pays off when answers get reused.

## Three ways a cache goes wrong

**1. It fills up.** Something must be evicted. **LRU** (least recently used) evicts whatever hasn't been touched for the longest. With room for three, A, B and C fill it; A is requested again and moves to the front; D arrives and B is evicted. One big scan, like a nightly report, can flush out all the popular items. **LFU** (least frequently used) keeps the most-hit items but is slow to forget yesterday's favorites. Real systems blend or approximate; Redis, for example, samples a few keys and evicts the oldest among them.

**2. It goes stale.** The price changes from $20 to $15 in the database, but the cache still says $20. Two fixes:
- A **TTL** (time to live): every entry expires after, say, 60 seconds, so it's at most 60 seconds wrong.
- **Invalidation**: when you write to the database, *delete* the cached key, don't update it. Two writers updating the cache can land in the wrong order.

There is still a race. A reader misses and fetches the old price; a writer updates the database and deletes the key; then the reader stores the old price. The TTL is the safety net, and some systems add leases or version numbers to reject late writes.

**3. It stampedes.** The home page needs a two-second query, it's cached, and a thousand requests per second read it. The entry expires. For the next two seconds every request misses and starts the same query, about 2,000 copies at once (illustrative). Fixes:
- **Request coalescing**: the first miss takes a lock and rebuilds; everyone else waits for that one result.
- **Stale-while-revalidate**: keep serving the old value while one worker refreshes it.
- **TTL jitter**: add a little randomness so keys written together don't expire together.

## Where writes go

- **Cache-aside**: the app writes the database and deletes the cached key.
- **Write-through**: every write goes to the cache and the database together. Reads stay fresh, but every write pays for both.
- **Write-back**: write to the cache now and flush later. Writes are fast, but if the cache dies before the flush, that data is gone.

## Common mistakes

- Watching the hit rate instead of the miss rate.
- Updating cached values on write instead of deleting them.
- Removing TTLs because "invalidation handles it".
- Giving every hot key the same TTL.
- Caching data that must never be stale, like an account balance before a transfer.
- Caching a personalized page under a shared key, so one user sees another user's data.

## Check yourself

1. **At 1,000 requests per second, how many reach the database at a 99% hit rate, and at 98%?**
   Ten per second at 99%, and twenty at 98%, because load is the request rate times (1 − h).
2. **Why delete the cached key on a write instead of writing the new value into the cache?**
   Two writers updating the cache can land in the wrong order. A delete just forces a fresh read.
3. **A hot key expires and the database is flooded with identical queries. Name two fixes.**
   Any two of: request coalescing, stale-while-revalidate, or jitter on TTLs.

## Watch the video

[Why Does a 1% Hit Rate Drop Double Database Load?](https://youtu.be/_7ZKsem-uzY)

*The video's narration is an AI-generated (synthetic) voice. This lesson is written from the video's script.*
