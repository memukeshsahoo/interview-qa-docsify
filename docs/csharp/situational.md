# C# — Situational questions

Each answer has a spoken explanation, a small code sample, and a real product story.

---

## Q1. A loop builds a log line with `s = s + chunk` and the API is slow. What do you do?

**Answer:**

Strings in C# cannot change. So each `+` creates a brand new string, and in a loop that means a lot of allocations. I use `StringBuilder` instead, or `$"..."` if there are only a few parts.

**Example:**

```csharp
var sb = new StringBuilder();
foreach (var line in lines)
    sb.AppendLine(line);
return sb.ToString();
```

**Real-world example:**

We exported 10,000 incident comments into one text file using `s += comment`, and the export API timed out. With `StringBuilder` it finished in milliseconds.

**Cross-question:** Two fields in a log?

**Cross-answer:**

`$"{id}:{status}"` is fine there. The loop is the real problem.

---

## Q2. You used `async void Save()` on a button handler in a library. Exceptions vanish. Why?

**Answer:**

An `async void` method cannot be awaited, so its errors never reach the caller. I use `async Task` everywhere except UI event handlers.

**Example:**

```csharp
public async Task SaveAsync(CancellationToken ct)
{
    await _db.SaveChangesAsync(ct);
}
```

**Real-world example:**

A “close incident” API swallowed failures because one helper was `async void`. The UI showed success, but the row stayed open. Once we changed it to `async Task`, the 500 showed up properly.

**Cross-question:** Event handler in WinForms?

**Cross-answer:**

That is the one place where `async void` is accepted. I still wrap it in a try/catch.

---

## Q3. Code does `var list = GetIncidents().Result;` and the site freezes under load. Why?

**Answer:**

That line blocks a thread-pool thread while it waits on async work. This is called sync-over-async. Under load you run out of threads. The fix is to use `await` all the way up.

**Example:**

```csharp
// bad
var items = _repo.GetOpenAsync().Result;

// good
var items = await _repo.GetOpenAsync(ct);
```

**Real-world example:**

A dashboard KPI used `.Result` “because the old method was sync”. At the 9am stand-up, the site hung. Switching to `await` and `async` controller actions fixed the stall.

**Cross-question:** ASP.NET Core deadlock?

**Cross-answer:**

You get the classic deadlock less often than in old ASP.NET, but thread-pool starvation is still very real.

---

## Q4. You `lock (_gate)` and inside you `await` HTTP. It will not compile / is dangerous. What instead?

**Answer:**

`lock` does not work with async code. I use `SemaphoreSlim.WaitAsync()` instead.

**Example:**

```csharp
private readonly SemaphoreSlim _mutex = new(1, 1);

public async Task RefreshTokenAsync()
{
    await _mutex.WaitAsync();
    try
    {
        await _http.PostAsync("/token", null);
    }
    finally
    {
        _mutex.Release();
    }
}
```

**Real-world example:**

Two API calls got a 401 at the same time, and both tried to refresh the token. Putting a `lock` around the `await` was not allowed. With `SemaphoreSlim`, only one refresh ran, and the second caller reused the new token.

**Cross-question:** Lock on `this`?

**Cross-answer:**

Never. Someone else can lock on your object too. Use a private `object` or a `SemaphoreSlim`.

---

## Q5. `foreach (var u in users.Where(u => IsVip(u)))` on an EF `IQueryable` throws. Why?

**Answer:**

`IsVip` is your own C# method. EF cannot turn it into SQL.

**Example:**

```csharp
// if VIP is a column
var vips = await db.Users.Where(u => u.IsVip).ToListAsync();

// if VIP is complex rules, filter in SQL first then in memory
var rows = await db.Users.Where(u => u.IsActive).ToListAsync();
var vips = rows.Where(IsVip).ToList();
```

**Real-world example:**

Our “VIP customers” query used a method that checked three flags in C#. EF threw a “translation failed” error. We fixed it with a computed `IsVip` column / a simple `Where` on those flags.

**Cross-question:** `ToList` then `Where` always?

**Cross-answer:**

Only after SQL has already cut down the rows. Don’t load the whole table.

---

## Q6. Two threads increment `int count`. Final number is wrong. Why?

**Answer:**

`count++` is not atomic — it is a read, an add, and a write, and two threads can mix those up. I use `Interlocked.Increment` or a `lock`.

**Example:**

```csharp
Interlocked.Increment(ref _openCount);
```

**Real-world example:**

An in-memory “open sockets” counter on a SignalR host came out too low under load. After switching to `Interlocked`, it matched the real connection count.

**Cross-question:** For a dictionary of tenants?

**Cross-answer:**

Use `ConcurrentDictionary`, or one lock. Don’t update a normal `Dictionary` from many threads.

---

## Q7. You put a `class User` in a `HashSet` and cannot find it after you change `Name`. Why?

**Answer:**

The hash comes from the fields you used in `GetHashCode`. If you change those after inserting, the set looks in the wrong place. I use immutable keys or an id.

**Example:**

