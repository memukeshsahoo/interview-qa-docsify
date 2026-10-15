# Programming questions (C# / JS / LINQ)

They ask you to write a small function. Talk while you type. Mention tests and complexity.

---

## Q1. Reverse a string (C#)

**Answer:**

I don't try to mutate the `string`. I use `Array.Reverse` on the chars, or a `StringBuilder`. For Unicode, I mention that I'm reversing **code units**, unless they want full rune support.

**Example:**

```csharp
static string Reverse(string s)
{
    if (string.IsNullOrEmpty(s)) return s;
    var chars = s.ToCharArray();
    Array.Reverse(chars);
    return new string(chars);
}
```

**Real-world example:** You rarely reverse strings in a ticket API. They ask this to see if you know strings are immutable.

**Cross-question:** In-place?

**Cross-answer:** Strings can't be changed in place. I reverse a `char[]` or a `Span<char>` instead.

---

## Q2. Palindrome (ignore case and spaces)

**Answer:**

Two pointers, one from each end. Skip anything that isn't a letter. Compare in lowercase.

**Example:**

```csharp
static bool IsPalindrome(string s)
{
    int i = 0, j = s.Length - 1;
    while (i < j)
    {
        if (!char.IsLetterOrDigit(s[i])) { i++; continue; }
        if (!char.IsLetterOrDigit(s[j])) { j--; continue; }
        if (char.ToLowerInvariant(s[i]) != char.ToLowerInvariant(s[j]))
            return false;
        i++; j--;
    }
    return true;
}
```

**Real-world example:** Validating a “code” field that must read the same forwards and backwards. Most of the time, though, they just want to see two pointers.

**Cross-question:** LINQ `s.Reverse().SequenceEqual(s)`?

**Cross-answer:** It works, but with extra allocations. Fine for small strings.

---

## Q3. Two sum — return indices of two numbers that add to target (C#)

**Answer:**

One pass with a dictionary of `value → index`. For each `x`, I look up `target - x`.

**Example:**

```csharp
static int[] TwoSum(int[] nums, int target)
{
    var seen = new Dictionary<int, int>();
    for (int i = 0; i < nums.Length; i++)
    {
        var need = target - nums[i];
        if (seen.TryGetValue(need, out var j))
            return [j, i];
        seen[nums[i]] = i;
    }
    throw new InvalidOperationException("No pair");
}
```

**Real-world example:** Matching invoice lines to a remaining amount. Same idea: a lookup instead of nested loops.

**Cross-question:** Nested loop O(n²)?

**Cross-answer:** It works, but it's too slow for large n. The hash map is O(n).

---

## Q4. LINQ: last 5 open incidents for a tenant (speak EF, not SQL)

**Answer:**

Filter, order, take, project to a DTO, async, no tracking.

**Example:**

```csharp
var latest = await db.Incidents
    .AsNoTracking()
    .Where(i => i.Status == "Open")
    .OrderByDescending(i => i.CreatedAt)
    .Take(5)
    .Select(i => new { i.Id, i.Title, i.CreatedAt })
    .ToListAsync(ct);
```

**Real-world example:** A dashboard “latest open” widget. Same query as REST optimization.

**Cross-question:** `ToList` then `Take`?

**Cross-answer:** That pulls every open row into memory. `Take` must stay on the `IQueryable`.

---

## Q5. Group by status and count (LINQ)

**Answer:**

```csharp
var counts = await db.Incidents
    .GroupBy(i => i.Status)
    .Select(g => new { Status = g.Key, Count = g.Count() })
    .ToListAsync(ct);
```

**Real-world example:** KPI cards for Open / Closed / OnHold. One query, not three.

**Cross-question:** Three `CountAsync` calls?

**Cross-answer:** That's three round trips. `GroupBy` is one.

---

## Q6. Debounce search (JavaScript)

**Answer:**

Wait until the user pauses typing, then fire the search. Abort any fetch that is still in flight.

**Example:**

```javascript
function debounce(fn, ms) {
  let t;
  return (...args) => {
    clearTimeout(t);
    t = setTimeout(() => fn(...args), ms);
  };
}

const search = debounce(async (q) => {
  const res = await fetch('/api/incidents?q=' + encodeURIComponent(q));
  render(await res.json());
}, 300);
```

