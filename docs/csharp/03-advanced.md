# C# — Advanced

These separate “I used the syntax” from “I know what the runtime is doing”. Still keep the words simple.

---

## Q1. How does garbage collection work?

**Answer:**

The GC (garbage collector) frees heap objects that nothing points to anymore. New, short-lived objects start in Gen 0. If they survive a collection, they move to Gen 1, and then to Gen 2. Big objects go to a separate large object heap. A collection can pause your threads while it runs.

In plain words: every time I write `new`, .NET puts that object in a shared area of memory called the heap. I never free it myself. Every so often the GC walks through what my program can still reach (static fields, local variables on running threads, and so on). Anything it cannot reach is rubbish, and that memory gets reused. Without this, I would have to free memory by hand, like in C, and one mistake would mean a crash or a slow leak.

The “generations” idea comes from a simple observation: most objects die young. A string built for one log line is gone a moment later. So the GC checks the young area (Gen 0) often, because that is cheap and frees a lot. Objects that keep surviving, like a cache that lives for the whole app, move up to Gen 1 and Gen 2, which are checked much less often. It is like cleaning your desk every day but clearing out the attic only once a year. Very big objects (85,000 bytes or more, such as large arrays) go on the large object heap, because moving them around would be expensive.

**Cross-question:** How can a GC language still “leak”?

**Cross-answer:**

You still hold a reference somewhere. Think of a static list, an event handler you never unsubscribed, or a cache with no limit. The GC cannot collect what you still point at. Unmanaged resources also leak if you skip `Dispose`.

So the GC only cleans what is truly forgotten. If a static list keeps growing, every item in it is still “in use” as far as the GC is concerned, and memory climbs until the app slows down or crashes. The event handler case is sneaky: when object A subscribes to an event on a long-lived object B, B now holds a reference to A, so A can never be collected until it unsubscribes. And things like file handles or database connections live outside .NET’s memory, so the GC does not know to close them quickly. That is the job of `Dispose`.

---

## Q2. Dispose vs async dispose vs finalizer?

**Answer:**

`Dispose` means “clean up now”. `DisposeAsync` is for cleanup that needs an await, like flushing a network buffer. A finalizer is the last chance. It is slow, and you cannot await inside it. I prefer `SafeHandle` over writing my own finalizer.

Why this matters: some objects hold things the GC does not manage, like an open file, a socket or a database connection. If I wait for the GC, that file might stay locked for a long time. So I call `Dispose` (usually with a `using` block) to release it the moment I am done, like turning off the tap instead of waiting for the water bill. `DisposeAsync` does the same job with `await using`, for cleanup that has to talk to the network or disk and should not block a thread while it waits.

A finalizer is a method the GC runs on an object before it reclaims it, just in case nobody called `Dispose`. It is a safety net, not the plan. `SafeHandle` is a ready-made class that wraps a native handle and already has a correct finalizer, so I don’t have to write that tricky code myself.

**Cross-question:** Why is a finalizer a last resort?

**Cross-answer:**

The object has to stay alive longer so the finalizer thread can run it. And finalizers are hard to write correctly.

In practice, an object with a finalizer cannot be freed in the first collection that finds it unreachable. It goes into a queue, a separate finalizer thread runs it later, and only after that can the memory be freed. So the object (and everything it points to) lives longer than it should. You also don’t know when the finalizer will run or in what order, and other objects it touches may already be finalized. That is why I clean up with `Dispose` and treat the finalizer only as a backup.

---

## Q3. `Span<T>` and `stackalloc`?

**Answer:**

`Span<T>` is a window over existing memory, so you don’t create an extra array when you don’t need one. `stackalloc` puts bytes on the stack. That is fast, but the space is limited. One rule to remember: a `Span` cannot be a class field.

Think of a long sentence written on paper. If I want to work with just one word, I don’t need to copy it onto a new sheet. I can put a frame over that word and work through the frame. That frame is `Span<T>`. For example, when parsing a big string, slicing it with a span gives me pieces without allocating new strings each time, which means less work for the GC.

