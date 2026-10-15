# Full-stack situational questions

Angular + ASP.NET Core together. Each one is a story you can tell in an interview.

---

## Q1. List is empty in Angular. API returns 200. Walk through debug.

**Answer:**

I open the Network tab and check the URL, the status, and the JSON.

- Body has rows but the UI is empty → it's the mapping, an `*ngIf`, or wrong property names (`title` vs `Title`).
- Body is `[]` → check the tenant, the filter, and the user. Debug the API next.
- CORS error → that's not an empty list; the call never succeeded.

**Example:**

The C# DTO has `Title`, but ASP.NET Core sends JSON in camelCase by default, so the API returns `title`. The Angular interface has to use `title` too. If it expects `Title`, the field comes back `undefined` and the list looks empty.

```csharp
public sealed class IncidentListDto
{
    public int Id { get; init; }
    public required string Title { get; init; }
}
```

```typescript
interface IncidentListDto { id: number; title: string; }
```

**Real-world example:**

Our incident dashboard cards showed 0. The Network tab showed `[]`. The tenant claim was missing after a refresh, and the API failed closed by returning an empty list instead of a 401. We changed it to return 401 when the tenant was missing, so the UI could log the user in again.

**Cross-question:** First tool?

**Cross-answer:**

The Network tab, not a rewrite of the component.

---

## Q2. How do you establish a relation between two unrelated Angular components **and** keep the API honest?

**Answer:**

On the UI side, I connect them with a shared service or the URL. On the API side, I still authorize every call, because a service is not security. If A “selects” incident 12, B should show it only after GET `/api/incidents/12` succeeds for **this** user.

**Example:**

A writes `selectedId` to a service. B calls the API with that id. If it gets a 404, B shows an empty state, not cached data from A.

**Real-world example:**

A malicious user could type another tenant's id into a service (or the query string). B must not show a DTO that A stuffed into memory. B loads it from the API.

**Cross-question:** Pass the whole object from A to B in the service to skip HTTP?

**Cross-answer:**

That's OK as a cache for the same user and the same tenant. But a refresh or a deep link still needs a GET by id.

---

## Q3. Login from Angular button to database. Where can it leak?

**Answer:**

It can leak through the password in a query string, in logs, or in error payloads, or a JWT in the URL. Also if HTTPS is off, if the token goes into `console.log`, or if the tenant is taken from the body.

**Example:**

```http
POST /api/auth/login
{ "email": "a@b.com", "password": "..." }
```

The server hashes and compares the password, then returns a token in JSON or in an httpOnly cookie. Angular stores it the way we agreed. Later calls send an `Authorization: Bearer` header, and the API uses the claims.

**Real-world example:**

A gateway logged the full request body, and passwords ended up in Seq. We redacted `password` and stopped logging `Authorization`.

**Cross-question:** JWT in localStorage?

**Cross-answer:**

It's common, but XSS can steal it. You should know the trade-off against httpOnly cookies, which then need CSRF protection.

---

## Q4. Create incident in Angular, then dashboard counts. Both must be tenant-safe.

**Answer:**

The POST has `[Authorize]`. The server sets the tenant from the request context, not from the JSON. The counts query uses the same tenant. The Angular service can fire a `changed` event, and the dashboard refetches its counts. Cache keys include the tenant.

**Example:**

```csharp
incident.TenantKey = _user.TenantKey; // ignore dto.TenantKey
```

**Real-world example:**

After a create, the KPI still showed “open = 4” instead of 5 until a refresh, because the dashboard never subscribed to `changed`. We emitted the event after the POST returned 201 and reloaded the counts. Putting the tenant key on the cache also stopped a later Redis mix-up.

**Cross-question:** Count in Angular from the list in memory?

**Cross-answer:**

That's wrong if the list is paged. I count on the server.

---

## Q5. File upload from Angular to .NET. Situational checks?

**Answer:**

`FormData` with `HttpClient`. The API checks size and extension, saves under a generated name, and checks permission on the parent incident. Private files don't go in git or in the public `wwwroot`.

