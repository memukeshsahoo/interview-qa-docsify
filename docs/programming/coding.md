# Programming questions (C# / JS / LINQ)

They ask you to write a small function. Talk while you type. Mention tests and complexity.

---

## Q1. Reverse a string (C#)

**Answer:**

I don't try to mutate the `string`. I use `Array.Reverse` on the chars, or a `StringBuilder`. For Unicode, I mention that I'm reversing **code units**, unless they want full rune support.

In C#, a `string` is immutable, which means once it's created it can never be changed. Any method that looks like it changes a string actually builds a new one. So I copy the characters into a `char[]` (a normal array I'm allowed to change), reverse that array, and build a new string from it. That's O(n) time: one pass over the characters.

The Unicode point is a detail that shows care. A C# `char` is one UTF-16 code unit, and some characters, like many emojis, take two of them. Reversing code units flips those two halves and breaks the emoji. For plain text that's fine, but if they care about every language and emoji, I'd need to reverse whole characters instead.

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

"In place" means changing the data where it already sits, without making a copy. A string doesn't allow that, but an array does. A `Span<char>` is a view over a block of characters that can be changed, so I can reverse it without extra copies.

---

## Q2. Palindrome (ignore case and spaces)

**Answer:**

Two pointers, one from each end. Skip anything that isn't a letter. Compare in lowercase.

A palindrome reads the same forwards and backwards, like "Race car". I put one finger on the first character and one on the last. If a finger is on a space or punctuation, I move it past. Then I compare the two characters in lowercase. If they differ, it's not a palindrome. If they match, both fingers move one step toward the middle, and I stop when they meet.

This is O(n) time and needs no extra memory, because I never build a cleaned-up copy of the string. Each character is looked at once.

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

An allocation means asking for new memory, which the garbage collector later has to clean up. The LINQ version creates extra objects to reverse and compare, while the two-pointer loop just reads the existing string. Also, as written, it doesn't skip spaces or ignore case, so it would answer a slightly different question.

---

## Q3. Two sum — return indices of two numbers that add to target (C#)

**Answer:**

One pass with a dictionary of `value → index`. For each `x`, I look up `target - x`.

The idea: for each number, I already know which partner it needs, `target - x`. So instead of searching the whole array for that partner, I keep a dictionary of every number I've seen so far and where it was. A dictionary lookup is close to instant, no matter how big it gets.

For example, with `[2, 7, 11]` and target 9: at 2, I need 7, which I haven't seen, so I store 2. At 7, I need 2, which is in the dictionary at index 0, so the answer is `[0, 1]`. One pass over the array, so O(n) time, and the dictionary uses O(n) extra memory. I check before storing the current number so a number doesn't get paired with itself.

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

A nested loop compares every number with every other number. With 10 items that's about 100 checks, but with 100,000 items it's about 10 billion. The dictionary version does roughly one lookup per item, so 100,000 items means about 100,000 steps.

---

## Q4. LINQ: last 5 open incidents for a tenant (speak EF, not SQL)

**Answer:**

Filter, order, take, project to a DTO, async, no tracking.

Each word is one step, and each has a reason:

- **Filter** (`Where`): only open incidents.
- **Order** (`OrderByDescending`): newest first, so "last 5" means the 5 most recent.
- **Take** (`Take(5)`): only five rows come back from the database.
- **Project** (`Select`): pick only the columns the screen needs, not the whole row. A DTO (data transfer object) is a small shape made just for sending data out.
- **Async** (`ToListAsync`): the server thread isn't blocked while waiting on the database, so it can serve other requests.
- **No tracking** (`AsNoTracking`): EF Core normally keeps a copy of every loaded row to spot changes later. For read-only data that's wasted work, so I turn it off.

The key point is that EF turns this whole chain into one SQL query, so the filtering, sorting and limiting all happen inside the database.

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

An `IQueryable` is a query that hasn't run yet. EF keeps adding steps to it and only sends SQL when I ask for results. `ToList` is that moment, so if I call it first, the database sends every open incident and only then does C# throw away all but five. It's like ordering every item in a catalogue just to keep five and send the rest back.

---

## Q5. Group by status and count (LINQ)

**Answer:**

I group the incidents by their status and count each group. EF turns this into one SQL `GROUP BY` query, so the database does the counting and only sends back one small row per status.