`stackalloc` gives me a small buffer on the stack instead of the heap. The stack is the method’s own scratch space, and it is cleaned up for free when the method returns, so the GC is not involved. But the stack is small, so this is only for small buffers. A `Span` cannot be a class field because it may point at stack memory, and that memory disappears when the method ends. The compiler blocks anything that could outlive it.

**Cross-question:** When do I need `Memory<T>`?

**Cross-answer:**

In async methods, in fields, and in queues — places where `Span` is not allowed. I take `.Span` only at the point where I actually read or write.

`Memory<T>` is the same “window over memory” idea, but it is allowed to live on the heap, so it can be stored in a field or survive across an `await`. I think of `Memory<T>` as the ticket I can carry around, and `.Span` as actually opening the door when I need to touch the data.

---

## Q4. Covariance and contravariance?

**Answer:**

Covariance (`out T`): an IEnumerable of string can be used as an IEnumerable of object. It works because the type only produces T.

Contravariance (`in T`): an Action of object can be used as an Action of string. It works because the type only consumes T.

`List<T>` is neither, because you both add to it and read from it.

In everyday terms: if a box only gives things out, and everything it gives is a string, it is safe to say “this box gives out objects”, because every string is an object. That is covariance. If a machine accepts anything (any object), then it is safe to feed it only strings. That is contravariance. A list both takes things in and gives things out, so neither swap is safe. If I could treat a list of strings as a list of objects, someone could add a number to it, and a string reader would later get a number. The `out` and `in` keywords on interfaces and delegates are how C# marks which direction is safe.

**Cross-question:** Why arrays are a trap?

**Cross-answer:**

A `string[]` can be used as an `object[]`. Then `o[0] = 123` compiles but blows up at runtime. Generics do not allow that hole.

Arrays got this covariance in early .NET, before generics existed, and it is unsafe for exactly the reason above. The compiler lets the code through, and the runtime has to check every write into the array. When the check fails, you get an `ArrayTypeMismatchException`. Generic interfaces like `IEnumerable<T>` avoid this because covariance is only allowed when the type can’t accept T as input.

---

## Q5. How is async compiled? What is a sync context?

**Answer:**

The compiler turns an async method into a state machine. Local variables that live across an `await` become fields on that state machine. A synchronization context (old ASP.NET, WinForms) says “continue on this same thread” after the await. ASP.NET Core does not install one by default.

A state machine here is just an object that remembers “where was I?” When my method hits an `await` on something that isn’t finished, the method returns early. Later, when the work completes, the state machine jumps back in at the right step with my local variables still there. It is like putting a bookmark in a book: you close it, do something else, and open it at the same page later. That is why the thread is free to do other work while waiting on I/O.

The synchronization context decides where the code after `await` runs. In a desktop app, UI controls can only be touched from the UI thread, so the context sends the rest of the method back to that thread. ASP.NET Core doesn’t have one, so the rest of the method just runs on whatever thread-pool thread is free.

**Cross-question:** When do I use `ConfigureAwait(false)`?

**Cross-answer:**

In libraries. That way I do not force the caller’s UI context, and I lower the deadlock risk. In ASP.NET Core app code it matters less. Don’t use it if the next line has to run on the UI thread.

`ConfigureAwait(false)` says “I don’t need to come back to the original context, any thread is fine”. The deadlock risk is a classic one: a UI thread blocks on `.Result`, while the awaited code is waiting to get back onto that same UI thread. Each waits for the other forever. A library that uses `ConfigureAwait(false)` doesn’t need the UI thread, so it doesn’t get stuck that way.

---

## Q6. Channels vs `Task.WhenAll`?

**Answer:**

`WhenAll` waits for a known set of tasks. A `Channel` is a pipeline: producers put items in, consumers take them out, and it may run forever. You can give it a size limit so you don’t flood memory.

`WhenAll` is like waiting at the table until all five dishes you ordered arrive. You know the list up front, and you continue when everything is done. A `Channel` is more like a conveyor belt in a factory. Some workers keep putting items on the belt, others keep taking them off, and neither side needs to know how many items there will be in total. A typical use is a background worker that processes jobs as they come in.