**Example:**

```typescript
const fd = new FormData();
fd.append('file', file);
this.http.post(`/api/incidents/${id}/files`, fd);
```

**Real-world example:**

Users attached photos to a ticket. The first version used the original file name, and one user uploaded `../../web.config`. Now we save the file as `guid + ext` in a folder outside the web root and serve downloads through an authorized action.

**Cross-question:** Progress bar?

**Cross-answer:**

`HttpClient` with `reportProgress` and `observe: 'events'`. The security checks stay the same.

---

## Q6. Slow page: is it Angular or the API?

**Answer:**

If TTFB (time to first byte) is high in the Network tab, it's the API or SQL. A huge download means the payload or the bundle. High CPU in the Performance panel means list rendering (`trackBy`, virtual scroll). Guessing “Angular is slow” without looking at the waterfall wastes time.

**Example:**

Waterfall: TTFB 3s, download 20ms → fix SQL/projection. TTFB 40ms, download 8MB → DTO too fat. TTFB 40ms, download 30kb, FPS drops → `@for` track id / virtual scroll.

**Real-world example:**

Our incident board felt laggy, but the API took 50ms. The page had 3,000 DOM rows without `track`. Virtual scroll plus `track id` made it smooth. We almost added Redis for no reason.

**Cross-question:** First fix in a 30-minute interview task?

**Cross-answer:**

Measure first, then fix the biggest bar in the waterfall.

---

## Q7. 401 after 20 minutes while the user is typing a form.

**Answer:**

The access token expired. The interceptor should refresh **once** and retry the save, or, if you cached the form, redirect to login without losing it. Don't fire two refreshes.

**Example:**

Queue the 401s on one `refresh$`, then retry the original POST. If the refresh fails, store the form in `sessionStorage` (no passwords) and go to login.

**Real-world example:**

We had a long incident form and a 15-minute token TTL. Save returned 401 and wiped the form. We added refresh and a message: “session expired, log in, we kept your draft.”

**Cross-question:** Put the access token in the URL to avoid expiry?

**Cross-answer:**

No. URLs leak through logs and browser history.

---

## Q8. Angular shows 403, Swagger with the same user can call it. What is different?

**Answer:**

It could be a different token, a different header, a different tenant host, or Swagger pointing at another environment. I compare the `Authorization` header (without pasting secrets anywhere), and I compare the origin and the route.

**Example:**

The Angular interceptor attached the token only to `/api` calls. A call to `/incidents` was missing the prefix, so it went to the Angular host and got a 403/404 from the SPA server, not from the API.

**Real-world example:**

`environment.apiUrl` was missing a slash: `https://api.site.com` + `incidents` instead of `https://api.site.com/api/incidents`. Swagger used the full path. We fixed `apiUrl` and the interceptor.

**Cross-question:** CORS only on Swagger?

**Cross-answer:**

Swagger runs on the same origin as the API, and Angular doesn't. That's a different issue from a 403.

---

## Q9. You must show live “ticket taken” on two agents’ boards.

**Answer:**

I put a SignalR hub on the API and one connection service in Angular. The server broadcasts only to the **tenant group**. The boards update a signal. The hub requires auth.

**Example:**

```csharp
await _hub.Clients.Group(tenantKey).SendAsync("taken", incidentId, agentName);
```

```typescript
conn.on('taken', (id, name) => this.board.markTaken(id, name));
```

**Real-world example:**

Two agents opened the same new ticket and both clicked Take. Without live updates, they both thought they owned it. The hub plus a row version on the API allowed only one Take, and the other agent's board moved the card.

**Cross-question:** Poll every 2 seconds instead?

**Cross-answer:**

That's OK for a small team. But a live Take needs faster feedback, and SignalR fits that.

---

## Q10. Product asks “just send TenantId from Angular, easier.” What do you say?

**Answer:**

