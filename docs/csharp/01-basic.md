# C# — Basic

If these feel weak, the interview often stops here. Speak in plain words, then add the name of the idea.

---

## Q1. What is C#? How is it different from .NET?

**Answer:**

C# is the **programming language** I write. .NET is the **platform** that provides the runtime and libraries I use for things like collections, files, networking, and web APIs. When I compile C# code, it is converted into **Intermediate Language (IL)**. When the application runs, the **Common Language Runtime (CLR)** uses the **Just-In-Time (JIT) compiler** to convert that IL into machine code that the computer can execute. So, simply: **C# is the language, and .NET is the platform that runs it and provides the libraries.**

**Cross-question:** .NET Framework vs .NET Core vs just “.NET”?

**Cross-answer:**

Old .NET Framework is Windows only. .NET Core was the cross-platform rewrite. From .NET 5 onward it is just called .NET. Today we target things like `net8.0` or `net10.0`.

---

## Q2. Value type vs reference type?

**Answer:**

A **value type** (`int`, `bool`, `struct`, `DateTime`) holds the data itself. Copy the variable, you copy the data.

A **reference type** (`class`, `string`, arrays) holds a pointer to the object. Copy the variable, both names point at the same object.

Do not say “structs always live on the stack”. A struct field on a class lives with that object on the heap.

**Cross-question:** Is `string` a value type because it cannot change?

**Cross-answer:**

No. `string` is a reference type. It is immutable, meaning each change makes a new string. Copying a string variable still copies the reference.

---

## Q3. Stack vs heap?

**Answer:**

The **stack** is for short-lived method data. When the method ends, it is gone.

The **heap** is for objects. The garbage collector cleans them when nothing points at them.

**Cross-question:** Where is a `struct` inside a class?

**Cross-answer:**

On the heap, as part of that object.

---

## Q4. Boxing and unboxing?

**Answer:**

Boxing is putting a value like `int` into an `object`. That makes a small heap object. Unboxing pulls it back out.

It is extra work. In a loop, prefer `List<int>` over an old `ArrayList` of ints.

**Cross-question:** If a method takes `object` and I pass an `int`, is that boxing?

**Cross-answer:**

Yes. Same idea with some interface calls on structs. Generics like `where T : IComparable<T>` help you skip boxing.

---

## Q5. `==` vs `Equals` vs `ReferenceEquals`?

**Answer:**

`ReferenceEquals` asks: same object in memory?

`==` on classes is usually the same, unless the type overloads it. `string` does.

`Equals` can be overridden to mean “same data”.

**Cross-question:** Two objects with the same fields: why is `==` false?

**Cross-answer:**

Because your class did not overload `==`. They are two objects. Strings look equal because `string` special-cases `==`.

---

## Q6. Access modifiers?

**Answer:**

`public` — anyone. `private` — only this type. `protected` — this type and children. `internal` — this project (assembly). There are also `protected internal` and `private protected` for mixed cases.

**Cross-question:** Default for a class? For a member?

**Cross-answer:**

A class is `internal` if you write nothing. A member is `private`.

---

## Q7. `class` vs `struct` vs `record`?

**Answer:**

A **class** is a reference type. Use it for things with identity, like a user in the database.

A **struct** is a value type. Keep it small.

A **record** is usually a class with “same data means equal”, and a nice `with` copy. Good for DTOs.

**Cross-question:** Record or class for an EF entity?

**Cross-answer:**

I use a class. EF likes mutable objects with an id. Records are nicer for request/response models.

---

## Q8. What is a constructor? Are they inherited?

**Answer:**

A constructor runs when you `new` an object. It sets it up. Child classes do **not** inherit constructors. If the parent has no empty constructor, the child must call `: base(...)`.

**Cross-question:** Static constructor?

**Cross-answer:**

It runs once for the type, before first use. No parameters. No `public`. Used to set static data.

---

## Q9. `const` vs `readonly`?

**Answer:**

`const` is baked in at compile time. It is always static. Only simple values like numbers and strings.

`readonly` can be set in the constructor. It can be runtime.

**Cross-question:** If I change a `const` in one DLL, does the other DLL need a rebuild?

**Cross-answer:**

Usually yes. The value was copied into the caller. `static readonly` is looked up at runtime, so the other DLL can see the new value after you replace the first DLL.

---

## Q10. Array vs `List<T>` vs `IEnumerable<T>`?

**Answer:**

An array has a fixed size. `List<T>` can grow. `IEnumerable<T>` is “I can foreach this”. It might not have run yet (LINQ). Do not loop it twice if it hits the database.

**Cross-question:** Why return `IReadOnlyList<T>`?

**Cross-answer:**

So the caller cannot `Add` or `Clear` my inner list. Smaller promise.

---

## Q11. `string` vs `StringBuilder`?

**Answer:**

Strings never change. `s = s + x` in a loop makes many extra strings. `StringBuilder` edits a buffer.

**Cross-question:** Always use `StringBuilder`?

**Cross-answer:**

No. A few pieces with `$"{a}-{b}"` is fine and easier to read. Use `StringBuilder` in loops.

---

## Q12. What can be null? What is `int?`?

**Answer:**

Classes can be null. `int` cannot. `int?` is a wrapper: either a number or “no value”.