```csharp
var counts = await db.Incidents
    .GroupBy(i => i.Status)
    .Select(g => new { Status = g.Key, Count = g.Count() })
    .ToListAsync(ct);
```

**Real-world example:** KPI cards for Open / Closed / OnHold. One query, not three.

**Cross-question:** Three `CountAsync` calls?

**Cross-answer:** That's three round trips. `GroupBy` is one.

A round trip is one request from the app to the database and back. Each one carries network delay, even if the query itself is quick. Three separate counts also scan the table three times, while one grouped query gets every count in one go. If a new status is added later, the `GroupBy` picks it up without any code change.

---

## Q6. Debounce search (JavaScript)

**Answer:**

Wait until the user pauses typing, then fire the search. Abort any fetch that is still in flight.

If I search on every key press, typing "network" sends seven requests, and only the last one matters. Debounce is like a lift door that waits a moment after the last person steps in before closing. Every key press restarts a short timer, and the search only runs once the timer finishes without being reset.

In the code, `t` remembers the current timer. Each call clears the old timer and starts a new one for `ms` milliseconds, so only the last call in a burst actually runs `fn`. Aborting old requests is the second half. An older, slower request could come back after a newer one and overwrite the results with stale data. Cancelling it, for example with an `AbortController`, prevents that.

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

Throttle runs regularly while something keeps happening, for example once every 200ms while the user scrolls, so the page can keep reacting. Debounce waits for things to stop. For search I only care about the final text, so debounce fits. For scroll, I want updates during the scroll, so throttle fits.

---

## Q7. Flatten nested comments (recursion vs stack)

**Answer:**

Comments form a tree. Recursion is fine if the depth is small. If the depth is unknown, I use an explicit stack.

Each comment can have replies, and each reply can have its own replies, so the data is shaped like a tree. Flattening means turning it into one simple list. Recursion means the function calls itself for each comment's children, which reads very naturally, as in the example.

The catch is that every nested call uses some space on the call stack, and that space is limited. A thread with thousands of levels of replies could crash with a stack overflow. With an explicit stack, I keep my own `Stack<Comment>` of "comments still to visit" and loop until it's empty. That uses normal memory, so depth isn't a problem. It's like keeping a to-do pile on your desk instead of trying to remember everything in your head.

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

Bad data can make a comment end up as a reply to one of its own replies, so walking the tree would go round in circles forever. Before visiting a comment, I check whether its id is already in the `HashSet`. If it is, I skip it. A `HashSet` lookup is fast, so the guard costs very little.

---

## Q8. Angular: bind two unrelated components (coding)

**Answer:**

They may ask you to type it out. A root service with a signal. Component A calls `select`, and component B reads the value.

"Unrelated" means neither component is inside the other, so they can't pass data through inputs and outputs. A service with `providedIn: 'root'` has only one copy for the whole app, so both components get the same object. It works like a shared noticeboard: one component pins a note, the other reads it.

The signal is private, so only the service can change it. Components get the read-only `id`, and when it changes, Angular updates every component that reads it. In the example, the method that changes it is called `set`.

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

`@Input` passes data from a parent's template down to a child. If two components sit in different parts of the page, I'd have to pass the value up to a shared parent and down again through every level in between. That gets messy fast, and the service avoids it.

---

## Q9. Find duplicates in a list (C#)

**Answer:**

`GroupBy` + `Where(g => g.Count() > 1)`, or a `HashSet` while looping through.

The `GroupBy` version puts equal values into the same bucket, keeps buckets with more than one item, and returns the value of each bucket. It's short and easy to read.

The `HashSet` version walks the list once. A `HashSet` only stores each value once, and its `Add` method returns `false` if the value is already there. So when `Add` returns false, I've found a duplicate. That version can also stop early if I only need to know whether any duplicate exists. Both are about O(n) time.

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

`Distinct` gives back each value once, so `[1, 2, 2, 3]` becomes `[1, 2, 3]`. That's useful for cleaning data, but in an import I need to tell the user "id 2 appears twice", and `Distinct` throws that information away.

---

## Q10. FizzBuzz (they still ask)

**Answer:**