I say no. The tenant comes from the authenticated host or from a claim the **server** issued. A `TenantId` sent by the client can be faked. Unrelated components sharing a `TenantService` in Angular is just for the UI, not isolation.

**Example:**

```csharp
var tenant = _tenantContext.Slug; // from validated claim / host map
// never: var tenant = dto.TenantId;
```

**Real-world example:**

A tester changed `tenantId` in DevTools and saw another org's list, until we made the API ignore the body and use the token. The Angular service still stores the tenant, but only for the UI label.

**Cross-question:** Super-admin switching tenant?

**Cross-answer:**

That's a **server** endpoint, “act as tenant X”, behind a permission check, which then issues a new token or server-side session. It's not a free field on every DTO.

---

## Q11. Wallet shows 500, booking says insufficient for 400. How do you debug?

**Answer:**

The card shows the **available** balance, which is `Balance - active holds`. The `Balance` column in the table is the gross amount. If Balance is 1000 and 600 is on hold, available is 400. I check `HoldBalances` for Active rows on that wallet, including expiry. Then I confirm whether the UI is bound to `available` or `balance`.

**Example:**

```csharp
Balance = result.WalletBalance - result.HoldAmount
```

**Real-world example:**

In NriCare, `GetWalletBalanceAsync` returns that subtracted number. The transactions page also returns `TotalHoldAmountCount`. A “wrong wallet” bug is often just a hold the UI didn't show.

**Cross-question:** Expired hold still blocking book?

**Cross-answer:**

It can. Booking create adds up all Active holds and ignores `ExpiresAt`, but the balance API skips expired ones. That mismatch makes a good real debug story.

---

## Q12. Angular got 401 while the user was on the booking chat. What should happen?

**Answer:**

The access JWT lasts about 20 minutes, and the chat hub also used `access_token` on the socket. The interceptor should refresh once, retry the REST call, and reconnect SignalR with the new token. If the refresh fails, keep the typed message in memory and send the user to login.

**Example:**

`POST /api/Auth/refresh-token` with the refresh string. The old refresh row gets revoked, and a new pair comes back.

**Real-world example:**

In NriCare, refresh rotates the token inside a transaction. If two tabs refresh the same token, the second one should fail, because the first one already revoked it. That's what rotation means. The user has to log in again on the tab that lost.

**Cross-question:** Put refresh token on the SignalR URL?

**Cross-answer:**

No. Only the short-lived access token goes there, and I don't log the full hub URL.

---

## Q13. Service provider marks complete on mobile. Association dashboard should show money moved. It does not. Where do you look?

**Answer:**

First the Network tab: did `change-status` return success? Then `ReleasePayment`: holds should be Released, the user's Balance should go down, and the SP/platform/MA ledgers should be created. Then the association filter: an MA user should see their own wallet, and SuperAdmin sees all. If FCM failed, the screen just didn't refresh, and pull-to-refresh should still load the new ledger from GET wallet.

**Example:**

Complete with no verifier → `Completed` + `ReleasePayment`. With a verifier → `UnderReview` and **no** payout yet.

**Real-world example:**

That verifier branch surprises people. The money moves later, when verification is approved. A reject goes through the refund path (`ReleaseHold`).

**Cross-question:** Count the payout in Angular from the old booking object?

**Cross-answer:**

No. Reload the wallet from the API. The in-memory booking DTO doesn't contain the split.


---
## Additional Questions
### Q14. A user sees data from another tenant. What is your first priority?
**Answer:** Stop the access, and check the server-side tenant filter and authorization. I don't rely on Angular hiding the screen. Then I add a regression test for cross-tenant access.
### Q15. The API returns 200 but Angular shows nothing. How do you debug?
**Answer:** I look at the actual response shape and the TypeScript model, then check the mapping and template bindings. I also make sure the request is hitting the environment I expect.
### Q16. The API is correct but the UI feels slow. What do you check?
**Answer:** Network payload, how many rows are rendered, change detection, methods called in templates, images and third-party components. I profile before adding another library.
