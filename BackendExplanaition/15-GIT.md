# LEVEL 15 — Git Deep Dive

## The Mental Model First

Git is fundamentally a graph of **commits**, where each commit is a snapshot of your entire
project at a point in time, pointing back to its parent(s). Branches, HEAD, merges — everything
else is just different ways of moving around or reshaping that graph.

```
   C1 ──▶ C2 ──▶ C3 ──▶ C4        each arrow points to the PARENT commit
                        ▲
                      main (branch = just a movable pointer to a commit)
```

---

## 1. Commit

A commit is an **immutable snapshot** — not a diff, though Git shows it to you as one. Internally,
it stores a full tree of the project state, a pointer to its parent commit(s), the author, a
message, and a hash (SHA) computed from all of that content.

```bash
git commit -m "Add user authentication"
git log --oneline    # a4f3c21 Add user authentication
```

```
Commit a4f3c21
  ├── tree: (snapshot of every file at this point)
  ├── parent: 9b2e114  (the commit before this one)
  ├── author: Alice
  └── message: "Add user authentication"
```

Because the hash is derived from the content, **changing anything about a commit — even just its
message — produces a brand-new hash.** This is why "editing history" always technically means
creating new commits, not literally modifying old ones.

## 2. Branch

A branch is nothing more than a **movable label pointing at a commit**. Creating a branch is
nearly instant precisely because it doesn't copy any files — it's just a new pointer.

```bash
git branch feature-x        # create a new branch (pointer) at the current commit
git checkout feature-x        # move HEAD to point at that branch
git switch -c feature-x         # modern equivalent of the two commands above combined
```

```
        C1 ──▶ C2 ──▶ C3
                       ▲
                     main, feature-x     (both branches point at the SAME commit right now)

# after committing on feature-x:
        C1 ──▶ C2 ──▶ C3 ──▶ C4
                       ▲       ▲
                     main   feature-x    (they've now diverged)
```

## 3. HEAD

HEAD is a pointer to **whatever you currently have checked out** — normally it points to a
branch (which itself points to a commit), so HEAD moves automatically as that branch advances.

```
HEAD ──▶ feature-x ──▶ C4

# "detached HEAD" — pointing directly at a commit instead of a branch (e.g. after
# `git checkout <commit-hash>`) — new commits here aren't attached to any branch and
# can be lost once you check out something else, unless you create a branch to save them
HEAD ──▶ C2   (feature-x branch is elsewhere, not tracking this)
```

```bash
git checkout a4f3c21          # HEAD now points directly at a commit ("detached")
git switch -c rescue-branch     # save your work by attaching a branch to it before leaving
```

## 4. Remote

A remote is just a **named reference to another copy of the repository**, usually on a server
(GitHub/GitLab). Your local repo tracks the remote's branches via refs like `origin/main`.

```bash
git remote -v                    # list configured remotes
git remote add origin https://github.com/you/repo.git
git fetch origin                   # download new commits, DON'T merge them into your branches
git pull origin main                 # fetch + merge (or rebase) in one step
git push origin main                  # upload your local commits to the remote
```

```
Local repo                              Remote (GitHub)
main ──▶ C1 ──▶ C2 ──▶ C3                origin/main ──▶ C1 ──▶ C2
                                          (before you push C3)

after `git push`:
Local repo                              Remote (GitHub)
main ──▶ C1 ──▶ C2 ──▶ C3                origin/main ──▶ C1 ──▶ C2 ──▶ C3
```

## 5. Merge

Combines two branches by creating a **new commit with two parents**, preserving the full history
of both branches exactly as it happened.

```bash
git checkout main
git merge feature-x
```

```
Before:
        C1 ──▶ C2 ──▶ C3 ──▶ C4  (main)
                 \
                  ▶ C5 ──▶ C6  (feature-x)

After `git merge feature-x` (while on main):
        C1 ──▶ C2 ──▶ C3 ──▶ C4 ──────▶ M   (M = merge commit, TWO parents: C4 and C6)
                 \                    ╱
                  ▶ C5 ──▶ C6 ───────╯
```

## 6. Rebase

Replays your branch's commits **on top of** another branch, rewriting them as brand-new commits
with new hashes — producing a linear history instead of a merge commit.

