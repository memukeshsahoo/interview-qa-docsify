# .NET — Situational questions

API, EF, auth, tenants. Say the answer first, then point to the example.

---

## Q1. The list endpoint is fast for 10 rows and dies for 200. SQL log shows hundreds of queries. What is it?

**Answer:**

That is the N+1 problem. The code loaded the parents, then loaded the children for each parent in a loop. So one list turns into one query plus one more per row. I fix it with a projection (`Select` into a DTO) or with `Include` / a split query.

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

---

## Q2. User A opens `/api/incidents/123` and sees tenant B’s ticket. How did you let that happen, and how do you fix it?

**Answer:**

That is IDOR (insecure direct object reference). We loaded the record by id only. But having the id does not mean you are allowed to see it. After loading, I check the tenant (and the role), and return 404 or 403 if it does not match.

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

---

## Q3. A singleton cache holds `DbContext`. After a few hours data is wrong or you get threading errors. Why?

**Answer:**

That is a captive dependency. A scoped `DbContext` got stuck inside a singleton, so it lives forever. `DbContext` is not thread-safe, and it keeps tracking old, stale entities. The fix is to inject `IServiceScopeFactory` and create a new scope for each operation.

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

---

## Q4. You cached incident counts with key `"counts"`. Tenant B sees tenant A’s numbers. Why?

**Answer:**

The cache key did not include the tenant id, so all tenants shared one entry. I always put the tenant in the key, and the user too if the data is personal.

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

---

## Q5. JWT is in the Angular app. API has `[Authorize]` but a method still trusts `request.UserId` from JSON. What is wrong?

**Answer:**

The client can send any user id it wants. I take the user id from the **claims** after the token is validated, not from the body.

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

---

## Q6. `SaveChanges` twice: create incident, then write outbox. Second fails. Incident exists, message never sent. How do you design it?

**Answer:**

I save the incident row and the outbox row in one transaction. A background worker publishes the message later. I don’t call the message bus and then save, and I don’t save and then call the bus without an outbox.

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

---

## Q7. GET `/api/incidents` takes 8 seconds. How do you debug in order?

**Answer:**

1. First, is it the server or the browser? I check TTFB (time to first byte) in the Network tab.
2. How many SQL statements run, and how long do they take?
3. Are we pulling too many columns? If so, `Select` into a DTO.
4. Is an index missing? I tell a human DBA; I don’t invent raw SQL in app code if the team doesn’t allow it.
5. Then N+1, then payload size, then JSON serialization.

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

---

## Q8. File upload: user sends `report.pdf.exe`. How do you stop it?

**Answer:**

I don’t trust the file name or the content-type header. I check the size, check the extension against an allow-list, and sniff the real file type if I can. I store the file under a generated name, outside the web root. Downloads go through an action that checks authorization.

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

---

## Q9. CORS works in Swagger but Angular on `:4200` fails. What do you check?

**Answer:**

Swagger usually runs on the same origin as the API. Angular runs on a different origin, so the browser applies CORS. I allow that exact origin. And I don’t use `AllowAnyOrigin` when cookies are involved.

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

---

## Q10. Two users edit the same incident. Last save wins and overwrites a comment. What do you add?

**Answer:**

I add a concurrency token (a row version). When two people save, the second save gets a concurrency exception, and we tell that user to reload.

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

---

## Q11. The same user double-clicks Book. Both requests should not hold the full wallet. How do you handle it?

**Answer:**

It’s the same wallet hit by two HTTP requests. Inside a transaction, I lock the wallet row with `FOR UPDATE`, work out `Balance - active holds`, and then insert the hold. The second request waits for the lock, then sees the first hold and gets “insufficient available balance”.

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

---

## Q12. Payment is captured at Razorpay, then the API crashes before wallet credit. What is the user state?

**Answer:**

Razorpay has the money, but the wallet is not credited. `RazorPayPaymentLog` may still be pending. There is no webhook to finish the job. If the user taps verify again, capture may say “already captured”, but `AddMoneyAsync` can run again and credit twice. I would store the Razorpay payment id as unique and credit only once.

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

---

## Q13. Background email job runs twice. Does the user get two emails?

**Answer:**

Yes, it can happen. `EmailJob` always inserts a new `EmailLog` and sends. It swallows exceptions, so TickerQ may **not** retry when sending fails — but if the job gets enqueued twice, it sends twice. Also, document expiry schedules a new daily ticker every time the app starts.

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

---

## Q14. Booking completed, then `ReleasePayment` is called twice. What happens?

**Answer:**

We look for a ledger entry with `IdempotencyKey == "release_{bookingId}"`. If we find one, we return success without paying again. So the second call does nothing — **if** the first one already committed.

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

---

## Q15. User accepts a quotation that needs extra money. Wallet is short. What do you return?

**Answer:**

Accept runs in `QuotationRepository.ChangeStatusAsync`. Available balance is the balance minus active holds. If `payable` is more than what’s available, we return a 400 that shows available vs required. The booking stays as it was. We don’t accept it and hope they recharge later.

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

---

## Q16. FCM is down. Should booking complete fail?

**Answer:**

No. Money and booking status are the source of truth. The push notification is extra. In the booking and wallet code we wrap `INotificationService` calls in `try/catch` and log the error. The HTTP call still succeeds. The user can still see the in-app `Notification` row if that save worked.

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

---

## Q17. How would you debug a slow wallet transaction list?

**Answer:**

I check network TTFB first, then the SQL. That endpoint counts incoming and outgoing transactions, sums them, gets the hold total, and then paginates — that’s several round trips. I would look at the indexes on `FromWalletId` / `ToWalletId` / `CreatedDate`, and ask whether we really need all those totals on every page request.

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

---

## Q18. SuperAdmin recharges a Main Association wallet. What must not happen?

**Answer:**

The association must be approved, the amount must be greater than 0, and the SuperAdmin wallet must have enough to cover it. Then we debit SuperAdmin, credit the Main Association, write a Transfer ledger entry, create a PDF receipt, and send an email. Two recharges running at the same time must not both pass on the same last rupee. Today there is no `FOR UPDATE` on that path, so I would add the same atomic debit we use for subscriptions.

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

---

## Q19. Chat message is saved, then the API node dies before FCM. Is the chat lost?

**Answer:**

No. The message is already saved in `ChatMessages`, and SignalR has already pushed it to the groups on that node. FCM runs fire-and-forget in a `Task.Run` with a new scope. Offline users might miss the push, but they still load the history through REST. I would not put FCM in the same transaction as the message.

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

---

## Q20. Code review: `FileController` is `[AllowAnonymous]`. What do you say?

**Answer:**

It means anyone can upload, get, and delete files without a token. Anyone who can reach the API can put objects in Spaces or delete them by id. I would put `[Authorize]` back, check that the user owns the file, and allow anonymous access only for a truly public asset if we ever need one.

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


---
## Additional Questions
### Q16. A production endpoint takes 5 seconds. What do you check?
**Answer:** I split the time into four parts: database, external APIs, application code, and serialization. Logs and traces show me where the time goes. Then I fix the real bottleneck.

### Q17. A user can access another user's record by changing the id in the URL. What do you do?
**Answer:** I add authorization on the server for that resource. Where possible, the query itself should be scoped to the current user/tenant. I don’t fetch the row first and trust the client.

### Q18. An API gets many duplicate requests. How do you protect it?
**Answer:** For expensive or sensitive operations, I use idempotency keys, validation, rate limiting where it makes sense, and unique constraints in the database. Just disabling the button on the client is not enough.
