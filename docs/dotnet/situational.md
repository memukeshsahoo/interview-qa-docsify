# .NET — Situational questions

API, EF, auth, tenants. Speak the answer, then point at the example.

---

## Q1. The list endpoint is fast for 10 rows and dies for 200. SQL log shows hundreds of queries. What is it?

**Answer:**

N+1. You loaded parents, then for each parent loaded children in a loop. Fix with a projection (`Select` to DTO) or `Include` / split query.

**Example:**

```csharp
var dto = await db.Incidents
    .Where(i => i.Status == "Open")
    .Select(i => new IncidentListDto
    {
        Id = i.Id,
        Title = i.Title,
        CommentCount = i.Comments.Count()
    })
    .ToListAsync(ct);
```

**Real-world example:**

Incident list showed comment counts. The code did `foreach` + `incident.Comments.Count`. 200 incidents = 201 SQL calls. One `Select` with `Count()` became one SQL.

**Cross-question:** `Include(i => i.Comments)` then `Count` in memory?

**Cross-answer:**

Pulls all comment rows. Heavier than `Count()` in the projection.

---

## Q2. User A opens `/api/incidents/123` and sees tenant B’s ticket. How did you let that happen, and how do you fix it?

**Answer:**

IDOR. You loaded by id only. Possession of the id is not permission. After load, check tenant (and role). Return 404 or 403.

**Example:**

```csharp
var incident = await db.Incidents.FirstOrDefaultAsync(i => i.Id == id, ct);
if (incident is null)
    return NotFound();

if (incident.TenantKey != _user.TenantKey)
    return NotFound(); // do not admit it exists
```

If the database **is** the tenant, still do not query another tenant’s connection.

**Real-world example:**

Support guessed sequential ids and saw another hospital’s incidents. Fix: tenant connection from the login claim, never from the body, plus tests “user A cannot read B”.

**Cross-question:** Hide with Angular `*ngIf`?

**Cross-answer:**

No. Postman still works. API must check.

---

## Q3. A singleton cache holds `DbContext`. After a few hours data is wrong or you get threading errors. Why?

**Answer:**

Captive dependency. Scoped `DbContext` became a singleton. It is not thread-safe and it tracks stale entities. Inject `IServiceScopeFactory` and create a scope per operation.

**Example:**

```csharp
public class NightlyJob(IServiceScopeFactory scopes)
{
    public async Task Run(CancellationToken ct)
    {
        await using var scope = scopes.CreateAsyncScope();
        var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        // use db, then dispose scope
    }
}
```

**Real-world example:**

Hangfire “send reminder” job injected `AppDbContext` into a singleton processor. Random “already tracked” errors. Creating a scope per job fixed it.

**Cross-question:** Transient DbContext?

**Cross-answer:**

Worse — two resolves in one request are two contexts. Keep **scoped** for HTTP, new scope for jobs.

---

## Q4. You cached incident counts with key `"counts"`. Tenant B sees tenant A’s numbers. Why?

**Answer:**

Cache key missed tenant id. Always include tenant (and user if the data is personal).

**Example:**

```csharp
var key = $"counts:{_user.TenantKey}";
if (!_cache.TryGetValue(key, out Counts? counts))
{
    counts = await LoadCounts(ct);
    _cache.Set(key, counts, TimeSpan.FromSeconds(30));
}
```

**Real-world example:**

Dashboard KPIs looked “shared” after Redis was added. Key was `"kpi-open"`. Adding tenant slug to the key isolated the numbers.

**Cross-question:** Memory cache on two servers?

**Cross-answer:**

Each server has its own memory. Use Redis if users bounce between nodes, still key by tenant.

---

## Q5. JWT is in the Angular app. API has `[Authorize]` but a method still trusts `request.UserId` from JSON. What is wrong?

**Answer:**

The client can send any user id. Take the id from **claims** after the token is validated.

**Example:**

```csharp
var userId = User.FindFirstValue(ClaimTypes.NameIdentifier);
dto.OwnerId = userId; // ignore body owner id for “me” actions
```

**Real-world example:**