```csharp
public readonly record struct UserId(Guid Value);

var set = new HashSet<UserId>();
set.Add(new UserId(id));
```

**Real-world example:**

We had a cache of “already notified users” keyed by a mutable `User` object. After a profile update, the lookup missed them, and they got duplicate SMS. Keying by `UserId` fixed it.

**Cross-question:** Records as keys?

**Cross-answer:**

Fine, as long as you don’t change the properties that make up equality.

---

## Q8. `using var http = new HttpClient()` inside a loop. After a day the server cannot connect out. Why?

**Answer:**

You end up with too many sockets stuck in TIME_WAIT. I use `IHttpClientFactory` with one typed client, not a `new` client per call.

**Example:**

```csharp
builder.Services.AddHttpClient<IWeatherClient, WeatherClient>();

public class WeatherClient(HttpClient http)
{
    public Task<string> GetAsync() => http.GetStringAsync("/forecast");
}
```

**Real-world example:**

A job called an SMS API 50,000 times with `new HttpClient()`. Outbound HTTP stopped working until the app was recycled. Moving to the factory fixed it.

In NriCare, `MailHelper` still does `new HttpClient()` for every ZeptoMail send. Same smell. I would inject `IHttpClientFactory` there.

**Cross-question:** Static singleton HttpClient?

**Cross-answer:**

It is better than `new` per call, but DNS changes can go stale. In ASP.NET Core the factory is the default choice.

---

## Q9. `catch (Exception) { }` around `SaveChanges`. Data is missing and nobody knows. What should you have done?

**Answer:**

Don’t swallow the error. Log it with context (the incident id, not the token). Fail the API with a 500 or 409, and let the user retry.

**Example:**

```csharp
try
{
    await db.SaveChangesAsync(ct);
}
catch (DbUpdateException ex)
{
    _log.LogError(ex, "Failed saving incident {IncidentId}", id);
    throw;
}
```

**Real-world example:**

Our close-ticket code swallowed unique-constraint errors. The UI said “closed”, but the DB row was still open. Removing the empty catch and returning a 409 stopped those support tickets.

**Cross-question:** Catch and return null?

**Cross-answer:**

Same smell. I prefer a result type, or an exception that the API maps to ProblemDetails.

---

## Q10. You need “optional” on an `int` property. You used `int` and `0` means missing. What goes wrong?

**Answer:**

`0` can be a real value, like a priority or a count. I use `int?` instead.

**Example:**

```csharp
public int? ParentIncidentId { get; set; }
```

**Real-world example:**

We sent parent ticket id `0` to mean “no parent”. That worked because real ids were never 0 — until an import used 0 as a legacy key. Switching to `int?` with `null` ended the mix-up.

**Cross-question:** `string` empty vs null?

**Cross-answer:**

Be consistent. For user input, I check `IsNullOrWhiteSpace`. In the database, null means unknown, and I use empty only if empty is a real value.

---

## Q11. Booking numbers use a static `ConcurrentDictionary` plus `SemaphoreSlim`. Two API servers. What goes wrong?

**Answer:**

The counter lives in process memory. So server A and server B each think the next number is 0001. You get duplicate `BK-2026-...-0001` numbers, or you skip numbers when one node restarts. The safe version is a database sequence, or `MAX` inside a transaction.

**Example:**

```csharp
private static readonly SemaphoreSlim _sequenceSemaphore = new(1, 1);
private static readonly ConcurrentDictionary<string, int> _dailySequenceCounter = new();
```

**Real-world example:**

In NriCare, `GenerateBookingNumber` works fine on one instance. The wallet’s `GenerateTransactionId` uses `Count() + 1` with no lock at all, so two top-ups can get the same TXN id. I would use a unique constraint plus a retry, or a sequence.

**Cross-question:** Is `SemaphoreSlim` useless then?

**Cross-answer:**

No. It still makes threads take turns **on that machine**. It just does not replace a uniqueness rule in the database.


---

## Additional Questions

### Q12. An API is slow only in production. How do you investigate?
**Answer:** First I check request duration, database duration, external API calls, logs and traces. Then I find the slowest step instead of guessing. If SQL is the problem, I check the generated query and the execution plan.

**Cross-question:** First thing you change?

**Cross-answer:** Nothing blindly. I measure first.

### Q13. Two requests can update the same wallet at the same time. What do you do?
**Answer:** I don’t rely only on an in-memory lock, because there may be more than one API server. I use a database transaction with the right concurrency control, plus a unique or idempotency rule where needed.

**Cross-question:** Why not just read balance, subtract, then save?

**Cross-answer:** Two requests can read the same old balance and both spend it. That is a race condition.

### Q14. An external payment call times out. Should you immediately retry?
**Answer:** Only if the operation is safe to retry. For payments, I send an idempotency key or use the provider’s own idempotency feature. Then I check the final status before creating another payment.

**Cross-question:** What if the provider actually processed it but my response timed out?

**Cross-answer:** Then a blind retry could create a duplicate payment. So I check the payment status using the same business or reference id.
