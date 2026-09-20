# C# — Advanced

These separate “I used the syntax” from “I know what the runtime is doing”. Still keep the words simple.

---

## Q1. How does garbage collection work?

**Answer:**

The GC frees heap objects that nothing points to. Short-lived objects are Gen 0. If they survive, they move to Gen 1, then Gen 2. Big objects go to the large object heap. Collections can pause threads.

**Cross-question:** How can a GC language still “leak”?

**Cross-answer:**

You still hold a reference: a static list, an event handler you never unsubscribed, a cache with no limit. The GC cannot collect what you still point at. Unmanaged stuff also leaks if you skip `Dispose`.

---

## Q2. Dispose vs async dispose vs finalizer?

**Answer:**

`Dispose` is “clean up now”. `DisposeAsync` is for cleanup that needs await, like flushing a network buffer. A finalizer is a last chance, slow, and you cannot await there. Prefer `SafeHandle` over a custom finalizer.

**Cross-question:** Why is a finalizer a last resort?

**Cross-answer:**

The object must live longer for the finalizer thread. It is hard to write correctly.

---

## Q3. `Span<T>` and `stackalloc`?

**Answer:**

`Span<T>` is a window over memory, no extra array if you do not need one. `stackalloc` puts bytes on the stack — fast and limited. `Span` cannot live on a class field.

**Cross-question:** When do I need `Memory<T>`?

**Cross-answer:**

Async methods, fields, queues. Take `.Span` only when you actually read or write.

---

## Q4. Covariance and contravariance?

**Answer:**

Covariance (`out T`): IEnumerable of string is IEnumerable of object. I only produce T.

Contravariance (`in T`): Action of object can be used as Action of string. I only consume T.

`List<T>` is neither, because you both add and read.

**Cross-question:** Why arrays are a trap?

**Cross-answer:**

`string[]` can be used as `object[]`. Then `o[0] = 123` blows up at runtime. Generics do not allow that hole.

---

## Q5. How is async compiled? What is a sync context?

**Answer:**

The compiler makes a state machine. Locals that live across `await` become fields. A synchronization context (old ASP.NET, WinForms) says “continue on this same thread”. ASP.NET Core does not install one by default.

**Cross-question:** When do I use `ConfigureAwait(false)`?

**Cross-answer:**

In libraries, so I do not force a UI context and I reduce deadlock risk. In ASP.NET Core app code it matters less. Do not use it if the next line must run on the UI thread.

---

## Q6. Channels vs `Task.WhenAll`?

**Answer:**

`WhenAll` waits for a known set of tasks. A `Channel` is a pipeline: producers and consumers, maybe forever, with a size limit so you do not flood memory.

**Cross-question:** `IAsyncEnumerable`?

**Cross-answer:**

`await foreach` — pull the next item when you are ready. Nice for paging an API.

---

## Q7. Interlocked vs lock vs ConcurrentDictionary?

**Answer:**

`Interlocked` is for simple atomic numbers. `lock` is the usual tool. ConcurrentDictionary is for many threads hitting a map. It is not always faster. If almost no extra threads, a simple lock can win. Measure.

**Cross-question:** Is `volatile` a lock?

**Cross-answer:**

No. It only limits some reordering and stale reads. I do not use it for business rules.

---

## Q8. Expression trees vs `Func`?

**Answer:**

`Expression<Func<...>>` is data. EF reads the tree and writes SQL. A `Func` is already code. EF cannot see inside it.

**Cross-question:** Can I compile an expression?

**Cross-answer:**

Yes, `.Compile()` makes a delegate. Then it is no longer a tree. Do not pass that to EF and expect SQL.

---

## Q9. Source generators — why do they exist?

**Answer:**

They write C# at compile time (JSON, logging, DI). You pay at build, not with slow reflection at startup. Good for AOT too.

**Cross-question:** Generator vs reflection?

**Cross-answer:**

Reflection is flexible and slower. Generators are rigid and fast.

---

## Q10. Nullable reference types — are they runtime safe?

**Answer:**

No. They are compiler warnings. `string` and `string?` are the same at runtime. You can still get null reference errors if you use `!` or turn warnings off.

**Cross-question:** Then why bother?

**Cross-answer:**

They catch a lot of bugs before production. I treat those warnings as errors on new code.

---

## Q11. Pattern matching that is more than `is int`?

**Answer:**

Property patterns, `>`, `and` / `or`, list patterns like `[1, .., var last]`. They make checks shorter and safer than a pile of ifs.

**Cross-question:** Switch vs visitor?

**Cross-answer:**

Switch is fine for a closed set of types. Visitor is for when many operations grow on a stable set of types. Most apps are fine with switch + records.

---

## Q12. Primary constructors — any catch?

**Answer:**

`class Foo(IClock clock)` is great for DI. The parameter is in scope for the whole class. Do not treat it like a casual local you can mutate without thought. For domain objects with rules, a normal constructor that validates can be clearer.

**Cross-question:** Collection expressions `[1,2,3]`?

**Cross-answer:**

Nice syntax for lists and spans. Same idea as collection initializers, shorter.

---

## Q13. Assembly vs module vs JIT?

**Answer:**

An assembly is the DLL you ship. The CLR loads it and JIT-compiles methods on first call (unless you used AOT / ReadyToRun).

**Cross-question:** Does a strong name make the app secure?

**Cross-answer:**

It proves the DLL identity and that it was not edited. It is not encryption and not auth for users.

---

## Q14. Generic math (`INumber<T>`)?

**Answer:**

