# How do you optimize a REST API? (.NET)

Interviewers want a **checklist**, then one story from a slow endpoint.

---

## Q1. How do you optimize REST API code in ASP.NET Core?

**Answer:**

I measure first, then fix the expensive part. I don't start with microservices.

Measuring first means I find out where the time actually goes before changing anything. Most slow APIs are slow because of simple things, like too many database queries or too much data in the response. Splitting the app into microservices or adding servers doesn't fix those; it just adds more moving parts. It's like a slow kitchen where the cook walks to the store room for every single ingredient: hiring a second cook won't help as much as bringing everything to the counter once.

This is the order I follow:

1. **Measure** — TTFB, SQL count, SQL duration, payload size. (TTFB is "time to first byte": how long the client waits before the first piece of the response arrives. Payload size is how big the response is.)
2. **Return less** — DTO projection, not full EF entities. No navigation graphs in JSON. (A DTO is a small class with only the fields the screen needs. Navigation graphs are the related objects hanging off an entity, like an incident's comments and their users.)
3. **Hit the database less** — no N+1, filter in SQL (`IQueryable`) before `ToList`, pagination (`Skip/Take` or keyset). (N+1 is one query for the list plus one more query per row.)
4. **Read path** — `AsNoTracking()` for lists. (EF doesn't need to watch read-only data for changes, so this saves memory and time.)
5. **Async I/O** — `ToListAsync`, no `.Result`. (While waiting for the database, the thread is free to serve other requests instead of sitting idle.)
6. **Cache what is safe** — memory/Redis, **key includes tenant** (and user if needed), short TTL. (TTL is "time to live": how long a cached value is kept before it's thrown away.)
7. **HTTP** — compression, `ETag` / `Cache-Control` for GETs that can cache, pagination headers.
8. **Do not over-serialize** — ignore nulls, do not send blobs in the list API (separate download). (A blob is a big chunk of binary data, like a file or image.)
9. **Pool and factory** — `IHttpClientFactory` for outbound calls. (It reuses network connections instead of opening a new one for every call.)
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

The slow version loads every incident and every comment into memory, then counts the comments in C#. The fast version lets the database do the work: it filters by status, sorts, counts the comments, and returns only 20 small rows. Only the three fields the list needs travel over the network.

**Real-world example:**

Our incident list had a TTFB of about 8 seconds. Each SQL query took 40 ms, but it ran **201 times** (N+1 comment counts), and the JSON included the comment bodies. Projection + `AsNoTracking` + a page size of 20 brought TTFB under 100 ms. We did not need to add Kafka.

This shows why measuring matters. Each query on its own looked fast at 40 ms, so looking at one query wouldn't have shown the problem. The real cost was the number of queries, and sending comment text that the list didn't even show.

**Cross-question:** Would you add raw SQL to go faster?

**Cross-answer:**

I stay on LINQ/EF. If a DBA says we need an index, that is a schema follow-up. I don't paste SQL strings into app code when the team forbids it.

LINQ is the C# way of writing queries, and EF turns it into SQL for me. In most cases, well-written LINQ already produces good SQL, so raw SQL isn't where the speed comes from. If the database needs an index to make a query fast, that's a change to the database structure (the schema), not a reason to rewrite the query by hand. And if the team's rule is "no SQL strings in app code", I follow it, because hand-written SQL is harder to maintain and easier to get wrong.

---

## Q2. Pagination: Skip/Take vs keyset?

**Answer:**

`Skip((page-1)*size).Take(size)` is simple, but deep pages (`Skip(100000)`) get slow. Keyset pagination means I continue from the last id I saw: `Where(i => i.Id < lastId).Take(20)`. That stays fast.

Why Skip gets slow: to skip 100,000 rows, the database still has to walk through those 100,000 rows just to throw them away, and then it returns the next 20. The deeper the page, the more work. It's like finding page 500 of a book by flipping one page at a time from the start.

Keyset pagination is more like using a bookmark. The client remembers the last ID it saw, and the next request asks for "the next 20 after this ID". With an index on the ID, the database jumps straight to that spot, so page 500 is about as fast as page 1. The trade-off is that you can't jump directly to a random page number; you move forward from where you are, which suits "load more" and infinite scroll.

**Example:**

```csharp
.Where(i => lastId == null || i.Id < lastId)
.OrderByDescending(i => i.Id)
.Take(20)
```

On the first call `lastId` is null, so it returns the newest 20. On the next call the client sends the smallest ID it got, and the query continues from there.

**Real-world example:**

In the admin screen, “page 500” of the audit logs timed out with Skip. Switching to a cursor (`lastId`) made infinite scroll on the Angular table smooth.

**Cross-question:** Always return total count?

**Cross-answer:**

No. `Count(*)` can be expensive. I return it when the UI needs “Page 3 of 90”. For infinite scroll, `hasMore` is enough.

To count the total, the database often has to go through every matching row, which on a big table can cost more than fetching the page itself. If the screen just shows "load more" as you scroll, all it needs to know is whether there's more data. A common way to get `hasMore` is to ask for one more row than the page size: if it comes back, there's more.

---

## Q3. How do you optimize JSON output?

**Answer:**

I keep DTOs small. I don't serialize loops of entities. For lists, I skip child collections and send only counts. Compression (`UseResponseCompression`) helps. And I don't send the same static payload every time — that's what `ETag` is for.

Serializing is turning C# objects into JSON text. "Loops of entities" means objects that point back to each other, like an incident that has comments, where each comment points back to the incident. If you serialize those directly, you either get an error or a huge response. For a list, the screen usually only needs "5 comments", not the full text of each comment.

Compression shrinks the response before it's sent, like zipping a file, and the browser unzips it automatically. JSON has a lot of repeated text, so it usually shrinks well. An `ETag` is a fingerprint of the response. The browser sends it back next time, and if the data hasn't changed, the server can reply "304 Not Modified" with no body, so the same data isn't downloaded again.

**Example:**

```csharp
builder.Services.AddResponseCompression();
app.UseResponseCompression();
```

The first line registers the compression service, and the second adds it to the request pipeline so responses get compressed.

**Real-world example:**

A dashboard returned 1.4 MB of incidents with nested users. The Angular app felt slow on mobile. A list DTO with 8 fields brought it down to 40 KB.

**Cross-question:** Gzip vs faster CPU?

**Cross-answer:**

For JSON APIs, compression is usually a win. If the CPU is already running hot, I measure first.

Compressing takes some CPU time on the server, but it saves much more time sending data over the network, especially on mobile. So in most cases it's worth it. If the server's CPU is already near its limit, adding compression could slow everything down, so in that case I measure before and after instead of assuming.

---

## Q4. Caching REST GET — what can go wrong?

**Answer:**

A wrong key, like one missing the tenant, leaks data. Caching POST is wrong. Caching user-specific data under a shared key is wrong. I use a short TTL, or explicit invalidation after writes.

A cache key is the label I store a cached value under, like the label on a drawer. If the label doesn't include who the data belongs to, everyone opens the same drawer. In a system used by many customers (tenants), that means one customer can see another's data. POST requests change data, so caching their response makes no sense; each one has to actually run. The same goes for anything personal: if I cache "my profile" under one shared key, the next user sees the first user's profile.

A short TTL means the cached value expires quickly, so stale data doesn't stay around long. Explicit invalidation means that when the data changes, I delete the cached value straight away so the next read gets fresh data.

**Example:**

```csharp
var key = $"incidents:open:{tenant}:{page}";
```

The key includes the tenant and the page number, so each tenant and each page gets its own cached entry.

**Real-world example:**

After we added Redis, hospital A saw hospital B’s open count. The key was `"open-count"`. Putting the tenant in the key fixed it.

**Cross-question:** Cache `[Authorize]` GET in the browser?

**Cross-answer:**

I use `Cache-Control: private`. Never `public` for tenant data on a shared CDN, unless there is a careful design behind it.

`[Authorize]` means the endpoint needs a logged-in user, so its data is personal or tenant-specific. `Cache-Control` is a header that tells browsers and other caches what they may store. "private" means only the user's own browser may keep a copy. "public" means shared caches, like a CDN (a network of servers that stores copies of content close to users), may store it too and hand that copy to other people. For one tenant's data, that could show it to the wrong customer.

---

## Q5. Many small HTTP calls from Angular vs one fat API?

**Answer:**

A chatty UI makes 20 calls to build one screen. Better is one **dashboard** endpoint that returns counts + the latest 5, or a BFF (backend for frontend). I don't bring in GraphQL just to hide 20 REST calls, unless the product needs it.

Every HTTP call has a fixed cost: the request travels to the server, waits, and the response travels back. On a slow mobile network that trip alone can take a noticeable amount of time. Twenty calls means paying that cost twenty times, and the browser can only run a limited number at once. It's like going to the shop twenty times for twenty items instead of once with a list.

A dashboard endpoint gathers everything one screen needs into a single response. A BFF is a backend built for a specific frontend, which shapes the data exactly the way that screen wants it. GraphQL lets the client ask for many things in one query, but it's a big change to how the API works, so I only add it if the product really needs that flexibility.

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

If I make one big endpoint for everything, screens that need only one piece still pay for loading all of it, and the endpoint becomes hard to change because many screens depend on it. So I keep normal list and detail endpoints, and only build a combined one when a screen always loads the same bundle together, like a home dashboard.

---

## Q6. How do you keep APIs fast under many concurrent users?

**Answer:**

Async all the way, no blocking of thread-pool threads, bounded pagination, timeouts on outbound HTTP, a rate limit on login, and connection pooling (EF + HttpClient factory). I scale **out** only after each request is cheap.

The server handles requests using a limited set of worker threads, called the thread pool. Async means that while a request is waiting for the database or another API, its thread goes back to the pool and serves someone else. Blocking means the thread just sits and waits, so under heavy load the pool runs out and new requests queue up. It's like waiters in a restaurant: a good waiter takes your order and serves other tables while the kitchen cooks, instead of standing at the kitchen door.

The rest keeps any single request from getting expensive. Bounded pagination means there's a maximum page size, so nobody can ask for a million rows. Timeouts stop a slow outside service from holding my requests forever. A rate limit on login stops someone from flooding it with attempts. Connection pooling reuses open database and HTTP connections instead of creating new ones every time. Scaling out means adding more servers, and I do that only once each request is efficient; otherwise I'm just paying for more servers to run the same waste.

**Example:**

```csharp
await http.GetAsync(url, ct); // pass cancellation
```

**Real-world example:**

A report used `.Result` on EF. At 9am the thread pool ran out of free threads. Switching to `await` fixed it without adding a server.

`.Result` forces the code to wait for an async task by blocking the thread. With many users at once, many threads were stuck waiting, so there were none left for new requests. With `await`, threads are released while the query runs.

**Cross-question:** `Task.Run` around EF?

**Cross-answer:**

That still ties up a thread. I prefer real async EF.

`Task.Run` moves work onto another thread-pool thread. If that work is a blocking EF call, the other thread now sits and waits instead, so I haven't saved anything; I've just moved the waiting. In a web app it can even make things worse. EF's own async methods, like `ToListAsync`, actually free the thread while waiting for the database.

---

## Q7. How would you optimize this action in a live review?

**Answer:**

I talk through the code out loud:

- Look at `Include` and loops (loading related data, or running a query inside a loop, often means too many queries or too much data)
- Look at `ToList` then `Where` (filtering after loading means the whole table came into memory first)
- Look at returning `Incident` entity (sends every field and related objects instead of a small DTO)
- Look at missing `AsNoTracking` (read-only lists don't need change tracking)
- Look at no paging (the response grows with the table)
- Look at sync-over-async (blocking calls like `.Result` or sync methods tie up threads)

Saying this out loud shows the interviewer how I think, not just the final code. I go through each smell, say why it's a problem, and then say what I'd change.

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

In this code, `ToList()` runs first, so every incident and all their comments come out of the database. Only then does `Where` keep the open ones, in memory. The fix is to move the `Where` before the query runs so the database only sends open incidents, return a small DTO with just the fields the list shows, use `ToListAsync` so the thread isn't blocked, and return one page at a time.

**Real-world example:**

That exact pattern was the first PR I was asked to speed up on an incidents API.

**Cross-question:** Change it in production without a test?

**Cross-answer:**

No. First I add a test that checks only open rows come back. Then I change the query.

The test captures what the endpoint is supposed to do. After I rewrite the query, I run the test again. If it still passes, I know I made it faster without changing the result. Without a test, a small mistake, like a wrong filter, could go out to production and return wrong data.

---

## Q8. How would you speed up NriCare booking or wallet list APIs?

**Answer:**

I'd use the same checklist, applied to this app. Project to a booking DTO (number, status, price, dates) — don't `Include` the fee breakdown, status history, chat, and holds for a list. Use `AsNoTracking()`. Filter by `_userSession` (NR user or service provider) **in SQL**. Use `ApplyPaginationAsync`. For the wallet, don't run four extra Count/Sum queries if the screen only needs the page of rows.

A booking list only shows a summary of each booking, so it only needs those few fields. The fee breakdown, history and chat belong on the booking detail screen. Loading them for every row in a list would multiply the data and the work. Filtering by the logged-in user in SQL means the database only sends that user's bookings, which is faster and also keeps one user's data away from another. Each extra Count or Sum is another trip to the database, so if the screen doesn't show those totals, there's no reason to run them on every call.

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

The key includes the user ID so each user has their own cached balance. Any action that changes the balance has to clear the cached value, otherwise the user might see money that's already been held or spent, and could try to spend it again. Because a mistake here affects real money, I'd only add caching if measurements show the balance query is actually a problem.


---
## Additional Questions
### Q16. How do you reduce API response size?
**Answer:** I return DTOs with only required fields, paginate collections, avoid unnecessary navigation graphs, and compress responses where it makes sense.

In simple terms, I send only what the screen will show. A DTO is a small class with just those fields. Pagination sends a list in pages instead of all at once. Navigation graphs are related objects, like an order's customer and items, and I leave them out unless they're needed. Compression shrinks the response before sending, like zipping a file. Smaller responses load faster, especially on mobile, and cost less server memory and network.

### Q17. What is the N+1 problem in an API?
**Answer:** It's when the API loads a list and then runs another database query for every row. I usually fix it by projecting the fields I need in one query, or with a carefully designed include.

For example, I load 100 incidents (1 query), then for each one I load its comment count (100 more queries). That's 101 queries for one screen. Each query might be quick, but the trips to the database add up, and the number grows with the list. It's like a shopkeeper going to the store room once for each item on your list instead of once for the whole list. Projecting with `Select` lets the database return the incidents and their counts together in a single query.

### Q18. Would you cache every API?
**Answer:** No. I cache data that is expensive and safe to reuse. For wallet or permission data, stale values can be dangerous, so I need a clear invalidation strategy first.

Caching means keeping a copy of a result so I don't have to work it out again, like keeping your most-used files on your desk instead of in the cupboard. It's a good fit for data that's slow to get and doesn't change often, like a list of countries. But a cached copy can go out of date (stale). If a stale wallet balance lets someone spend money they no longer have, or a stale permission lets a user do something they've just lost access to, that's a real problem. So before caching that kind of data, I decide exactly when the cached copy gets cleared.
