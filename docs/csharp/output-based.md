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

Two ideas are at work here. First, there are two kinds of "equal": same value (the same letters) and same instance (the very same object in memory). Two copies of a book have the same text but are two different books.

Second, when you write the same text literal (a string typed directly in the code, like `"inc"`) more than once, .NET keeps only one copy of it in an intern pool and reuses it. So `a` and `b` point at the same object. `c` is built at runtime with `new string(...)`, so it is a separate object that just happens to hold the same letters.

Going line by line:
- `a == b`: `string` changes what `==` means so it compares the letters. Same letters, so `True`.
- `(object)a == (object)b`: casting to `object` makes `==` compare instances instead. `a` and `b` are the same interned object, so `True`.
- `a == c`: compares the letters again, so `True`.
- `(object)a == (object)c`: compares instances, and `c` is a different object, so `False`.

**Example / why:** intern pool + overloaded `==`. The intern pool decides which variables share one object, and the overloaded `==` decides whether letters or objects are compared.

**Real-world example:** Putting user names in a `Dictionary` keyed by string still works because equality is by value. Do not use `object` reference equality for strings. Whether two strings are the same object depends on how they were created, so the result can change for text that looks identical.

**Cross-question:** `ReferenceEquals(a, c)`?

**Cross-answer:** False. `ReferenceEquals` always checks whether two variables point at the same object, and `c` was created separately with `new string(...)`.

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

`==` on `object` compares references. Two boxes means two heap objects. `Equals` is overridden on `int` and called virtually, so it compares the value.

Boxing is what happens when you store a value type like `int` in an `object` variable. .NET wraps the number in a small object on the heap (the area of memory for objects), like putting a coin into a box. Each assignment makes its own box. So `o1` and `o2` are two different boxes that both hold the number 1.

- `o1 == o2`: the variables are typed as `object`, so `==` asks "is this the same box?". It isn't, so `False`.
- `o1.Equals(o2)`: `Equals` is a virtual method, so the call goes to the version written for `int`, even though the variable is `object`. That version opens the boxes and compares the numbers inside. Both are 1, so `True`.

**Real-world example:** `ArrayList` of ints — `==` on boxed values fails in tests. `List<int>` does not box. `ArrayList` stores everything as `object`, so every int gets boxed and `==` compares boxes. `List<int>` stores plain ints, so `==` compares the numbers, and you also skip the extra memory for the boxes.

**Cross-question:** `object.Equals(x, x)`?

**Cross-answer:** True (value Equals). `x` gets boxed twice, into two separate boxes, but `object.Equals` then calls the value `Equals` of `int`, which compares the numbers inside, not the boxes.

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

Step by step:
- `Add(a)`: by default C# passes a copy of the value. `n` is a copy of `a`, so `n++` changes only the copy. `a` is still 1, and the first line prints `1`.
- `AddRef(ref a)`: `ref` passes the variable itself, not a copy. `n` is just another name for `a`, so `n++` changes `a` to 2, and the second line prints `2`.

It is like giving someone a photocopy of a form versus handing them the original. Writing on the photocopy doesn't change your original.

**Real-world example:** A helper `Normalize(page)` that does `page = page < 1 ? 1 : page` without `ref` does not change the caller’s page. In APIs, I return a new value instead of using `ref`. Something like `page = Normalize(page)` makes the change obvious to anyone reading the calling code.

**Cross-question:** `in` parameter?

**Cross-answer:** Read-only by-ref. You cannot `n++`. Like `ref`, it passes the variable itself without copying it, but the method is not allowed to change it. That is mostly useful for large structs, where copying would cost something.

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

The query does not run until `ToList`. Two enumerations means two runs. `Range(1,3)` has 3 items, and running it twice gives 6.

"Deferred" means the query is only a recipe, not the cooked meal. Writing `Where(...)` just saves the instructions in `q`. Nothing has been checked yet.

Step by step:
1. After building `q`, the filter has not run once, so `n` is 0 and the first line prints `0`.
2. The first `ToList()` runs the recipe: the filter runs for 1, 2 and 3, so `n` becomes 3.
3. The second `ToList()` runs the whole recipe again from the start, because `q` doesn't remember earlier results. Three more runs, so `n` becomes 6, and the second line prints `6`.

