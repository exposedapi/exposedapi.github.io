# Finding My First Bug: A Small Bypass That Meant a Lot

This was one of those moments where things looked completely locked down at first, and then a tiny inconsistency opened everything up.

I was testing an application where different roles had clearly defined permissions. As a viewer, I wasn’t supposed to see the list of users. That part was working as expected. When I hit the endpoint:

`GET /api/v3/users`

I got a clean `401 Unauthorized`. No surprises there.

## Starting With the Obvious

My first instinct was to compare behavior with a higher-privileged role. So I captured a valid request from an admin account and replayed it with my viewer token. I expected either partial data or maybe a misconfigured authorization check.

Nothing changed.

Still blocked. Same response. Same behavior.

At this point, it felt like proper role-based access control was in place.

## Trying Things That Didn’t Work

Before finding the actual issue, I went through a bunch of common bypass attempts. Most of them failed, but they helped confirm what wasn’t broken.

I tried encoding tricks:

```/api/v3/%75sers  
/api/v3/users%2f```

No luck. Same unauthorized responses.

Then I played with HTTP methods:

```POST /api/v3/users  
PUT /api/v3/users```

Still blocked. No unexpected behavior.

I also tested header-based tricks, hoping something upstream might trust client input:

```X-Forwarded-For: 127.0.0.1  
X-Original-URL: /api/v3/users  
X-Rewrite-URL: /api/v3/users```

Again, nothing. Either properly ignored or validated.

At this point, I knew the application wasn’t falling for the usual low-effort tricks.

## The Small Detail That Changed Everything

Out of habit more than expectation, I tried slightly modifying the endpoint:

`GET /api/v3/uSers`

This time, the response came back `200 OK`.

And not just that. I got the full list of users.

That was the moment it clicked.

## What Actually Happened

The backend was treating `/users` and `/uSers` differently when it came to authorization, but not when it came to routing or data handling.

So effectively:

- Authorization checks were case-sensitive
- Resource handling was case-insensitive

That mismatch created a bypass.

The system blocked:

`/api/v3/users`

But allowed:

`/api/v3/uSers`

Even though both resolved to the same underlying functionality.

## Why This Was Interesting

This wasn’t some complex exploit or deep logic flaw. It was a small inconsistency. But it showed something important.

Security controls are only as strong as their weakest assumption.

In this case, the assumption was that endpoint casing would always be consistent. The application didn’t normalize input before applying authorization checks, and that gap was enough.

## What I Took Away From This

This was my first real bug, and what stuck with me wasn’t just the finding itself, but the process.

Most of the things I tried didn’t work. Encoding tricks, HTTP verb tampering, header manipulation. All dead ends. But each one helped narrow down the possibilities and build confidence that the system was mostly solid.

The actual bypass came from something simple that I almost didn’t try.

That’s the part I enjoyed the most. Not just finding the bug, but understanding why everything else failed and why this worked.

It reminded me that sometimes the interesting issues aren’t hidden behind complexity. They sit in small inconsistencies that only show up when you look just a bit differently.

---

*Triaged as P1 · Resolved · Bounty awarded.*
