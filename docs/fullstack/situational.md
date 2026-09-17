# Full-stack situational questions

Angular + ASP.NET Core together. Each one is a story you can tell in an interview.

---

## Q1. List is empty in Angular. API returns 200. Walk through debug.

**Answer:**

Open Network. Check URL, status, JSON.

- Body has rows, UI empty → mapping, `*ngIf`, wrong property names (`title` vs `Title`).
- Body `[]` → tenant, filter, user. Debug the API next.
- CORS error → not an empty list; the call never succeeded.

**Example:**

C# DTO `Title`, JSON default is `title` in newer System.Text.Json. Angular interface expects `Title` — mismatch.

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

Incident dashboard cards were 0. Network showed `[]`. Tenant claim was missing after refresh. API fail-closed to empty instead of 401. We returned 401 when tenant was missing so the UI could log in again.

**Cross-question:** First tool?

**Cross-answer:**

Network tab, not a rewrite of the component.

---

## Q2. How do you establish a relation between two unrelated Angular components **and** keep the API honest?

**Answer:**

UI: shared service or URL. API: still authorize. A service is not security. If A “selects” incident 12, B may show it only after GET `/api/incidents/12` succeeds for **this** user.

**Example:**

A writes `selectedId` to a service. B calls the API with that id. If 404, B shows empty, not cached data from A.

**Real-world example:**

A malicious user types another tenant’s id in a service (or query string). B must not show a DTO that A stuffed in memory. B loads from the API.

**Cross-question:** Pass the whole object from A to B in the service to skip HTTP?

**Cross-answer:**

OK as a cache for the same user, same tenant. Refresh and deep link still need GET by id.

---

## Q3. Login from Angular button to database. Where can it leak?

**Answer:**

Password in query string, logs, error payloads, or JWT in the URL. HTTPS off. Token in `console.log`. Tenant from the body.

**Example:**

```http
POST /api/auth/login
{ "email": "a@b.com", "password": "..." }
```

Server hashes, compares, returns token in JSON or httpOnly cookie. Angular stores as agreed. Later: `Authorization: Bearer` header. API uses claims.

**Real-world example:**

A gateway logged the full request body and passwords landed in Seq. We redacted `password` and never logged `Authorization`.

**Cross-question:** JWT in localStorage?

**Cross-answer:**

Common; XSS can steal it. Know the trade vs httpOnly cookies + CSRF.

---

## Q4. Create incident in Angular, then dashboard counts. Both must be tenant-safe.

**Answer:**

POST with `[Authorize]`. Server sets tenant from context, not from JSON. Counts query the same tenant. Angular service may fire `changed`; dashboard refetches counts. Cache keys include tenant.

**Example:**

```csharp
incident.TenantKey = _user.TenantKey; // ignore dto.TenantKey
```

**Real-world example:**

KPI “open = 5” after create still showed 4 until refresh — dashboard never subscribed to `changed`. We emitted after POST 201 and reloaded counts. Tenant key on cache stopped a later Redis mix-up.

**Cross-question:** Count in Angular from the list in memory?

**Cross-answer:**

Wrong if the list is paged. Count on the server.

---

## Q5. File upload from Angular to .NET. Situational checks?

**Answer:**

`FormData` + `HttpClient`. API: size, extension, generated name, permission on the parent incident. Do not put files in git or public `wwwroot` if they are private.

**Example:**

```typescript
const fd = new FormData();
fd.append('file', file);
this.http.post(`/api/incidents/${id}/files`, fd);
```

**Real-world example:**

Users attached photos to a ticket. First version used the original file name and one user uploaded `../../web.config`. We now save `guid + ext` under a non-web folder and download through an authorized action.

**Cross-question:** Progress bar?

**Cross-answer:**

`HttpClient` `reportProgress` + `observe: 'events'`. Still the same security checks.

---

## Q6. Slow page: is it Angular or the API?

**Answer:**

Network TTFB high → API/SQL. Huge download → payload or bundle. CPU in Performance → list render (`trackBy`, virtual scroll). Guessing “Angular is slow” without the waterfall wastes time.

**Example:**

Waterfall: TTFB 3s, download 20ms → fix SQL/projection. TTFB 40ms, download 8MB → DTO too fat. TTFB 40ms, download 30kb, FPS drops → `@for` track id / virtual scroll.

**Real-world example:**

Incident board felt laggy. API was 50ms. 3,000 DOM rows without `track`. Virtual scroll + `track id` made it smooth. We almost added Redis for no reason.

**Cross-question:** First fix in a 30-minute interview task?

**Cross-answer:**

Measure, then the biggest bar in the waterfall.

---

## Q7. 401 after 20 minutes while the user is typing a form.

**Answer:**

Access token expired. Interceptor should refresh **once**, retry the save, or redirect to login without losing the form if you cached it. Do not fire two refreshes.

**Example:**

Queue 401s on one `refresh$`. Retry original POST. On refresh fail, store form in `sessionStorage` (no passwords) and go to login.

