# .NET — Situational questions

API, EF, auth, tenants. Say the answer first, then point to the example.

---

## Q1. The list endpoint is fast for 10 rows and dies for 200. SQL log shows hundreds of queries. What is it?

**Answer:**

That is the N+1 problem. The code loaded the parents, then loaded the children for each parent in a loop. So one list turns into one query plus one more per row. I fix it with a projection (`Select` into a DTO) or with `Include` / a split query.

In plain words: imagine asking a shop for a list of 200 orders, then phoning them 200 more times to ask "how many items are in this one?". Each call is quick, but together they add up. With 10 rows nobody notices. With 200 rows, every extra trip to the database adds delay, and the page slows to a crawl.

A projection means I ask the database for exactly the shape I need in one go. A DTO (data transfer object) is just a small class that holds only the fields the screen shows. EF Core turns the whole `Select`, including the `Count()`, into one SQL query, so the database does the counting and sends back one small result.

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

Our incident list showed comment counts. The code did a `foreach` and read `incident.Comments.Count` for each one. 200 incidents meant 201 SQL calls. We changed it to one `Select` with `Count()`, and it became a single SQL query.

**Cross-question:** `Include(i => i.Comments)` then `Count` in memory?

**Cross-answer:**

That pulls every comment row into memory just to count them. It is much heavier than doing `Count()` inside the projection.

It is like carrying every box out of a warehouse just to count how many boxes there are. With `Count()` in the projection, the database counts and sends back one number per incident. With `Include`, it sends every comment's full text over the network, and the app then holds all of it in memory.

---

## Q2. User A opens `/api/incidents/123` and sees tenant B’s ticket. How did you let that happen, and how do you fix it?

**Answer:**

That is IDOR (insecure direct object reference). We loaded the record by id only. But having the id does not mean you are allowed to see it. After loading, I check the tenant (and the role), and return 404 or 403 if it does not match.

A tenant is one customer organisation sharing the same app, for example one hospital. IDOR is like a hotel where every room key opens any door as long as you know the room number. Ids are often just 1, 2, 3, so anyone can change `123` to `124` and try. If the server only asks "does this id exist?" and never "does it belong to you?", private data leaks between customers.

Returning 404 ("not found") instead of 403 ("forbidden") is a small extra step. A 403 tells the caller "this record exists, you just can't see it", which is already a hint. A 404 tells them nothing.

**Example:**

```csharp
var incident = await db.Incidents.FirstOrDefaultAsync(i => i.Id == id, ct);
if (incident is null)
    return NotFound();

if (incident.TenantKey != _user.TenantKey)
    return NotFound(); // do not admit it exists
```

If each tenant has its own database, the same rule applies: never query another tenant’s connection.

**Real-world example:**

A support user guessed sequential ids and saw another hospital’s incidents. The fix was to pick the tenant connection from the login claim, never from the request body. We also added tests like “user A cannot read B”.

**Cross-question:** Hide with Angular `*ngIf`?

**Cross-answer:**

No. Someone can still call the API from Postman. The API itself must do the check.

`*ngIf` only hides a button or a section on the screen. That is like hiding the door behind a curtain but leaving it unlocked. Anyone can send the HTTP request directly with a tool like Postman or the browser's dev tools, so the security check has to live on the server.

---

## Q3. A singleton cache holds `DbContext`. After a few hours data is wrong or you get threading errors. Why?

**Answer:**

That is a captive dependency. A scoped `DbContext` got stuck inside a singleton, so it lives forever. `DbContext` is not thread-safe, and it keeps tracking old, stale entities. The fix is to inject `IServiceScopeFactory` and create a new scope for each operation.

Some background. In .NET dependency injection, a **singleton** is created once and shared by the whole app for its whole life. A **scoped** service is created fresh for each HTTP request and thrown away at the end. `DbContext` is meant to be scoped: it is like a notepad for one conversation with the database. It remembers every row it has loaded (this is called tracking).

