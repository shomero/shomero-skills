---
name: open-pr
description: Open a GitHub pull request in the house format — branch off main, run the review-pr gate first, a [KEY-123] title, a body that leads with the ticket title and link then What's this / Changes / Testing / Notes, and the review posted as a PR comment. Use whenever asked to open or create a PR, "PR this", or ship a branch for review.
---

# open-pr

`main` is protected: never push to it. Branch, push, PR, review, merge.

## Steps

1. **Find the ticket**: from the branch name (`feat/abc-26-search` → `ABC-26`), else the tracker, else ask. Get its exact title.
2. **Off `main`**: `git checkout -b <type>/<key>-<slug>` if needed.
3. **Commit** pending work only if the user wants it, with the attribution trailer the session gives.
4. **Review first**: run `review-pr` on `git diff main...HEAD`.
   - No BLOCKER/HIGH → continue.
   - Any BLOCKER/HIGH → stop, report, offer to fix. If the user ships anyway, the review comment starts with `⚠️ Review gate overridden by the author: N BLOCKER, M HIGH outstanding.`
5. **Push**: `git push -u origin <branch>`.
6. **Create** with the body in a file so headings survive: `gh pr create --base main --title "[KEY-123] <imperative summary>" --body-file <path>`.
7. **Post the review** as a PR comment: `gh pr comment <n> --body-file <path>`.
8. **Report the URL.** Never merge; a person reviews first.

## Body

```markdown
# <Ticket title>

<ticket url>

# What's this?

<2–4 plain sentences: what and why, enough to review without the diff.>

# Changes

- <one bullet per meaningful change, grouped by module>

# Testing

- <commands run with result, platforms compiled, manual checks>

# Notes

<follow-ups, known gaps, screenshots or before/after link. Drop if empty.>
```

End the body with the attribution line the session gives. Projects with no ticket tracker drop the key and the link; the title is the task name.

Your CLAUDE.md names the tracker, the key prefix and its base URL.
