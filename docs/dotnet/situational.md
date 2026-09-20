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

---

## Q11. The same user double-clicks Book. Both requests should not hold the full wallet. How do you handle it?

**Answer:**

Same wallet, two HTTP requests. I lock the wallet row with `FOR UPDATE` inside a transaction, compute `Balance - active holds`, then insert the hold. The second waiter sees the first hold and gets “insufficient available balance”.

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

NriCare booking create does this. Available is not `Wallet.Balance` alone. Holds sit in `HoldBalances` with status Active.

**Cross-question:** In-memory lock instead?

**Cross-answer:**

`SemaphoreSlim` only works on one server. Two API instances would both pass. Database lock is the right place.

---

## Q12. Payment is captured at Razorpay, then the API crashes before wallet credit. What is the user state?

**Answer:**

Razorpay has the money. `RazorPayPaymentLog` may still be pending. Wallet is not credited. There is no webhook to finish the job. If the user taps verify again, capture may say “already captured”, but `AddMoneyAsync` can run again and double credit. I would store Razorpay payment id uniquely and credit only once.

**Example:**

```csharp
// risky: latest log with no payment id, any user
var log = await _context.RazorPayPaymentLogs
    .Where(x => x.RazorpayPaymentId == null)
    .OrderByDescending(x => x.Id)
    .FirstOrDefaultAsync();
```

**Real-world example:**

`PaymentController` verify captures, then separately calls `AddMoneyAsync`. Those are not one transaction with Razorpay.

**Cross-question:** Outbox?

**Cross-answer:**

After capture, write “credit wallet” in the same DB as the log, then a job credits. Or a Razorpay webhook that is idempotent on `pay_...`.

---

## Q13. Background email job runs twice. Does the user get two emails?

**Answer:**

Yes, it can. `EmailJob` always inserts a new `EmailLog` and sends. It swallows exceptions, so TickerQ may **not** retry on failure — but a duplicate enqueue will send twice. Document expiry also schedules a new daily ticker on every app start.

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

NriCare emails go ZeptoMail through this job. I would rethrow after logging if I want retries, and skip send if a log for that receipt already succeeded.

**Cross-question:** Review-request job?

**Cross-answer:**

That one is safer. `CreateReviewRequestAsync` returns early if a request already exists. Retries are ok.

---

## Q14. Booking completed, then `ReleasePayment` is called twice. What happens?

**Answer:**

We look for ledger `IdempotencyKey == "release_{bookingId}"`. If found, we return success without paying again. The second call is a no-op **if** the first already committed.

**Example:**

```csharp
var idempotencyKey = $"release_{bookingId}";
if (await _context.WalletTransactions.AnyAsync(x => x.IdempotencyKey == idempotencyKey))
    return ApiResponse<string>.Success("Already released");
```

**Real-world example:**

Complete booking and verifier-approved both call `ReleasePayment`. The key is how we avoid paying the provider twice.

**Cross-question:** Two requests in the same millisecond?

**Cross-answer:**

Both can pass the `AnyAsync` check. Unique index on the key is the missing piece.

---

## Q15. User accepts a quotation that needs extra money. Wallet is short. What do you return?

**Answer:**

Accept runs in `QuotationRepository.ChangeStatusAsync`. Available = balance minus active holds. If `payable` is greater than available, we return 400 with available vs required. Booking stays as it was. We do not accept and hope they recharge later.

**Example:**

```csharp
if (availableBalance < payable)
    return ApiResponse<string>.BadRequest(
        $"Insufficient balance. Available: {availableBalance}, Required: {payable}");
```

**Real-world example:**

NriCare extra quotation amount becomes another `HoldBalance` on the same booking, expiry 7 days, key `quotation_hold_{quotationId}_{nrUserId}`.

**Cross-question:** Transaction around accept?

**Cross-answer:**

Not today. I would wrap hold + quotation status + booking Active in one `BeginTransactionAsync`.

---

## Q16. FCM is down. Should booking complete fail?

**Answer:**

No. Money and booking status are the source of truth. Push is extra. In booking/wallet code we `try/catch` around `INotificationService` and log. The HTTP call still succeeds. The user can see the in-app `Notification` row if that save worked.

**Example:**