When a singleton holds a `DbContext`, that one notepad is shared by every request and never thrown away. Two requests can write on it at the same time, which causes threading errors, because it is built for one user at a time. It also keeps old copies of rows it loaded hours ago, so you see stale data. Creating a scope per operation gives each job its own fresh notepad that gets thrown away when the job is done.

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

We had a Hangfire “send reminder” job that injected `AppDbContext` into a singleton processor. We kept getting random “already tracked” errors. Creating a scope per job fixed it.

**Cross-question:** Transient DbContext?

**Cross-answer:**

That is worse. If you resolve it twice in one request, you get two different contexts. I keep it **scoped** for HTTP requests and create a new scope for background jobs.

Transient means "a new one every time someone asks for it". So if a service and a repository in the same request each get their own `DbContext`, a change saved through one is not part of the other's work. You can no longer save related changes together in one `SaveChanges` call. Scoped gives one shared context per request, which is what we want.

---

## Q4. You cached incident counts with key `"counts"`. Tenant B sees tenant A’s numbers. Why?

**Answer:**

The cache key did not include the tenant id, so all tenants shared one entry. I always put the tenant in the key, and the user too if the data is personal.

A cache is like a set of labelled boxes on a shelf. The key is the label. If every customer's numbers go into a box labelled just "counts", whoever fills the box first decides what everyone else sees until it expires. Putting the tenant in the key (like `counts:hospitalA`) gives each customer their own box. This is both a correctness bug and a data leak, because one customer sees another customer's figures.

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

After we added Redis, dashboard KPIs looked “shared” between customers. The key was just `"kpi-open"`. Adding the tenant slug to the key kept each tenant’s numbers separate.

**Cross-question:** Memory cache on two servers?

**Cross-answer:**

Each server has its own memory. If users bounce between nodes, I use Redis. And I still key it by tenant.

With an in-memory cache, server 1 and server 2 each keep their own copy. A user might see one number on one request and a different number on the next, depending on which server answered. Redis is a separate shared cache that all servers read from, so everyone sees the same value.

---

## Q5. JWT is in the Angular app. API has `[Authorize]` but a method still trusts `request.UserId` from JSON. What is wrong?

**Answer:**

The client can send any user id it wants. I take the user id from the **claims** after the token is validated, not from the body.

A JWT (JSON Web Token) is a signed pass the server gives you when you log in. Claims are the facts written inside it, like "user id = 42" or "role = Admin". Because the server signed it, nobody can change those facts without the signature breaking. `[Authorize]` only proves the caller is logged in as *someone*. It does not stop them typing a different user id into the JSON body.

It is like showing your ID card at the door, then writing someone else's name on the form you hand in. The bouncer checked your card, but the form is still a lie. So for "me" actions, I read who you are from the token, never from what you typed.

**Example:**

```csharp
var userId = User.FindFirstValue(ClaimTypes.NameIdentifier);
dto.OwnerId = userId; // ignore body owner id for “me” actions
```

**Real-world example:**

Our “Update my profile” endpoint accepted `userId` in the body. A user changed someone else’s email with it. We stopped reading the body id and used the token’s `sub` claim instead.

**Cross-question:** Admin updating another user?

**Cross-answer:**

Then the target id comes from the route. I check that the caller is an **Admin** and in the same tenant. I don’t just accept any id from the JSON.

So the admin case has two checks. Who is calling still comes from the token, and the role and tenant in the token decide whether they may edit others. Which user they want to edit comes from the URL, and I still check that this user belongs to the admin's tenant.

---

## Q6. `SaveChanges` twice: create incident, then write outbox. Second fails. Incident exists, message never sent. How do you design it?

**Answer:**

I save the incident row and the outbox row in one transaction. A background worker publishes the message later. I don’t call the message bus and then save, and I don’t save and then call the bus without an outbox.

A transaction means "all of these changes happen, or none of them do". The outbox is just a table of "messages still to send". Writing the incident and its outbox row together means you can never end up with one and not the other. Then a background worker reads the outbox, sends each message, and marks it done. If the worker crashes, the row is still there and it tries again later.