For each number from 1 to 100, I print "Fizz" if it divides by 3, "Buzz" if it divides by 5, "FizzBuzz" if it divides by both, and the number itself otherwise. `%` gives the remainder, so `i % 3 == 0` means "divides evenly by 3". I work out the two checks once, then pick the output. The "both" case has to be checked first, otherwise 15 would stop at "Fizz".

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

For each number, I go through the rules, collect the word of every rule that divides it, and join them. If nothing matched, I print the number. Adding 7 → "Qix" is then one new line in the rules list, and 21 automatically prints "FizzQix". With hard-coded `if`s, every new rule means more combinations to write by hand.

---

## Q11. Check balanced parentheses

**Answer:**

Use a stack. Push on `(`, pop on `)`. The stack must be empty at the end.

A stack is like a pile of plates: you can only add to the top or take from the top, so the last one in is the first one out. Every `(` is a promise that a `)` will come later, so I put it on the pile. Every `)` keeps one promise, so I take one off.

Two things make the string unbalanced. A `)` arrives when the pile is empty, meaning there's nothing to close, as in `")("`. Or the pile still has items at the end, meaning something was opened and never closed. One pass over the string, so O(n).

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

Step by step: the first `(` is pushed, so the stack holds one. The second `(` is pushed, so it holds two. The `)` pops one, leaving one. The string ends with one `(` still on the stack, so one bracket was never closed and the method returns false.

---

## Q12. REST: design `GET /api/incidents` on a whiteboard

**Answer:**

Query params: `status`, `q`, `page`, `pageSize`. It needs auth. The tenant comes from context. It returns a DTO list. No results means 200 with an empty array. 401 if there is no token. Never a 500 for “no rows”.

Here's the reasoning behind each choice:

- **Query params:** `status` filters, `q` is the search text, and `page` with `pageSize` split results into pages so we never send thousands of rows at once.
- **Tenant from context:** the tenant (the customer company the user belongs to) is read from the logged-in user's token on the server, not from a URL parameter. If the client could send it, someone could change it and see another company's data.
- **DTO list:** we send a shape made for the screen, not the raw database entity, so internal fields don't leak.
- **200 with an empty array:** "no incidents match" is a normal answer, not an error. The front end just shows "nothing found".
- **401 without a token:** the server doesn't know who is asking.
- **Never 500:** a 500 means the server broke. Empty results aren't a failure, and a false 500 would set off alerts and confuse anyone debugging.

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

PUT means "here is the full new version, replace what you have". If I sent only the status with PUT, the other fields could be treated as missing or wiped. PATCH means "change just these fields", which matches closing an incident. The request is smaller too, and there's less chance of overwriting someone else's edit to another field.

---

## Q13. Write available wallet balance (C#)

**Answer:**

Available = gross balance minus the active, non-expired holds. Say it out loud while you type. Don't change Balance here.

A hold is money that's reserved but not spent yet, like when a hotel blocks an amount on your card at check-in. The money is still in your account, but you can't use it for something else. So the gross balance is everything in the wallet, and the available balance is what's free to spend right now.

In the code, I only count holds that are `Active` and haven't expired. A hold with no `ExpiresAt` never expires on its own. This method only calculates a number and doesn't save anything. Changing the stored balance here would mix up "reading" with "moving money", and a simple screen refresh could end up changing someone's balance.

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

A user seeing "-500" would be confused, so the display shows 0. The real number still matters for the decision: before creating a booking, I compare the actual available amount with the price and stop the booking if there isn't enough.

---

## Q14. Group ledger rows into credit vs debit totals

**Answer:**

One pass with `GroupBy`, or two sums. In an interview, one loop or one LINQ query is enough.

A ledger is the list of every money movement in and out of a wallet. Each row has a flow type, credit (money in) or debit (money out). Grouping by `FlowType` gives one result per type with how many rows there are and the total amount. The two-sums way would be one `Sum` filtered to credits and another filtered to debits.

The interviewer mainly wants to see that I don't loop over the data many times or query the database once per number I need.

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

When `txns` is still an `IQueryable`, EF turns the `GroupBy` into a SQL `GROUP BY`, and the database sends back one small row per flow type. If I call `ToList` first, every transaction is downloaded into the app and grouped there. For a wallet with years of history, that's a lot of data moved for two numbers.

---

## Q15. Make “release payment” safe to retry (small design)

**Answer:**