```csharp
try { await _notificationService.NotifyWalletCredit(userId, amount, txnId); }
catch (Exception ex) {
    _logger.LogError(ex, "Push notification failed for wallet top-up. UserId={UserId}", userId);
}
```

**Real-world example:**

`WalletRepository.AddMoneyAsync` already does this. Booking notification job also isolates DB insert vs FCM in two try blocks.

**Cross-question:** Hide the failure forever?

**Cross-answer:**

Log it. A later retry on FCM is nice. Do not roll back a completed booking because Firebase blinked.

---

## Q17. How would you debug a slow wallet transaction list?

**Answer:**

Network TTFB first. Then SQL. That endpoint counts incoming, outgoing, sums, hold total, then paginates — several round trips. I would look at indexes on `FromWalletId` / `ToWalletId` / `CreatedDate`, and whether we need all those totals on every page request.

**Example:**

```csharp
var incoming = await baseQuery.Where(...Credit...).CountAsync();
var outgoing = await baseQuery.Where(...Debit...).CountAsync();
// then SumAsync twice, then paginated Select
```

**Real-world example:**

`GetWalletTransactionAsync` does this. Fine for a small ledger. Under load I would one grouped query or cache the totals for a few seconds per user.

**Cross-question:** `Include` the wallets?

**Cross-answer:**

No. Project names in `Select`. `Include` pulls full graphs.

---

## Q18. SuperAdmin recharges a Main Association wallet. What must not happen?

**Answer:**

The association must be approved. Amount > 0. SuperAdmin wallet must cover it. We debit SA, credit MA, write a Transfer ledger, PDF receipt, email. Two concurrent recharges should not both pass on the same last rupee — today there is no `FOR UPDATE` on that path, so I would add the same atomic debit as subscriptions.

**Example:**

```csharp
var rows = await _context.Wallets
    .Where(w => w.Id == saWalletId && w.Balance >= amount)
    .ExecuteUpdateAsync(s => s.SetProperty(w => w.Balance, w => w.Balance - amount));
if (rows == 0) return BadRequest("Insufficient Super Admin balance");
```

**Real-world example:**

`MainAssociationRepository.RechargeWalletAsync` is the admin recharge, not Razorpay. Modes like Bank Transfer / UPI are stored on the ledger.

**Cross-question:** Unique transaction reference?

**Cross-answer:**

The field exists. A unique index would stop duplicate bank refs. I would add it.

---

## Q19. Chat message is saved, then the API node dies before FCM. Is the chat lost?

**Answer:**

The message is already in `ChatMessages`. SignalR already pushed to groups on that node. FCM is fire-and-forget `Task.Run` with a new scope. Offline users might miss the push, but they still load history from REST. I would not put FCM in the same transaction as the message.

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

NriCare `ChatHub.SendMessage` does exactly this. Online check uses in-memory connections, so a second server may still send FCM, which is ok.

**Cross-question:** `Clients.All` for the message?

**Cross-answer:**

No. We send to participant user groups. `Clients.All` is only used for online/offline presence, which I would also scope down.

---

## Q20. Code review: `FileController` is `[AllowAnonymous]`. What do you say?

**Answer:**

Upload, get, and delete files without a token. Anyone who can hit the API can put objects in Spaces or delete by id. I would put `[Authorize]` back, check the user owns the file, and keep anonymous only for a true public asset if we ever need one.

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

This is the current NriCare common file API. I would treat it as a security fix, not a feature.

**Cross-question:** Presigned download instead of public get?

**Cross-answer:**

Yes for private docs. Helper already builds presigned URLs (~50 minutes). The delete still must be authenticated.


---
## Additional Questions
### Q16. A production endpoint takes 5 seconds. What do you check?
**Answer:** I split the time into database, external APIs, application code and serialization. Logs/traces tell me where the time is going; then I optimize the actual bottleneck.

### Q17. A user can access another user's record by changing the id in the URL. What do you do?
**Answer:** I add server-side resource authorization. The query itself should be scoped to the current user/tenant where possible, not fetch the row first and trust the client.

### Q18. An API gets many duplicate requests. How do you protect it?
**Answer:** For expensive or sensitive operations I use idempotency keys, validation, rate limiting where appropriate, and database uniqueness constraints. Client-side button disabling alone is not enough.
