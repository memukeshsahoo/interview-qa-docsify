# Refresh token, JWT, and “is it still stateless?”

Say this out loud in an interview. Use the **real-world** part if they ask about your last project.

---

## Q1. How do you implement refresh tokens in a .NET + Angular app?

**Answer:**

I use **two tokens**. Think of it like a hotel. The access token is your room key card: you show it at every door, and it stops working after a short time. The refresh token is your booking at the front desk: when the key card expires, you go to the desk once, they check your booking, and they give you a new card. You don't have to check in again from scratch.

1. **Access token** — a JWT with a short life (5–15 minutes). A JWT (JSON Web Token) is a small signed piece of text that says who the user is, for example their user id and tenant. Angular sends it on every API call as `Authorization: Bearer ...`. The API **only validates** the signature, issuer, audience, and expiry. It does **not** look this token up in the database. That part is **stateless**, which means the server doesn't need to remember anything about the user between calls. The token itself carries everything the API needs.

2. **Refresh token** — a long life (days). It’s random bytes (or a JWT that I still treat as just a handle). Angular sends it **only** to `/api/auth/refresh` (and to logout/revoke). The **auth** side stores a **hash** of it, along with the family id, user, tenant, expiry, and a revoked flag. On every refresh I rotate it: the old token dies and a new pair is issued.

Why two tokens and not one? If I only had one long-lived token, anyone who stole it could use it for days. If I only had one short-lived token, the user would have to log in again every 10 minutes. Two tokens give me both: a short risk window and a smooth experience for the user.

On the Angular side I use an interceptor. An interceptor is a piece of code that sits between the app and every HTTP call, so it can look at each request and response in one place. If a call returns 401 (meaning "you're not authorised", usually because the access token expired), I call refresh **once**, queue up any other 401s, and retry them. If refresh fails, I log the user out.

I never put the refresh token in a query string, and I never log it, because URLs and logs get copied around and stored in many places. For the refresh token I prefer an httpOnly cookie, or memory plus silent refresh. The access token usually lives in memory.

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

**The access token is still stateless.** A normal REST call (`GET /api/incidents`) does **not** touch any session table. The API checks the JWT signature and trusts the claims until the token expires. It's like a bouncer checking the hologram on a ticket: he doesn't phone the box office for every person, he just checks the ticket is real and not expired.

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

I do **not** store every access token. That would make the whole API stateful and slower, because every single request would need a database lookup.

If they push for “pure stateless refresh”: yes, you *can* make the refresh token a JWT and store nothing. But then you **cannot** revoke a stolen refresh token until it expires. For a real product, I don’t do that.

**Follow-up:** Is ASP.NET Core cookie auth stateless?

**Follow-up answer:** No. Cookie auth is a **session** (or a protected cookie that the server issued). The browser sends the cookie automatically, and the server decides what it means. JWT access tokens are the stateless style, where the client sends the token on purpose in a header. Mixing cookies and JWT without a clear plan confuses everyone, because nobody is sure which one actually logged the user in.

---

## Q2. Why hash the refresh token in the database?

**Answer:**

If the database leaks, raw tokens could be used to log in. Anyone holding a copy of the table could just copy a token and pretend to be that user.

A hash is one-way. It turns the token into a fixed string of characters, and you can't turn that string back into the original token. When a token comes in, I hash it and compare it to the stored hash. If they match, the token is valid. It’s the same idea as passwords. This is a random token, not a user password, but I still hash it.

SHA-256 is common for high-entropy tokens like this; passwords still need a slow hasher like bcrypt/argon. "High-entropy" means the token is long and fully random, so nobody can guess it by trying lots of values. Passwords are different: people pick short, guessable words, so we use a deliberately slow hasher to make guessing expensive.

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

I don’t. Access tokens are short-lived and checked by their signature. If you store them, the API has to look them up on every call — and that’s a session, not JWT. You lose the main benefit of JWT (no database hit per request) and you get a very busy table full of tokens that expire in minutes anyway.

---

## Q3. What is refresh token rotation and reuse detection?

**Answer:**