Now you can write one method that adds `int` or `double` because numbers share interfaces with static operators. Old C# had no “T must support `+`”.

**Cross-question:** Boxing in generic math?

**Cross-answer:**

The point is to avoid boxing. Constraints use those interfaces so the JIT can call the right operators.

---

## Q15. `unsafe` and pointers — when?

**Answer:**

Interop, image buffers, very hot parsers. Never in normal business code. Pinning objects for too long is bad for GC.

**Cross-question:** `fixed` vs `GCHandle`?

**Cross-answer:**

`fixed` is scoped. `GCHandle` pinned is for longer native calls; you must `Free`.

---

## Q16. How would you make an immutable model?

**Answer:**

Records or `init` properties, constructor checks, `with` for copies, read-only lists. Value objects compare by data. Entities usually compare by id.

**Cross-question:** EF and immutability?

**Cross-answer:**

EF likes to set properties. Fully frozen entities fight the ORM. Common pattern: mutable entity for storage, immutable events/DTOs around it.

---

## Q17. Locking on a boxed int or a string?

**Answer:**

Boxing makes a new object each time — the lock does nothing useful. Strings can be interned — you might lock the same string as some other library. Always `private readonly object _gate = new()`.

**Cross-question:** `System.Threading.Lock` in newer .NET?

**Cross-answer:**

A real lock type so you do not lock the wrong thing. Still know classic `lock`.

---

## Q18. What is a memory barrier in simple words?

**Answer:**

It is a “do not reorder this” fence for the CPU and compiler. You almost never write one by hand. `lock` and `Interlocked` already include the needed fences.

**Cross-question:** Should I use `Thread.MemoryBarrier` in a web app?

**Cross-answer:**

No. Use higher-level tools.

---

## Q19. `ref` returns and `ref` fields — why do they exist?

**Answer:**

So hot code can pass around a location without copying a big struct. Easy to create bugs (dangling refs). I do not use them in normal API code.

**Cross-question:** Can a `ref` return point at a local that is gone?

**Cross-answer:**

The compiler tries to stop that. That is why `ref struct` and ref safety rules exist.

---

## Q20. How do you think about allocation in a hot path?

**Answer:**

I look for: boxing, extra strings, LINQ in a tight loop, lambdas that allocate, `ToList` for no reason. I only go to `Span` after measuring. Clear code first.

**Cross-question:** Is LINQ always slow?

**Cross-answer:**

No. It is extra delegates and sometimes extra collections. On a 20-item list nobody cares. On a 10-million loop in a game tick, maybe.

---

## Q21. `ConfigureAwait` on ASP.NET Core — still needed?

**Answer:**

App code: usually not, because there is no sync context. Library code: still a good habit. Never block with `.Result` either way.

**Cross-question:** Can `await` resume on another thread?

**Cross-answer:**

Yes. After I/O, a different thread-pool thread may continue. Do not keep thread-local data without care. Do not use `Thread.CurrentPrincipal` as your only user store.

---

## Q22. What is IL and why would you look at it?

**Answer:**

IL is the bytecode C# compiles to. I look at it (SharpLab, ILSpy) when I need to know if something boxes or allocates. I do not read IL every day.

**Cross-question:** Does `async` make huge IL?

**Cross-answer:**

The state machine is bigger than a sync method. That is normal. I still use async for I/O.

---

## Q23. Records and `with` — is it a deep copy?

**Answer:**

No. It copies the top level. Nested objects are still shared unless you replace them too.

**Cross-question:** Record equality with a list inside?

**Cross-answer:**

The generated equality uses the list’s own Equals, which for `List<T>` is not “same items”. People get surprised. For value-like lists, think carefully or compare yourself.

---

## Q24. How would you cancel async work?

**Answer:**

Pass a `CancellationToken` from the HTTP request or a timeout. Check it in loops. Pass it to EF and HttpClient. Do not ignore it.

**Cross-question:** What if I swallow `OperationCanceledException`?

**Cross-answer:**

The caller thinks the work finished. For HTTP, cancellation is normal when the user navigates away. I let it bubble as cancel, not as a 500.

---

## Q25. Difference between parallelism and concurrency?

**Answer:**

Concurrency: many tasks in progress (async I/O). Parallelism: many CPUs running at the same time (`Parallel.For`, `Task.Run` on CPU work). A web API is mostly concurrent I/O, not “use all cores to compute one request”.

**Cross-question:** Should an API `Parallel.ForEach` a database save?

**Cross-answer:**

Usually no. `DbContext` is not thread-safe. You can hammer the database and make things worse.


---

## Additional Questions

### Q26. What is `IAsyncEnumerable<T>` useful for?
**Answer:** It gives me a stream of async results. I can process each item with `await foreach` instead of waiting for the whole collection.

**Cross-question:** Does it automatically make database queries cheap?

**Cross-answer:** No. I still need sensible filtering, indexes, and paging. Streaming only changes how results are consumed.

### Q27. What happens when `Task.WhenAll` has multiple failures?
**Answer:** The combined task becomes faulted. I can inspect the individual tasks if I need all failures; I do not assume only the first operation failed.

**Cross-question:** Would you use it for 10,000 database writes?

**Cross-answer:** No. I would control concurrency and avoid overwhelming the database.

### Q28. When is `ValueTask` actually worth using?
**Answer:** When a method very often completes synchronously and allocation matters. Otherwise I prefer `Task` because it is simpler and easier to compose.

**Cross-question:** Would you use it everywhere for performance?

**Cross-answer:** No. I measure first. `ValueTask` has usage rules and can make code harder to reason about.
