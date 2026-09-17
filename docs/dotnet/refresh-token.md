# Refresh token, JWT, and “is it still stateless?”

Speak this in an interview. Use the **real-world** bit if they ask about your last project.

---

## Q1. How do you implement refresh tokens in a .NET + Angular app?

**Answer:**

I use **two tokens**.

1. **Access token** — JWT, short life (5–15 minutes). Angular sends it on every API call: `Authorization: Bearer ...`. The API **only validates** the signature, issuer, audience, and expiry. It does **not** look this token up in the database. That part is **stateless**.

2. **Refresh token** — long life (days), random bytes (or a JWT that I still treat as a handle). Angular sends it **only** to `/api/auth/refresh` (and logout/revoke). The **auth** side stores a **hash** of it (and family id, user, tenant, expiry, revoked flag). On refresh I rotate: old token dies, new pair is issued.

Angular: interceptor. If a call returns 401, I call refresh **once**, queue other 401s, retry. If refresh fails, logout.

I never put the refresh token in a query string. I never log it. Prefer httpOnly cookie for refresh, or memory + silent refresh. Access token often in memory.

**Example:**

```csharp
public sealed class RefreshToken
{
    public Guid Id { get; set; }
    public required string UserId { get; set; }
    public required string TenantKey { get; set; }
    public required string TokenHash { get; set; }  // hash, not raw
    public DateTimeOffset ExpiresAt { get; set; }
    public DateTimeOffset? RevokedAt { get; set; }
    public string? ReplacedByHash { get; set; }
}

// login
var access = _jwt.CreateAccessToken(user, tenant, TimeSpan.FromMinutes(10));
var rawRefresh = Convert.ToBase64String(RandomNumberGenerator.GetBytes(64));
db.RefreshTokens.Add(new RefreshToken
{
    UserId = user.Id,
    TenantKey = tenant,
    TokenHash = Sha256(rawRefresh),
    ExpiresAt = DateTimeOffset.UtcNow.AddDays(14),
});
await db.SaveChangesAsync(ct);
return new TokenResponse(access, rawRefresh);

// refresh
var row = await db.RefreshTokens
    .FirstOrDefaultAsync(x => x.TokenHash == Sha256(request.RefreshToken), ct);
if (row is null || row.RevokedAt is not null || row.ExpiresAt < DateTimeOffset.UtcNow)
    return Unauthorized();

row.RevokedAt = DateTimeOffset.UtcNow;
// issue new pair, set ReplacedByHash, save
```

```typescript
// Angular interceptor (sketch)
if (err.status === 401 && !req.url.includes('/auth/refresh')) {
  return this.auth.refreshOnce().pipe(
    switchMap(token => next(req.clone({ setHeaders: { Authorization: `Bearer ${token}` } })))
  );
}
```

**Real-world example:**

In an incident product, agents kept a ticket form open for 40 minutes. Access JWT died at 15 minutes. Without refresh, Save returned 401 and the form looked “logged out”. Refresh rotated the token, retried Save, and the ticket closed. Refresh hashes lived in the tenant/auth database so we could **logout all devices** after a stolen laptop.

**Cross-question:** If you insert the refresh token in the DB, how is JWT still **stateless**?

**Cross-answer:**

**The access token is still stateless.** Every normal REST call (`GET /api/incidents`) does **not** hit a session table. The API checks the JWT cryptographically and trusts the claims until it expires.

**The refresh token is the stateful exception**, on purpose. Stateless JWT cannot be revoked early (unless you add a denylist, which is also state). We store refresh tokens so we can:

- rotate them
- revoke one device
- detect reuse (theft)
- force logout

So the design is:

| Piece | Stateful? | Used on |
| --- | --- | --- |
| Access JWT | No | Almost every API call |
| Refresh token row | Yes | Only `/refresh`, `/logout` |

I do **not** store every access token. That would make the whole API stateful and slower.

If they say “pure stateless refresh”: you *can* make the refresh token a JWT and store nothing. Then you **cannot** revoke a stolen refresh token until it expires. For a real product I do not do that.

**Follow-up:** Is ASP.NET Core cookie auth stateless?

**Follow-up answer:** No. Cookie auth is a **session** (or a protected cookie the server issued). JWT access tokens are the stateless style. Mixing cookie + JWT without a plan confuses everyone.

---

## Q2. Why hash the refresh token in the database?

**Answer:**

If the DB leaks, raw tokens would work as login. A hash is one-way. I compare hash(incoming) to the stored hash. Same idea as passwords, but it is a random token, not a user password — still hash it (SHA-256 is common for high-entropy tokens; passwords still need a slow hasher like bcrypt/argon).

**Example:**

```csharp
static string Sha256(string raw)
{
    var bytes = SHA256.HashData(Encoding.UTF8.GetBytes(raw));
    return Convert.ToHexString(bytes);
}
```

**Real-world example:**

A backup of the auth table was restored on a dev machine. Hashes were useless as tokens. If we had stored raw refresh tokens, every user session would be stolen.

**Cross-question:** Can I store the JWT access token in the same table?

**Cross-answer:**

I do not. Access tokens are short and validated by signature. Storing them makes the API look them up every call — that is a session, not JWT.