The client or server sends a key like `release_{bookingId}`, kept in a store where it must be unique. If the key already exists, return the first result. If not, run the split in one transaction.

The problem: networks fail. The server might release the payment, but the response gets lost, so the client retries. Without protection, the money is released twice. An idempotency key is a unique label for one operation. Idempotent means doing it twice has the same effect as doing it once, like pressing a lift button again: the lift still comes only once.

Before doing the work, I check if this key has been used. If it has, I return the earlier result and move no money. If not, I do all the money movements inside one transaction, so either every part (debit the user, credit the others) is saved, or none of it is. That way a crash halfway can't leave the money split wrongly.

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

GET requests are expected to be safe to repeat, so browsers, proxies and crawlers may send them again, prefetch them or cache them without asking. If GET released payments, a link preview or a refresh could move money. POST tells everyone along the way that this request changes something and shouldn't be repeated casually.

---

## Q16. LINQ: bookings UnderDiscussion for this NR user, last 10

**Answer:**

Filter by user and status, order, take, project. Async, no tracking.

It's the same pattern as the incidents query. `Where` keeps only this user's bookings that are in the `UnderDiscussion` status. `OrderByDescending` puts the newest first. `Take(10)` limits it to ten rows. `Select` picks only the fields the list shows.

`AsNoTracking` skips EF's change tracking because I'm only reading. `ToListAsync` keeps the server free while the database works. All of it becomes one SQL query, so the database does the filtering and sends back at most ten small rows.

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

Once `ToList` runs, the query is sent and everything matching so far comes back from the database. Any filter after that runs in C# memory, after all that data has already travelled over the network. Keeping every filter before `ToListAsync` lets the database send only the rows that are needed.


---
## Additional Questions
### Q17. Find the first non-repeating character.
**Answer:** First I count each character, then I scan the string again and return the first character with a count of one. That is O(n) time.

Two passes are needed because I can't know a character is unique until I've seen the whole string. In "swiss", `s` looks unique at first but shows up again later. The first pass builds a dictionary from each character to how many times it appears. The second pass goes through the string in its original order, so the first character with a count of 1 is the answer, `w` in this case. If no character is unique, `FirstOrDefault` returns the default `char`, `'\0'`, so the caller should check for that.

```csharp
var counts = s.GroupBy(c => c).ToDictionary(g => g.Key, g => g.Count());
var first = s.FirstOrDefault(c => counts[c] == 1);
```

### Q18. Remove duplicates while keeping order.
**Answer:** I use a HashSet to remember what I've already seen, and keep only the first occurrence of each value.

The trick is that `HashSet.Add` returns `true` when the value is new and `false` when it's already in the set. Passing `seen.Add` to `Where` means "keep this item only if it was added just now", so the first time a value appears it stays, and every later copy is dropped. Because I walk the list from start to end, the original order is kept. For example, `[3, 1, 3, 2, 1]` becomes `[3, 1, 2]`. It's one pass, so O(n).

```csharp
var seen = new HashSet<int>();
var result = numbers.Where(seen.Add).ToList();
```

### Q19. Binary search — when would you use it?
**Answer:** When the input is sorted and I need to find a value quickly. Each step throws away about half of what's left to search, so it is O(log n).

It's how you look up a word in a paper dictionary. You open near the middle, see if your word comes before or after that page, and ignore the other half. Then you repeat on the half that's left. With a million sorted items, that takes about 20 steps instead of up to a million. It only works on sorted data, because the "before or after" decision depends on the order. In C#, `Array.BinarySearch` and `List<T>.BinarySearch` already do this.

### Q20. How would you retry an external API safely?
**Answer:** I retry only transient failures, with a timeout and backoff. For operations that can create money or orders, I use an idempotency key. I never blindly retry a payment that isn't idempotent.

A transient failure is a short-lived problem that will probably fix itself, like a timeout, a dropped connection, or a 503 "service busy". A 400 "bad request" will fail the same way every time, so retrying it is pointless. A timeout stops me from waiting forever on one call. Backoff means waiting a bit longer between each try, like 1s, then 2s, then 4s, so I don't hammer a service that's already struggling.

The danger with payments is that a timeout doesn't tell me whether the payment went through. If I just send it again, the customer might be charged twice. With an idempotency key, the provider sees the same key, knows it's the same request, and doesn't charge again.
