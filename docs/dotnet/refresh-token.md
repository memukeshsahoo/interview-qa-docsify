# Refresh token, JWT, and “is it still stateless?”

Say this out loud in an interview. Use the **real-world** part if they ask about your last project.

---

## Q1. How do you implement refresh tokens in a .NET + Angular app?

**Answer:**

I use **two tokens**.

1. **Access token** — a JWT with a short life (5–15 minutes). Angular sends it on every API call as `Authorization: Bearer ...`. The API **only validates** the signature, issuer, audience, and expiry. It does **not** look this token up in the database. That part is **stateless**.

2. **Refresh token** — a long life (days). It’s random bytes (or a JWT that I still treat as just a handle). Angular sends it **only** to `/api/auth/refresh` (and to logout/revoke). The **auth** side stores a **hash** of it, along with the family id, user, tenant, expiry, and a revoked flag. On every refresh I rotate it: the old token dies and a new pair is issued.

On the Angular side I use an interceptor. If a call returns 401, I call refresh **once**, queue up any other 401s, and retry them. If refresh fails, I log the user out.

I never put the refresh token in a query string, and I never log it. For the refresh token I prefer an httpOnly cookie, or memory plus silent refresh. The access token usually lives in memory.

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

In an incident product, agents kept a ticket form open for 40 minutes. The access JWT expired after 15 minutes. Without refresh, Save returned 401 and it looked like the user was “logged out”. With refresh, the token was rotated, Save was retried, and the ticket closed. We kept the refresh hashes in the tenant/auth database, so we could **log out all devices** after a laptop was stolen.

**Cross-question:** If you insert the refresh token in the DB, how is JWT still **stateless**?

**Cross-answer:**

**The access token is still stateless.** A normal REST call (`GET /api/incidents`) does **not** touch any session table. The API checks the JWT signature and trusts the claims until the token expires.

**The refresh token is the stateful exception**, and that’s on purpose. A stateless JWT can’t be revoked early (unless you add a denylist, which is also state). We store refresh tokens so we can:

- rotate them
- revoke one device
- detect reuse (theft)
- force logout

So the design looks like this:

| Piece | Stateful? | Used on |
| --- | --- | --- |
| Access JWT | No | Almost every API call |
| Refresh token row | Yes | Only `/refresh`, `/logout` |

I do **not** store every access token. That would make the whole API stateful and slower.

If they push for “pure stateless refresh”: yes, you *can* make the refresh token a JWT and store nothing. But then you **cannot** revoke a stolen refresh token until it expires. For a real product, I don’t do that.

**Follow-up:** Is ASP.NET Core cookie auth stateless?

**Follow-up answer:** No. Cookie auth is a **session** (or a protected cookie that the server issued). JWT access tokens are the stateless style. Mixing cookies and JWT without a clear plan confuses everyone.

---

## Q2. Why hash the refresh token in the database?

**Answer:**

If the database leaks, raw tokens could be used to log in. A hash is one-way. When a token comes in, I hash it and compare it to the stored hash. It’s the same idea as passwords. This is a random token, not a user password, but I still hash it. SHA-256 is common for high-entropy tokens like this; passwords still need a slow hasher like bcrypt/argon.

**Example:**

```csharp
static string Sha256(string raw)
{
    var bytes = SHA256.HashData(Encoding.UTF8.GetBytes(raw));
    return Convert.ToHexString(bytes);
}
```

**Real-world example:**

Someone restored a backup of the auth table on a dev machine. The hashes were useless as tokens. If we had stored raw refresh tokens, every user session could have been stolen.

**Cross-question:** Can I store the JWT access token in the same table?

**Cross-answer:**

I don’t. Access tokens are short-lived and checked by their signature. If you store them, the API has to look them up on every call — and that’s a session, not JWT.

---

## Q3. What is refresh token rotation and reuse detection?

**Answer:**

Rotation means each refresh issues a **new** refresh token and **kills** the old one. Reuse detection means: if someone sends a refresh token that is **already revoked**, I assume it was stolen and revoke the **whole family** of tokens for that user/device.

**Example:**

```csharp
if (row.RevokedAt is not null)
{
    await RevokeFamily(row.UserId, row.TenantKey, ct);
    return Unauthorized();
}
```

**Real-world example:**

An attacker stole an old refresh token from logs. The user had already refreshed, so that token was revoked. When the attacker used it, the whole family got locked, and we asked the user to log in again on all devices.

**Cross-question:** Sliding expiry without rotation?

**Cross-answer:**

That’s weaker. A stolen token keeps working until its long expiry. Rotation makes that window much smaller.

---

## Q4. Where does Angular keep the tokens?

**Answer:**