Think of it like dropping a letter into the office out-tray at the same moment you file the paperwork. Even if you go home sick, the letter is in the tray and someone will post it. Without the tray, you would have to remember to post it yourself, and if you got interrupted, it would never go.

**Example:**

```csharp
await using var tx = await db.Database.BeginTransactionAsync(ct);
db.Incidents.Add(incident);
db.Outbox.Add(new OutboxMessage { Type = "IncidentCreated", Payload = payload });
await db.SaveChangesAsync(ct);
await tx.CommitAsync(ct);
```

**Real-world example:**

The “ticket created” email sometimes never arrived. The Hangfire enqueue ran after the save, and sometimes the process died in between. With an outbox and a worker, the email became eventually consistent: it may be a bit late, but it always goes out.

**Cross-question:** Two databases?

**Cross-answer:**

You can’t easily share one EF transaction across two databases. So the outbox table must live in the **same** database as the business row.

The whole idea depends on one database promising "both rows or neither". If the outbox sits in a different database, you are back to two separate saves, and one can succeed while the other fails.

---

## Q7. GET `/api/incidents` takes 8 seconds. How do you debug in order?

**Answer:**

I work from the outside in, so I don't spend time fixing the wrong layer:

1. First, is it the server or the browser? I check TTFB (time to first byte) in the Network tab.
2. How many SQL statements run, and how long do they take?
3. Are we pulling too many columns? If so, `Select` into a DTO.
4. Is an index missing? I tell a human DBA; I don’t invent raw SQL in app code if the team doesn’t allow it.
5. Then N+1, then payload size, then JSON serialization.

TTFB is how long the browser waits before the server starts sending anything back. If TTFB is small but the page is still slow, the problem is the download size or the browser. If TTFB is large, the server is slow, so I look at the SQL next. An index is like the index at the back of a book: without it, the database reads every row to find a match. Serialization is turning C# objects into JSON text, which also takes time when the data is large.

**Example:**

```csharp
// log in Development
options.LogTo(Console.WriteLine);
var sql = query.ToQueryString();
```

**Real-world example:**

Our incident list had a TTFB of 8s, but the SQL took only 40ms. The payload included every comment body. A projection without comments brought TTFB down to 80ms. The problem was selecting too much data, not the database server.

**Cross-question:** Add Redis first?

**Cross-answer:**

No. Not until I know the query itself is sensible and the data is keyed by tenant.

Adding a cache before finding the cause just hides the problem. The first request is still slow, the cache may hold far more data than needed, and if the key is wrong it can show one tenant's data to another. I fix the query first, then cache only if it is still needed.

---

## Q8. File upload: user sends `report.pdf.exe`. How do you stop it?

**Answer:**

I don’t trust the file name or the content-type header. I check the size, check the extension against an allow-list, and sniff the real file type if I can. I store the file under a generated name, outside the web root. Downloads go through an action that checks authorization.

Everything about an upload comes from the user, including the name and the content-type header (the label that says "this is a PDF"). Both are easy to fake. An allow-list means "only these extensions are accepted", which is safer than trying to list every bad one. Sniffing means looking at the first bytes of the file, because real PDFs and PNGs start with known patterns.

Storing outside the web root means the web server will never hand the file out or run it directly. It is like keeping parcels from strangers in a locked back room, not on the shop counter. Anyone who wants one has to ask at the desk, and the desk checks who they are first.

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

Someone uploaded a “PDF” that was really an exe with a double extension. We saved files as a random id plus an allowed extension only, and we never executed uploads.

**Cross-question:** `Path.Combine(folder, file.FileName)`?

**Cross-answer:**

A name with `../` can escape the folder. I never use the raw file name as part of the path.

`../` means "go up one folder". A file named something like `../../web.config` could end up written outside the uploads folder and overwrite an important file. This is called path traversal. A generated name like a GUID cannot contain those characters, so the problem goes away.

---

## Q9. CORS works in Swagger but Angular on `:4200` fails. What do you check?

**Answer:**

Swagger usually runs on the same origin as the API. Angular runs on a different origin, so the browser applies CORS. I allow that exact origin. And I don’t use `AllowAnyOrigin` when cookies are involved.