“Update my profile” accepted `userId` in the body. A user changed someone else’s email. We ignored the body id and used the token `sub`.

**Cross-question:** Admin updating another user?

**Cross-answer:**

Then the target id is in the route, and you check **Admin** plus tenant, not “any id in JSON”.

---

## Q6. `SaveChanges` twice: create incident, then write outbox. Second fails. Incident exists, message never sent. How do you design it?

**Answer:**

One transaction: incident row + outbox row. A worker publishes later. Do not call the bus then save, or save then bus with no outbox.

**Example:**

```csharp
await using var tx = await db.Database.BeginTransactionAsync(ct);
db.Incidents.Add(incident);
db.Outbox.Add(new OutboxMessage { Type = "IncidentCreated", Payload = payload });
await db.SaveChangesAsync(ct);
await tx.CommitAsync(ct);
```

**Real-world example:**

Email “ticket created” sometimes never arrived because Hangfire enqueue ran after save and the process died. Outbox + worker made email eventually consistent.

**Cross-question:** Two databases?

**Cross-answer:**

You cannot share one EF transaction easily. Outbox in the **same** DB as the business row.

---

## Q7. GET `/api/incidents` takes 8 seconds. How do you debug in order?

**Answer:**

1. Is it the server or the browser? Network TTFB.
2. How many SQL statements? Duration?
3. Too many columns? `Select` DTO.
4. Missing index (tell a human DBA; do not invent raw SQL in app code if the team forbids it).
5. N+1, then payload size, then JSON.

**Example:**

```csharp
// log in Development
options.LogTo(Console.WriteLine);
var sql = query.ToQueryString();
```

**Real-world example:**

Incident list TTFB was 8s. SQL was 40ms. The payload included every comment body. Projection without comments made TTFB 80ms. The bug was over-select, not the database server.

**Cross-question:** Add Redis first?

**Cross-answer:**

Not before you know the query is honest and keyed by tenant.

---

## Q8. File upload: user sends `report.pdf.exe`. How do you stop it?

**Answer:**

Do not trust the file name or content-type header. Check size, extension allow-list, and sniff the real type if you can. Store a generated name outside the web root. Serve through an authorized download action.

**Example:**

```csharp
var ext = Path.GetExtension(file.FileName).ToLowerInvariant();
if (ext is not ".pdf" and not ".png")
    return BadRequest();

if (file.Length > 5_000_000)
    return BadRequest();

var name = $"{Guid.NewGuid():N}{ext}";
```

**Real-world example:**

A “PDF” upload was an exe with a double extension. We saved as a random id + allowed ext only, and never executed uploads.

**Cross-question:** `Path.Combine(folder, file.FileName)`?

**Cross-answer:**

`../` can escape the folder. Never use the raw name as the path.

---

## Q9. CORS works in Swagger but Angular on `:4200` fails. What do you check?

**Answer:**

Swagger is often the same origin as the API. Angular is another origin. Allow that exact origin. Do not use `AllowAnyOrigin` with cookies.

**Example:**

```csharp
policy.WithOrigins("http://localhost:4200")
      .AllowAnyHeader()
      .AllowAnyMethod();
```

**Real-world example:**

Devs thought the API was “down”. Postman worked. Browser console said CORS. Adding `localhost:4200` (or a dev proxy) fixed it. Production used the real HTTPS origin.

**Cross-question:** Is CORS login?

**Cross-answer:**

No. It is a browser rule. Auth is still required.

---

## Q10. Two users edit the same incident. Last save wins and overwrites a comment. What do you add?

**Answer:**

Concurrency token (row version). Second save gets an exception. Tell the user to reload.

**Example:**

```csharp
public byte[] RowVersion { get; set; } = [];

// config
builder.Property(x => x.RowVersion).IsRowVersion();
```

**Real-world example:**

Two agents typed a resolution. The slower save wiped the first note. RowVersion + “someone else changed this” toast saved the notes.

**Cross-question:** Last write wins for a timestamp field?

**Cross-answer:**

OK for “last seen at”. Not for money, status, or comments.