Rotation means each refresh issues a **new** refresh token and **kills** the old one. So a refresh token can only be used once. It's like a single-use voucher: once you've handed it in, it's worthless, and you get a new voucher for next time.

Reuse detection means: if someone sends a refresh token that is **already revoked**, I assume it was stolen and revoke the **whole family** of tokens for that user/device. A "family" is the chain of tokens that came from one login: token 1 was swapped for token 2, token 2 for token 3, and so on.

Why it matters: the real user and a thief can't both keep using the same chain. If the real user refreshed first, the thief's copy is already dead, and the moment the thief tries it, we know something is wrong. If the thief refreshed first, the real user's next refresh uses an old token, and that also trips the alarm. Either way, the whole family gets revoked and the user logs in again.

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

That’s weaker. Sliding expiry means the same refresh token keeps getting its expiry date pushed forward each time it's used. A stolen token keeps working until its long expiry, and you can't even tell that two people are using it. Rotation makes that window much smaller, and it also gives you reuse detection.

---

## Q4. Where does Angular keep the tokens?

**Answer:**

- **Refresh:** an httpOnly Secure cookie (best protection against XSS), or memory. httpOnly means JavaScript on the page can't read the cookie at all; only the browser sends it. Secure means it only goes over HTTPS.
- **Access:** memory (a variable or a signal). When the user reloads the SPA, I get a new access token silently using the refresh cookie.

`localStorage` is easy, but anything in it can be **stolen by XSS**. Many apps still use it; if I do, I call out the risk clearly.

XSS (cross-site scripting) is when an attacker manages to run their own JavaScript inside your page, for example through a bad third-party script or input that wasn't cleaned. That script can read anything JavaScript can read, including `localStorage`. An httpOnly cookie is out of reach for it. Think of `localStorage` as leaving your key under the doormat, and an httpOnly cookie as a key that only the lock itself can see.

**Example:**

The API sends `Set-Cookie: refresh=...; HttpOnly; Secure; SameSite=Lax; Path=/api/auth`. Angular never reads that cookie. Refresh calls use `withCredentials: true`, which tells the browser to include cookies on that call. CORS must allow that exact origin, not `*`. CORS is the browser rule that decides which other websites may call your API; when cookies are involved, the browser won't accept a wildcard, so you have to name the exact site. `Path=/api/auth` means the browser only sends the cookie to the auth endpoints, not to every API call.

**Real-world example:**

Our incident SPA was on `app.company.com` and the API on `api.company.com`. We used SameSite and a specific CORS origin. The access token stayed in a service signal. After F5, the header still showed the user, because `/refresh` used the cookie.

**Cross-question:** Put JWT in the SignalR query string?

**Cross-answer:**

Browsers can’t easily set headers on a WebSocket, so people pass `?access_token=` instead. The problem is it leaks into logs, because web servers and proxies often write the full URL, query string included, into their log files. Keep that token short-lived, never log the full URL, and prefer a negotiate step that uses the header. That way, even if a token does end up in a log, it's already expired by the time anyone reads it.

---

## Q5. Login, refresh, logout — write the three endpoints.

**Answer:**

- `POST /api/auth/login` — takes credentials and returns access + refresh (and sets the cookie if that’s the design). The tenant comes from the host/claim mapping, not from a free field in the body. Otherwise a user could type in another company's tenant and try to log in there.
- `POST /api/auth/refresh` — takes a valid refresh token and returns a new pair; the DB row is rotated.
- `POST /api/auth/logout` — revokes the refresh row (this device) or all rows (log out everywhere).

The access token is **not** stored, so logout can’t kill it instantly. That’s why the access token life is **short**. If you need an instant kill, you need a denylist (which is state), or you wait for it to expire.

In simple terms: logout stops the user from getting any *new* access tokens. The one they already hold keeps working for a few more minutes, like a day pass that's still valid until the end of the day even after you've cancelled your membership.

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

Then a stolen access token works for 8 hours, and you have no way to revoke it. Logout wouldn't help either, because there's nothing on the server to cancel. Short access tokens plus refresh is the usual compromise: the user stays logged in all day, but any single stolen access token is only useful for a few minutes.

---

