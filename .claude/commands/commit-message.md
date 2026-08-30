---
description: Phase 7 of ASDD — generate commit message(s) for whichever repo(s) have changes, plus a summary and PR description.
---

# /commit-message

`fengshui-shifu-api` and `fengshui-shifu-ui` are separate git repos. Check both
independently — do not assume the change is confined to one.

## 1. Locate the repos, then read their state

**Do not assume the shell's working directory.** Implementation usually leaves
it inside one of the sub-repos, so a bare relative path like
`git -C fengshui-shifu-ui` will fail with "cannot change to
'fengshui-shifu-ui'". Resolve an absolute path to the top-level folder first —
the one containing both sub-repos and this `.claude/` directory — and build the
repo paths from it.

Then, for **each** repo, read (all read-only):

```
git -C <root>/<repo> branch --show-current
git -C <root>/<repo> status --short
git -C <root>/<repo> diff
git -C <root>/<repo> diff --staged
git -C <root>/<repo> log --oneline -10
```

Untracked files do not appear in `git diff` — check `status --short` for `??`
entries and account for them, or a new file will be missing from the message.

## 2. Write the message(s)

1. If a repo has no changes, skip it entirely — do not generate a message for
   an empty diff.
2. For each repo that **does** have changes, write a separate, self-contained
   commit message based only on that repo's diff. Do not blend the two repos'
   changes into one message.
3. If **both** repos have changes, call this out explicitly as a cross-cutting
   change and note the dependency between them (e.g. "API adds field X; UI
   commit consumes it") so the user commits them in the right order.
4. Finish with:
   - **Summary** — plain-language recap of what was built, across both repos
     if applicable.
   - **PR description(s)** — one per repo with changes, using that repo's
     commit message as the title and a short body. Include how the change was
     verified, and state plainly anything that was *not* verified.

## 3. Hand over — never run it

Do not run `git add`, `git commit`, or `git push`. Draft the message(s) and give
the user the exact commands to run. Per `CLAUDE.md`, every git write is theirs.
