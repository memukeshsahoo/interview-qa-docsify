# C# — Situational questions

Each answer has a spoken explanation, a small code sample, and a real product story.

---

## Q1. A loop builds a log line with `s = s + chunk` and the API is slow. What do you do?

**Answer:**

Strings cannot change. Each `+` makes a new string. In a loop that is many allocations. Use `StringBuilder`, or `$"..."` if there are only a few parts.

**Example:**

```csharp
var sb = new StringBuilder();
foreach (var line in lines)
    sb.AppendLine(line);
return sb.ToString();
```

**Real-world example:**

Exporting 10,000 incident comments into one text file with `s += comment` made the export API time out. `StringBuilder` finished in milliseconds.

**Cross-question:** Two fields in a log?

**Cross-answer:**

`$"{id}:{status}"` is fine. The loop is the problem.

---

## Q2. You used `async void Save()` on a button handler in a library. Exceptions vanish. Why?

**Answer:**

`async void` cannot be awaited. Errors do not land on the caller. Use `async Task` except for UI event handlers.

**Example:**

```csharp
public async Task SaveAsync(CancellationToken ct)
{
    await _db.SaveChangesAsync(ct);
}
```

**Real-world example:**

A “close incident” API swallowed failures because a helper was `async void`. The UI showed success. The row stayed open. Changing to `async Task` made the 500 visible.

**Cross-question:** Event handler in WinForms?

**Cross-answer:**

That is the one place `async void` is accepted. Still wrap in try/catch.

---

## Q3. Code does `var list = GetIncidents().Result;` and the site freezes under load. Why?

**Answer:**

You blocked a thread-pool thread waiting on async work (sync-over-async). Under load, threads run out. Use `await` all the way.

**Example:**

```csharp
// bad
var items = _repo.GetOpenAsync().Result;

// good
var items = await _repo.GetOpenAsync(ct);
```

**Real-world example:**

Dashboard KPI used `.Result` “because the old method was sync”. At 9am stand-up the site hung. Replacing with `await` and `async` controller actions fixed the stall.

**Cross-question:** ASP.NET Core deadlock?

**Cross-answer:**

Less classic deadlock than old ASP.NET, but thread-pool starvation is real.

---

## Q4. You `lock (_gate)` and inside you `await` HTTP. It will not compile / is dangerous. What instead?

**Answer:**

`lock` is not async-safe. Use `SemaphoreSlim.WaitAsync()`.

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

Two API calls 401 at once and both tried to refresh the token. A `lock` around `await` was invalid. `SemaphoreSlim` made one refresh, the second waiter reused the new token.

**Cross-question:** Lock on `this`?

**Cross-answer:**

Never. Someone else can lock your object. Private `object` or `SemaphoreSlim`.

---

## Q5. `foreach (var u in users.Where(u => IsVip(u)))` on an EF `IQueryable` throws. Why?

**Answer:**

`IsVip` is your C# method. EF cannot turn it into SQL.

**Example:**

```csharp
// if VIP is a column
var vips = await db.Users.Where(u => u.IsVip).ToListAsync();

// if VIP is complex rules, filter in SQL first then in memory
var rows = await db.Users.Where(u => u.IsActive).ToListAsync();
var vips = rows.Where(IsVip).ToList();
```

**Real-world example:**

“VIP customers” used a method that checked three flags in C#. EF threw translation failed. We mapped a computed `IsVip` column / simple `Where` on those flags.

**Cross-question:** `ToList` then `Where` always?

**Cross-answer:**

Only after the SQL already reduced rows. Do not load the whole table.

---

## Q6. Two threads increment `int count`. Final number is wrong. Why?

**Answer:**

`count++` is not atomic. Use `Interlocked.Increment` or a `lock`.

**Example:**

```csharp
Interlocked.Increment(ref _openCount);
```

**Real-world example:**

An in-memory “open sockets” counter on a SignalR host was short under load. `Interlocked` matched the real connection count.