## Q6. How did you implement refresh tokens in NriCare? Be honest.

**Answer:**

Login is phone + BCrypt password. We return a JWT (~20 minutes) and a refresh token: 64 random bytes in Base64, stored **as the raw string** in `RefreshTokens`, with a default life of about 1 hour. On refresh, we load that row and check `IsActive`. Then we start a transaction, set `RevokedAt` on the old row, insert a new refresh token with the same `SessionId`, and issue a new JWT.

So rotation is already there: each refresh kills the old token and creates a new one. The transaction makes sure both steps happen together. If something fails halfway, we don't end up with the old token revoked and no new one created, which would log the user out for no reason. Keeping the same `SessionId` ties all the tokens from one login together, so they form one family.

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

`AuthService.RefreshTokenAsync` does the rotation. Logout only stamps `UserLoginActivity.LogoutTimeUtc`. It does **not** revoke the refresh rows, so I would also revoke by `SessionId` on logout. Right now, logout records *when* the user left, but the refresh token from that session could still be used until it expires.

**Cross-question:** Do you hash the refresh token?

**Cross-answer:**

Not in the current table. I would hash it (SHA-256) and store a `TokenHash`, like I described in Q2. If the DB leaks, raw tokens should not work. The change is small: hash the token before saving, and hash the incoming token before looking it up.

---

## Q7. After login, how does the rest of the API know the user?

**Answer:**

The access JWT is still stateless. On each `/api` call, the Bearer middleware validates the signature and expiry. Middleware is code that runs on every request before it reaches the controller, like a security desk every visitor passes. Then `SessionMiddleware` fills `IUserSession` from the claims. Claims are the facts stored inside the token, such as the user id and access type. Repositories never look up the refresh row on a normal GET.

So every request carries its own proof of who the user is. The API reads the token, copies the important facts into `IUserSession`, and the rest of the code (services, repositories, filters) reads from `IUserSession` instead of asking the database again. This keeps normal calls fast and keeps the "who is this user" logic in one place.

**Example:**

```csharp
user.UserId = userId;
user.UserAccessType = parsedAccessType;
if (parsedAccessType == UserAccessType.SuperAdmin)
    user.DisableMainAssociationFilter = true;
```

**Real-world example:**

That’s why a SuperAdmin can see all associations and an NR user cannot. The filter reads the same scoped session. "Scoped" here means one `IUserSession` is created per request, so each request only sees its own user.

**Cross-question:** `SaveToken = true` on JWT bearer?

**Cross-answer:**

It stores the token on `HttpContext`, so SignalR or later middleware can read it. It is **not** a server-side session table checked on every API call. The token is only kept for the life of that one request, and then it's gone.


---
## Additional Questions
### Q8. What happens if two requests get 401 at the same time?
**Answer:** I don’t want both of them to refresh on their own. Angular should share one refresh request, queue the failed API calls, and retry them once the new access token is ready.

Here's why. With rotation, a refresh token can only be used once. If two calls both try to refresh with the same token, the first one wins and the second one sends a token that's already revoked. That looks exactly like theft, so reuse detection could revoke the whole family and log the user out. It's like two people in a family both going to the counter to renew the same ticket: one person should go, and everyone else waits for them.

### Q9. Why rotate refresh tokens?
**Answer:** After a refresh, the old token stops working. If someone uses an old token again, I can spot possible token theft and revoke the whole token family.

Without rotation, a refresh token that leaks (from a log, a backup, a shared computer) keeps working quietly until it expires, and I'd never know two people were using it. With rotation, each token is single-use, so a leaked copy is either already dead or it triggers an alarm the moment it's used.

### Q10. Can logout instantly invalidate an access JWT?
**Answer:** Not if the access token is fully stateless. I usually keep it short-lived and revoke the refresh token. An instant denylist for access tokens adds server-side state.

The API checks an access token only by its signature and expiry, so it has no list to remove it from. Logout cuts off the refresh token, which means no new access tokens, and the current one runs out within a few minutes. If a product truly needs instant logout (for example, after a security incident), I'd add a denylist of token ids that the API checks on each call, but then every request has to look something up, and the API is no longer fully stateless.