- **Refresh:** an httpOnly Secure cookie (best protection against XSS), or memory.
- **Access:** memory (a variable or a signal). When the user reloads the SPA, I get a new access token silently using the refresh cookie.

`localStorage` is easy, but anything in it can be **stolen by XSS**. Many apps still use it; if I do, I call out the risk clearly.

**Example:**

The API sends `Set-Cookie: refresh=...; HttpOnly; Secure; SameSite=Lax; Path=/api/auth`. Angular never reads that cookie. Refresh calls use `withCredentials: true`. CORS must allow that exact origin, not `*`.

**Real-world example:**

Our incident SPA was on `app.company.com` and the API on `api.company.com`. We used SameSite and a specific CORS origin. The access token stayed in a service signal. After F5, the header still showed the user, because `/refresh` used the cookie.

**Cross-question:** Put JWT in the SignalR query string?

**Cross-answer:**

Browsers can’t easily set headers on a WebSocket, so people pass `?access_token=` instead. The problem is it leaks into logs. Keep that token short-lived, never log the full URL, and prefer a negotiate step that uses the header.

---

## Q5. Login, refresh, logout — write the three endpoints.

**Answer:**

- `POST /api/auth/login` — takes credentials and returns access + refresh (and sets the cookie if that’s the design). The tenant comes from the host/claim mapping, not from a free field in the body.
- `POST /api/auth/refresh` — takes a valid refresh token and returns a new pair; the DB row is rotated.
- `POST /api/auth/logout` — revokes the refresh row (this device) or all rows (log out everywhere).

The access token is **not** stored, so logout can’t kill it instantly. That’s why the access token life is **short**. If you need an instant kill, you need a denylist (which is state), or you wait for it to expire.

**Example:**

Logout everywhere:

```csharp
await db.RefreshTokens
    .Where(x => x.UserId == userId && x.RevokedAt == null)
    .ExecuteUpdateAsync(s => s.SetProperty(x => x.RevokedAt, DateTimeOffset.UtcNow), ct);
```

**Real-world example:**

A user clicked “log out all sessions” after using a shared kiosk. We revoked the refresh rows. The old access JWTs expired within 10 minutes. Support was fine with that window.

**Cross-question:** Why not 8-hour access JWT and no refresh?

**Cross-answer:**

Then a stolen access token works for 8 hours, and you have no way to revoke it. Short access tokens plus refresh is the usual compromise.

---

## Q6. How did you implement refresh tokens in NriCare? Be honest.

**Answer:**

Login is phone + BCrypt password. We return a JWT (~20 minutes) and a refresh token: 64 random bytes in Base64, stored **as the raw string** in `RefreshTokens`, with a default life of about 1 hour. On refresh, we load that row and check `IsActive`. Then we start a transaction, set `RevokedAt` on the old row, insert a new refresh token with the same `SessionId`, and issue a new JWT.

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

`AuthService.RefreshTokenAsync` does the rotation. Logout only stamps `UserLoginActivity.LogoutTimeUtc`. It does **not** revoke the refresh rows, so I would also revoke by `SessionId` on logout.

**Cross-question:** Do you hash the refresh token?

**Cross-answer:**

Not in the current table. I would hash it (SHA-256) and store a `TokenHash`, like I described in Q2. If the DB leaks, raw tokens should not work.

---

## Q7. After login, how does the rest of the API know the user?

**Answer:**

The access JWT is still stateless. On each `/api` call, the Bearer middleware validates the signature and expiry. Then `SessionMiddleware` fills `IUserSession` from the claims. Repositories never look up the refresh row on a normal GET.

**Example:**

```csharp
user.UserId = userId;
user.UserAccessType = parsedAccessType;
if (parsedAccessType == UserAccessType.SuperAdmin)
    user.DisableMainAssociationFilter = true;
```

**Real-world example:**

That’s why a SuperAdmin can see all associations and an NR user cannot. The filter reads the same scoped session.

**Cross-question:** `SaveToken = true` on JWT bearer?

**Cross-answer:**

It stores the token on `HttpContext`, so SignalR or later middleware can read it. It is **not** a server-side session table checked on every API call.


---
## Additional Questions
### Q8. What happens if two requests get 401 at the same time?
**Answer:** I don’t want both of them to refresh on their own. Angular should share one refresh request, queue the failed API calls, and retry them once the new access token is ready.

### Q9. Why rotate refresh tokens?
**Answer:** After a refresh, the old token stops working. If someone uses an old token again, I can spot possible token theft and revoke the whole token family.

### Q10. Can logout instantly invalidate an access JWT?
**Answer:** Not if the access token is fully stateless. I usually keep it short-lived and revoke the refresh token. An instant denylist for access tokens adds server-side state.
