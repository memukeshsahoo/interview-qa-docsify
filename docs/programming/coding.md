# Programming questions (C# / JS / LINQ)

They ask you to write a small function. Talk while you type. Mention tests and complexity.

---

## Q1. Reverse a string (C#)

**Answer:**

Do not mutate `string`. Use `Array.Reverse` on chars, or a `StringBuilder`. For Unicode, mention you are reversing **code units** unless they want full rune support.

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

**Real-world example:** Rarely reverse strings in a ticket API. They use this to see if you know immutability.

**Cross-question:** In-place?

**Cross-answer:** Strings cannot. Reverse a `char[]` or `Span<char>`.

---

## Q2. Palindrome (ignore case and spaces)

**Answer:**

Two pointers from both ends. Skip non-letters. Compare lowercase.

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

**Real-world example:** Validating a “code” field that must read the same forward and back. More often they just want two pointers.

**Cross-question:** LINQ `s.Reverse().SequenceEqual(s)`?

**Cross-answer:** Works, extra allocations. Fine for small strings.

---

## Q3. Two sum — return indices of two numbers that add to target (C#)

**Answer:**

One pass, dictionary `value → index`. For each `x`, look for `target - x`.

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

**Real-world example:** Matching invoice lines to a remaining amount. Same idea as a lookup instead of nested loops.

**Cross-question:** Nested loop O(n²)?

**Cross-answer:** Works, too slow for large n. Hash map is O(n).

---

## Q4. LINQ: last 5 open incidents for a tenant (speak EF, not SQL)

**Answer:**

Filter, order, take, project DTO, async, no tracking.

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

**Real-world example:** Dashboard “latest open” widget. Same query as REST optimization.

**Cross-question:** `ToList` then `Take`?

**Cross-answer:** Pulls all open rows. `Take` must stay on `IQueryable`.

---

## Q5. Group by status and count (LINQ)

**Answer:**

```csharp
var counts = await db.Incidents
    .GroupBy(i => i.Status)
    .Select(g => new { Status = g.Key, Count = g.Count() })
    .ToListAsync(ct);
```

**Real-world example:** KPI cards Open / Closed / OnHold. One query, not three.

**Cross-question:** Three `CountAsync` calls?

**Cross-answer:** Three round trips. `GroupBy` is one.

---

## Q6. Debounce search (JavaScript)

**Answer:**

Wait until the user pauses, then fire. Abort in-flight fetch.

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

**Real-world example:** Header search on the incident board. Same as Angular `debounceTime` + `switchMap`.

**Cross-question:** Throttle?

**Cross-answer:** At most once per interval (scroll). Search wants debounce.

---

## Q7. Flatten nested comments (recursion vs stack)

**Answer:**

Tree of comments. Recursion is fine if depth is small. For unknown depth, explicit stack.

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

**Real-world example:** Incident comment thread export to a flat activity log.

**Cross-question:** Infinite parent loop?

**Cross-answer:** Guard with a `HashSet` of visited ids.

---

## Q8. Angular: bind two unrelated components (coding)

**Answer:**

They may ask you to type it. Root service + signal. A calls `select`. B reads.

**Example:**

```typescript
@Injectable({ providedIn: 'root' })
export class SelectionService {
  private readonly _id = signal<number | null>(null);
  readonly id = this._id.asReadonly();
  set(id: number) { this._id.set(id); }
}
```

**Real-world example:** List and header badge. Full write-up: [Angular situational Q1](../angular/situational.md).

**Cross-question:** `@Input` between them?

**Cross-answer:** Only if parent-child. Unrelated → service.

---

## Q9. Find duplicates in a list (C#)

**Answer:**

`GroupBy` + `Where(g => g.Count() > 1)`, or `HashSet` while iterating.

**Example:**

```csharp
var dupes = ids.GroupBy(x => x)
    .Where(g => g.Count() > 1)
    .Select(g => g.Key)
    .ToList();
```

**Real-world example:** Importing tickets from Excel — duplicate external ids before insert.

**Cross-question:** `Distinct`?

**Cross-answer:** Removes dupes. It does not **list** which values were duplicated.

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

**Real-world example:** None. They want clean conditions, not nested mess.

