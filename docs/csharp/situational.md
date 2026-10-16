# C# — Situational questions

Each answer has a spoken explanation, a small code sample, and a real product story.

---

## Q1. A loop builds a log line with `s = s + chunk` and the API is slow. What do you do?

**Answer:**

Strings in C# cannot change. Once a string is made, it stays exactly as it is. So each `+` does not add to the old string. It creates a brand new string, copies the old text and the new chunk into it, and throws the old one away. In a loop that means a lot of allocations (new pieces of memory) and a lot of copying, and the copying gets bigger every time round because the text keeps growing.

That is why the API gets slow. With a few hundred lines you won't notice. With thousands, the program spends most of its time copying the same text again and again, and the garbage collector has to clean up all those thrown-away strings.

I use `StringBuilder` instead. It is like writing on a notepad where you keep adding lines to the same page, instead of rewriting the whole letter on a fresh sheet every time you add a sentence. At the end I call `ToString()` once. If there are only a few parts, `$"..."` is fine and easier to read.

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

`$"{id}:{status}"` is fine there. It builds the string once, and the cost is tiny. The loop is the real problem, because that is where the copying repeats thousands of times.

---

## Q2. You used `async void Save()` on a button handler in a library. Exceptions vanish. Why?

**Answer:**

An `async` method normally returns a `Task`. A `Task` is like a receipt for work that is still going on: the caller can wait on it with `await`, and if the work fails, the error is stored in the receipt and thrown when the caller awaits it.

An `async void` method gives back no receipt. Nobody can await it, so when it fails there is no caller waiting to catch the error. The error does not come back to your code at all. It gets thrown on the side, where it can crash the process or just get lost in a log nobody reads. To the caller it looks like everything worked.

So I use `async Task` everywhere except UI event handlers. Then the caller can await it, catch errors, and report them properly.

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

That is the one place where `async void` is accepted, because the event system expects a `void` method and there is no caller to return a `Task` to. I still wrap the body in a try/catch, so any error is logged or shown to the user instead of escaping.

---

## Q3. Code does `var list = GetIncidents().Result;` and the site freezes under load. Why?

**Answer:**

`.Result` means "stop here and wait until this async work is finished". While it waits, it blocks a thread-pool thread. The thread pool is the small team of worker threads that the server uses to handle requests. A blocked thread just sits there doing nothing, but it still can't take any other work.

This is called sync-over-async: calling async code and then waiting on it in a blocking way. With a few users it works. Under load, many requests block their threads at the same time, the pool runs out of free threads, and new requests have to queue. The site looks frozen even though the CPU is mostly idle.

Think of a restaurant where each waiter stands at the kitchen door until the food is ready, instead of serving other tables meanwhile. With enough customers, there are no waiters left. The fix is to use `await` all the way up, so the thread goes back to the pool while the database or API is working.

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

You get the classic deadlock less often than in old ASP.NET, because ASP.NET Core doesn't force the code after an `await` to come back to the same request thread. But thread-pool starvation is still very real. Blocking with `.Result` still ties up threads, so the site can still stall under load.

---

## Q4. You `lock (_gate)` and inside you `await` HTTP. It will not compile / is dangerous. What instead?

**Answer:**

A `lock` makes sure only one thread at a time runs a piece of code. It belongs to the thread that took it, and the same thread must release it. But after an `await`, the rest of the method can continue on a different thread. So `lock` does not work with async code, and the C# compiler won't let you put an `await` inside a `lock` block.

I use `SemaphoreSlim.WaitAsync()` instead. A `SemaphoreSlim` created with a count of 1 is like a single key to a room: whoever has the key goes in, and everyone else waits outside. The difference is that you can wait for it with `await`, so the waiting code doesn't block a thread. I always release it in a `finally` block, so the key is given back even when the HTTP call fails.

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

Never. Your object is visible to other code, so someone else can lock on your object too. Then their code and yours wait on each other, and you can get a deadlock you can't see from your own class. Use a private `object` or a `SemaphoreSlim` that only your class can reach.

---

## Q5. `foreach (var u in users.Where(u => IsVip(u)))` on an EF `IQueryable` throws. Why?

**Answer:**

When `users` is an EF `IQueryable`, the `Where` does not run in C#. EF reads the filter and translates it into SQL, so the database does the filtering. That only works for things EF knows how to turn into SQL, like comparing a column with a value.

`IsVip` is your own C# method. The database has no idea what that method does, so EF cannot turn it into SQL, and it throws a "could not be translated" error instead of quietly guessing.