The size limit is called a bounded channel. If the belt is full, the producer has to wait until there is room. That stops a fast producer from piling up millions of items in memory while a slow consumer falls behind.

**Cross-question:** `IAsyncEnumerable`?

**Cross-answer:**

You use it with `await foreach` and pull the next item when you are ready. It works nicely for paging through an API.

The difference from a channel is who is in charge. With `IAsyncEnumerable<T>`, the consumer asks for the next item, and only then does the producer go and fetch it. For paging, that means I load page 1, handle it, and only load page 2 when my loop asks for more.

---

## Q7. Interlocked vs lock vs ConcurrentDictionary?

**Answer:**

`Interlocked` is for simple atomic operations on numbers. `lock` is the usual tool. ConcurrentDictionary is for many threads hitting the same map. But it is not always faster. If there is hardly any contention, a simple lock can win. So I measure.

“Atomic” means it happens all at once, with no chance for another thread to sneak in halfway. Something as small as `count++` is actually read, add, write. If two threads do that at the same moment, one update can get lost. `Interlocked.Increment` does it as one step. A `lock` is a bigger tool: only one thread at a time can be inside the locked block, like a single key to a bathroom. `ConcurrentDictionary` is a dictionary built so many threads can read and write it without me writing locks around it.

“Contention” means threads fighting for the same thing at the same time. If that rarely happens, taking a plain lock is cheap, and the extra machinery in a concurrent collection may cost more than it saves. That’s why I measure instead of guessing.

**Cross-question:** Is `volatile` a lock?

**Cross-answer:**

No. It only limits some reordering and stale reads. I don’t use it for business rules.

`volatile` makes sure a thread sees a fairly fresh value of a field and that some reads and writes aren’t moved around by the compiler or CPU. But it doesn’t make a group of steps happen together. “Check the balance, then subtract” can still be broken by another thread in between. For rules like that, I need a real lock.

---

## Q8. Expression trees vs `Func`?

**Answer:**

An `Expression<Func<...>>` is data — a tree that describes the code. EF reads that tree and writes SQL from it. A `Func` is already compiled code, so EF cannot see inside it.

Compare a recipe with a cooked meal. An expression tree is the recipe: EF can read each step (“the `Age` property, greater than, 18”) and translate it into SQL, so the filtering happens in the database. A `Func` is the cooked meal: you can eat it, but you can’t read the steps back out of it.

Why it matters: if I pass a `Func` to a query where EF expects a filter, the filter can’t become SQL. Then I may end up loading far more rows than I need and filtering them in memory, which is slow on a big table.

**Cross-question:** Can I compile an expression?

**Cross-answer:**

Yes. `.Compile()` turns it into a delegate. But then it is no longer a tree, so don’t pass that to EF and expect SQL.

This is useful when I want to run the same logic in memory, for example in a unit test or to check an object in C#. I keep the expression for EF and use the compiled version only for in-memory work.

---

## Q9. Source generators — why do they exist?

**Answer:**

They write C# for you at compile time — for example for JSON, logging, and DI. You pay the cost at build time instead of with slow reflection at startup. They are also good for AOT.

Reflection is code that inspects types while the app is running: “what properties does this class have?” It is powerful but slow, and it happens on every start. A source generator looks at my code during the build and writes the extra code ahead of time, like the JSON reading and writing for a class. At runtime, the app just calls normal code that already exists.

AOT (ahead-of-time compilation) turns the app into native machine code before it ships. In that mode, some reflection tricks don’t work well, because unused code gets trimmed away. Generated code is plain C#, so it compiles and trims cleanly.

**Cross-question:** Generator vs reflection?

**Cross-answer:**

Reflection is flexible but slower. Generators are rigid but fast.

Reflection can deal with any type it meets at runtime, even one loaded from a plugin. A generator only knows about the code it saw at build time, so it can’t adapt later, but there’s no lookup cost when the app runs.

---

## Q10. Nullable reference types — are they runtime safe?

**Answer:**

