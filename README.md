# Full-Stack Interview Q&A (Docsify)

Interview questions from **basic → advanced**, with **cross-questions** and **model answers**.

Topics:

- Tell me about yourself (HR / intro)
- C#
- .NET (ASP.NET Core, EF Core, architecture)
- Angular
- JavaScript

This is a **separate project** from Elevate Solutions. It is Markdown-only and is meant to be read in the browser with **Docsify** (`docsify serve`).

> People often type **docify**. The tool used here is **[Docsify](https://docsify.js.org/)**.

---

## Run locally

### 1. Install Node.js

You need Node.js 18+ and npm.

### 2. Install dependencies

```powershell
cd D:\Techriff-Project\interview-qa-docsify
npm install
```

### 3. Start Docsify

```powershell
npm start
```

Open:

**http://localhost:3000**

Use the left sidebar to browse every topic. Search is enabled in the top bar.

If `docsify` is already installed globally:

```powershell
npx docsify serve docs --port 3000 --open
```

or:

```powershell
docsify serve docs --port 3000
```

---

## GitHub (private — not on the portfolio)

This project is **not** part of `memukeshsahoo.github.io`. That repo stays your public portfolio.

Interview notes go in a **separate private repo**. GitHub Pages is **off**, so nothing appears at:

- `https://memukeshsahoo.github.io/`
- `https://memukeshsahoo.github.io/interview-qa-docsify/`

Read the notes on your machine:

```powershell
cd D:\Techriff-Project\interview-qa-docsify
npm start
```

Then open **http://localhost:3000**.

Do **not** commit `node_modules`. `.gitignore` already excludes it.

Hosted Docsify site (separate from the portfolio home page):

**https://memukeshsahoo.github.io/interview-qa-docsify/**

---

## How to use these answers

- Read the **main answer** first (60–90 seconds in a real interview).
- Then read the **cross-question**. Interviewers usually drill one layer deeper.
- Prefer explaining **why**, not only **what**.
- When a snippet is shown, be ready to talk through it without looking.

---

## Folder map

```text
interview-qa-docsify/
  README.md              ← this file
  package.json
  docs/
    index.html           ← Docsify entry
    README.md            ← home page
    _sidebar.md          ← left navigation
    hr/
    csharp/
    dotnet/
    angular/
    javascript/
```
