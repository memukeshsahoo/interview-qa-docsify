# How do you optimize a REST API? (.NET)

Interviewers want a **checklist**, then one story from a slow endpoint.

---

## Q1. How do you optimize REST API code in ASP.NET Core?

**Answer:**

I measure first, then fix the expensive part. I don't start with microservices.

This is the order I follow:

1. **Measure** — TTFB, SQL count, SQL duration, payload size.
2. **Return less** — DTO projection, not full EF entities. No navigation graphs in JSON.
3. **Hit the database less** — no N+1, filter in SQL (`IQueryable`) before `ToList`, pagination (`Skip/Take` or keyset).
4. **Read path** — `AsNoTracking()` for lists.
5. **Async I/O** — `ToListAsync`, no `.Result`.
6. **Cache what is safe** — memory/Redis, **key includes tenant** (and user if needed), short TTL.
7. **HTTP** — compression, `ETag` / `Cache-Control` for GETs that can cache, pagination headers.
8. **Do not over-serialize** — ignore nulls, do not send blobs in the list API (separate download).
9. **Pool and factory** — `IHttpClientFactory` for outbound calls.
10. **Only then** — extra server, Redis, CQRS. Not first.

**Example (slow list → fast list):**

```csharp
// slow: Include everything, then map in memory
var rows = await db.Incidents.Include(i => i.Comments).ToListAsync();
return rows.Select(i => new { i.Id, i.Title, Count = i.Comments.Count });

// fast: one SQL, no tracking, page
var rows = await db.Incidents
    .AsNoTracking()
    .Where(i => i.Status == status)
    .OrderByDescending(i => i.CreatedAt)
    .Select(i => new IncidentListDto
    {
        Id = i.Id,
        Title = i.Title,
        CommentCount = i.Comments.Count()
    })
    .Skip((page - 1) * 20)
    .Take(20)
    .ToListAsync(ct);
```

**Real-world example:**

Our incident list had a TTFB of about 8 seconds. Each SQL query took 40 ms, but it ran **201 times** (N+1 comment counts), and the JSON included the comment bodies. Projection + `AsNoTracking` + a page size of 20 brought TTFB under 100 ms. We did not need to add Kafka.

**Cross-question:** Would you add raw SQL to go faster?

**Cross-answer:**

I stay on LINQ/EF. If a DBA says we need an index, that is a schema follow-up. I don't paste SQL strings into app code when the team forbids it.

---

## Q2. Pagination: Skip/Take vs keyset?

**Answer:**

`Skip((page-1)*size).Take(size)` is simple, but deep pages (`Skip(100000)`) get slow. Keyset pagination means I continue from the last id I saw: `Where(i => i.Id < lastId).Take(20)`. That stays fast.

**Example:**

```csharp
.Where(i => lastId == null || i.Id < lastId)
.OrderByDescending(i => i.Id)
.Take(20)
```

**Real-world example:**

In the admin screen, “page 500” of the audit logs timed out with Skip. Switching to a cursor (`lastId`) made infinite scroll on the Angular table smooth.

**Cross-question:** Always return total count?

**Cross-answer:**

No. `Count(*)` can be expensive. I return it when the UI needs “Page 3 of 90”. For infinite scroll, `hasMore` is enough.

---

## Q3. How do you optimize JSON output?

**Answer:**

I keep DTOs small. I don't serialize loops of entities. For lists, I skip child collections and send only counts. Compression (`UseResponseCompression`) helps. And I don't send the same static payload every time — that's what `ETag` is for.

**Example:**

```csharp
builder.Services.AddResponseCompression();
app.UseResponseCompression();
```

**Real-world example:**

A dashboard returned 1.4 MB of incidents with nested users. The Angular app felt slow on mobile. A list DTO with 8 fields brought it down to 40 KB.

**Cross-question:** Gzip vs faster CPU?

**Cross-answer:**

For JSON APIs, compression is usually a win. If the CPU is already running hot, I measure first.

---

## Q4. Caching REST GET — what can go wrong?

**Answer:**

A wrong key, like one missing the tenant, leaks data. Caching POST is wrong. Caching user-specific data under a shared key is wrong. I use a short TTL, or explicit invalidation after writes.

**Example:**

```csharp
var key = $"incidents:open:{tenant}:{page}";
```

**Real-world example:**

