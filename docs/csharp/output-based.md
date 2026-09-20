# C# — Output-based questions

They show a snippet. You say what prints, then **why**. Then a real place it bites.

---

## Q1. Strings and `==`

```csharp
string a = "inc";
string b = "inc";
string c = new string("inc".ToCharArray());
Console.WriteLine(a == b);
Console.WriteLine((object)a == (object)b);
Console.WriteLine(a == c);
Console.WriteLine((object)a == (object)c);
```

**Answer (output):**

```text
True
True
True
False
```

(`(object)a == (object)b` is often **True** because literals are interned — same instance.)

`a == c` is **True** because `string` overloads `==` as value. `(object)a == (object)c` is **False** — different instances.

**Example / why:** intern pool + overloaded `==`.

**Real-world example:** Putting user names in a `Dictionary` keyed by string still works because equality is by value. Do not use `object` reference equality for strings.

**Cross-question:** `ReferenceEquals(a, c)`?

**Cross-answer:** False.

---

## Q2. Boxing

```csharp
int x = 1;
object o1 = x;
object o2 = x;
Console.WriteLine(o1 == o2);
Console.WriteLine(o1.Equals(o2));
```

**Answer (output):**

```text
False
True
```

`==` on `object` is reference. Two boxes = two heap objects. `Equals` is overridden on `int` via virtual call and compares value.

**Real-world example:** `ArrayList` of ints — `==` on boxed values fails in tests. `List<int>` does not box.

**Cross-question:** `object.Equals(x, x)`?

**Cross-answer:** True (value Equals).

---

## Q3. `ref` vs copy

```csharp
void Add(int n) { n++; }
void AddRef(ref int n) { n++; }

var a = 1;
Add(a);
Console.WriteLine(a);
AddRef(ref a);
Console.WriteLine(a);
```

**Answer (output):**

```text
1
2
```

**Real-world example:** A helper `Normalize(page)` that does `page = page < 1 ? 1 : page` without `ref` does not change the caller’s page. Return a new value instead of `ref` in APIs.

**Cross-question:** `in` parameter?

**Cross-answer:** Read-only by-ref. You cannot `n++`.

---

## Q4. LINQ deferred

```csharp
var n = 0;
var q = Enumerable.Range(1, 3).Where(_ => { n++; return true; });
Console.WriteLine(n);
_ = q.ToList();
_ = q.ToList();
Console.WriteLine(n);
```

**Answer (output):**

```text
0
6
```

The query does not run until `ToList`. Two enumerations = two runs. `Range(1,3)` has 3 items, twice = 6.

**Real-world example:** `IQueryable` enumerated twice = two SQL calls. I materialize once (`ToList`) if I need two loops.

**Cross-question:** `Count()` then `ToList()` on EF?

**Cross-answer:** Two SQL statements unless you cache the list.

---

## Q5. `async` method without await

```csharp
async Task<int> Get()
{
    return 5;
}
Console.WriteLine(Get().Result);
```

**Answer (output):**

```text
5
```

It still returns a `Task`. Compiler warns. Extra state machine for no reason.

**Real-world example:** Someone wrote `async Task<User> GetUser() => _cache.User;` — async with no await. Change to `Task.FromResult` or return `User` sync.

**Cross-question:** `async void`?

**Cross-answer:** Cannot `.Result`. Exceptions are nasty. Avoid.

---

## Q6. `string` vs `StringBuilder` identity

```csharp
var s = "a";
s += "b";
Console.WriteLine(s);
```

**Answer (output):** `ab`

`s` now points at a **new** string. `"a"` is unchanged (immutable).

**Real-world example:** Loop `s += line` for a report. Output is correct but slow. `StringBuilder` for the loop.

**Cross-question:** Does `+=` mutate?

**Cross-answer:** No. It replaces the variable.

---

## Q7. `const` vs `readonly` in a trick

```csharp
class A
{
    public const int C = 1;
    public static readonly int R = 1;
}
Console.WriteLine(A.C + A.R);
```

**Answer (output):** `2`

**Real-world example:** Changing `const` in a DLL does not update other assemblies until they rebuild. `readonly` is runtime.

**Cross-question:** Can `R` be 2 after static ctor?

**Cross-answer:** Yes, if the static constructor sets it. `const` cannot.

---

## Q8. Virtual vs `new`

```csharp
class Base { public virtual void P() => Console.Write("B"); }
class D : Base { public override void P() => Console.Write("D"); }
class E : Base { public new void P() => Console.Write("E"); }

Base x = new D();
Base y = new E();
x.P();
y.P();
```

**Answer (output):** `DB`

`override` is polymorphic. `new` hides; the variable is `Base`, so `B`.

**Real-world example:** A “logging” child method with `new` never ran when the code held a `Base` reference. Use `override`.

**Cross-question:** `((E)y).P()`?

**Cross-answer:** `E`

---

## Q9. `finally` and return

```csharp
int F()
{
    try { return 1; }
    finally { Console.Write("F"); }
}
Console.Write(F());
```

**Answer (output):** `F1`

`finally` runs before the value is fully returned to the caller.

**Real-world example:** `Dispose` in `finally` still runs when you `return` from `try`. `using` is that pattern.