**Real-world example:** The header search on the incident board. Same as Angular `debounceTime` + `switchMap`.

**Cross-question:** Throttle?

**Cross-answer:** Throttle fires at most once per interval, which suits scroll. Search wants debounce.

---

## Q7. Flatten nested comments (recursion vs stack)

**Answer:**

Comments form a tree. Recursion is fine if the depth is small. If the depth is unknown, I use an explicit stack.

**Example:**

```csharp
IEnumerable<Comment> Flatten(IEnumerable<Comment> roots)
{
    foreach (var c in roots)
    {
        yield return c;
        foreach (var x in Flatten(c.Children))
            yield return x;
    }
}
```

**Real-world example:** Exporting an incident comment thread to a flat activity log.

**Cross-question:** Infinite parent loop?

**Cross-answer:** I guard against it with a `HashSet` of visited ids.

---

## Q8. Angular: bind two unrelated components (coding)

**Answer:**

They may ask you to type it out. A root service with a signal. Component A calls `select`, and component B reads the value.

**Example:**

```typescript
@Injectable({ providedIn: 'root' })
export class SelectionService {
  private readonly _id = signal<number | null>(null);
  readonly id = this._id.asReadonly();
  set(id: number) { this._id.set(id); }
}
```

**Real-world example:** A list and a header badge. Full write-up: [Angular situational Q1](../angular/situational.md).

**Cross-question:** `@Input` between them?

**Cross-answer:** Only if they are parent and child. For unrelated components → a service.

---

## Q9. Find duplicates in a list (C#)

**Answer:**

`GroupBy` + `Where(g => g.Count() > 1)`, or a `HashSet` while looping through.

**Example:**

```csharp
var dupes = ids.GroupBy(x => x)
    .Where(g => g.Count() > 1)
    .Select(g => g.Key)
    .ToList();
```

**Real-world example:** Importing tickets from Excel and catching duplicate external ids before insert.

**Cross-question:** `Distinct`?

**Cross-answer:** It removes the dupes, but it does not **tell you** which values were duplicated.

---

## Q10. FizzBuzz (they still ask)

**Answer:**

```csharp
for (int i = 1; i <= 100; i++)
{
    var f = i % 3 == 0;
    var b = i % 5 == 0;
    Console.WriteLine(f && b ? "FizzBuzz" : f ? "Fizz" : b ? "Buzz" : i.ToString());
}
```

**Real-world example:** None. They want clean conditions, not a nested mess.

**Cross-question:** Extend to 7 → “Qix”?

**Cross-answer:** Keep a list of rules like `(3,"Fizz"), (5,"Buzz")` and join the matches. It shows you don't hard-code just two cases.

---

## Q11. Check balanced parentheses

**Answer:**

Use a stack. Push on `(`, pop on `)`. The stack must be empty at the end.

**Example:**

```csharp
static bool Balanced(string s)
{
    var st = new Stack<char>();
    foreach (var c in s)
    {
        if (c == '(') st.Push(c);
        else if (c == ')')
        {
            if (st.Count == 0) return false;
            st.Pop();
        }
    }
    return st.Count == 0;
}
```

**Real-world example:** Validating a filter expression the user typed. Same idea works for `{}` `[]`.

**Cross-question:** `"(()"`?

**Cross-answer:** False, because the stack is not empty at the end.

---

## Q12. REST: design `GET /api/incidents` on a whiteboard

**Answer:**

Query params: `status`, `q`, `page`, `pageSize`. It needs auth. The tenant comes from context. It returns a DTO list. No results means 200 with an empty array. 401 if there is no token. Never a 500 for “no rows”.

**Example:**

```http
GET /api/incidents?status=Open&page=1&pageSize=20
Authorization: Bearer ...
```

```json
{ "items": [ { "id": 1, "title": "VPN down", "commentCount": 3 } ], "page": 1, "hasMore": true }
```

**Real-world example:** The first load of the incident board. Same as [REST optimization](../dotnet/rest-api-optimization.md).

**Cross-question:** PUT vs PATCH for status?

**Cross-answer:** PATCH `{ "status": "Closed" }` if we only change the status. PUT if we replace the whole resource.

