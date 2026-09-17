# How do you optimize a REST API? (.NET)

Interviewers want a **checklist**, then one story from a slow endpoint.

---

## Q1. How do you optimize REST API code in ASP.NET Core?

**Answer:**

I do not start with microservices. I measure, then fix the expensive part.

Order I use:

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

Incident list TTFB was ~8 seconds. SQL was 40 ms but ran **201 times** (N+1 comment counts) and JSON included comment bodies. Projection + `AsNoTracking` + page size 20 brought TTFB under 100 ms. We did not add Kafka.

**Cross-question:** Would you add raw SQL to go faster?

**Cross-answer:**

I stay on LINQ/EF. If a DBA needs an index, that is a schema follow-up. I do not paste SQL strings in app code when the team forbids it.

---

## Q2. Pagination: Skip/Take vs keyset?

**Answer:**

`Skip((page-1)*size).Take(size)` is simple. Deep pages (`Skip(100000)`) get slow. Keyset: `Where(i => i.Id < lastId).Take(20)` stays fast.

**Example:**

```csharp
.Where(i => lastId == null || i.Id < lastId)
.OrderByDescending(i => i.Id)
.Take(20)
```

**Real-world example:**

Admin “page 500” of audit logs timed out with Skip. Cursor (`lastId`) made infinite scroll on the Angular table smooth.

**Cross-question:** Always return total count?

**Cross-answer:**

`Count(*)` can be expensive. Return it when the UI needs “Page 3 of 90”. For infinite scroll, `hasMore` is enough.

---

## Q3. How do you optimize JSON output?

**Answer:**

Small DTOs. Do not serialize loops of entities. For lists, no child collections except counts. Compression (`UseResponseCompression`) helps. Don’t send the same static payload every time — `ETag`.

**Example:**

```csharp
builder.Services.AddResponseCompression();
app.UseResponseCompression();
```

**Real-world example:**

Dashboard returned 1.4 MB of incidents with nested users. Angular felt slow on mobile. List DTO of 8 fields → 40 KB.

**Cross-question:** Gzip vs faster CPU?

**Cross-answer:**

For JSON APIs, compression is usually a win. Measure if CPU is already hot.

---

## Q4. Caching REST GET — what can go wrong?

**Answer:**

Wrong key (missing tenant) leaks data. Caching POST is wrong. Caching user-specific data with a shared key is wrong. Short TTL or explicit invalidation after writes.

**Example:**

```csharp
var key = $"incidents:open:{tenant}:{page}";
```

**Real-world example:**

After Redis, hospital A saw hospital B’s open count. Key was `"open-count"`. Tenant in the key fixed it.

**Cross-question:** Cache `[Authorize]` GET in the browser?

**Cross-answer:**

`Cache-Control: private`. Never `public` for tenant data on a shared CDN without a careful design.

---

## Q5. Many small HTTP calls from Angular vs one fat API?

**Answer:**

Chatty UI: 20 calls to build a screen. Better: one **dashboard** endpoint that returns counts + latest 5, or BFF. Do not invent GraphQL only to hide 20 REST calls unless the product needs it.

**Example:**

```csharp
GET /api/dashboard/summary
→ { open, closed, latest: [ ... ] }
```

**Real-world example:**

Home screen called counts, latest, flags, profile separately. Four round trips on mobile. One summary DTO cut wait time.

**Cross-question:** Always one big endpoint?

**Cross-answer:**

No. Reuse list/detail. Combine only when the screen always needs the bundle.

---

## Q6. How do you keep APIs fast under many concurrent users?

**Answer:**

Async all the way, no thread-pool blocking, bounded pagination, timeouts on outbound HTTP, rate limit login, connection pooling (EF + HttpClient factory). Scale **out** only after the request is cheap.

**Example:**

```csharp
await http.GetAsync(url, ct); // pass cancellation
```

**Real-world example:**

A report used `.Result` on EF. At 9am the thread pool starved. `await` fixed it without a new server.

**Cross-question:** `Task.Run` around EF?

**Cross-answer:**

That still uses a thread. Prefer real async EF.

---

## Q7. How would you optimize this action in a live review?

**Answer:**

Talk through the code out loud:

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

Filter in SQL, project DTO, async, paging, no Include of comments for a list.

**Real-world example:**

That exact pattern was the first PR I was asked to speed up on an incidents API.

**Cross-question:** Change it in production without a test?

**Cross-answer:**

I add a test that open-only rows return, then change the query.