An origin is the combination of scheme, host and port, for example `http://localhost:4200`. If the page and the API differ in any of those, the browser treats it as a cross-origin call. CORS (Cross-Origin Resource Sharing) is how the API tells the browser "pages from this origin are allowed to call me". If the API doesn't say so, the browser blocks the response, even though the server may have handled the request.

It's like a building that only lets in visitors whose names are on the guest list. Swagger is already inside the building, so it walks straight through. Angular is outside on a different port, so its name has to be on the list. Allowing any origin together with cookies would let any website make logged-in calls for the user, which is why I name the exact origin.

**Example:**

```csharp
policy.WithOrigins("http://localhost:4200")
      .AllowAnyHeader()
      .AllowAnyMethod();
```

**Real-world example:**

Developers thought the API was “down”. Postman worked fine, but the browser console showed a CORS error. Adding `localhost:4200` (or using a dev proxy) fixed it. In production we allowed the real HTTPS origin.

**Cross-question:** Is CORS login?

**Cross-answer:**

No. CORS is just a browser rule about which origins can call the API. You still need authentication.

Only browsers follow CORS. Postman, curl or another server ignore it completely. So CORS decides which websites may call the API from a browser. It does not decide who the user is or what they may do. That is the job of authentication and authorization.

---

## Q10. Two users edit the same incident. Last save wins and overwrites a comment. What do you add?

**Answer:**

I add a concurrency token (a row version). When two people save, the second save gets a concurrency exception, and we tell that user to reload.

Without it, both users load the same version, both make changes, and whoever saves last quietly replaces the other's work. Nobody gets an error, so nobody knows anything was lost.

A row version is a value the database changes automatically every time the row is saved. When EF Core saves, it adds "only if the row version is still what I loaded" to the update. If someone else saved in the meantime, the version no longer matches, no row is updated, and EF throws a `DbUpdateConcurrencyException`. It's like a shared document that says "this file was changed since you opened it" instead of silently replacing the other person's edits.

**Example:**

```csharp
public byte[] RowVersion { get; set; } = [];

// config
builder.Property(x => x.RowVersion).IsRowVersion();
```

**Real-world example:**

Two agents were typing a resolution at the same time. The slower save wiped out the first note. With RowVersion and a “someone else changed this” toast, no notes were lost.

**Cross-question:** Last write wins for a timestamp field?

**Cross-answer:**

That’s fine for something like “last seen at”. It is not fine for money, status, or comments.

For "last seen at", the newest value is the only one that matters, so losing an older one does no harm. For a balance, a status change or a comment, the older save holds real information, and losing it is a bug.

---

## Q11. The same user double-clicks Book. Both requests should not hold the full wallet. How do you handle it?

**Answer:**

It’s the same wallet hit by two HTTP requests. Inside a transaction, I lock the wallet row with `FOR UPDATE`, work out `Balance - active holds`, and then insert the hold. The second request waits for the lock, then sees the first hold and gets “insufficient available balance”.

Why this matters: without a lock, both requests read the balance at the same moment, both see enough money, and both create a hold. The user has now reserved twice what they actually have. This is a race condition, where the result depends on which request happens to get there first.

`FOR UPDATE` is a PostgreSQL instruction that says "I'm about to change this row, nobody else may lock it until I'm done". It's like a single bathroom key at a petrol station. The second person has to wait until the first gives the key back, and by then the first hold already exists, so the second check sees the reduced available balance. A hold is money set aside for a booking but not yet paid out.

**Example:**

```csharp
var wallet = await _context.Wallets
    .FromSqlRaw(
        "SELECT * FROM \"Wallets\" WHERE \"UserId\" = {0} AND NOT \"IsDeleted\" FOR UPDATE",
        userId)
    .FirstOrDefaultAsync();

var activeHolds = await _context.HoldBalances
    .Where(h => h.WalletId == wallet.Id && h.Status == HoldStatus.Active)
    .SumAsync(h => (decimal?)h.Amount) ?? 0m;

if (wallet.Balance - activeHolds < totalAmount)
    return ApiResponse.BadRequest("Insufficient available balance.");
```

