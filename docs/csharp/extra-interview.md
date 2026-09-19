# C# — Extra Interview Questions

## Q1. First vs FirstOrDefault vs Single?

**Answer:** `First` returns the first match and throws if there is none. `FirstOrDefault` returns the default when there is none. `Single` expects exactly one match and throws if there are zero or more than one.

**Cross-question:** When would you use `Single`?

**Cross-answer:** When the business rule says the result must be unique, like a lookup by a unique email or ID.

## Q2. What is SelectMany?

**Answer:** `Select` maps one item to one result. `SelectMany` flattens nested collections into one sequence.

**Cross-question:** Is it the same as Select?

**Cross-answer:** No. Select can produce a sequence of collections; SelectMany produces one flat sequence.

## Q3. Why avoid multiple enumeration?

**Answer:** If the source is expensive, every enumeration can repeat the work. With EF Core, that can mean multiple database queries. I materialize once when I actually need to reuse the results.