**Cross-question:** For a dictionary of tenants?

**Cross-answer:**

`ConcurrentDictionary`, or one lock. Do not increment a normal `Dictionary` from many threads.

---

## Q7. You put a `class User` in a `HashSet` and cannot find it after you change `Name`. Why?

**Answer:**

Hash is from fields you used in `GetHashCode`. Mutating after insert breaks lookup. Use immutable keys or an id.

**Example:**

```csharp
public readonly record struct UserId(Guid Value);

var set = new HashSet<UserId>();
set.Add(new UserId(id));
```

**Real-world example:**

A cache of “already notified users” keyed by a mutable `User` object. After a profile update, “already notified” missed them and they got duplicate SMS. Key by `UserId`.

**Cross-question:** Records as keys?

**Cross-answer:**

Fine if you do not mutate the properties that make equality.

---

## Q8. `using var http = new HttpClient()` inside a loop. After a day the server cannot connect out. Why?

**Answer:**

Too many sockets in TIME_WAIT. Use `IHttpClientFactory`, one typed client, not `new` per call.

**Example:**

```csharp
builder.Services.AddHttpClient<IWeatherClient, WeatherClient>();

public class WeatherClient(HttpClient http)
{
    public Task<string> GetAsync() => http.GetStringAsync("/forecast");
}
```

**Real-world example:**

A job called a SMS API 50,000 times with `new HttpClient()`. Outbound HTTP died until recycle. Factory fixed it.

In NriCare, `MailHelper` still does `new HttpClient()` per ZeptoMail send. Same smell. I would inject `IHttpClientFactory`.

**Cross-question:** Static singleton HttpClient?

**Cross-answer:**

Better than per-call `new`, but DNS can go stale. Factory is the default in ASP.NET Core.

---

## Q9. `catch (Exception) { }` around `SaveChanges`. Data is missing and nobody knows. What should you have done?

**Answer:**

Do not swallow. Log with context (incident id, not the token). Fail the API with a 500/409. Let the user retry.

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

Close-ticket swallowed unique-constraint errors. UI said closed. DB still open. Removing the empty catch and returning 409 fixed support tickets.

**Cross-question:** Catch and return null?

**Cross-answer:**

Same smell. Prefer a result type or an exception the API maps to ProblemDetails.

---

## Q10. You need “optional” on an `int` property. You used `int` and `0` means missing. What goes wrong?

**Answer:**

`0` is a valid number (priority, count). Use `int?`.

**Example:**

```csharp
public int? ParentIncidentId { get; set; }
```

**Real-world example:**

Parent ticket id `0` was sent as “no parent”. A real id never 0 until an import used 0 as a legacy key. `int?` + `null` ended the mix-up.

**Cross-question:** `string` empty vs null?

**Cross-answer:**

Be consistent. User input: `IsNullOrWhiteSpace`. Database: null for unknown, empty only if empty is a real value.

---

## Q11. Booking numbers use a static `ConcurrentDictionary` plus `SemaphoreSlim`. Two API servers. What goes wrong?

**Answer:**

The counter lives in process memory. Server A and server B each think the next number is 0001. You get duplicate `BK-2026-...-0001` or you skip numbers when one node restarts. A database sequence, or `MAX` inside a transaction, is the safe version.

**Example:**

```csharp
private static readonly SemaphoreSlim _sequenceSemaphore = new(1, 1);
private static readonly ConcurrentDictionary<string, int> _dailySequenceCounter = new();
```

**Real-world example:**

NriCare `GenerateBookingNumber` works on one instance. Wallet `GenerateTransactionId` uses `Count() + 1` with no lock — two top-ups can get the same TXN id. I would use a unique constraint plus retry, or a sequence.

**Cross-question:** Is `SemaphoreSlim` useless then?

**Cross-answer:**

It still serializes threads **on that machine**. It does not replace a database uniqueness rule.