**Real-world example:**

NriCare’s booking create does exactly this. The available amount is not just `Wallet.Balance`. Holds sit in `HoldBalances` with status Active, and we subtract them.

**Cross-question:** In-memory lock instead?

**Cross-answer:**

`SemaphoreSlim` only works within one server. With two API instances, both requests could pass. The database lock is the right place for this.

`SemaphoreSlim` is a lock that lives in the memory of one running app. If a load balancer sends the two clicks to two different servers, each server has its own lock, and neither knows about the other. The database is the one place both servers share, so that's where the lock has to be.

---

## Q12. Payment is captured at Razorpay, then the API crashes before wallet credit. What is the user state?

**Answer:**

Razorpay has the money, but the wallet is not credited. `RazorPayPaymentLog` may still be pending. There is no webhook to finish the job. If the user taps verify again, capture may say “already captured”, but `AddMoneyAsync` can run again and credit twice. I would store the Razorpay payment id as unique and credit only once.

In plain words, the user has paid but their wallet shows nothing, which is the worst kind of bug for trust. "Capture" means telling Razorpay to actually take the money. A webhook is Razorpay calling our server back to say "this payment succeeded". Without one, nothing finishes the job after a crash unless the user retries.

The retry path has the opposite risk: crediting twice. Storing the Razorpay payment id with a unique rule means the database itself refuses a second credit row for the same payment. This makes the credit idempotent, meaning running it again has no extra effect.

**Example:**

```csharp
// risky: latest log with no payment id, any user
var log = await _context.RazorPayPaymentLogs
    .Where(x => x.RazorpayPaymentId == null)
    .OrderByDescending(x => x.Id)
    .FirstOrDefaultAsync();
```

**Real-world example:**

In `PaymentController`, verify captures the payment and then separately calls `AddMoneyAsync`. Those two steps are not one transaction with Razorpay.

**Cross-question:** Outbox?

**Cross-answer:**

After capture, I write a “credit wallet” record in the same DB as the log, and a job does the credit. Or I use a Razorpay webhook that is idempotent on the `pay_...` id, so running it twice is safe.

Either way, the goal is that once Razorpay has the money, something reliable finishes the credit even if the API crashes, and the `pay_...` id makes sure it only happens once.

---

## Q13. Background email job runs twice. Does the user get two emails?

**Answer:**

Yes, it can happen. `EmailJob` always inserts a new `EmailLog` and sends. It swallows exceptions, so TickerQ may **not** retry when sending fails — but if the job gets enqueued twice, it sends twice. Also, document expiry schedules a new daily ticker every time the app starts.

"Swallows exceptions" means the `catch` block logs the error and then carries on as if all was fine. TickerQ, the job scheduler, only retries a job when it sees it fail, so a swallowed error looks like success. That gives two problems at once. A failed email is never retried, and nothing checks whether this email was already sent, so a job queued twice sends twice.

The daily ticker point is the same kind of issue. If the app adds a new scheduled job on every start without checking for an existing one, each restart adds one more copy.

**Example:**

```csharp
[TickerFunction("SendEmail")]
public async Task SendEmail(TickerFunctionContext<AddEmailLogRequestDto> trequest)
{
    try
    {
        var mailId = await _mailRepository.AddEmailLogs(trequest.Request);
        await _mailHelper.SentEMail(mailId);
    }
    catch (Exception ex)
    {
        _logger.LogError(ex, "Failed to send email to {Recipient}", recipient);
        // swallowed — TickerQ will not retry
    }
}
```

**Real-world example:**

NriCare emails go out through ZeptoMail via this job. If I want retries, I would rethrow after logging. And I would skip the send if a log for that receipt already succeeded.

**Cross-question:** Review-request job?

**Cross-answer:**

That one is safer. `CreateReviewRequestAsync` returns early if a request already exists, so retries are fine.

It checks "have I already done this?" before doing anything. That check is what makes a job safe to run more than once.

---

## Q14. Booking completed, then `ReleasePayment` is called twice. What happens?

