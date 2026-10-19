---
title: "Cookies and Sessions: How Web Applications Remember Users"
description: "Why HTTP applications need state management, how browser cookies work with server-side sessions, and which security properties matter."
publishDate: 2025-05-14
updatedDate: 2026-10-19
tags: [web, authentication, security]
draft: true
---

> Translated and technically reviewed from [the original Korean post](https://tyge-ottesen-brahe.tistory.com/36).

When studying web development, we encounter cookies, sessions, and JWTs all the time. Yet explaining how a cookie actually works can be harder than using one. I wanted to understand their relationship instead of treating them as three disconnected buzzwords.

## Why start with HTTP?

HTTP is designed so that each request can be understood independently. An ordinary HTTP server does not automatically know that two different requests came from the same logged-in user. Applications add mechanisms to associate those requests with a user or an ongoing interaction.

Traditional HTTP/1.1 and HTTP/2 frequently run over TCP, which maintains transport connection state—including sequence numbers and acknowledgments. This is not the same as **application session state**. HTTP/3 uses QUIC instead of TCP, but the distinction remains: transport connections do not, by themselves, establish an application login session.

That is why cookies and sessions are useful.

## Cookies

A cookie is a small piece of data that a server can ask a browser to store, commonly through a `Set-Cookie` response header. On later requests matching its domain, path, and other rules, the browser can send it in a `Cookie` request header.

Cookies can support preferences, analytics, and authentication workflows. Because they are stored on the client side, a user or attacker with sufficient access may delete or manipulate them. A server must not trust arbitrary cookie values as proof of authorization.

Cookies can be hardened with attributes such as:

- `HttpOnly`: prevents ordinary JavaScript access through `document.cookie`.
- `Secure`: restricts transmission to secure connections, subject to browser rules.
- `SameSite`: limits when the browser includes a cookie in certain cross-site requests.

These measures are important but not a complete defense against cross-site scripting, cross-site request forgery, or session theft.

## Server-side sessions

With a common server-side session design, the server creates session state and assigns an unpredictable session identifier. The browser stores that identifier, often in a cookie. On later requests, the server uses the identifier to find the corresponding session state.

The browser does not need to hold the entire session object. This makes server-side revocation and state management possible. However, **a stolen session identifier can still let an attacker impersonate a user**; keeping the session data on the server does not make the identifier harmless.

In a distributed service, session state may be shared through a central store such as Redis, or otherwise managed so requests can be handled by multiple instances. Such a store is a deployment choice, not an inherent requirement of every session-based system.

## Why are session IDs so often sent with cookies?

This was the part I found especially interesting. We can have *both* a server-side session and a client-side cookie in the same authentication design.

1. The user logs in.
2. The server creates session state and an opaque session ID.
3. The server sends the ID in a cookie.
4. The browser includes the cookie in eligible subsequent requests.
5. The server looks up the session and applies authorization rules.

Why not put the session ID in a URL? URLs can be copied, shared, stored in history, and recorded in logs, making accidental exposure a serious risk. A manually managed request header is possible, but cookies provide built-in browser handling and useful security attributes.

The convenience of automatic transmission also requires CSRF defenses when cookies authenticate state-changing requests. See the [OWASP Session Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html).

## What I took away

Cookies are a browser storage and transmission mechanism; sessions are a way to maintain application state across requests. They solve related but different problems, and they are frequently used **together**. Understanding that relationship helped me move beyond simply implementing a login feature.