# Tell me about yourself

This is usually the first question. They want to hear if you can talk clearly and stay on topic.

Speak for **60–90 seconds**. Do not read your whole CV.

---

## Q1. Tell me something about yourself.

**Answer:**

You can fill in the blanks. Keep it short.

> I am a full-stack developer. I work with C#, ASP.NET Core, and Angular. In my day job I build APIs and screens — login, lists, dashboards, that kind of work.
>
> Lately I have worked on incident-style products: a dashboard, filters, background jobs, and a database per customer. Before that I built business apps with forms, reports, and Angular plus .NET.
>
> I like taking a feature from the API all the way to the UI. I also like writing code that the next person can read. I am looking for a role where I can go deeper on the backend and still stay close to Angular.

**Cross-question:** That is a bit general. Tell me **one** project in 30 seconds.

**Cross-answer:**

> I built the incident dashboard. List, filters, and a few count cards. The API had to return totals without showing another customer’s data. The hard part was not the chart. The hard part was login checks, keeping tenants apart, and making the query fast with EF Core.

---

## Q2. Walk me through your resume.

**Answer:**

Pick three points, newest first. Skip school unless they ask.

> I will keep this to recent work. Right now I am a full-stack engineer on .NET APIs and Angular. I owned feature X — for example the incident dashboard — API, database queries, and the screen. Before that I worked on Y, where I learned Z, like auth or background jobs.

**Cross-question:** Why are you looking / why did you leave?

**Cross-answer:**

Stay kind. Do not blame people.

> I want a place where I can own bigger backend work — design, speed, helping others — and still ship Angular screens. I am not running away from a person. I want a better fit for growth.

---

## Q3. What are your strengths?

**Answer:**

Give two, with a small proof.

> First, I can turn a fuzzy request into a small slice that ships: table, API, screen, and who is allowed to use it. Second, I debug in a calm order. I reproduce it, I find which layer is wrong — UI, API, EF, or database — then I fix the real cause, not a patch on top.

**Cross-question:** When did one of those strengths go wrong?

**Cross-answer:**

> I once built a big abstraction before the need was clear. “Clean architecture” slowed us down. Now I ship the simple version first. I split it out only when a second real use case shows up.

---

## Q4. What is your biggest weakness?

**Answer:**

Use a real one, not “I work too hard”. Then say what you changed.

> I used to start coding before I wrote what “done” means. I missed empty lists and “user has no permission”. Now I write three paths first: happy path, empty path, and not allowed. Then I open the editor.

Do not say you are bad at the main skill of the job.

**Cross-question:** How would your last lead describe you in one line?

**Cross-answer:**

> Delivers what he promises, asks questions early, sometimes too picky in code review. I am trying to keep comments in line with real risk.

---

## Q5. Why should we hire you?

**Answer:**

Match **their** job, not a generic speech.

> You need someone who can ship a .NET API and an Angular screen in the same sprint, and who does not treat security as extra homework. That is how I already work: check the tenant, use `[Authorize]`, send DTOs not full database rows. I can start helping in week one because this is my stack.

**Cross-question:** What would your first 30 days look like?

**Cross-answer:**

> Week 1: run the app, read the docs, fix a small bug. Week 2 and 3: own one small feature end to end. Week 4: suggest one real improvement based on what I saw, not a random blog idea.

---

## Q6. Describe a conflict at work.

**Answer:**

Use a story: situation, what you did, result.

> A product manager wanted a “quick export” with every column, including internal ids. I wanted to keep private fields out. I offered a safe file with allowed columns, and a follow-up ticket for extra fields. We shipped on time. The extra columns came later when we had a real need.

**Cross-question:** What if they still say “just do it”?

**Cross-answer:**

> I write down the risk in plain words — “this field is personal data” — and I offer a cheaper option. I will not quietly ship a security hole to keep the peace.

---

## Q7. Tell me about a production bug you fixed.

**Answer:**

> After login, users saw an empty list. The screen was fine. The API returned 200 with no rows. The tenant id on the token was wrong after a refresh. I logged the tenant id (not the token), added a test that user A cannot see tenant B, and shipped a fix. After that, if tenant context is missing, we fail closed. We do not return “all rows”.

**Cross-question:** How did you stop it happening again?

**Cross-answer:**

> A test for cross-tenant access, and a rule: no tenant means 401 or 403, never a full dump.

---

## Q8. Where do you see yourself in 3 to 5 years?

**Answer:**

> I want to be the person the team trusts with the hard path: speed, security, and design reviews. I still want to write code, not only slides. Lead is fine if it means helping the team, not leaving the keyboard.

**Cross-question:** Do you want to be a manager?

**Cross-answer:**

> I am open to tech lead. People management is a different job. I would take it only if I am ready to be judged on other people’s success.

---

## Q9. Do you have any questions for us?

**Answer:**

Always yes. Try three:

1. What does success look like in six months for this role?
2. How do you handle production issues? Who is on call?
3. How do frontend and backend share the API contract?

**Cross-question:** Why those?

**Cross-answer:**

> They tell me if I will be blocked, if quality is real, and if the API is treated like a product.

---

## Q10. How do you explain a technical thing to a non-tech person?

**Answer:**

> I drop the jargon. I say what the user will see, what could go wrong, and how long it takes. Example: instead of “we need a distributed cache”, I say “the second server does not see the first server’s memory, so users may get logged out. Redis is shared memory both servers can use.”

**Cross-question:** Give a 20-second version of multi-tenant.

**Cross-answer:**

> Each customer’s data lives in its own box. User from company A should never see company B, even if they guess an id.

---

## Q11. What are you most proud of?

**Answer:**

Pick one story with a result.

> I am proud of the incident dashboard. Users used to export Excel to count tickets. We gave them live numbers, with the right permissions. Support calls about “wrong counts” went down because we counted in one place on the server.

**Cross-question:** What would you do differently?

**Cross-answer:**

> I would agree the meaning of each number with the business first. “Open” meant different things to two teams. The code was fine. The definition was not.

---

## Q12. How do you handle a tight deadline?

**Answer:**

> I cut scope, not quality of the risky parts. We ship the smallest useful version: one screen, one API, auth on. Nice-to-haves go to the next sprint. I say the trade-off out loud so nobody is surprised.

**Cross-question:** Have you ever missed a date?

**Cross-answer:**

> Yes. I flagged it as soon as I knew, with a new date and what we could still ship. Silent delay is worse than an honest slip.

---

## 90-second script (fill this in)

> I am **[name]**. I have **[X years]** as a full-stack developer in **C# / ASP.NET Core / Angular**.
> Right now I **[one line]**.
> A project I am proud of is **[name]**. The problem was **[…]**. I **[what you did]**. The result was **[…]**.
> I am strongest at **[skill]**.
> I want this role because **[why it matches]**.