**Answer:**

We look for a ledger entry with `IdempotencyKey == "release_{bookingId}"`. If we find one, we return success without paying again. So the second call does nothing — **if** the first one already committed.

An idempotency key is a unique label for one action, here "release the payment for booking X". Before paying, the code checks whether a ledger row with that label already exists. It's like a cashier stamping "PAID" on an invoice: if someone brings the same invoice back, the stamp shows it's already been paid.

The "if" matters. The check only sees the first payment after it has been saved to the database. If the first call hasn't finished saving yet, the second call won't see it.

**Example:**

```csharp
var idempotencyKey = $"release_{bookingId}";
if (await _context.WalletTransactions.AnyAsync(x => x.IdempotencyKey == idempotencyKey))
    return ApiResponse<string>.Success("Already released");
```

**Real-world example:**

Both “complete booking” and “verifier approved” call `ReleasePayment`. This key is how we avoid paying the provider twice.

**Cross-question:** Two requests in the same millisecond?

**Cross-answer:**

Then both can pass the `AnyAsync` check. A unique index on the key is the missing piece.

Both requests ask "does this key exist?" before either has saved, so both hear "no". A unique index makes the database refuse a second row with the same key, so the second insert fails instead of paying twice. The app check is a polite first filter. The unique index is the real guarantee.

---

## Q15. User accepts a quotation that needs extra money. Wallet is short. What do you return?

**Answer:**

Accept runs in `QuotationRepository.ChangeStatusAsync`. Available balance is the balance minus active holds. If `payable` is more than what’s available, we return a 400 that shows available vs required. The booking stays as it was. We don’t accept it and hope they recharge later.

A quotation here is an extra charge on top of the original booking. The reason to refuse straight away is simple. If we accepted it without the money, the work could go ahead with nothing set aside to pay for it, and we'd have to chase the user afterwards. A 400 (bad request) with both numbers tells the user exactly how much more they need to add, so they can top up and try again.

**Example:**

```csharp
if (availableBalance < payable)
    return ApiResponse<string>.BadRequest(
        $"Insufficient balance. Available: {availableBalance}, Required: {payable}");
```

**Real-world example:**

In NriCare, the extra quotation amount becomes another `HoldBalance` on the same booking. It expires in 7 days and uses the key `quotation_hold_{quotationId}_{nrUserId}`.

**Cross-question:** Transaction around accept?

**Cross-answer:**

Not today. I would wrap the hold, the quotation status, and setting the booking to Active in one `BeginTransactionAsync`.

These three changes belong together. Without a transaction, a failure halfway could leave a hold with no accepted quotation, or an accepted quotation with no hold. One transaction means they all save or none do.

---

## Q16. FCM is down. Should booking complete fail?

**Answer:**

No. Money and booking status are the source of truth. The push notification is extra. In the booking and wallet code we wrap `INotificationService` calls in `try/catch` and log the error. The HTTP call still succeeds. The user can still see the in-app `Notification` row if that save worked.

FCM (Firebase Cloud Messaging) is Google's service that sends push notifications to phones. It's a nice extra, like a text message telling you your parcel has arrived. If the text doesn't come, the parcel still arrived. Failing the whole booking because the phone alert failed would punish the user for something outside our control, and could leave money and status in a confusing state.

**Example:**

```csharp
try { await _notificationService.NotifyWalletCredit(userId, amount, txnId); }
catch (Exception ex) {
    _logger.LogError(ex, "Push notification failed for wallet top-up. UserId={UserId}", userId);
}
```

**Real-world example:**

`WalletRepository.AddMoneyAsync` already does this. The booking notification job also keeps the DB insert and the FCM call in two separate try blocks.

**Cross-question:** Hide the failure forever?

**Cross-answer:**

No, I log it. Retrying FCM later would be nice. But I won’t roll back a completed booking just because Firebase had a hiccup.

Logging means we can see how often pushes fail and spot a real outage. The important work is done either way, and the user can still find the update inside the app.

---

## Q17. How would you debug a slow wallet transaction list?

**Answer:**