**Real-world example:** Enumerating an `IQueryable` twice means two SQL calls. I materialize once (`ToList`) if I need two loops. Materialize means running the query once and keeping the results in a list in memory. Then both loops use that list and the database is hit only once.

**Cross-question:** `Count()` then `ToList()` on EF?

**Cross-answer:** Two SQL statements unless you cache the list. `Count()` sends a count query, and `ToList()` sends another query for the rows. If I already need the rows, I call `ToList()` once and use the list's `Count`.

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

It still returns a `Task`. The compiler warns you, and you get an extra state machine for no reason.

The `async` keyword tells the compiler to rewrite the method into a state machine, which is the hidden code that lets a method pause at an `await` and carry on later. Here there is no `await`, so nothing ever pauses. The method returns a `Task` that is already finished with the value 5, and `.Result` just reads that 5 out of it.

So the output is correct, but the method pays for machinery it never uses. The compiler's warning is a hint that you probably forgot an `await` or didn't need `async` at all.

**Real-world example:** Someone wrote `async Task<User> GetUser() => _cache.User;` — async with no await. I'd change it to `Task.FromResult` or return `User` synchronously. `Task.FromResult` hands back a finished `Task` without the extra state machine. If no caller really needs a `Task`, returning `User` directly is simpler still.

**Cross-question:** `async void`?

**Cross-answer:** You cannot call `.Result` on it, and exceptions are nasty. I avoid it. There is no `Task` to wait on, so the caller can't get the result, can't know when it finished, and can't catch its errors.

---

## Q6. `string` vs `StringBuilder` identity

```csharp
var s = "a";
s += "b";
Console.WriteLine(s);
```

**Answer (output):** `ab`

`s` now points at a **new** string. `"a"` is unchanged (immutable).

Strings in C# are immutable, which means they can never be changed after they are created. `s += "b"` is short for `s = s + "b"`. It builds a brand new string `"ab"` and then points `s` at it. The old `"a"` still exists, untouched. Only the variable moved. It is like writing a new name tag instead of editing the old one.

**Real-world example:** Doing `s += line` in a loop for a report gives the right output but is slow. I use `StringBuilder` for the loop. Every `+=` copies all the text built so far into a new string, so the work keeps growing with each line. `StringBuilder` keeps adding to the same buffer and makes the final string once.

**Cross-question:** Does `+=` mutate?

**Cross-answer:** No. It just points the variable at a new string. Any other variable that pointed at the old string still sees the old text.

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

Both values are 1, so the sum is 2. The trick is not in the output but in how each one is stored. A `const` value is baked into the calling code when it is compiled: the compiler copies the number 1 straight into the place where `A.C` is used. A `static readonly` field is a real field that is read when the program runs.

**Real-world example:** Changing `const` in a DLL does not update other assemblies until they rebuild. `readonly` is read at runtime. So if a library changes `C` from 1 to 5 and you only replace the DLL, other projects that use it still see 1 until they are compiled again, because the 1 was copied into them. With `readonly`, they read the new value straight away. That is why I use `const` only for values that truly never change, like the number of days in a week.

**Cross-question:** Can `R` be 2 after static ctor?

**Cross-answer:** Yes, if the static constructor sets it. `const` cannot. A `readonly` field can be set in its declaration or in a constructor, and after that it is locked. A `const` is fixed at compile time and can never be set anywhere else.

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

`override` is polymorphic. `new` only hides the method, and the variable is `Base`, so it prints `B`.

Polymorphic means "the real object decides which method runs", not the type of the variable.
- `x.P()`: `x` is declared as `Base` but holds a `D`. `D` overrides `P`, so .NET checks the real object at runtime and runs `D`'s version. It prints `D`.
- `y.P()`: `y` holds an `E`, but `E` uses `new`. That doesn't replace the `Base` method. It adds a second, separate method with the same name. Which one runs depends on the variable's type, and the variable is `Base`, so `Base`'s version runs and prints `B`.

Together on one line: `DB`.

**Real-world example:** A “logging” child method marked `new` never ran when the code held a `Base` reference. Use `override`. Most code passes objects around as their base type, so a `new` method gets skipped silently, without any error.

**Cross-question:** `((E)y).P()`?

**Cross-answer:** `E`. The cast makes the variable's type `E`, so the compiler picks `E`'s own `P`.

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