---

## Q13. Write available wallet balance (C#)

**Answer:**

Available = gross balance minus the active, non-expired holds. Say it out loud while you type. Don't change Balance here.

**Example:**

```csharp
static decimal Available(decimal balance, IEnumerable<Hold> holds, DateTime utcNow)
{
    var held = holds
        .Where(h => h.Status == HoldStatus.Active
                    && (h.ExpiresAt == null || h.ExpiresAt > utcNow))
        .Sum(h => h.Amount);
    return balance - held;
}
```

**Real-world example:**

This is `GetWalletBalanceAsync` in NriCare. Booking create uses a stricter version: it counts all Active holds and doesn't skip expired ones.

**Cross-question:** Negative available?

**Cross-answer:**

Show it as 0 on screen. For a booking, fail if `available < required`.

---

## Q14. Group ledger rows into credit vs debit totals

**Answer:**

One pass with `GroupBy`, or two sums. In an interview, one loop or one LINQ query is enough.

**Example:**

```csharp
var totals = txns.GroupBy(t => t.FlowType)
    .Select(g => new { Flow = g.Key, Count = g.Count(), Sum = g.Sum(x => x.Amount) })
    .ToList();
```

**Real-world example:**

Wallet history in NriCare works out the incoming/outgoing counts and sums with separate queries. Interviewers like to see one `GroupBy` instead of four round trips.

**Cross-question:** Do this in SQL?

**Cross-answer:**

Yes, if `txns` is an `IQueryable`. If you `ToList` first, the grouping happens in memory.

---

## Q15. Make “release payment” safe to retry (small design)

**Answer:**

The client or server sends a key like `release_{bookingId}`, kept in a store where it must be unique. If the key already exists, return the first result. If not, run the split in one transaction.

**Example:**

```csharp
if (await db.WalletTransactions.AnyAsync(x => x.IdempotencyKey == key, ct))
    return "Already released";

await using var tx = await db.Database.BeginTransactionAsync(ct);
// debit user, credit SP / platform / association
await db.SaveChangesAsync(ct);
await tx.CommitAsync(ct);
```

**Real-world example:**

`BookingRepository.ReleasePayment` already uses keys like `release_{bookingId}` and `release_sp_{bookingId}`. The next step is a unique index.

**Cross-question:** GET vs POST for release?

**Cross-answer:**

POST. It moves money, and GET should never change anything.

---

## Q16. LINQ: bookings UnderDiscussion for this NR user, last 10

**Answer:**

Filter by user and status, order, take, project. Async, no tracking.

**Example:**

```csharp
var rows = await db.Bookings
    .AsNoTracking()
    .Where(b => b.NRUserId == nrUserId && b.Status == BookingStatus.UnderDiscussion)
    .OrderByDescending(b => b.CreatedDate)
    .Take(10)
    .Select(b => new { b.Id, b.BookingNumber, b.Price, b.Status })
    .ToListAsync(ct);
```

**Real-world example:**

Same shape as the NR user home list in NriCare. Status is an enum, not a magic string.

**Cross-question:** `ToList` then `Where` status?

**Cross-answer:**

That loads every booking for the user. Keep the `Where` on the `IQueryable`.


---
## Additional Questions
### Q17. Find the first non-repeating character.
**Answer:** First I count each character, then I scan the string again and return the first character with a count of one. That is O(n) time.

```csharp
var counts = s.GroupBy(c => c).ToDictionary(g => g.Key, g => g.Count());
var first = s.FirstOrDefault(c => counts[c] == 1);
```

### Q18. Remove duplicates while keeping order.
**Answer:** I use a HashSet to remember what I've already seen, and keep only the first occurrence of each value.

```csharp
var seen = new HashSet<int>();
var result = numbers.Where(seen.Add).ToList();
```

### Q19. Binary search — when would you use it?
**Answer:** When the input is sorted and I need to find a value quickly. Each step throws away about half of what's left to search, so it is O(log n).

### Q20. How would you retry an external API safely?
**Answer:** I retry only transient failures, with a timeout and backoff. For operations that can create money or orders, I use an idempotency key. I never blindly retry a payment that isn't idempotent.