```bash
git checkout feature-x
git rebase main
```

```
Before:
        C1 ──▶ C2 ──▶ C3 ──▶ C4   (main)
                 \
                  ▶ C5 ──▶ C6      (feature-x)

After `git rebase main` (while on feature-x):
        C1 ──▶ C2 ──▶ C3 ──▶ C4 ──▶ C5' ──▶ C6'   (feature-x, now linear)

Note: C5' and C6' are NEW commits (new hashes) — the original C5/C6 are abandoned.
This is why you should NEVER rebase commits that have already been pushed and
that other people might have based their own work on — you'd be rewriting history
out from under them.
```

## 7. Cherry-pick

Copies **one specific commit** from anywhere in the history onto your current branch, without
bringing along everything else from its original branch.

```bash
git checkout main
git cherry-pick a4f3c21    # apply just this one commit's changes onto main
```

```
        C1 ──▶ C2 ──▶ C3           (main)

  feature-x:  C1 ──▶ Cx ──▶ Cy ──▶ Cz

After `git cherry-pick Cy` onto main:
        C1 ──▶ C2 ──▶ C3 ──▶ Cy'    (main — just Cy's changes, as a new commit, Cx and Cz never came along)
```

Use case: you have one urgent bug fix buried in an otherwise-unfinished feature branch and need
it on `main` immediately without merging the whole branch.

## 8. Stash

Temporarily shelves uncommitted changes so you can switch branches (or pull) with a clean working
directory, then reapply them later.

```bash
git stash              # save uncommitted changes, revert working directory to last commit
git stash list           # see all stashed changesets
git stash pop             # reapply the most recent stash AND remove it from the stash list
git stash apply            # reapply without removing from the list (in case you need it again)
```

```
Working directory: [dirty: uncommitted changes to file.py]
          │  git stash
          ▼
Working directory: [clean, matches last commit]     Stash: {file.py changes saved here}
          │  git stash pop
          ▼
Working directory: [dirty again: file.py changes restored]     Stash: {empty}
```

## 9. Conflict

Happens when Git can't automatically reconcile changes to the **same lines** of the same file
across two branches being merged/rebased — it needs a human to decide the correct result.

```bash
git merge feature-x
# CONFLICT (content): Merge conflict in app.py
```

```python
# app.py — Git marks the conflicting section
<<<<<<< HEAD
def greet():
    return "Hello from main"
=======
def greet():
    return "Hello from feature-x"
>>>>>>> feature-x
```

```bash
# After manually editing the file to resolve it:
git add app.py
git commit          # (for a merge) or `git rebase --continue` (for a rebase)
```

---

## ⚠️ `git reset` vs `git revert`

Both "undo" something, but in fundamentally different ways — this is one of the most important
distinctions in Git.

### `git reset` — rewrites history, moves the branch pointer BACKWARD

```
Before:
        C1 ──▶ C2 ──▶ C3 ──▶ C4
                              ▲
                            main

git reset --hard C2   (moves main's pointer back to C2, C3 and C4 are ABANDONED)

After:
        C1 ──▶ C2      C3 ──▶ C4   (orphaned — will be garbage collected eventually)
                ▲
              main
```

```bash
git reset --soft  C2     # move the branch pointer back, but KEEP changes staged
git reset --mixed C2     # move the branch pointer back, KEEP changes but unstaged (default)
git reset --hard  C2     # move the branch pointer back, DISCARD all changes entirely — destructive!
```

**Danger:** `git reset --hard` on commits **already pushed and shared** with others rewrites
history they've based their own work on — their local `main` will diverge and conflict badly with
yours on next push/pull. Safe on local, unpushed commits; risky on shared branches.

### `git revert` — adds a NEW commit that undoes a previous one, history stays intact

```
Before:
        C1 ──▶ C2 ──▶ C3 ──▶ C4
                              ▲
                            main

git revert C3   (creates a NEW commit that applies the INVERSE of C3's changes)

After:
        C1 ──▶ C2 ──▶ C3 ──▶ C4 ──▶ C5   (C5 = "Revert C3", undoes C3's changes)
                                    ▲
                                  main

Nothing is deleted — the fact that C3 happened AND was later undone is preserved in history.
```

