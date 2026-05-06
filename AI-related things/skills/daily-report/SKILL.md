---
name: daily-report
description: Generate a daily work report of codebase changes for manager review. Manual invocation only — do NOT auto-trigger. Invoke only when user explicitly runs /daily-report or says "daily report", "work report", "what did I change today/yesterday", or "generate report for manager".
argument-hint: [since]
allowed-tools: [Bash]
---

# Daily Work Report

Produce a bullet-pointed summary of codebase changes for a given day. One bullet per logical area. Written for a non-technical manager — plain English, no jargon, no file names, no class names.

Works on any git repository or monorepo.

## When invoked

- `/daily-report` → today (since midnight)
- `/daily-report yesterday` → previous calendar day
- `/daily-report 2026-05-05` → specific date
- `/daily-report this week` → last 7 days

## Process

### Step 1 — get commits from ALL branches

```bash
git log --all --since="<period>" --pretty=format:"COMMIT|||%H|||%s" --name-only
```

Period rules:

- "today" / no argument: `--since=midnight`
- "yesterday": `--since="yesterday 00:00" --until="today 00:00"`
- specific date `2026-05-05`: `--since="2026-05-05 00:00" --until="2026-05-06 00:00"`
- "this week": `--since="7 days ago"`

Also get the repo root name:

```bash
basename $(git rev-parse --show-toplevel)
```

If no commits found, output: "No commits found for [date]. Nothing to report."

### Step 2 — detect projects

Look at the changed file paths and decide if this is a monorepo (multiple sub-projects) or a single project.

**Monorepo signal:** Changed files come from 2+ distinct top-level directories that each look like independent projects — i.e., they each contain files like `package.json`, `composer.json`, `*.php`, `*.py`, `go.mod`, or similar project roots. Common monorepo top-level names: `packages/`, `apps/`, `plugins/`, `services/`, `libs/`.

- If monorepo: treat each top-level sub-directory as a separate project. Use `###` headings to separate them in the output.
- If single project: no headings, just bullets.

### Step 3 — derive area labels from file paths

Do NOT use a hardcoded mapping. Instead, derive the area label dynamically from the file path:

1. Strip the project root prefix (e.g., `src/`, `lib/`, `includes/`, `app/`)
2. Take the first remaining path segment as the raw area (e.g., `Admin`, `Frontend`, `components`, `utils`)
3. If that segment is a catch-all like `index`, `main`, `helpers`, go one level deeper
4. Convert to title-case human-readable label: replace hyphens and underscores with spaces, capitalise each word

**Examples of derived labels:**

- `includes/Admin/Recovery.php` → "Admin"
- `src/components/Button.tsx` → "Components"
- `app/services/PaymentService.php` → "Services"
- `lib/utils/date.js` → "Utils"
- `templates/emails/welcome.php` → "Emails"
- `assets/css/main.css` → "Styling"
- `assets/js/checkout.js` → "Styling" (group css+js assets together as "Interface" or "Styling")
- `tests/unit/AuthTest.php` → _(skip — test files, do not include)_
- `.github/`, `.claude/`, `openspec/`, `docs/` → _(skip — tooling/docs, do not include)_
- `package.json`, `composer.json`, root config files → _(skip)_

When multiple files in a commit touch different segments, assign the commit to whichever segment has the most files. If tied, use the first one.

### Step 4 — build bullets

Each commit (or pair of commits that are trivially the same fix) gets its own bullet. Do not collapse multiple distinct changes into one. Only merge if two commits are literally the same thing (e.g., two consecutive margin tweaks on the same screen).

For each bullet:

1. Strip commit type prefixes (`fix: 🐛`, `feat: ✨`, `chore:`, `refactor:`, etc.) — keep only the plain description text
2. Write one short sentence in plain English for a non-technical reader
3. Describe what changed or what is better — not how it was done

**Tone rules:**

- Describe what the user or admin experiences, not what the code does
- "Fixed a display issue on the dashboard" not "resolved CSS margin regression in Stats.php"
- "Customers can now download invoices" not "added invoice endpoint to REST API"
- "Improved how subscription statuses appear in the list" not "updated SubscriptionStatusHandler"
- No file names, class names, function names, or tech terms

Each area gets an H3 heading. Each commit gets its own flat bullet beneath it. One sentence per bullet. Do not consolidate unless two commits are near-identical. Always add a blank line between sections.

**Format — single project:**

```
### [Area]: [Plain English description.]
```

**Format — multiple changes in same area:**

```
### [Area]
- [First change.]
- [Second change.]
- [Third change.]
```

**Format — monorepo (multiple projects):** use H2 for project name, H3 for areas:

```
## [Project Name]

### [Area]: [Change.]
### [Area]
- [First change.]
- [Second change.]

## [Project Name B]

### [Area]: [Change.]
### [Area]
- [First change.]
- [Second change.]
```

This flat structure is required for platform compatibility — some messaging platform does not render nested markdown lists reliably.

### Step 5 — output

Always send raw markdown in a codeblock so the user can copy and paste the markdown.

Single project:

```
Work report for [date]:

- Area 1: ...
- Area 2: ...
```

Monorepo with multiple projects touched:

```
Work report for [date]:

### Project Name A
- Area 1: ...

### Project Name B
- Area 1: ...
```

Range argument: head as "Work report for [start] – [end]:".
