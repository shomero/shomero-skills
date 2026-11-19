---
name: code-comments
description: The house rule on code comments — a comment says why (a constraint, a reason, a non-obvious choice) and KDoc on a public declaration says what it is for; nothing narrates what the next line does, no history, no commented-out code, no TODO without a ticket. Use whenever writing or reviewing code, and whenever tempted to explain code with a comment.
---

# Code comments

Code says what it does. A comment says what the code can't: why.

## Write

- **Why, next to the code it explains**: a constraint, a reason, a choice a reader would otherwise undo.
  `// Five seconds is Android's ANR deadline, so a rotation doesn't restart the load.`
- **KDoc on public declarations**: what the thing is for, for its callers, in a sentence or two. Not how it's built.
  `/** The notes the person can see, newest first. A screen's load() collects it. */`
- **A smell we'll fix later**, with its ticket: `// Smell: <what>. Fixed in <ticket url>`
- **A workaround we can't avoid**, with what forces it: `// Workaround: <constraint>, <why there's no alternative today>.`

## Never

- **Narrating what the next line does** (`// increment the counter`, `// call the API`). Rename or split the code instead; names, types and small functions carry meaning, and a test name carries the case.
- **History**: "was X, changed to Y", "added for the redesign". That's the commit message.
- **Commented-out code**: git keeps it.
- **`TODO` without a ticket**: it never gets done and nobody owns it.
- **A comment that repeats the name or the type** (`/** The note repository. */` on `NoteRepository`).

## When you touch a file

- Fix the comments on the lines you change: a comment that no longer holds is worse than none.
- A smell whose ticket is closed goes, with the code it excused.
- Leave other comments alone unless they're wrong. Licence headers stay.

Long explanations (alternatives weighed, how a bug was found) go in the commit message, the PR body or the ticket, where they're dated.