**Cross-question:** Extend to 7 → “Qix”?

**Cross-answer:** List of rules `(3,"Fizz"), (5,"Buzz")` and concatenate. Shows you do not hard-code only two cases.

---

## Q11. Check balanced parentheses

**Answer:**

Stack. Push `(`, pop on `)`. Empty at end.

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

**Real-world example:** Validating a filter expression the user typed. Same idea for `{}` `[]`.

**Cross-question:** `"(()"`?

**Cross-answer:** False — stack not empty.

---

## Q12. REST: design `GET /api/incidents` on a whiteboard

**Answer:**

Query: `status`, `q`, `page`, `pageSize`. Auth. Tenant from context. DTO list. 200 + empty array. 401 if no token. Never 500 for “no rows”.

**Example:**

```http
GET /api/incidents?status=Open&page=1&pageSize=20
Authorization: Bearer ...
```

```json
{ "items": [ { "id": 1, "title": "VPN down", "commentCount": 3 } ], "page": 1, "hasMore": true }
```

**Real-world example:** Incident board first paint. Same as [REST optimization](../dotnet/rest-api-optimization.md).

**Cross-question:** PUT vs PATCH for status?

**Cross-answer:** PATCH `{ "status": "Closed" }` if we only change status. PUT if we replace the whole resource.

---

## Q13. Write available wallet balance (C#)

**Answer:**

Available = gross balance minus active, non-expired holds. Speak it while you type. Do not mutate Balance here.

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

This is `GetWalletBalanceAsync` in NriCare. Booking create uses a stricter version: all Active holds, no expiry skip.

**Cross-question:** Negative available?

**Cross-answer:**

Treat as 0 for display. For booking, fail if `available < required`.

---

## Q14. Group ledger rows into credit vs debit totals

**Answer:**

One pass with `GroupBy`, or two sums. In an interview, one loop or LINQ is enough.

**Example:**

```csharp
var totals = txns.GroupBy(t => t.FlowType)
    .Select(g => new { Flow = g.Key, Count = g.Count(), Sum = g.Sum(x => x.Amount) })
    .ToList();
```

**Real-world example:**

Wallet history in NriCare computes incoming/outgoing counts and sums with separate queries. Interviewers like one `GroupBy` instead of four round trips.

**Cross-question:** Do this in SQL?

**Cross-answer:**

Yes if `txns` is `IQueryable`. If you `ToList` first, grouping is in memory.

---

## Q15. Make “release payment” safe to retry (small design)

**Answer:**

Client or server sends a key `release_{bookingId}`. Unique store. If it exists, return the first result. If not, run the split in one transaction.

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

`BookingRepository.ReleasePayment` already uses keys like `release_{bookingId}` and `release_sp_{bookingId}`. Next step: unique index.

**Cross-question:** GET vs POST for release?

**Cross-answer:**

POST. It changes money. GET should not.

---

## Q16. LINQ: bookings UnderDiscussion for this NR user, last 10

**Answer:**

Filter by user, status, order, take, project. Async. No tracking.

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

Same shape as an NR user home list in NriCare. Status is an enum, not a magic string.

**Cross-question:** `ToList` then `Where` status?

**Cross-answer:**

Loads every booking for that user. Keep `Where` on `IQueryable`.


---
## Additional Questions
### Q17. Find the first non-repeating character.
**Answer:** Count each character first, then scan the string again and return the first character whose count is one. This is O(n) time.

```csharp
var counts = s.GroupBy(c => c).ToDictionary(g => g.Key, g => g.Count());
var first = s.FirstOrDefault(c => counts[c] == 1);
```

### Q18. Remove duplicates while keeping order.
**Answer:** Use a HashSet to remember what I already saw, then add only the first occurrence.

```csharp
var seen = new HashSet<int>();
var result = numbers.Where(seen.Add).ToList();
```

### Q19. Binary search — when would you use it?
**Answer:** When the input is sorted and I need to find a value efficiently. Each step removes about half the search space, so it is O(log n).

### Q20. How would you retry an external API safely?
**Answer:** Retry only transient failures, use a timeout and backoff, and use an idempotency key for operations that can create money/orders. Never blindly retry a non-idempotent payment.