**Cross-question:** Exception in `finally`?

**Cross-answer:** It can hide the original exception. Keep `finally` simple.

---

## Q10. Nullable

```csharp
int? a = null;
int b = a ?? 3;
Console.WriteLine(b);
Console.WriteLine(a.HasValue);
```

**Answer (output):**

```text
3
False
```

**Real-world example:** `ParentIncidentId ?? 0` looked like “parent 0” in the UI. We showed “none” when `null`.

**Cross-question:** `int b = (int)a` when `a` is null?

**Cross-answer:** Throws `InvalidOperationException`.

---

## Q11. `lock` and strings (concept)

```csharp
lock ("incident") { }
```

**Answer:**

It compiles. It is **dangerous**. Interned string is shared. Another library can lock the same string and deadlock.

**Output:** no print; the trap is design.

**Real-world example:** Two modules locked `"cache"` and froze the API. Private `readonly object _gate = new()`.

**Cross-question:** `lock (this)`?

**Cross-answer:** Also bad. Outsiders can lock your instance.

---

## Q12. Array covariance

```csharp
string[] s = { "a" };
object[] o = s;
try { o[0] = 1; Console.Write("ok"); }
catch (ArrayTypeMismatchException) { Console.Write("fail"); }
```

**Answer (output):** `fail`

**Real-world example:** Passing `string[]` to a method that takes `object[]` and writes a number. Prefer `IReadOnlyList<string>`.

**Cross-question:** `List<string>` to `List<object>`?

**Cross-answer:** Does not compile. Generics are safer here.

---

## Q13. Available wallet balance (holds)

```csharp
decimal balance = 1000m;
var holds = new[]
{
    (Amount: 200m, Active: true,  Expired: false),
    (Amount: 100m, Active: true,  Expired: true),
    (Amount: 50m,  Active: false, Expired: false),
};

var available = balance - holds
    .Where(h => h.Active && !h.Expired)
    .Sum(h => h.Amount);

Console.WriteLine(available);
```

**Answer (output):** `800`

Only the 200 hold is active and not expired. 100 is expired. 50 is not active.

**Real-world example:**

`GetWalletBalanceAsync` is `Balance` minus active holds that are not past `ExpiresAt`. Booking create currently counts **all** Active holds, even expired ones. Interviewers like that difference.

**Cross-question:** If Balance was already reduced by the hold?

**Cross-answer:**

Then subtracting holds again would double-count. In NriCare, create-hold does **not** reduce `Balance`. Available must subtract holds.

---

## Q14. Deferred LINQ vs one ToList

```csharp
var n = 0;
var q = new[] { 1, 2, 3 }.Where(x => { n++; return x > 1; });
Console.WriteLine(n);
Console.WriteLine(q.Count());
Console.WriteLine(n);
Console.WriteLine(q.Sum());
Console.WriteLine(n);
```

**Answer (output):**

```text
0
2
2
5
4
```

`Where` does not run until `Count` / `Sum`. Each enumeration runs the predicate again. `Count` sees 2 and 3. `Sum` is 5. `n` becomes 2 then 4.

**Real-world example:**

If I pass an `IQueryable` of wallet rows to `CountAsync` and then `ToListAsync`, that is two SQL calls. Materialize once if I need both.

**Cross-question:** `q.ToList()` first?

**Cross-answer:**

Then `n` would be 3 after ToList (all items visited once), and Count/Sum would be in memory.

---

## Q15. `async` + `Task.Run` around sync SDK

```csharp
async Task<int> CreateOrder()
{
    return await Task.Run(() => 42);
}

Console.WriteLine(CreateOrder().Result);
```

**Answer (output):** `42`

It works, but you moved blocking work onto the thread pool. The method looks async. Under load those threads still get used.

**Real-world example:**

`RazorpayService.CreateOrderAsync` does `await Task.Run(() => _client.Order.Create(options))` because the Razorpay SDK call is sync. Honest answer: wrap it, but do not pretend it is non-blocking I/O.

**Cross-question:** `.Result` on that in a controller?

**Cross-answer:**

Do not. `await CreateOrderAsync()` in the action. `.Result` can stall the thread pool.


---

## Additional Questions

### Q16. What is printed?
```csharp
var numbers = new[] { 1, 2, 3 };
var query = numbers.Where(x => x > 1);
numbers[1] = 5;
Console.WriteLine(query.First());
```
**Answer:** `5`.

**Why:** `Where` is deferred. The array is changed before enumeration, so the query sees the new value.

### Q17. What is printed?
```csharp
int x = 10;
var a = x;
a++;
Console.WriteLine(x);
Console.WriteLine(a);
```
**Answer:**
```text
10
11
```

**Why:** `int` is a value type. Assigning it copies the value.

### Q18. What is printed?
```csharp
var tasks = Enumerable.Range(1, 3)
    .Select(async x => { await Task.Delay(10); return x * 2; });
var result = await Task.WhenAll(tasks);
Console.WriteLine(string.Join(",", result));
```
**Answer:** `2,4,6`.

**Why:** Each async lambda returns a Task, and `WhenAll` waits for all of them before returning the results.
