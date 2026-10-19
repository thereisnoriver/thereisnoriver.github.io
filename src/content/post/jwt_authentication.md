---
title: "JWT Authentication: Tokens, Sessions, and Refresh Strategies"
description: "A study note on JWT-based authentication, token theft, refresh-token rotation, and the differences from server-side sessions."
publishDate: 2025-05-22
updatedDate: 2026-10-19
tags: [authentication, jwt, security]
draft: true
---

> Translated and technically reviewed from [the original Korean post](https://tyge-ottesen-brahe.tistory.com/38). This is a conceptual guide, not a claim that the described controls were deployed in production.

In the previous post, I looked at cookies and server-side sessions. A session-based design usually stores session state on the server and sends the browser an opaque session identifier. JWT-based authentication raises a different question: what changes when the client carries a token containing claims that the server can verify cryptographically?

## What is a JWT?

A JSON Web Token (JWT) is a structured token format. A common signed JWT contains a header, a claims payload, and a signature. An application may issue a JWT after login and require the client to present it with later requests, often through an `Authorization: Bearer ...` header or, depending on the architecture, a cookie.

An example claims payload might contain a subject identifier, issuer, audience, and expiration time. **A signed JWT is not inherently encrypted**. Its claims can normally be decoded by anyone who possesses the token, so secrets and unnecessary personal data should not be placed in the payload.

### Useful properties

- **Verifiable integrity:** a correct signature-verification process detects unauthorized modification of signed claims.
- **Potentially stateless validation:** a server may validate a token without looking up a server-side session for each request.
- **Scalability:** independent service instances can verify tokens if they have the appropriate key material and validation rules.

Stateless verification is a design option rather than a universal advantage. Revocation, risk assessment, and refresh-token management can reintroduce server-side state.

## The problem of token theft

Possession of a valid bearer token may be sufficient to act as the authenticated user within the token's authority. Signature verification detects tampering but **does not prove that the presenter is the legitimate user**. If an attacker steals a valid access token, the token may be usable until it expires or is otherwise invalidated.

Shorter expiration times reduce the window of misuse, but they also mean legitimate clients need a way to renew access.

## Access tokens and refresh tokens

An access token is used to access protected resources. A refresh token is used to request new access tokens, typically from a dedicated authorization endpoint.

A simplified flow is:

1. The client submits login credentials over a protected connection.
2. The server validates the credentials and issues an access token and, where appropriate, a refresh token.
3. The client presents the access token for ordinary API calls.
4. When the access token expires, the client uses the refresh token to request another one.
5. The server validates the refresh token and may **rotate** it, invalidating the previous value.

In a stateful refresh-token design, the server may store a hash or record of refresh tokens or token families to support revocation and replay detection. A stolen refresh token is serious, but its theft does **not** necessarily permit unlimited renewal: expiration, rotation, replay detection, and revocation can constrain it.

### Comparing risks

| | Access token | Refresh token |
|---|---|---|
| Purpose | Authorize API access | Obtain new access tokens |
| Typical lifetime | Shorter | Longer, with controls |
| Theft impact | Access within token scope until expiry/revocation | Potential renewal or session hijack unless detected/blocked |
| Protections | Least privilege, short lifetime, secure transport/storage | Rotation, reuse detection, revocation, secure transport/storage |

There is no blanket rule that refresh tokens are less likely to be stolen. Exposure depends on the client, storage mechanism, and attack scenario.

## Cookie storage and CSRF

A JWT is a token **format**, while a cookie is a **browser transport and storage mechanism**. A JWT can be carried by a cookie or an authorization header. `HttpOnly`, `Secure`, and appropriate `SameSite` settings can help protect cookie-based tokens, but automatically attached cookies also create CSRF considerations. A token in JavaScript-accessible browser storage has different exposure to malicious scripts.

## Sessions versus JWTs

| Question | Server-side session | Self-contained signed JWT access token |
|---|---|---|
| What does the client carry? | Usually an opaque session ID | A structured token with claims |
| Where is current user state? | Often in a server-side session store | Some claims carried in token; additional state may still exist |
| How can it be revoked? | Server deletes or invalidates the session | Expiry, denylist, key rotation, or other validation mechanisms |
| What happens after theft? | Stolen ID can be replayed | Stolen bearer token can be replayed |
| How does scaling work? | Shared or otherwise distributed session management | Shared verification keys and consistent validation policies |

The practical lesson for me is that JWTs do not remove the need to design **identity, expiration, revocation, storage, and monitoring**. They change the location and form of some state; they are not automatically more secure than sessions.

Further reading: [OWASP JWT Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/JSON_Web_Token_Cheat_Sheet.html) and [OWASP OAuth2 Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/OAuth2_Cheat_Sheet.html).