---

## Q3. What is refresh token rotation and reuse detection?

**Answer:**

Each refresh issues a **new** refresh token and **kills** the old one. If someone sends an **already revoked** refresh token, I assume theft: revoke the **whole family** for that user/device.

**Example:**

```csharp
if (row.RevokedAt is not null)
{
    await RevokeFamily(row.UserId, row.TenantKey, ct);
    return Unauthorized();
}
```

**Real-world example:**

Attacker stole an old refresh token from logs. User already refreshed, so that token was revoked. Using it again locked the family and we asked the user to log in on all devices.

**Cross-question:** Sliding expiry without rotation?

**Cross-answer:**

Weaker. A stolen token works until the long expiry. Rotation shrinks the window.

---

## Q4. Where does Angular keep the tokens?

**Answer:**

- **Refresh:** httpOnly Secure cookie (best against XSS) or memory.
- **Access:** memory (variable / signal). Survive refresh of the SPA by using the refresh cookie silently.

`localStorage` is easy and **XSS-stealable**. Many apps still use it; I say the risk out loud.

**Example:**

API `Set-Cookie: refresh=...; HttpOnly; Secure; SameSite=Lax; Path=/api/auth`. Angular never reads that cookie. `withCredentials: true` on refresh calls. CORS must allow that exact origin, not `*`.

**Real-world example:**

Incident SPA on `app.company.com`, API on `api.company.com`. We used SameSite and a specific CORS origin. Access token stayed in a service signal. Header still showed the user after F5 because `/refresh` used the cookie.

**Cross-question:** Put JWT in the SignalR query string?

**Cross-answer:**

Browsers cannot set headers on WebSocket easily, so people pass `?access_token=`. It leaks in logs. Keep it short-lived, never log the full URL, prefer a negotiate step that uses the header.

---

## Q5. Login, refresh, logout — write the three endpoints.

**Answer:**

- `POST /api/auth/login` — credentials → access + refresh (and set cookie if that is the design). Tenant from host/claim map, not from a free body field.
- `POST /api/auth/refresh` — valid refresh → new pair; rotate DB row.
- `POST /api/auth/logout` — revoke refresh row (this device) or all rows (logout everywhere).

Access token is **not** stored, so logout cannot kill it instantly. That is why access life is **short**. For instant kill you need a denylist (state) or wait for expiry.

**Example:**

Logout everywhere:

```csharp
await db.RefreshTokens
    .Where(x => x.UserId == userId && x.RevokedAt == null)
    .ExecuteUpdateAsync(s => s.SetProperty(x => x.RevokedAt, DateTimeOffset.UtcNow), ct);
```

**Real-world example:**

User clicked “log out all sessions” after a shared kiosk. We revoked refresh rows. Old access JWTs died within 10 minutes. Support accepted that window.

**Cross-question:** Why not 8-hour access JWT and no refresh?

**Cross-answer:**

Stolen access token works for 8 hours with no revoke story. Short access + refresh is the usual compromise.

---

## Q6. How did you implement refresh tokens in NriCare? Be honest.

**Answer:**

Login is phone + BCrypt password. We return a JWT (~20 minutes) and a refresh token: 64 random bytes, Base64, stored **as the raw string** in `RefreshTokens`, default about 1 hour. Refresh loads that row, checks `IsActive`, starts a transaction, sets `RevokedAt` on the old row, inserts a new refresh with the same `SessionId`, and issues a new JWT.

**Example:**

```csharp
var refreshToken = await _dbContext.RefreshTokens
    .Include(rt => rt.User)
    .FirstOrDefaultAsync(rt => rt.Token == token);

if (refreshToken == null || !refreshToken.IsActive)
    return null;

refreshToken.RevokedAt = DateTime.UtcNow;
var newRefresh = await GenerateRefreshToken(refreshToken.UserId, ipAddress, refreshToken.SessionId!);
```

**Real-world example:**

`AuthService.RefreshTokenAsync` does rotation. Logout only stamps `UserLoginActivity.LogoutTimeUtc`. It does **not** revoke refresh rows, so I would also revoke by `SessionId` on logout.

**Cross-question:** Do you hash the refresh token?

**Cross-answer:**

Not in the current table. I would hash it (SHA-256) and store `TokenHash`, like I described in Q2. If the DB leaks, raw tokens should not work.

---

## Q7. After login, how does the rest of the API know the user?

**Answer:**

The access JWT is still stateless. Each `/api` call: Bearer middleware validates signature and expiry, then `SessionMiddleware` fills `IUserSession` from claims. Repositories never look up the refresh row on a normal GET.

**Example:**

```csharp
user.UserId = userId;
user.UserAccessType = parsedAccessType;
if (parsedAccessType == UserAccessType.SuperAdmin)
    user.DisableMainAssociationFilter = true;
```

**Real-world example:**

That is why a SuperAdmin can see all associations and an NR user cannot. The filter reads the same scoped session.

**Cross-question:** `SaveToken = true` on JWT bearer?

**Cross-answer:**

It stores the token on `HttpContext` so SignalR or later middleware can read it. It is **not** a server-side session table for every API call.