There are two fixes. If VIP is stored as a column, filter on that column, and SQL does all the work. If the VIP rule is complex C# logic, first filter in SQL as much as you can (for example, only active users), load those rows, and then run `IsVip` on them in memory.

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

Only after SQL has already cut down the rows. Don’t load the whole table. If the table has a million rows, `ToList` first pulls all of them over the network into memory just to throw most of them away.

---

## Q6. Two threads increment `int count`. Final number is wrong. Why?

**Answer:**

`count++` looks like one step, but it is not atomic (done in one go). It is three steps: read the value, add 1, write it back. Two threads can mix those up. For example, both read 5, both add 1, and both write 6. Two increments happened, but the count only went up by one.

It is like two people updating the same paper tally. Both look at it, see 5, and both write 6. One of the updates is lost. With many threads and many increments, the final number ends up lower than the real one.

I use `Interlocked.Increment`, which does the read, add and write as one step that no other thread can get in the middle of. For more complex updates, I use a `lock` so only one thread changes the value at a time.

**Example:**

```csharp
Interlocked.Increment(ref _openCount);
```

**Real-world example:**

An in-memory “open sockets” counter on a SignalR host came out too low under load. After switching to `Interlocked`, it matched the real connection count.

**Cross-question:** For a dictionary of tenants?

**Cross-answer:**

Use `ConcurrentDictionary`, or one lock. Don’t update a normal `Dictionary` from many threads. A normal `Dictionary` is not built for that: updates can be lost, and its internal state can get corrupted. `ConcurrentDictionary` is made to be shared between threads safely.

---

## Q7. You put a `class User` in a `HashSet` and cannot find it after you change `Name`. Why?

**Answer:**

A `HashSet` stores each item in a "bucket" chosen by its hash code. The hash comes from the fields you used in `GetHashCode`. When you look an item up, the set calculates the hash again and goes straight to that bucket.

If you change those fields after inserting, the hash changes, so the set looks in the wrong place. The object is still inside the set, just in the old bucket, and the set can't find it. It is like filing a document under "S" for Sharma, then changing the name on it to Verma and looking under "V".