Step by step: the `try` block hits `return 1`. The method remembers the value 1, but before it actually hands it back, the `finally` block runs and prints `F`. Then the method returns 1, and the caller prints it. So `F` appears first, then `1`.

A `finally` block always runs when the code leaves the `try`, whether it leaves normally, through a `return`, or because of an exception. It is like locking the door on your way out, whichever way you leave the house.

**Real-world example:** `Dispose` in `finally` still runs when you `return` from `try`. `using` is that pattern. A `using` block is the compiler writing that `try`/`finally` for you, so files, connections and other resources are always released.

**Cross-question:** Exception in `finally`?

**Cross-answer:** It can hide the original exception. Keep `finally` simple. If the `try` threw an error and then the `finally` throws too, the second error replaces the first one, and you lose the information about what really went wrong.

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

`int?` is a nullable int: it can hold a number or `null` (no value). `??` is the null-coalescing operator, which means "use the left side, but if it's null, use the right side instead".
- `a` is `null`, so `a ?? 3` gives 3, and `b` prints `3`.
- `HasValue` tells you whether a nullable has a number in it. `a` is `null`, so it prints `False`.

**Real-world example:** `ParentIncidentId ?? 0` looked like “parent 0” in the UI. We showed “none” when `null`. A fallback value with `??` is handy, but if the fallback looks like real data, users read it as real data.

**Cross-question:** `int b = (int)a` when `a` is null?

**Cross-answer:** Throws `InvalidOperationException`. A plain `int` must hold a number, so there is nothing valid to convert `null` into. Check `HasValue` or use `??` first.

---

## Q11. `lock` and strings (concept)

```csharp
lock ("incident") { }
```

**Answer:**

It compiles, but it is **dangerous**. An interned string is shared, so another library can lock the same string and deadlock.

A `lock` needs an object to lock on, like a single key that only one thread can hold at a time. Text literals like `"incident"` are interned, so every piece of code in the process that writes `"incident"` gets the very same string object. That means any other code, including libraries you didn't write, that happens to lock on `"incident"` is using the same key as you. Your code and theirs then block each other for no good reason, and if each is waiting on something the other holds, you get a deadlock (both wait forever).

**Output:** nothing prints; the trap is the design.

**Real-world example:** Two modules locked `"cache"` and froze the API. The fix is a private `readonly object _gate = new()`. Because the object is private, no code outside the class can ever get hold of it to lock on.

**Cross-question:** `lock (this)`?

**Cross-answer:** Also bad. Code outside your class can lock your instance. Anyone holding a reference to your object can use it as a lock, so you have the same shared-key problem.

---

## Q12. Array covariance

```csharp
string[] s = { "a" };
object[] o = s;
try { o[0] = 1; Console.Write("ok"); }
catch (ArrayTypeMismatchException) { Console.Write("fail"); }
```

**Answer (output):** `fail`

Array covariance means C# lets you treat a `string[]` as an `object[]`, because every string is an object. But the array underneath is still a `string[]`. `o` is just another view of the same array.

Step by step: `o[0] = 1` tries to put an int into what is really a string array. The compiler allows it, because `o` is typed as `object[]`. At runtime, .NET checks the real array type, sees that an int isn't a string, and throws `ArrayTypeMismatchException`. So `"ok"` is never printed, the `catch` runs, and it prints `fail`.

It is like a box labelled "Things" that is really an egg carton. The label says you can put anything in it, but a shoe won't go.

**Real-world example:** Passing `string[]` to a method that takes `object[]` and writes a number. Prefer `IReadOnlyList<string>`. If the method only needs to read the items, a read-only type makes the wrong write impossible, and the bug shows up at compile time instead of in production.

**Cross-question:** `List<string>` to `List<object>`?

**Cross-answer:** It does not compile. Generics are safer here. The compiler blocks the conversion, because otherwise you could add an int to what is really a list of strings, which is exactly the array problem above.

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

A hold is money set aside for something that hasn't been charged yet, like a hotel blocking an amount on your card at check-in. The money is still in the balance, but you can't spend it.

Step by step: the `Where` keeps only holds that are `Active` and not `Expired`. The 200 hold passes. The 100 hold fails because it is expired. The 50 hold fails because it is not active. `Sum` adds up what is left, which is 200. Then 1000 − 200 = 800.

**Real-world example:**

