---
id: token-expires-tech
topic: Very common JWT problem
category: Design & Architecture
---


## The core issue

A **JWT access token is usually stateless**. That means once the server issues it, the token can often remain valid until its expiration time, unless you add extra server-side checks. OWASP notes that JWTs do not have a built-in user-driven revocation feature, and Auth0 likewise states that stateless access tokens generally cannot be directly revoked before expiry. 


## Option 1: Short-lived access token + revoke refresh tokens

This is the most common and practical answer.

Example:

* access token: 5–15 minutes
* refresh token: days or weeks
* password change -> revoke all refresh tokens

Result:

* user is kicked out everywhere after access tokens expire
* worst-case delay is the remaining lifetime of current access tokens 

This is often enough for many systems.

### Why this works

Because access tokens are intentionally short-lived, even if they cannot be instantly revoked, the risk window is small.

## How the flow works

### 1. User logs in

The server authenticates username/password and returns:

* `access_token` — short lifetime
* `refresh_token` — longer lifetime

This matches the OAuth/OIDC token model used by Microsoft and Auth0. 

### 2. Client calls APIs with the access token

The access token is included in API requests, usually in the `Authorization: Bearer ...` header.

### 3. Access token expires

When the access token expires, the client sends the refresh token to the auth server and asks for a new access token. Microsoft explicitly documents that when an access token expires, the client uses the refresh token to silently acquire a new access token.

### 4. Auth server checks the refresh token

If the refresh token is still valid and not revoked, the server issues a new access token, and sometimes a new refresh token too. Microsoft and Auth0 both document refresh token replacement/rotation behavior.

---

## Option 2: Token version / security stamp check

This is the usual way to get closer to **immediate invalidation**.

### Idea

Store a field on the user record, for example:

```text
Users
- Id
- PasswordHash
- TokenVersion
```

When issuing JWT, include `TokenVersion` in the token claims.

Example claim:

```json
{
  "sub": "123",
  "token_version": 5
}
```

When password changes:

* increment `TokenVersion` from 5 to 6

On every authenticated API request:

* read user ID from token
* load current `TokenVersion` from database or cache
* compare with token claim
* if token says 5 but DB says 6 -> reject token

That makes all previously issued access tokens invalid immediately after password change.

### Cost

This removes part of the “fully stateless” benefit, because now the API must check server-side state on requests.

### Why this pattern is good

It is simple, works across web/mobile/desktop, and invalidates **all sessions for that user** at once.

Also, in the ASP.NET world, the idea is very similar to a **security stamp**: changing the stamp invalidates prior authentication state. Microsoft community guidance explicitly references changing the user’s security stamp for session invalidation scenarios. 

---

## Option 3: Store sessions/tokens server-side per device

Another strong design is to treat each login as a server-side session.

Example table:

```text
UserSessions
- SessionId
- UserId
- DeviceType
- RefreshTokenHash
- IsRevoked
- CreatedAt
- ExpiresAt
```

JWT contains:

* `sub`
* `session_id`

On each request:

* optionally verify that `session_id` is still active

When password changes:

* revoke all sessions for that user

This gives you:

* revoke all devices
* revoke one specific device
* show “logged in devices”
* audit trail

This is often the best design if you want strong session control.

---