**Real-world example:**

Long incident form. Token TTL 15 minutes. Save returned 401 and wiped the form. We added refresh + “session expired, log in, we kept your draft.”

**Cross-question:** Put the access token in the URL to avoid expiry?

**Cross-answer:**

No. URLs leak via logs and history.

---

## Q8. Angular shows 403, Swagger with the same user can call it. What is different?

**Answer:**

Different token, different header, different tenant host, or Swagger hits another environment. Compare the `Authorization` header (do not paste secrets). Compare origin and route.

**Example:**

Angular interceptor attached the token only to `/api`. A call to `/incidents` missed the prefix and went to the Angular host — 403/404 from the SPA server, not the API.

**Real-world example:**

`environment.apiUrl` missing slash: `https://api.site.com` + `incidents` vs `https://api.site.com/api/incidents`. Swagger used the full path. We fixed `apiUrl` and interceptor.

**Cross-question:** CORS only on Swagger?

**Cross-answer:**

Swagger same origin. Angular not. Different issue from 403.

---

## Q9. You must show live “ticket taken” on two agents’ boards.

**Answer:**

SignalR hub on the API, one connection service in Angular. Server broadcasts to the **tenant group** only. Boards update a signal. Auth on the hub.

**Example:**

```csharp
await _hub.Clients.Group(tenantKey).SendAsync("taken", incidentId, agentName);
```

```typescript
conn.on('taken', (id, name) => this.board.markTaken(id, name));
```

**Real-world example:**

Two agents opened the same new ticket. Both clicked Take. Without live updates they both thought they owned it. Hub + row version on the API allowed only one Take; the other board moved the card.

**Cross-question:** Poll every 2 seconds instead?

**Cross-answer:**

OK for a small team. Live Take needs faster feedback; SignalR fits.

---

## Q10. Product asks “just send TenantId from Angular, easier.” What do you say?

**Answer:**

No. Tenant comes from the authenticated host or a claim the **server** issued. Client `TenantId` is spoofable. Unrelated components sharing a `TenantService` in Angular is UX only, not isolation.

**Example:**

```csharp
var tenant = _tenantContext.Slug; // from validated claim / host map
// never: var tenant = dto.TenantId;
```

**Real-world example:**

A tester changed `tenantId` in DevTools and saw another org’s list until the API ignored the body and used the token. The Angular service still stores tenant for the UI label only.

**Cross-question:** Super-admin switching tenant?

**Cross-answer:**

A **server** endpoint “act as tenant X” after a permission check, then a new token or server-side session. Not a free field on every DTO.

---

## Q11. Wallet shows 500, booking says insufficient for 400. How do you debug?

**Answer:**

The card is **available** balance: `Balance - active holds`. The table column `Balance` is gross. If 1000 sits in Balance and 600 is held, available is 400. Check `HoldBalances` for Active rows on that wallet, including expiry. Then confirm the UI bound `available` vs `balance`.

**Example:**

```csharp
Balance = result.WalletBalance - result.HoldAmount
```

**Real-world example:**

NriCare `GetWalletBalanceAsync` returns that subtracted number. The transactions page also returns `TotalHoldAmountCount`. A “wrong wallet” bug is often a hold the UI did not show.

**Cross-question:** Expired hold still blocking book?

**Cross-answer:**

It can. Booking create sums all Active holds and ignores `ExpiresAt`. The balance API skips expired ones. That mismatch is a real debug story.

---

## Q12. Angular got 401 while the user was on the booking chat. What should happen?

**Answer:**

Access JWT is ~20 minutes. Chat hub also used `access_token` on the socket. Interceptor should refresh once, retry REST, and reconnect SignalR with the new token. If refresh fails, keep the typed message in memory and send them to login.

**Example:**

`POST /api/Auth/refresh-token` with the refresh string. Old refresh row is revoked. New pair comes back.

**Real-world example:**

NriCare refresh rotates in a transaction. If two tabs refresh the same token, the second should fail because the first revoked it. That is rotation. The user must log in again on the losing tab.

**Cross-question:** Put refresh token on the SignalR URL?

**Cross-answer:**

No. Only the short access token, and do not log the full hub URL.

---

## Q13. Service provider marks complete on mobile. Association dashboard should show money moved. It does not. Where do you look?

**Answer:**

Network: did `change-status` return success? Then `ReleasePayment` — holds Released, user Balance down, SP/platform/MA ledgers created. Association filter: MA user should see their wallet, SuperAdmin sees all. If FCM failed, the screen just did not refresh — pull-to-refresh should still load the new ledger from GET wallet.

**Example:**

Complete with no verifier → `Completed` + `ReleasePayment`. With verifier → `UnderReview` and **no** payout yet.

**Real-world example:**

That verifier branch surprises people. Money moves later when verification is approved. Reject uses the refund path (`ReleaseHold`).

**Cross-question:** Count the payout in Angular from the old booking object?

**Cross-answer:**

No. Reload wallet from the API. The in-memory booking DTO does not contain the split.