I check network TTFB first, then the SQL. That endpoint counts incoming and outgoing transactions, sums them, gets the hold total, and then paginates — that’s several round trips. I would look at the indexes on `FromWalletId` / `ToWalletId` / `CreatedDate`, and ask whether we really need all those totals on every page request.

A round trip is one question sent to the database and one answer back. Each one adds waiting time, so a page that needs several of them gets slower as the table grows. Paginating means showing 20 rows at a time instead of all of them. Indexes on the columns we filter and sort by let the database jump straight to the right rows instead of reading the whole table.

The last question is about need. The totals usually don't change when you move from page 1 to page 2, so working them out again on every page is wasted effort.

**Example:**

```csharp
var incoming = await baseQuery.Where(...Credit...).CountAsync();
var outgoing = await baseQuery.Where(...Debit...).CountAsync();
// then SumAsync twice, then paginated Select
```

**Real-world example:**

`GetWalletTransactionAsync` works this way. It’s fine for a small ledger. Under load, I would use one grouped query, or cache the totals for a few seconds per user.

**Cross-question:** `Include` the wallets?

**Cross-answer:**

No. I project just the names in the `Select`. `Include` pulls in full object graphs.

The list only needs to show a name like "From: Ravi". `Include` would load the whole wallet record for every row, with all its fields. Picking just the name in `Select` keeps the query and the response small.

---

## Q18. SuperAdmin recharges a Main Association wallet. What must not happen?

**Answer:**

The association must be approved, the amount must be greater than 0, and the SuperAdmin wallet must have enough to cover it. Then we debit SuperAdmin, credit the Main Association, write a Transfer ledger entry, create a PDF receipt, and send an email. Two recharges running at the same time must not both pass on the same last rupee. Today there is no `FOR UPDATE` on that path, so I would add the same atomic debit we use for subscriptions.

The danger is the same race as a double-click. Two recharges read the SuperAdmin balance at the same time, both see enough, and both take money out. The balance can then go below zero.

An atomic debit fixes it by doing the check and the subtraction in one SQL statement: "reduce the balance by X, but only if the balance is at least X". The database handles that row update as one step, so a second recharge sees the new, lower balance. If it no longer has enough, zero rows are updated and we return an error. It's like a cashier who checks the till and takes the money in the same movement, so nobody can grab the same note in between.

**Example:**

```csharp
var rows = await _context.Wallets
    .Where(w => w.Id == saWalletId && w.Balance >= amount)
    .ExecuteUpdateAsync(s => s.SetProperty(w => w.Balance, w => w.Balance - amount));
if (rows == 0) return BadRequest("Insufficient Super Admin balance");
```

**Real-world example:**

`MainAssociationRepository.RechargeWalletAsync` is the admin recharge, not Razorpay. Payment modes like Bank Transfer / UPI are stored on the ledger.

**Cross-question:** Unique transaction reference?

**Cross-answer:**

The field already exists. A unique index on it would stop duplicate bank references. I would add one.

A bank or UPI reference should only ever be used once. If an admin enters the same reference twice by mistake, the unique index makes the database reject the second entry, so the same bank payment can't be credited twice.

---

## Q19. Chat message is saved, then the API node dies before FCM. Is the chat lost?

**Answer:**

No. The message is already saved in `ChatMessages`, and SignalR has already pushed it to the groups on that node. FCM runs fire-and-forget in a `Task.Run` with a new scope. Offline users might miss the push, but they still load the history through REST. I would not put FCM in the same transaction as the message.

The order is what keeps the chat safe. SignalR is the live connection that delivers messages instantly to people who have the chat open. The database save comes first, so the message is safe before anything else happens. Fire-and-forget means the push notification is started in the background and the code doesn't wait for it. If the server dies at that point, only the phone alert is lost. When the user opens the app, it fetches the saved history and the message is there.

Putting FCM inside the message transaction would make a slow or failed push block or undo saving the message, which is the wrong way round.

**Example:**