So I use immutable keys (keys that can't change after creation) or an id. An id like a `Guid` never changes, so the lookup always lands in the right bucket.

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

Fine, as long as you don’t change the properties that make up equality. Records compare by their values, so their hash code comes from those values. Change one after inserting and you get the same lost-item problem.

---

## Q8. `using var http = new HttpClient()` inside a loop. After a day the server cannot connect out. Why?

**Answer:**

Each new `HttpClient` opens its own network connection (a socket). When you dispose it, the socket isn't freed right away. The operating system keeps it in a TIME_WAIT state for a while, to make sure late packets don't get mixed up with a new connection.

If you create a new client for every call, you end up with too many sockets stuck in TIME_WAIT. The machine only has a limited number of ports for outgoing connections. Once they are used up, new outbound calls fail, even though your code looks correct. It is like taking a new phone line for every call and not being able to give the old lines back for a while.

I use `IHttpClientFactory` with one typed client, not a `new` client per call. The factory reuses connections from a pool and manages their lifetime, so you don't run out of sockets.

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

It is better than `new` per call, because connections get reused. But DNS changes can go stale: a long-lived client can keep talking to the old server address after the DNS record has changed. In ASP.NET Core the factory is the default choice, because it recycles the underlying connections from time to time and avoids both problems.

---

## Q9. `catch (Exception) { }` around `SaveChanges`. Data is missing and nobody knows. What should you have done?

**Answer:**

An empty `catch` swallows the error: the save failed, but the code carries on as if it worked. The user sees success, the data never reaches the database, and there is nothing in the logs to explain it. It is like a smoke alarm with the battery taken out. It is quiet, but that doesn't mean nothing is burning.

Don’t swallow the error. Log it with context (the incident id, not the token), so someone can find out what failed and for which record. Never put secrets like tokens in logs. Then fail the API with a 500 or 409, so the caller knows the save didn't happen, and let the user retry.

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

Same smell. `null` hides why it failed, and the caller has to guess, or forgets to check and crashes later somewhere else. I prefer a result type that says clearly whether it worked and why not, or an exception that the API maps to ProblemDetails (the standard error format ASP.NET Core returns to clients).

---

## Q10. You need “optional” on an `int` property. You used `int` and `0` means missing. What goes wrong?

**Answer:**

A plain `int` always has a value. It can't say "nothing here", so people pick a special number like `0` to mean missing. The trouble is that `0` can be a real value, like a priority or a count. Then the code can't tell "the value is zero" apart from "no value was given", and sooner or later something treats one as the other.

I use `int?` instead. `int?` is a nullable int: it holds either a number or `null`, and `null` clearly means "no value". It is like leaving a form field blank instead of writing 0 in it.

**Example:**

```csharp
public int? ParentIncidentId { get; set; }
```

**Real-world example:**

We sent parent ticket id `0` to mean “no parent”. That worked because real ids were never 0 — until an import used 0 as a legacy key. Switching to `int?` with `null` ended the mix-up.

**Cross-question:** `string` empty vs null?

**Cross-answer:**

Be consistent. For user input, I check `IsNullOrWhiteSpace`, because a user who typed only spaces really entered nothing. In the database, null means unknown, and I use empty only if empty is a real value. If both mean "missing" in different places, every check has to handle both, and someone will forget.

---

## Q11. Booking numbers use a static `ConcurrentDictionary` plus `SemaphoreSlim`. Two API servers. What goes wrong?

**Answer:**

The counter lives in process memory. Each server has its own copy, and neither one can see the other's. So server A and server B each think the next number is 0001. The `SemaphoreSlim` only makes threads take turns inside one server, so it can't stop this.

You get duplicate `BK-2026-...-0001` numbers, or you skip numbers when one node restarts. It is like two counters at a bank each handing out tokens from their own roll: both give out token 1.

The safe version is a database sequence, or `MAX` inside a transaction. The database is the one place both servers share, so it can hand out each number only once.

**Example:**

```csharp
private static readonly SemaphoreSlim _sequenceSemaphore = new(1, 1);
private static readonly ConcurrentDictionary<string, int> _dailySequenceCounter = new();
```

**Real-world example:**

In NriCare, `GenerateBookingNumber` works fine on one instance. The wallet’s `GenerateTransactionId` uses `Count() + 1` with no lock at all, so two top-ups can get the same TXN id. I would use a unique constraint plus a retry, or a sequence.

**Cross-question:** Is `SemaphoreSlim` useless then?

**Cross-answer:**

No. It still makes threads take turns **on that machine**, which stops two requests on the same server from clashing. It just does not replace a uniqueness rule in the database, which is the only thing that protects you across all servers.


---

## Additional Questions

### Q12. An API is slow only in production. How do you investigate?
**Answer:** First I check request duration, database duration, external API calls, logs and traces. Then I find the slowest step instead of guessing. If SQL is the problem, I check the generated query and the execution plan.

Production is different from my machine: more data, more users at once, real network delays, and other services that may be slow. So I can't just reproduce it locally and hope. Traces show how long each step of one request took, like a timeline. That tells me whether the time goes in my code, the database, or a call to another service. The execution plan shows how the database runs the query, for example whether it uses an index or scans the whole table.

It is like a delivery that arrives late. Before blaming the driver, you check each stop on the route and see where the delay actually happened.

**Cross-question:** First thing you change?

**Cross-answer:** Nothing blindly. I measure first. If I change things on a guess, I may fix the wrong part, add new risk, and still not know what the real problem was.

### Q13. Two requests can update the same wallet at the same time. What do you do?
**Answer:** I don’t rely only on an in-memory lock, because there may be more than one API server. A lock in one server's memory can't stop a request running on another server. I use a database transaction with the right concurrency control, plus a unique or idempotency rule where needed.

A transaction groups the read and the write so they succeed or fail together. Concurrency control is how the database stops two updates from overwriting each other. For example, it can lock the row while one request is changing it, or reject a save if the row changed since it was read. An idempotency rule means that if the same request arrives twice, it only takes effect once.

Without this, money can be spent twice or a top-up can be lost. For a wallet, that is a real financial error, not just a display bug.

**Cross-question:** Why not just read balance, subtract, then save?

**Cross-answer:** Two requests can read the same old balance and both spend it. That is a race condition. For example, the balance is 100, two requests for 80 each both see 100, both decide there is enough, and both save 20. 160 was spent from a balance of 100.

### Q14. An external payment call times out. Should you immediately retry?
**Answer:** Only if the operation is safe to retry. For payments, I send an idempotency key or use the provider’s own idempotency feature. Then I check the final status before creating another payment.

A timeout only means I didn't get an answer in time. It doesn't tell me whether the payment failed or went through. An idempotency key is a unique id I send with the request. If the provider sees the same key again, it knows it is the same payment and doesn't charge twice.

It is like pressing "Pay" in an app when the screen freezes. Before pressing again, you check your bank to see whether the money already went.

**Cross-question:** What if the provider actually processed it but my response timed out?

**Cross-answer:** Then a blind retry could create a duplicate payment, and the customer gets charged twice. So I check the payment status using the same business or reference id, and only create a new payment if the first one really didn't happen.