No. They are only compiler warnings. At runtime, `string` and `string?` are the same type. You can still get null reference errors if you use `!` or turn the warnings off.

The `?` is a note to the compiler that says “this one might be null”. The compiler then warns me if I use it without checking. But nothing stops a null from arriving at runtime, for example from JSON, from an old library, or from code where I wrote `!`. The `!` (the “null-forgiving” operator) tells the compiler “trust me, it’s not null”, and the compiler stops warning even if I’m wrong.

So it’s a smoke alarm, not a fireproof wall. It warns me early, but I still validate input at the edges of my app.

**Cross-question:** Then why bother?

**Cross-answer:**

They catch a lot of bugs before production. On new code I treat those warnings as errors.

`NullReferenceException` is one of the most common crashes in .NET. Getting a warning while I type is much cheaper than finding the crash in production. Treating the warnings as errors means the build fails, so nobody can ignore them.

---

## Q11. Pattern matching that is more than `is int`?

**Answer:**

There are property patterns, relational patterns like `>`, `and` / `or`, and list patterns like `[1, .., var last]`. They make checks shorter and safer than a pile of ifs.

A property pattern checks values inside an object, like “an order whose `Status` is Paid”. A relational pattern compares, like “age is `>= 18`”. `and` / `or` let me combine checks in one line. A list pattern checks the shape of a list: `[1, .., var last]` means “starts with 1, anything in the middle, and give me the last item”.

They are safer because the compiler understands them. In a `switch` expression it can warn me if I missed a case, or if one case can never be reached because an earlier one already covers it. With a long chain of ifs, I only find that out when something breaks.

**Cross-question:** Switch vs visitor?

**Cross-answer:**

A switch is fine for a closed set of types. A visitor is for when you keep adding operations over a stable set of types. Most apps are fine with switch plus records.

A visitor is a design pattern where each type has an `Accept` method that hands itself to a “visitor” object, and each new operation is a new visitor class. It keeps each operation in one place, but it is a lot of ceremony. With records and switch expressions, I get most of the same benefit with much less code.

---

## Q12. Primary constructors — any catch?

**Answer:**

`class Foo(IClock clock)` is great for DI. The catch is that the parameter stays in scope for the whole class. So don’t treat it like a casual local you can change without thinking. For domain objects with rules, a normal constructor that validates can be clearer.

With a primary constructor, I declare the parameters right after the class name and can use them anywhere in the class. For services that just receive dependencies through DI (dependency injection, where the framework passes in the things a class needs), this removes a lot of boilerplate fields.

The catch: the parameter is not a `readonly` field. Any method can assign a new value to it, and that change is seen by every other method in the class. That can lead to surprising bugs. Also, if a class has rules like “price must be positive”, a normal constructor body with checks makes those rules easy to see.

**Cross-question:** Collection expressions `[1,2,3]`?

**Cross-answer:**

Nice syntax for lists and spans. Same idea as collection initializers, just shorter.

So instead of `new List<int> { 1, 2, 3 }`, I can write `[1, 2, 3]`, and the compiler builds the right type based on what the variable expects.

---

## Q13. Assembly vs module vs JIT?

**Answer:**

An assembly is the DLL you ship. The CLR (the .NET runtime) loads it, and the JIT compiles each method to machine code the first time it is called — unless you used AOT or ReadyToRun.

When I build a C# project, I don’t get machine code. I get IL, a middle-step language, packed into a DLL (or EXE) along with information about the types. That package is the assembly, and it’s also the unit of versioning. A module is the actual file inside an assembly that holds the IL. Almost every assembly has exactly one module, so day to day the two feel like the same thing.

The JIT (“just-in-time” compiler) turns IL into real machine code for the CPU, and it does that lazily, the first time each method runs. That is part of why the first request to an app is slower. AOT and ReadyToRun do some or all of that compiling at build time instead, which makes startup faster.

**Cross-question:** Does a strong name make the app secure?

**Cross-answer:**

It proves the DLL’s identity and that nobody edited it. It is not encryption, and it is not authentication for users.