`GetWalletBalanceAsync` is `Balance` minus active holds that are not past `ExpiresAt`. Booking create currently counts **all** Active holds, even expired ones. Interviewers like that difference. Two places that calculate "available money" with different rules can give the user two different numbers, and the booking side would block money that should already be free.

**Cross-question:** If Balance was already reduced by the hold?

**Cross-answer:**

Then subtracting holds again would double-count. In NriCare, create-hold does **not** reduce `Balance`. Available must subtract holds. So the rule depends on where the hold is recorded: if it is already taken out of `Balance`, you must not subtract it a second time.

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
3
5
6
```

`Where` does not run until I call `Count` or `Sum`, and each of those runs the predicate again over all three items. `Count` keeps 2 and 3, so it prints 2, and `Sum` prints 5. Because `n++` runs before the filter check, it counts every item on every pass: `n` is 3 after `Count` and 6 after `Sum`.

The predicate is the little function inside `Where` that decides yes or no for each item. Line by line:
1. `n` is still 0, because building `q` doesn't run anything yet. Prints `0`.
2. `Count()` runs the filter over 1, 2, 3. Only 2 and 3 pass `x > 1`, so it prints `2`.
3. That pass ran the predicate three times, so `n` is now 3. Prints `3`.
4. `Sum()` runs the filter again from scratch and adds 2 + 3. Prints `5`.
5. Three more predicate runs, so `n` is 6. Prints `6`.

**Real-world example:**

If I pass an `IQueryable` of wallet rows to `CountAsync` and then `ToListAsync`, that is two SQL calls. Materialize once if I need both. Load the rows once with `ToListAsync`, then take the count from the list in memory.

**Cross-question:** `q.ToList()` first?

**Cross-answer:**

Then `n` would be 3 after ToList (all items visited once), and Count/Sum would be in memory. They would work on the saved list of `[2, 3]` and wouldn't run the predicate again, so `n` would stay at 3.

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

It works, but you only moved blocking work onto the thread pool. The method looks async. Under load those threads still get used.

`Task.Run` takes some work and runs it on a thread-pool thread, one of the shared worker threads. The method awaits that, gets 42, and returns it, and `.Result` reads 42. So the output is simple.

The point is what happens with real work. A truly async call, like a well-written HTTP or database call, frees the thread while it waits on the network. `Task.Run` around a blocking call doesn't do that. The blocking still happens; it is just on another thread now. It is like handing your waiting-in-line job to a colleague: you are free, but someone is still standing in the queue.

**Real-world example:**

`RazorpayService.CreateOrderAsync` does `await Task.Run(() => _client.Order.Create(options))` because the Razorpay SDK call is sync. Honest answer: wrap it, but do not pretend it is non-blocking I/O.

**Cross-question:** `.Result` on that in a controller?

**Cross-answer:**

Don't. I `await CreateOrderAsync()` in the action. `.Result` can stall the thread pool. You would block the request thread while another pool thread does the blocking SDK call, so one request ties up two threads.


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

Step by step: the second line only saves the filter. It does not look at the array yet. Then `numbers[1] = 5` changes the array to `{ 1, 5, 3 }`. Only when `First()` is called does the query actually run. It checks 1 (not greater than 1, skipped), then 5 (passes), and returns it. If the query had run straight away, the answer would have been 2. The query reads the data when you use it, not when you write it.

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

`var a = x;` gives `a` its own copy of 10. After that, `a` and `x` have nothing to do with each other. `a++` changes only `a` to 11, and `x` stays 10. It is like copying a phone number onto a new sheet: changing the copy doesn't change the original. With a class (a reference type), both variables would point at the same object, and a change through one would show through the other.

### Q18. What is printed?
```csharp
var tasks = Enumerable.Range(1, 3)
    .Select(async x => { await Task.Delay(10); return x * 2; });
var result = await Task.WhenAll(tasks);
Console.WriteLine(string.Join(",", result));
```
**Answer:** `2,4,6`.

**Why:** Each async lambda returns a Task, and `WhenAll` waits for all of them before returning the results.

Step by step: `Select` turns 1, 2, 3 into three tasks. Each waits 10 ms and then returns double its number. `WhenAll` starts waiting on all three together and finishes when the last one is done. It returns the results in the same order as the tasks were given, not in the order they finished, so the array is `[2, 4, 6]`. `string.Join` puts commas between them, giving `2,4,6`.