**Cross-question:** `int?` vs `string?`?

**Cross-answer:**

`int?` is a real wrapper at runtime. `string?` is mostly a compiler warning. At runtime it is still a string and can still be null if you ignore warnings.

---

## Q13. `if` vs `switch`?

**Answer:**

`if` is for conditions. `switch` is nice for a list of cases. A switch **expression** returns a value in one go.

**Cross-question:** Give a simple pattern match.

**Cross-answer:**

`if (user is { IsActive: true, Role: "Admin" })` — “if this user is active and admin”.

---

## Q14. try / catch / finally?

**Answer:**

`try` is the risky code. `catch` handles a **specific** error if you can. `finally` always runs, for cleanup. Do not swallow errors. Do not use exceptions as normal flow.

**Cross-question:** `throw ex;` vs `throw;`?

**Cross-answer:**

`throw;` keeps the original stack. `throw ex;` hides where it first failed. I use `throw;` or wrap it: `throw new AppException("...", ex)`.

---

## Q15. Namespaces and `using`?

**Answer:**

Namespaces group types so names do not clash. `using System.Linq;` brings names in. `using var x = ...` is different — that **disposes** `x` at the end of the block.

**Cross-question:** File-scoped namespace?

**Cross-answer:**

`namespace MyApp.Services;` at the top. Same meaning, less nesting. C# 10+.

---

## Q16. `ref`, `out`, `in`?

**Answer:**

They pass a variable by reference.

- `ref`: already has a value; method can change it
- `out`: method **must** set it
- `in`: read-only reference, useful for big structs

**Cross-question:** `out` or a tuple?

**Cross-answer:**

`TryParse` style `out` is fine. For two results I prefer a small type or `(bool ok, User user)`. Many `out` parameters are hard to read.

---

## Q17. Overload vs override vs `new`?

**Answer:**

**Overload:** same name, different parameters.

**Override:** child replaces a `virtual` method. Calls use the real type.

**`new`:** hides the parent method. Easy to get wrong. I avoid it.

**Cross-question:** Variable type is `Base`, object is `Derived`. Which method runs?

**Cross-answer:**

`override` → child method. `new` → parent method, because the variable is `Base`. That surprise is why `new` is a smell.

---

## Q18. What does `static` mean?

**Answer:**

It belongs to the type, not one object. No `this`. A static class cannot be created with `new`.

**Cross-question:** Why are static classes hard to test?

**Cross-answer:**

I cannot easily fake `DateTime.Now` or `File.ReadAllText`. I wrap them in a small interface and inject it.

---

## Q19. What is a property?

**Answer:**

A property looks like a field (`user.Name`) but it is get/set methods. I can add checks later without changing callers.

**Cross-question:** `{ get; set; }` vs a full backing field?

**Cross-answer:**

Start with auto properties. Add a backing field when I need extra logic.

---

## Q20. `if (x == null)` vs `is null` vs `??`?

**Answer:**

`is null` is nice because it cannot be overloaded. `??` means “if left is null, use the right side”. `?.` stops if the left is null.

**Cross-question:** `??` vs `||`?

**Cross-answer:**

C# does not use `||` for defaults the way JavaScript does. Use `??` for null. For `int?`, `value ?? 0` is the usual pattern.

---

## Q21. What is an enum?

**Answer:**

A named set of numbers: `Open`, `Closed`. Easier to read than magic ints. I still store them carefully in the database and do not assume the number will never change.

**Cross-question:** Enum vs a lookup table?

**Cross-answer:**

Enum: closed list that rarely changes, used in code. Table: business can add values without a new build. Status values that product changes often belong in data.

---

## Q22. What is LINQ in one sentence?

**Answer:**

LINQ is a way to filter and map lists in C#, like `Where` and `Select`. In memory it loops objects. With EF it can become SQL.

**Cross-question:** `Count()` vs `.Count` on a list?

**Cross-answer:**

On `List<T>`, `.Count` is a property and is cheap. `.Count()` is LINQ and still fine on a list, but on an `IQueryable` it can run `SELECT COUNT` in the database. Know which type you have.

---

## Q23. `string.IsNullOrEmpty` vs `IsNullOrWhiteSpace`?

**Answer:**

Empty means `""`. White space means `"   "` too. For user input I usually use `IsNullOrWhiteSpace`.

**Cross-question:** Does it catch `" null "` as text?

**Cross-answer:**

No. That is a real string of characters. Only trim and extra checks would catch junk like that.

---

## Q24. What is an interface in simple words?

**Answer:**

It is a contract: “you must have these methods”. A class can implement many interfaces. I program against `IEmailSender`, not `SmtpEmailSender`, so tests can pass a fake.

**Cross-question:** Can an interface have code?

**Cross-answer:**

Yes, default interface methods exist. I keep them thin. Shared heavy code still belongs in a class.

---

## Q25. `DateTime` vs `DateTimeOffset` vs `DateOnly`?

**Answer:**

`DateTime` is a date and time, but timezone is easy to mess up. `DateTimeOffset` includes the offset from UTC. `DateOnly` is just a date, like a birthday.

**Cross-question:** What do you store in the database for “when was this created”?

**Cross-answer:**

UTC. I convert for the screen. Mixing local times from servers in two countries is a classic bug.