A strong name is a signature added to an assembly with a key. It gives the DLL a unique identity and lets the runtime notice if the file was changed. But anyone can still open the DLL and read the code, and it has nothing to do with who is logged in to the app.

---

## Q14. Generic math (`INumber<T>`)?

**Answer:**

Now you can write one method that adds `int` or `double`, because number types share interfaces with static operators. Old C# had no way to say “T must support `+`”.

Before this, if I wanted a `Sum` method for `int`, `long`, `double` and `decimal`, I had to write four copies or use slow workarounds. Generics could not require “T has a `+` operator”, because operators are static and interfaces could not describe static members. Newer .NET added static abstract members to interfaces, and the number types now implement interfaces like `INumber<T>`. So I can write `where T : INumber<T>` and then use `+`, `T.Zero` and so on inside one method.

**Cross-question:** Boxing in generic math?

**Cross-answer:**

Avoiding boxing is the whole point. The constraints use those interfaces, so the JIT can call the right operators directly.

Boxing means wrapping a value like an `int` in an object on the heap, which costs an allocation each time. With generic math, the JIT creates a separate version of the method for each value type (one for `int`, one for `double`), so the `+` becomes a plain CPU add with no wrapping.

---

## Q15. `unsafe` and pointers — when?

**Answer:**

Interop, image buffers, and very hot parsers. Never in normal business code. Also, pinning objects for too long is bad for the GC.

`unsafe` code lets me use raw pointers, like in C. I get direct memory access, but I lose the safety checks, such as array bounds checking. One wrong pointer can corrupt memory and cause crashes that are very hard to trace. That’s why I keep it to the few places where it really helps: talking to native libraries (interop), working with raw pixel buffers, or squeezing speed out of code that runs millions of times.

Pinning means telling the GC “don’t move this object, a pointer is looking at it”. The GC normally moves objects together to remove gaps. A pinned object is like a parked car in the middle of a road the GC wants to sweep, so pinning things for a long time makes memory messier.

**Cross-question:** `fixed` vs `GCHandle`?

**Cross-answer:**

`fixed` is scoped to a block. A pinned `GCHandle` is for longer native calls, and you must call `Free` yourself.

`fixed` pins an object only while the code is inside that block, and it’s released automatically when the block ends, so it’s hard to get wrong. A pinned `GCHandle` stays pinned until I call `Free`. That’s needed when native code holds the pointer after my method returns, but if I forget `Free`, the object stays pinned and is never collected.

---

## Q16. How would you make an immutable model?

**Answer:**

I use records or `init` properties, checks in the constructor, `with` for copies, and read-only lists. Value objects compare by their data. Entities usually compare by id.

Immutable means once the object is created, it can’t change. If I want a different value, I create a new object. That makes code easier to trust: if I pass an object to another method or thread, I know nobody can change it behind my back. `init` properties can only be set while the object is being created. The constructor checks make sure an invalid object can never exist. `with` gives me a copy with one or two values changed.

A value object is something defined only by its data, like an amount of money: two “100 INR” values are the same thing. An entity has an identity that stays the same even when its data changes, like a customer who changes their address but is still the same customer, so it compares by id.

**Cross-question:** EF and immutability?

**Cross-answer:**

EF likes to set properties. Fully frozen entities fight the ORM. A common pattern is a mutable entity for storage, with immutable events or DTOs around it.