```bash
git revert C3    # safe to use even on shared/pushed history — doesn't rewrite anything
```

### The Decision

```
Do other people already have these commits (pushed to a shared branch)?
   │
   ├── YES ──▶ git revert   (adds a new "undo" commit, safe for shared history)
   │
   └── NO, it's only local/unpushed ──▶ git reset is fine
                                        (rewriting your own private history is safe)
```

| | `git reset` | `git revert` |
|---|---|---|
| History | Rewrites it (commits disappear) | Preserves it (adds a new commit) |
| Safe on shared/pushed branches? | ❌ No | ✅ Yes |
| Use case | Cleaning up local, not-yet-pushed mistakes | Undoing something already public |

---

## ⚠️ `merge` vs `rebase`

Both integrate changes from one branch into another — the difference is entirely about the
**shape of the resulting history**.

```
Starting point:
        C1 ──▶ C2 ──▶ C3 ──▶ C4   (main)
                 \
                  ▶ C5 ──▶ C6      (feature-x)
```

### Merge — preserves exact history, non-linear

```bash
git checkout main
git merge feature-x
```

```
        C1 ──▶ C2 ──▶ C3 ──▶ C4 ──────▶ M
                 \                    ╱
                  ▶ C5 ──▶ C6 ───────╯
```

- ✅ Non-destructive — original commits (C5, C6) are untouched, exact history preserved.
- ✅ Safe to do on shared branches, no rewriting.
- ❌ History becomes a graph with merge commits, can get visually noisy with many branches.

### Rebase — linear history, rewrites commits

```bash
git checkout feature-x
git rebase main
git checkout main
git merge feature-x    # now a "fast-forward" merge — just moves the pointer, no merge commit
```

```
        C1 ──▶ C2 ──▶ C3 ──▶ C4 ──▶ C5' ──▶ C6'
                                              ▲
                                     main, feature-x (after fast-forward merge)
```

- ✅ Clean, linear, easy-to-read history — looks like the feature was built sequentially on top
  of the latest main, even though it wasn't originally.
- ❌ Rewrites commit hashes — **never rebase commits already pushed/shared** unless you're certain
  no one else has based work on them (and even then, coordinate carefully).
- ❌ Conflicts can need resolving multiple times (once per replayed commit) instead of once.

### The Decision

```
Is this a local feature branch, not yet pushed / not shared with others?
   │
   ├── YES ──▶ rebase onto main freely — get a clean linear history before you share it
   │
   └── NO, already pushed / others are working from it ──▶ merge instead
                                                            (or coordinate very carefully
                                                             before a shared rebase)
```

| | Merge | Rebase |
|---|---|---|
| History shape | Non-linear, preserves branch structure | Linear, rewritten |
| Rewrites commits? | No | Yes |
| Safe on shared branches? | ✅ Yes | ❌ Risky |
| Common team convention | "Merge feature branches into main via PR" | "Rebase your feature branch onto main before opening a PR, to keep it up to date" |

**A common real workflow combines both:** rebase your own local, unpushed feature branch onto the
latest `main` to stay current and keep history clean, then merge that feature branch into `main`
via a pull request (often as a single squashed commit) once it's shared and reviewed.

---

## Summary Table

| Command | What it actually does |
|---|---|
| `git commit` | Creates a new immutable snapshot, pointing at the previous commit as parent |
| `git branch` | Creates a new movable pointer to a commit |
| `HEAD` | Points to whatever you currently have checked out (usually a branch) |
| `git remote` | A named reference to another copy of the repo (e.g. `origin`) |
| `git merge` | Combines branches via a new commit with two parents; preserves history |
| `git rebase` | Replays commits onto a new base; rewrites hashes, linear history |
| `git cherry-pick` | Copies one specific commit onto the current branch |
| `git reset` | Moves the branch pointer backward; can discard commits/changes — rewrites history |
| `git revert` | Adds a new commit that undoes a previous one — safe, preserves history |
| `git stash` | Temporarily shelves uncommitted changes |
| Conflict | Git needs a human decision when the same lines changed differently on two branches |