```csharp
await _chatRepository.SaveMessageAsync(...);
await Clients.Groups(participantIds).ReceiveMessage(messageDto);
_ = Task.Run(async () => {
    using var scope = _scopeFactory.CreateScope();
    var handler = scope.ServiceProvider.GetRequiredService<IChatNotificationHandler>();
    await handler.HandleNewMessageAsync(...);
});
```

**Real-world example:**

NriCare’s `ChatHub.SendMessage` does exactly this. The online check uses in-memory connections, so a second server may still send FCM. That’s ok.

**Cross-question:** `Clients.All` for the message?

**Cross-answer:**

No. We send to the participants’ user groups. `Clients.All` is only used for online/offline presence, and I would narrow that down too.

`Clients.All` sends to every connected user of the app. For a chat message that would leak a private conversation to everyone. Even for presence, everyone only needs to know about their own contacts, not the whole user base.

---

## Q20. Code review: `FileController` is `[AllowAnonymous]`. What do you say?

**Answer:**

It means anyone can upload, get, and delete files without a token. Anyone who can reach the API can put objects in Spaces or delete them by id. I would put `[Authorize]` back, check that the user owns the file, and allow anonymous access only for a truly public asset if we ever need one.

`[AllowAnonymous]` turns off the login check for the whole controller. It's like leaving the office filing cabinet unlocked in the street: anyone passing can add papers, read them or throw them away. Ids are usually simple numbers, so someone could delete files one after another just by counting. Spaces is the cloud storage where the files actually live, so an open endpoint also means strangers can fill it up at our cost.

**Example:**

```csharp
[AllowAnonymous]
public class FileController : BaseCommonController
{
    [HttpDelete("{id:long}")]
    public async Task<IActionResult> DeleteFile(long id) { ... }
}
```

**Real-world example:**

This is how the NriCare common file API is today. I would treat it as a security fix, not a feature.

**Cross-question:** Presigned download instead of public get?

**Cross-answer:**

Yes, for private documents. There’s already a helper that builds presigned URLs (valid ~50 minutes). But delete still must require authentication.

A presigned URL is a temporary link with a signature built in, like a guest pass that stops working after a set time. The API checks the user first and then hands out the link, so the file can be downloaded straight from storage without making it public forever. That only covers reading, though. Deleting must always go through a logged-in, checked request.


---
## Additional Questions
### Q16. A production endpoint takes 5 seconds. What do you check?
**Answer:** I split the time into four parts: database, external APIs, application code, and serialization. Logs and traces show me where the time goes. Then I fix the real bottleneck.

The idea is to measure before guessing. A trace is a timeline of one request that shows how long each step took, like a receipt that lists every item and its price. If the database part takes 4 of the 5 seconds, tuning the C# code won't help. Fixing the wrong part wastes time and can add new bugs while the real problem stays.

### Q17. A user can access another user's record by changing the id in the URL. What do you do?
**Answer:** I add authorization on the server for that resource. Where possible, the query itself should be scoped to the current user/tenant. I don’t fetch the row first and trust the client.

Scoping the query means the database search itself includes "and it belongs to this user", for example filtering by both the record id and the logged-in user's id. If the record belongs to someone else, the query simply finds nothing. It's like a locker room where your key only opens your own locker, instead of every locker opening and someone checking afterwards whose it was. Anything the browser sends, including the id in the URL, can be changed, so the server has to decide.

### Q18. An API gets many duplicate requests. How do you protect it?
**Answer:** For expensive or sensitive operations, I use idempotency keys, validation, rate limiting where it makes sense, and unique constraints in the database. Just disabling the button on the client is not enough.

Duplicates come from double-clicks, slow networks where the app retries, or scripts calling the API again and again. Each tool covers a different part:

- An idempotency key is a unique id the client sends with the request. If the same key comes again, the server returns the first result instead of doing the work twice.
- Rate limiting caps how many requests one user can make in a short time, like a shop letting in only so many people per minute.
- A unique constraint is a rule in the database that refuses a second row with the same value. It is the final safety net if two requests slip past the other checks together.

Disabling the button only helps honest users on that one screen. A retry from the network or a direct API call skips it completely.