After we added Redis, hospital A saw hospital B’s open count. The key was `"open-count"`. Putting the tenant in the key fixed it.

**Cross-question:** Cache `[Authorize]` GET in the browser?

**Cross-answer:**

I use `Cache-Control: private`. Never `public` for tenant data on a shared CDN, unless there is a careful design behind it.

---

## Q5. Many small HTTP calls from Angular vs one fat API?

**Answer:**

A chatty UI makes 20 calls to build one screen. Better is one **dashboard** endpoint that returns counts + the latest 5, or a BFF (backend for frontend). I don't bring in GraphQL just to hide 20 REST calls, unless the product needs it.

**Example:**

```csharp
GET /api/dashboard/summary
→ { open, closed, latest: [ ... ] }
```

**Real-world example:**

Our home screen called counts, latest, flags, and profile separately. That was four round trips on mobile. One summary DTO cut the wait time.

**Cross-question:** Always one big endpoint?

**Cross-answer:**

No. I reuse the list and detail endpoints. I combine only when the screen always needs the whole bundle.

---

## Q6. How do you keep APIs fast under many concurrent users?

**Answer:**

Async all the way, no blocking of thread-pool threads, bounded pagination, timeouts on outbound HTTP, a rate limit on login, and connection pooling (EF + HttpClient factory). I scale **out** only after each request is cheap.

**Example:**

```csharp
await http.GetAsync(url, ct); // pass cancellation
```

**Real-world example:**

A report used `.Result` on EF. At 9am the thread pool ran out of free threads. Switching to `await` fixed it without adding a server.

**Cross-question:** `Task.Run` around EF?

**Cross-answer:**

That still ties up a thread. I prefer real async EF.

---

## Q7. How would you optimize this action in a live review?

**Answer:**

I talk through the code out loud:

- Look at `Include` and loops
- Look at `ToList` then `Where`
- Look at returning `Incident` entity
- Look at missing `AsNoTracking`
- Look at no paging
- Look at sync-over-async

**Example they might paste:**

```csharp
[HttpGet]
public List<Incident> Get()
{
    var all = _db.Incidents.Include(x => x.Comments).ToList();
    return all.Where(x => x.Status == "Open").ToList();
}
```

**What I say:**

Filter in SQL, project to a DTO, make it async, add paging, and don't Include comments for a list.

**Real-world example:**

That exact pattern was the first PR I was asked to speed up on an incidents API.

**Cross-question:** Change it in production without a test?

**Cross-answer:**

No. First I add a test that checks only open rows come back. Then I change the query.

---

## Q8. How would you speed up NriCare booking or wallet list APIs?

**Answer:**

I'd use the same checklist, applied to this app. Project to a booking DTO (number, status, price, dates) — don't `Include` the fee breakdown, status history, chat, and holds for a list. Use `AsNoTracking()`. Filter by `_userSession` (NR user or service provider) **in SQL**. Use `ApplyPaginationAsync`. For the wallet, don't run four extra Count/Sum queries if the screen only needs the page of rows.

**Example:**

```csharp
var rows = await _context.Bookings
    .AsNoTracking()
    .Where(b => b.NRUserId == nrUserId)
    .OrderByDescending(b => b.CreatedDate)
    .Select(b => new BookingListDto
    {
        Id = b.Id,
        BookingNumber = b.BookingNumber,
        Status = b.Status,
        Price = b.Price
    })
    .ApplyPaginationAsync(request);
```

**Real-world example:**

NriCare already paginates wallet transactions with a `Select` DTO. The extra totals on that same call are what I would measure first if the page felt slow.

**Cross-question:** Cache the wallet balance?

**Cross-answer:**

Only with a short TTL and the key `wallet-available:{userId}`. I'd invalidate it on hold, release, and top-up. A wrong cache here is a money bug, so I would measure before caching.


---
## Additional Questions
### Q16. How do you reduce API response size?
**Answer:** I return DTOs with only required fields, paginate collections, avoid unnecessary navigation graphs, and compress responses where it makes sense.

### Q17. What is the N+1 problem in an API?
**Answer:** It's when the API loads a list and then runs another database query for every row. I usually fix it by projecting the fields I need in one query, or with a carefully designed include.

### Q18. Would you cache every API?
**Answer:** No. I cache data that is expensive and safe to reuse. For wallet or permission data, stale values can be dangerous, so I need a clear invalidation strategy first.