EF (Entity Framework, an ORM that maps C# classes to database tables) loads rows and tracks changes by setting and watching properties. If every property is locked, I have to fight it with extra configuration. So I let the entity change in a controlled way, and keep the objects I pass between layers (DTOs, events) immutable.

---

## Q17. Locking on a boxed int or a string?

**Answer:**

Boxing makes a new object every time, so the lock does nothing useful. Strings can be interned, so you might end up locking the same string as some other library. I always use `private readonly object _gate = new()`.

A `lock` works on one specific object, like a single key that threads pass around. The compiler won’t let me lock on an `int` directly, but if I cast it to `object`, it gets boxed into a brand-new object each time, so every thread gets its own key and nobody actually waits. The lock protects nothing.

Strings have the opposite problem. .NET may keep only one shared copy of identical string literals (that is interning). So if my code locks on `"orders"` and some other code also locks on `"orders"`, we are fighting over the same key without knowing it, which can cause slowdowns or deadlocks. A private object that only my class can see avoids both problems.

**Cross-question:** `System.Threading.Lock` in newer .NET?

**Cross-answer:**

It is a real lock type, so you don’t lock on the wrong thing by accident. You should still know the classic `lock`.

Instead of using any random object as the key, I create a dedicated `Lock` object, and the `lock` statement works with it directly. It makes the intent clear. Most existing code still uses the classic `object` approach, though, so I need to understand both.

---

## Q18. What is a memory barrier in simple words?

**Answer:**

It is a “do not reorder this” fence for the CPU and the compiler. You almost never write one by hand. `lock` and `Interlocked` already include the fences you need.

To run faster, the compiler and the CPU may change the order of reads and writes, as long as the result looks the same to a single thread. With several threads, that can break things. For example, thread A sets `data = 5` and then `ready = true`, but thread B might see `ready` as true before it sees the new `data`. A memory barrier is a line that says “everything before this must be done before anything after it”. Since `lock` and `Interlocked` already include these fences, using them correctly takes care of it.

**Cross-question:** Should I use `Thread.MemoryBarrier` in a web app?

**Cross-answer:**

No. Use the higher-level tools.

Hand-placed barriers are easy to get wrong, and the bugs only show up rarely, under load, on certain machines. In a web app, `lock`, `Interlocked`, concurrent collections and async code cover what I need.

---

## Q19. `ref` returns and `ref` fields — why do they exist?

**Answer:**

So hot code can pass around a location in memory without copying a big struct. They make it easy to create bugs, like dangling refs. I don’t use them in normal API code.

Normally, when a method returns a struct, the caller gets a copy. For a big struct used millions of times, that copying adds up. A `ref` return gives back the address of the original instead, like giving someone directions to your house rather than a photo of it. They can then read or change the real thing directly. `ref` fields let a `ref struct` (like `Span<T>`) hold such an address inside it.

A dangling ref is an address that points at memory that is no longer valid, like directions to a house that has been knocked down. That kind of bug is why I keep these features for performance-critical library code only.

**Cross-question:** Can a `ref` return point at a local that is gone?

**Cross-answer:**

The compiler tries to stop that. That is why `ref struct` and the ref safety rules exist.

A local variable lives in the method’s stack space, which is gone once the method returns. So the compiler refuses to let me return a `ref` to a plain local. The ref safety rules track how long each reference is allowed to live, and `ref struct` types are kept on the stack so they can’t outlive the memory they point at.

---

## Q20. How do you think about allocation in a hot path?

**Answer:**

I look for boxing, extra strings, LINQ in a tight loop, lambdas that allocate, and `ToList` calls for no reason. I only move to `Span` after measuring. Clear code comes first.

A hot path is code that runs extremely often, like the inner loop of a parser or a method called on every request. Every allocation there creates work for the GC, and more GC work means more pauses. The usual hidden sources: boxing a value type into an object, building strings with `+` in a loop, LINQ that creates iterators and delegates, lambdas that capture variables (each capture creates an object), and `ToList` copying data that is only read once.

But I don’t rewrite everything in `Span` on a hunch. I profile first, find the few places that actually matter, and fix those. The rest of the code stays simple and easy to read.

**Cross-question:** Is LINQ always slow?

**Cross-answer:**

No. It adds extra delegates and sometimes extra collections. On a 20-item list nobody cares. In a 10-million-item loop inside a game tick, maybe it matters.

The overhead is small and fixed per call. On normal business code, the time is spent waiting on the database or network, not on LINQ. It only becomes visible when the same code runs a huge number of times under a tight time budget.

---

## Q21. `ConfigureAwait` on ASP.NET Core — still needed?

**Answer:**

In app code, usually not, because there is no sync context. In library code it is still a good habit. Either way, never block with `.Result`.

`ConfigureAwait(false)` only matters when there is a synchronization context to return to. ASP.NET Core has none, so in my controllers and services it changes nothing. A library doesn’t know where it will be used, though. It might end up in a WPF or WinForms app that does have a UI context, so adding `ConfigureAwait(false)` there is still the safe choice.

`.Result` (and `.Wait()`) make the current thread sit and wait for the task. That wastes a thread that could be serving other requests, and in apps with a sync context it can cause a deadlock. I use `await` all the way through.

**Cross-question:** Can `await` resume on another thread?

**Cross-answer:**

Yes. After I/O, a different thread-pool thread may continue the work. So be careful with thread-local data. And don’t use `Thread.CurrentPrincipal` as your only place to store the user.

Thread-local data is stored per thread, like a note stuck to one desk. If my method starts on one thread and continues on another after `await`, that note isn’t there anymore. So I pass the data I need through parameters, or use things designed to flow with async code. In ASP.NET Core, the current user is on `HttpContext.User`.

---

## Q22. What is IL and why would you look at it?

**Answer:**

IL is the bytecode that C# compiles to. I look at it (with SharpLab or ILSpy) when I need to know if something boxes or allocates. I don’t read IL every day.

IL (Intermediate Language) is the in-between form of my code: lower-level than C#, but not yet machine code for a specific CPU. The JIT turns it into machine code when the app runs. Reading it shows what the compiler actually produced from my C#. For example, a `box` instruction tells me a value type is being wrapped into an object, and a `newobj` tells me something is being allocated. SharpLab is a website that shows the IL as you type; ILSpy is a tool that opens a compiled DLL.

**Cross-question:** Does `async` make huge IL?

**Cross-answer:**

The state machine is bigger than a sync method. That is normal. I still use async for I/O.

The extra code is the bookkeeping that lets the method pause and resume. That cost is tiny compared with what async gives back: while the I/O is waiting, the thread is free to serve other work, so the server can handle many more requests at once.

---

## Q23. Records and `with` — is it a deep copy?

**Answer:**

No. It copies only the top level. Nested objects are still shared unless you replace them too.

Say I have a record `Order` with a `Customer` name and a `List<Item>` of items. `order with { Customer = "Ravi" }` creates a new `Order` with the new name, but the items property in both orders points to the very same list. If I add an item through the copy, the original sees it too. It’s like photocopying a page that says “see the folder on shelf 3”: you get a new page, but both pages point to the same folder. If I want them separate, I give the copy its own new list.

**Cross-question:** Record equality with a list inside?

**Cross-answer:**

The generated equality calls the list’s own Equals, and for `List<T>` that does not mean “same items”. People get surprised by this. For lists that should act like values, think it through or write the comparison yourself.

`List<T>.Equals` only checks whether it’s the same list object, not whether the contents match. So two records with the same data but two separate lists containing `[1, 2, 3]` are not equal. If I need content-based equality, I override `Equals` and `GetHashCode` in the record and compare the items myself, for example with `SequenceEqual`.

---

## Q24. How would you cancel async work?

**Answer:**

I pass a `CancellationToken` from the HTTP request or from a timeout. I check it inside loops. I pass it on to EF and HttpClient. I don’t ignore it.

A `CancellationToken` is like a shared “stop” flag. Whoever started the work can flip it, and the code doing the work checks it and stops early. In ASP.NET Core, the framework gives me one that fires when the client disconnects. Why it matters: without it, if a user closes the page, the server keeps running a heavy report or database query that nobody will ever see, which wastes CPU and database time.

Passing the token down to EF and `HttpClient` matters because they can stop the actual database query or network call. Checking it in my own loops lets long work end at a sensible point.

**Cross-question:** What if I swallow `OperationCanceledException`?

**Cross-answer:**

Then the caller thinks the work finished. For HTTP, cancellation is normal — the user just navigated away. I let it bubble up as a cancel, not as a 500.

`OperationCanceledException` is how .NET reports “this was stopped on purpose”. If I catch it and carry on, the code above me may treat half-done work as complete, for example saving partial results. And it shouldn’t be logged or returned as a server error (500), because nothing actually broke. It would just fill the error logs with noise.

---

## Q25. Difference between parallelism and concurrency?

**Answer:**

Concurrency means many tasks are in progress at once, like async I/O. Parallelism means many CPUs are running at the same time, like `Parallel.For` or `Task.Run` on CPU work. A web API is mostly concurrent I/O, not “use all cores to compute one request”.

A kitchen example: one cook who puts rice on to boil, then chops vegetables while it cooks, then checks the oven, is doing concurrency. Several things are in progress, but one person handles them by switching while things wait. Four cooks each chopping at the same moment is parallelism, more hands doing actual work together.

Most of a web API’s time goes into waiting on the database, other APIs or disk. Async lets one thread start many of those waits and move on. Parallelism helps when there is heavy computing to split up, like resizing many images, and that’s less common inside a single web request.

**Cross-question:** Should an API `Parallel.ForEach` a database save?

**Cross-answer:**

Usually no. `DbContext` is not thread-safe. And you can hammer the database and make things worse.

“Not thread-safe” means two threads using the same `DbContext` at once can corrupt its internal tracking or throw errors. Even with a separate context per thread, firing many saves at once just makes them compete for the same database connections and locks, which can be slower. A batched save or a simple loop is usually better.


---

## Additional Questions

### Q26. What is `IAsyncEnumerable<T>` useful for?
**Answer:** It gives me a stream of async results. I can handle each item with `await foreach` instead of waiting for the whole collection.

The difference is like streaming a video versus downloading the whole file first. With a normal `Task<List<T>>`, I wait until every item is loaded into memory, and only then start working. With `IAsyncEnumerable<T>`, I can start on the first item as soon as it arrives, and I don’t have to hold everything in memory at once. That helps with large result sets, reading a big file line by line, or pulling pages from an API.

**Cross-question:** Does it automatically make database queries cheap?

**Cross-answer:** No. I still need sensible filtering, indexes, and paging. Streaming only changes how I consume the results.

If the query is slow or returns a million rows I don’t need, the database still does all that work. Streaming just means my app handles the rows one by one instead of all at once. The cost of the query itself stays the same.

### Q27. What happens when `Task.WhenAll` has multiple failures?
**Answer:** The combined task becomes faulted. If I need all the failures, I look at the individual tasks. I don’t assume only the first operation failed.

“Faulted” means the task ended with an error. When I `await` the combined task, it throws only the first exception, which can hide the rest. For example, if three API calls fail for three different reasons, a log of just the first one tells only a third of the story. To see everything, I can check each task’s `Exception` property, or look at the combined task’s `Exception`, which holds all the errors together in an `AggregateException`.

**Cross-question:** Would you use it for 10,000 database writes?

**Cross-answer:** No. I would control how many run at once so I don’t overwhelm the database.

`Task.WhenAll` starts nothing itself. It just waits for tasks that are already running, so creating 10,000 write tasks means 10,000 writes hit the database at roughly the same time. That can use up the connection pool and slow everything down. Instead, I limit it, for example with `Parallel.ForEachAsync` and a max degree of parallelism, or a `SemaphoreSlim`, or I batch the writes into fewer, bigger operations.

### Q28. When is `ValueTask` actually worth using?
**Answer:** When a method very often completes synchronously and the allocation matters. Otherwise I prefer `Task`, because it is simpler and easier to compose.

Every `Task` is an object on the heap. If a method usually has the answer ready right away, for example reading from an in-memory cache and only going to the database on a miss, creating a `Task` object on every call is wasted work. `ValueTask` is a struct that can hold the ready result directly, so the common fast path allocates nothing. That only matters when the method is called a huge number of times.

**Cross-question:** Would you use it everywhere for performance?

**Cross-answer:** No. I measure first. `ValueTask` has usage rules and can make code harder to reason about.

The main rule: a `ValueTask` should be awaited only once. I shouldn’t await it twice, await it from two places, or call `.Result` on it before it finishes. A `Task` allows all of that safely. If I need those things, I convert it with `.AsTask()`. For most code, the saved allocation isn’t worth the extra care.