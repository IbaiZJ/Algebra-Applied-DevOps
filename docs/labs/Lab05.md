# Lab 5: Branching strategies and merge conflicts

Use the results of **Lab 4** as the starting point for this lab.

In [Lab 4](Lab04.md), you learned the everyday git commands: staging, committing, stashing, tagging and synchronizing with a remote repository. You also resolved a simple merge conflict. In this lab, you will learn how teams **organize** their branches. A branching strategy is an agreement on which branches exist, how long they live, where work is merged and how releases are produced. Choosing one directly affects how often conflicts happen and how your CI/CD pipeline (coming in the following labs) will be triggered.

You will work with four strategies:

- **Trunk-based development**
- **Feature branching**
- **Git Flow**
- **Forking strategy**

and you will practice **merge conflicts** in each of them, including conflicts between branches with **related** and **unrelated** histories.

> **No application code is written in this lab.** You will create practice repositories and use *mock* commits: small edits to text files (`features.md`, `flags.txt`, `CHANGELOG.md`, `VERSION`) and empty commits created with `git commit --allow-empty -m "message"`. The goal is to build and read the **commit graph**, not to build an application.

> **Prerequisites:** the skills from Lab 4, git 2.28 or newer (`git init -b main` and `git switch --orphan` are used), a GitHub account, and the GitHub CLI (`gh`) is recommended but not required. For Task 4 you need a teammate (or a second GitHub account).

---

## Overview: the strategies at a glance

```text
Trunk-based            main  ──●──●──●──●──●──●──●──   (tiny branches live for hours, merged back quickly)
                              \/  \/  \/
Feature branching      main  ──●─────────●─────●──     (one branch per feature/issue, merged by pull request)
                              \       /  \   /
                               ●──●──●    ●─●
Git Flow         main    ──●────────────●────────●──   (release and hotfix branches, tags on main)
                 develop ──●──●──●──●──●──●──●──●──
                              \  /    \  / 
                               ●●      ●●  (feature/*, release/*, hotfix/*)
Forking                upstream main ──●──●──●──       (contributors work in their own copy)
                       your fork     ──●──●──●──●──    and send pull requests to upstream
```

| | Trunk-based | Feature branching | Git Flow | Forking |
|---|---|---|---|---|
| Long-lived branches | `main` only | `main` only | `main` + `develop` | `main` in each fork |
| Typical branch lifetime | hours to 1–2 days | days to a sprint | features: days; releases: weeks | until the PR is merged |
| Release approach | tag on `main` (optionally short `release/*` branch) | tag on `main` | `release/*` branch, tag on `main` | maintainers tag upstream |
| Merge conflict risk | low (small, frequent merges) | medium | higher (many parallel branches) | medium to high (forks drift) |
| Needs strong CI / feature flags | **yes** | recommended | optional | recommended |
| Fits | experienced teams with good automated tests, continuous delivery | most student and small-team projects | versioned software with scheduled releases | open source, external contributors |

---

## Task 0: Preparation

Create a folder for the practice repositories and, for each scenario below, a **separate public GitHub repository**:

| Repository | Scenario |
|------------|----------|
| `branching-trunk-demo` | Task 1 |
| `branching-feature-demo` | Task 2 |
| `branching-gitflow-demo` | Task 3 |
| `branching-fork-demo` | Task 4 (shared in a pair) |
| `branching-conflict-lab` | Task 5 |

A quick way to create each repository from the terminal:

```bash
mkdir branching-trunk-demo && cd branching-trunk-demo
git init -b main
echo "# Trunk-based demo" > README.md
git add README.md
git commit -m "Initial commit"
gh repo create branching-trunk-demo --public --source=. --remote=origin --push
```

(Without `gh`: create the empty repository on GitHub **without** a README, then `git remote add origin <url>` and `git push -u origin main`.)

Useful aliases to read the graph quickly (use them throughout the lab):

```bash
git config --global alias.graph "log --oneline --graph --decorate --all"
git config --global merge.conflictStyle zdiff3   # shows the common ancestor inside conflict markers (git 2.35+; use "diff3" on older versions)
```

---

## Task 1: Trunk-based development

### Idea

Everyone integrates into a single branch, the **trunk** (`main`), at least once a day. Branches, if used at all, are **short-lived** (hours, at most a day or two) and contain a small change. Unfinished functionality is hidden behind a **feature flag** instead of living in a long branch, so `main` is always releasable.

### 1.1 Direct commits to trunk and short-lived branches

In `branching-trunk-demo`, create `features.md` and `flags.txt`:

```bash
printf "# Features\n" > features.md
printf "new-checkout=off\ndark-mode=off\n" > flags.txt
git add . && git commit -m "Add feature list and flags (mock)"
git push
```

Small change, committed straight to trunk (acceptable for tiny, safe changes):

```bash
echo "- search" >> features.md
git commit -am "Add search to feature list"
git push
```

Now a **short-lived branch** with a pull request:

```bash
git switch -c tbd/new-checkout
echo "- new checkout (hidden behind flag)" >> features.md
git commit -am "Add new checkout skeleton, flag is OFF"
git push -u origin tbd/new-checkout
gh pr create --fill       # or open the PR in the browser
```

Merge it the same day (use **Squash and merge** on GitHub), delete the branch and update your local `main`:

```bash
git switch main && git pull --prune
git branch -d tbd/new-checkout
```

### 1.2 Feature flag instead of a long branch

Turn the functionality on in a separate, tiny commit:

```bash
sed -i.bak 's/new-checkout=off/new-checkout=on/' flags.txt && rm flags.txt.bak
git commit -am "Enable new-checkout flag"
```

Releasing the feature is now a one-line commit, and turning it off again (rollback) is too. Write one sentence in your report explaining why this is safer than keeping the checkout feature in a branch for two weeks.

### 1.3 Conflict: two people change the same trunk line

Simulate two developers by making two clones (`git clone <url> trunk-anna` and `git clone <url> trunk-marko`) or by using two branches.

1. **Anna** creates `tbd/anna-dark-mode`, sets `dark-mode=on` in `flags.txt`, commits and pushes.
2. **Marko** creates `tbd/marko-dark-mode`, sets `dark-mode=beta` in `flags.txt`, commits and pushes.
3. Anna's PR is squashed and merged into `main`.
4. Marko updates his branch with the new trunk, which gives a conflict:

```bash
git fetch origin
git rebase origin/main
# CONFLICT (content): Merge conflict in flags.txt
```

5. Resolve it. Open `flags.txt`, choose the correct final value (talk it over, as in a real team), remove the markers, then:

```bash
git add flags.txt
git rebase --continue
git push --force-with-lease     # required, because rebase rewrote Marko's own branch
```

> **Rebase quirk.** During a `rebase`, the meaning of *ours* and *theirs* is reversed: **ours** is the branch you are rebasing *onto* (`origin/main`, Anna's change), **theirs** is *your* commit being replayed (Marko's change). During a normal `merge` it is the opposite.

Why were conflicts rare and small here? Because of small, frequent integration. Note this in your report.

### 1.4 Releases from trunk and a fix via cherry-pick

Tag the release on `main`:

```bash
git tag -a v0.1.0 -m "First release"
git push origin v0.1.0
```

Trunk-based teams sometimes cut a short-lived **release branch** from the tag when a version must be maintained. Simulate a bug fix that needs to go into the release too:

```bash
git switch -c release/0.1 v0.1.0
git push -u origin release/0.1

git switch main
echo "- bugfix: search crash" >> features.md
git commit -am "Fix search crash"           # fix on trunk FIRST
git push

git switch release/0.1
git cherry-pick -x <hash-of-the-fix>        # copy the fix to the release branch; -x records the original commit in the message
git tag -a v0.1.1 -m "Patch release"
git push origin release/0.1 v0.1.1
```

The rule in trunk-based development is **fix forward**: the fix always lands on trunk first and is then copied to the release branch, never the other way around.

If the cherry-pick conflicts, it is resolved just like a merge: edit the file, `git add`, then `git cherry-pick --continue` (or `git cherry-pick --abort`).

### Expected result of Task 1

- `git graph` output showing a mostly linear `main` with short side branches, the `v0.1.0` and `v0.1.1` tags and the `release/0.1` branch.
- One conflict that appeared during a rebase, with a description of how it was resolved.
- A short description of the role of feature flags.

---

## Task 2: Feature branching

### Idea

Every feature, bug fix or issue gets **its own branch** created from `main`. Work is merged back through a **pull request** after a review. `main` always contains finished work. This is the strategy you started using in Lab 3 and Lab 4, and the most common choice for student projects.

### 2.1 Set up the repository

In `branching-feature-demo`, create `README.md`, `routes.md` and `CHANGELOG.md`, push, and protect `main` (**Settings → Rules → Rulesets** or **Branches**: require a pull request before merging).

```bash
printf "# Routes\n- GET /\n" > routes.md
printf "# Changelog\n" > CHANGELOG.md
git add . && git commit -m "Add routes and changelog (mock)" && git push
```

Create three issues in the repository (use the issue templates from Lab 3): `Register user`, `Login user`, `Item list`.

### 2.2 One branch per issue

Use the naming convention from Lab 3 (`<issue-number>-<description>`). Create the branches **from the issues** (Development → Create a branch) or with:

```bash
gh issue develop 1 --name 1-register-user --checkout
```

Add a mock commit to each branch:

| Branch | Mock change |
|--------|-------------|
| `1-register-user` | add `- POST /register` to `routes.md` and a line to `CHANGELOG.md` |
| `2-login-user` | add `- POST /login` to `routes.md` and a line to `CHANGELOG.md` |
| `3-item-list` | add `- GET /items` to `routes.md` |

Make two or three commits per branch, each referencing its issue (`#1`) and push all branches. Open pull requests with `Closes #<n>` in the description.

### 2.3 Conflict in a pull request, resolved in two different ways

`1-register-user` and `2-login-user` both appended a line at the **same place** in `routes.md` and `CHANGELOG.md`.

1. Review and merge the PR for `1-register-user` into `main`.
2. Look at the PR for `2-login-user`: GitHub now reports *"This branch has conflicts that must be resolved"*.

Resolve this conflict **using method A: merge `main` into your feature branch** (history is preserved, a merge commit is added):

```bash
git switch 2-login-user
git fetch origin
git merge origin/main
# fix both files so that BOTH routes and BOTH changelog lines remain, then:
git add routes.md CHANGELOG.md
git commit
git push
```

3. Merge the PR for `2-login-user`.
4. Now `3-item-list` conflicts too. Resolve it **using method B: rebase on `main`** (linear history, commits are rewritten):

```bash
git switch 3-item-list
git fetch origin
git rebase origin/main
# resolve, git add <file>, git rebase --continue
git push --force-with-lease
```

(A third option is GitHub's **Resolve conflicts** button in the PR, which edits the files in the browser and creates a merge commit. Try it for one extra conflict if you wish.)

### 2.4 Merge methods on GitHub

Merge the three PRs using a **different merge method each** (Merge commit, Squash and merge, Rebase and merge) and inspect `git graph` afterwards.

| Method | Result on `main` | Use when |
|--------|------------------|----------|
| Merge commit | All branch commits plus a merge commit; graph shows branches | You want the full history of the branch |
| Squash and merge | One new commit with all changes of the PR; the branch's commits are not in `main` | Branches have noisy "WIP" commits |
| Rebase and merge | Branch commits replayed on `main`, no merge commit | You want a linear history and clean commits |

Note which method you would pick for your team project and why.

### Expected result of Task 2

- Three merged PRs, each linked to an issue.
- Screenshot or text of a PR showing the conflict message.
- One conflict resolved by merging `main` into the branch and another by rebasing, with a comparison of the two resulting graphs.
- A short comparison of the three merge methods based on what you saw in `git graph`.

---

## Task 3: Git Flow

### Idea

Git Flow is a **model with two permanent branches and three types of temporary ones**:

| Branch | Created from | Merged into | Purpose |
|--------|--------------|-------------|---------|
| `main` | | | Only released code; every commit on it is a tagged release |
| `develop` | `main` | | Integration branch for the next release |
| `feature/*` | `develop` | `develop` | One new feature |
| `release/x.y.z` | `develop` | `main` **and** `develop` | Stabilization before a release (version bump, last fixes) |
| `hotfix/x.y.z` | `main` | `main` **and** `develop` | Urgent fix of a released version |

```text
main      ──●────────────────────●───────────────●──────  (v1.0.0 and v1.0.1 tags here)
             \                  / \             /
release       \       ●──●────●    \           /
               \     /                \         /
develop  ───────●──●──●──●─────────●────●───────●──────
                 \   /   \        /
feature           ●─●     ●──●───●
```

### 3.1 Set up

In `branching-gitflow-demo`:

```bash
git init -b main
printf "0.0.0\n" > VERSION
printf "# Changelog\n" > CHANGELOG.md
git add . && git commit -m "Initial commit"
git switch -c develop
gh repo create branching-gitflow-demo --public --source=. --remote=origin --push
git push -u origin develop
```

(`gh repo create ... --push` pushes the current branch. Also push `main`: `git push -u origin main`.)

### 3.2 Feature branches from `develop`

```bash
git switch develop
git switch -c feature/user-accounts
echo "- Add user accounts" >> CHANGELOG.md
git commit -am "Mock: user accounts"
git switch develop
git merge --no-ff feature/user-accounts -m "Merge feature/user-accounts"
git branch -d feature/user-accounts
```

`--no-ff` forces a merge commit even if a fast-forward is possible, so the feature remains visible as a branch in the history. Repeat the procedure for a second feature, `feature/item-catalog`.

On GitHub you would open the pull request **into `develop`**, not `main`.

### 3.3 Release branch

```bash
git switch develop
git switch -c release/1.0.0
printf "1.0.0\n" > VERSION
git commit -am "Bump version to 1.0.0"
echo "- fix typo in release notes" >> CHANGELOG.md
git commit -am "Release fix: typo"           # only stabilization work happens here
```

Meanwhile, new work for the **next** release continues on `develop` (do not block it):

```bash
git switch develop
git switch -c feature/notifications
echo "- Add notifications" >> CHANGELOG.md   # same place in the same file as the release fix → will conflict later
git commit -am "Mock: notifications"
git switch develop && git merge --no-ff feature/notifications
```

Finish the release:

```bash
git switch main
git merge --no-ff release/1.0.0 -m "Release 1.0.0"
git tag -a v1.0.0 -m "Release 1.0.0"

git switch develop
git merge --no-ff release/1.0.0        # bring the release fixes back to develop
```

The last merge **conflicts in `CHANGELOG.md`**, because `develop` got a new line at the same place where the release branch got its fix. Resolve it by keeping both lines, commit, delete the release branch (`git branch -d release/1.0.0`) and push everything with the tags:

```bash
git push origin main develop --follow-tags
```

### 3.4 Hotfix from `main`

A critical bug is reported in `v1.0.0`:

```bash
git switch main
git switch -c hotfix/1.0.1
printf "1.0.1\n" > VERSION
echo "- Hotfix: login crash" >> CHANGELOG.md
git commit -am "Hotfix: login crash, bump to 1.0.1"

git switch main
git merge --no-ff hotfix/1.0.1 -m "Hotfix 1.0.1"
git tag -a v1.0.1 -m "Hotfix 1.0.1"

git switch develop
git merge --no-ff hotfix/1.0.1         # conflicts in VERSION / CHANGELOG.md are expected
```

Resolve the conflict. **Careful here:** `VERSION` on `develop` is `1.0.0`, `1.0.1` on the hotfix, and `develop` is working towards the next version. Decide which version number `develop` should carry and justify it in your report. Then:

```bash
git branch -d hotfix/1.0.1
git push origin main develop --follow-tags
```

> The rule that causes most conflicts in Git Flow: **every fix that lands on `main` (release or hotfix) must also be merged into `develop`.** Forgetting it makes branches diverge, so the next release brings the old bug back.

### 3.5 Optional: the `git flow` helper

There are command line extensions (`git flow init`, `git flow feature start <name>`, `git flow release finish <version>`) that run the sequences above for you. Compare the result with your manual repository if you try them. They are not required, because the point of the task is to understand what the commands do.

### Expected result of Task 3

- `git graph` output showing `main`, `develop`, at least two features, one release and one hotfix, with tags `v1.0.0` and `v1.0.1` on `main`.
- Two conflicts resolved: one when merging the release back into `develop`, one when merging the hotfix.
- A short answer: which branches were merged into both `main` and `develop`, and why?

---

## Task 4: Forking strategy

### Idea

Contributors do **not** have write access to the main repository. Each contributor creates a personal **fork** (a server-side copy of the repository on GitHub), works there and proposes changes through a **pull request from the fork to the original repository**. The original is called **upstream**, the fork is **origin** in the contributor's clone. This is the standard model for open source projects, and it is also used inside companies that want strict control over the main repository.

```text
  upstream (maintainer)  github.com/anna/branching-fork-demo
        ^   |
   PR   |   | fork (copy on GitHub), fetch upstream
        |   v
  origin (contributor)   github.com/marko/branching-fork-demo   ──clone──>  local repository
```

This task is done in **pairs**: one person is the **maintainer** (Anna), the other the **contributor** (Marko). Then swap roles and repeat the key steps.

### 4.1 Maintainer: prepare the upstream repository

Anna creates `branching-fork-demo` with a `README.md`, `CONTRIBUTING.md` (one paragraph on how to contribute) and `contributors.md` (a list of names). The repository stays on `main`, and Anna **does not** add Marko as a collaborator.

### 4.2 Contributor: fork, clone, add upstream

```bash
# fork in the browser (Fork button) or:
gh repo fork anna/branching-fork-demo --clone=true

cd branching-fork-demo
git remote -v
# origin    https://github.com/marko/branching-fork-demo.git   (your fork)
git remote add upstream https://github.com/anna/branching-fork-demo.git   # needed only if `gh` did not add it
git remote -v
```

You now have two remotes: `origin` (your fork, you can push) and `upstream` (Anna's repository, read-only for you).

### 4.3 Contributor: work on a branch and open a pull request

Never work on `main` of the fork. Use a topic branch:

```bash
git switch -c add-marko-to-contributors
echo "- Marko" >> contributors.md
git commit -am "Add Marko to contributors"
git push -u origin add-marko-to-contributors
```

Open a pull request from `marko:add-marko-to-contributors` to `anna:main`. Keep **Allow edits by maintainers** enabled.

### 4.4 Conflict: the upstream moved on

While Marko's PR is open, Anna merges another change at the same place in `contributors.md` directly into upstream `main` (for example, adds herself on the same line position). Marko's PR now shows a conflict.

Marko resolves it by bringing the new upstream commits into his branch, **rebase** variant:

```bash
git fetch upstream
git rebase upstream/main
# resolve conflicts in contributors.md (keep both names), then
git add contributors.md
git rebase --continue
git push --force-with-lease origin add-marko-to-contributors
```

The pull request updates automatically. Anna reviews it (leaves one review comment, and Marko adds a follow-up commit to answer it) and merges it.

(You may also bring the changes in with `git merge upstream/main`, which does not require a force push.)

### 4.5 Contributor: sync the fork

After the merge, the `main` of Marko's fork is **behind** upstream. Synchronize it (use either way):

- On GitHub, on the fork's page, click **Sync fork → Update branch**, then `git pull` locally, or
- in the terminal:

```bash
git switch main
git fetch upstream
git merge --ff-only upstream/main      # fails if your main has diverged, which is a signal you committed to main by mistake
git push origin main
git branch -d add-marko-to-contributors
git push origin --delete add-marko-to-contributors
```

### 4.6 Swap roles

Marko becomes the maintainer of his own `branching-fork-demo` repository and Anna the contributor. This time, Anna must also try **a conflict while the fork's own `main` has diverged**: she accidentally commits directly to her fork's `main` before syncing. Fix it **without losing the commit**:

```bash
git fetch upstream
git switch -c rescue/my-change            # keep the stray commit on a proper branch
git switch main
git reset --hard upstream/main            # make fork's main equal to upstream again
git push --force-with-lease origin main   # fork's main is YOUR copy, so this is acceptable here
```

Then open the pull request from `rescue/my-change`.

### Expected result of Task 4

- A pull request from a fork to the upstream repository, merged (link to it in your report).
- Output of `git remote -v` showing `origin` and `upstream`.
- A resolved conflict caused by upstream changes, with the commands used.
- A synchronized fork (fork's `main` equals upstream `main`).

---

## Task 5: Merge conflicts in depth

In the previous tasks, the conflicts were simple. In this task, you will systematically explore the **kinds of conflicts**, **ways to resolve them**, and what happens when the histories are **related or unrelated**. Work in `branching-conflict-lab`.

```bash
mkdir branching-conflict-lab && cd branching-conflict-lab
git init -b main
printf "line 1\nline 2\nline 3\n" > notes.txt
git add . && git commit -m "Add notes.txt"
gh repo create branching-conflict-lab --public --source=. --remote=origin --push
```

### 5.1 Anatomy of a conflict

Create two branches that change the **same line** differently:

```bash
git switch -c branch-a && sed -i.bak 's/line 2/line 2 (A)/' notes.txt && rm notes.txt.bak && git commit -am "A changes line 2"
git switch main
git switch -c branch-b && sed -i.bak 's/line 2/line 2 (B)/' notes.txt && rm notes.txt.bak && git commit -am "B changes line 2"
git merge branch-a        # fast-forward from main? No: we are on branch-b, so this is a real merge
```

Result:

```text
Auto-merging notes.txt
CONFLICT (content): Merge conflict in notes.txt
Automatic merge failed; fix conflicts and then commit the result.
```

Read the file (with `zdiff3` set, you also see the common ancestor):

```text
line 1
<<<<<<< HEAD
line 2 (B)
||||||| 3f2c1ab
line 2
=======
line 2 (A)
>>>>>>> branch-a
line 3
```

| Marker | Meaning |
|--------|---------|
| `<<<<<<< HEAD` ... `|||||||` | **Ours**: your current branch (`branch-b`) |
| `|||||||` ... `=======` | The common ancestor (only shown with `diff3` / `zdiff3`) |
| `=======` ... `>>>>>>> branch-a` | **Theirs**: the branch being merged in (`branch-a`) |

Useful commands while a conflict is in progress:

```bash
git status                                  # "Unmerged paths: both modified"
git diff --name-only --diff-filter=U        # list only conflicted files
git log --merge --oneline                   # commits from both sides that touch the conflicted files
git show :1:notes.txt                       # stage 1: common ancestor
git show :2:notes.txt                       # stage 2: ours
git show :3:notes.txt                       # stage 3: theirs
```

### 5.2 Five ways to resolve a content conflict

Reproduce the conflict from 5.1 each time (`git merge --abort` or `git reset --hard` plus a repeat of the merge) and resolve it in a **different way**. After each, record the final content of `notes.txt` and run `git graph`.

**A) Manual edit.** Open the file, produce the correct combination (e.g. `line 2 (A and B)`), delete the markers, then `git add notes.txt` and `git commit`.

**B) Take one whole side for the file.**

```bash
git checkout --ours   -- notes.txt     # keep the version from the current branch
# or
git checkout --theirs -- notes.txt     # keep the version from the incoming branch
git add notes.txt && git commit
```

(`git restore --ours notes.txt` / `--theirs` is the newer equivalent.)

**C) Resolve at merge time with a strategy option.**

```bash
git merge -X theirs branch-a     # conflicting hunks: prefer the incoming side. Non-conflicting changes from BOTH sides are still merged
git merge -X ours   branch-a     # conflicting hunks: prefer our side
```

**D) Use a merge tool.**

```bash
git config --global merge.tool vscode
git config --global mergetool.vscode.cmd 'code --wait $MERGED'
git mergetool                    # opens each conflicted file
```

(VS Code also offers *Accept Current / Accept Incoming / Accept Both* buttons above each conflict. Use any tool your team prefers.)

**E) Abort and rethink.**

```bash
git merge --abort                # restore the state before the merge
```

Typical reason: you realize the branches need to be coordinated with a teammate first.

Don't confuse the **strategy option** `-X ours` with the **strategy** `-s ours`:

| Command | Effect |
|---------|--------|
| `git merge -X ours <branch>` | Real merge. Only *conflicting hunks* take our side. Other changes of `<branch>` are included. |
| `git merge -s ours <branch>` | Records a merge commit but **discards every change** of `<branch>`. |

Write two or three sentences on when each of the five ways is appropriate and when it is dangerous (hint: B and C can silently throw away a teammate's work).

### 5.3 Other kinds of conflicts

Content conflicts are only one type. Create and resolve each of the following (new branches from `main`, repeat the merge):

| Type | How to create it | How to resolve |
|------|------------------|----------------|
| **modify/delete** | On branch X delete `notes.txt` (`git rm`), on branch Y modify it, merge | Decide: keep the file (`git add notes.txt`) or delete it (`git rm notes.txt`), then commit |
| **add/add** | On two branches, create a **new** file `todo.txt` with different content, merge | Edit to combine, `git add`, commit |
| **rename/modify** | On X rename `notes.txt` to `docs.txt` (`git mv`), on Y edit `notes.txt` | Git often follows the rename and merges automatically. Check `git status`. If not, resolve manually |

### 5.4 Conflicts during rebase and cherry-pick

The same conflicts can occur whenever git replays commits:

```bash
# rebase
git switch branch-a
git rebase main            # conflict → fix → git add → git rebase --continue
                           # alternatives: git rebase --abort | git rebase --skip

# cherry-pick
git switch main
git cherry-pick <hash-from-branch-b>   # conflict → fix → git add → git cherry-pick --continue
                                       # alternative: git cherry-pick --abort
```

Remember that during a rebase **ours/theirs are swapped** (see Task 1).

### 5.5 Related and unrelated histories

Two branches have **related histories** if they share at least one common ancestor commit. Git uses the nearest common ancestor (the **merge base**) to calculate a three-way merge.

```bash
git merge-base main branch-a        # prints the hash of the common ancestor
git graph                           # you can SEE where the lines split
```

Two branches have **unrelated histories** if there is no common ancestor, which means they started as two separate repositories (or an orphan branch). Git refuses to merge them by default.

**Scenario 1: an orphan branch (quick demonstration).**

```bash
git switch --orphan standalone         # a new branch with no parent commit
printf "I come from another history\n" > other.txt
printf "# A different README\n" > README.md
git add . && git commit -m "Root commit of a separate history"
git merge-base main standalone         # prints nothing: no common ancestor
git switch main
git merge standalone
# fatal: refusing to merge unrelated histories
```

**Scenario 2: the classic real-life mistake.**

1. On GitHub, create a new repository **with** a README and a `.gitignore` (so it has a root commit).
2. Locally, `git init -b main`, create a *different* `README.md`, commit, `git remote add origin <url>`.
3. `git pull origin main` → `fatal: refusing to merge unrelated histories`. Both repositories have their own root commit.

**Ways to resolve unrelated histories** (try at least three on scenario 2, recreating the setup if needed):

| Way | Commands | Result | Use when |
|-----|----------|--------|----------|
| **1. Merge anyway** | `git pull origin main --allow-unrelated-histories` (or `git merge <branch> --allow-unrelated-histories`) | A merge commit with two roots. Files that exist on both sides (e.g. `README.md`) produce **add/add conflicts**, which you resolve as in 5.2 | You really want to keep both histories |
| **2. Rebase local work on top of the remote** | `git fetch origin` then `git rebase --onto origin/main --root` | Linear history: remote's commits first, then your commits replayed. `README.md` conflicts appear during the rebase | Your local history is small and not yet shared |
| **3. Throw away the local history and use the remote** | `git fetch origin` then `git reset --hard origin/main` (copy your local files elsewhere first!) | Local becomes identical to the remote. Local commits are lost | Local work is only a scaffold |
| **4. Keep the local history, replace the remote** | `git push --force-with-lease origin main` | The remote now has your history, the GitHub-created root commit is gone | You are the only user of the new, empty remote repository |
| **5. Copy selected commits** | `git fetch origin`, `git switch -c from-remote origin/main`, `git cherry-pick <hash>...` | Chosen commits are replayed onto the remote's history | Only some commits are worth keeping |

> Way 4 and `--allow-unrelated-histories` are **not** first aids for every refusal. Ask first: *why* are there two histories? In a team project, unrelated histories usually mean someone created the repository twice, and the right fix is to agree which one is the real one.

Prevent the problem: either create the repository on GitHub **empty** and push your local history to it, or **clone** the repository on GitHub instead of running `git init`.

### 5.6 Preventing conflicts

Finish the task with a short list (in your report) of what reduces conflicts in a team, e.g.:

- integrate often: pull or rebase before you start and before you push;
- small, focused branches and commits, as in trunk-based development;
- agree who owns which files (split work by feature/module, as in your user stories);
- avoid reformatting whole files in unrelated commits;
- use `.gitattributes`/`.gitignore` for generated files and lock files;
- talk to the teammate whose change you are overriding.

### Expected result of Task 5

- `git graph` outputs of the merge in 5.2 for at least three of the five resolutions.
- Resolved modify/delete and add/add conflicts.
- One conflict resolved during a rebase and one during a cherry-pick.
- A demonstration of `refusing to merge unrelated histories` and its resolution using at least three of the five ways in 5.5, with the `git merge-base` output before and after.

---

## Task 6: Choosing a strategy for your project

Based on what you experienced, decide which strategy your team will use in the **project repository** from Lab 1.

Create `docs/branching-strategy.md` in your project repository and include:

1. **Chosen strategy** (trunk-based, feature branching, Git Flow, or a described hybrid) and a justification that uses at least three criteria: team size, release frequency, experience with automated tests, expected CI/CD setup, number of parallel features.
2. **Branch types and naming rules** (e.g. `main`, `<issue-number>-<short-description>`, `hotfix/<version>`), and which branches are permanent.
3. **Merge rules:** who reviews, which merge method (merge, squash, rebase), whether `main` is protected, and what must pass before merging.
4. **Release rules:** how versions are numbered, where tags are created, and who creates them.
5. **Conflict policy:** how and by whom conflicts are resolved (e.g. "the author of the later PR rebases onto `main` and resolves"), and what to do in case of doubt.
6. **Diagram:** a small commit graph of your strategy (ASCII or a `.drawio` file in `docs/`, as in Lab 2).

Apply what you decided: set the branch protection rules on `main` of your project repository accordingly.
---

## Command cheat sheet

| Goal | Command |
|------|---------|
| Show graph | `git log --oneline --graph --decorate --all` |
| Find the common ancestor | `git merge-base <a> <b>` |
| Merge keeping a merge commit even if fast-forward is possible | `git merge --no-ff <branch>` |
| Abort an operation | `git merge --abort`, `git rebase --abort`, `git cherry-pick --abort` |
| List conflicted files | `git diff --name-only --diff-filter=U` |
| Take one side of a conflicted file | `git checkout --ours -- <file>` / `git checkout --theirs -- <file>` |
| Auto-prefer a side for conflicting hunks | `git merge -X ours <branch>` / `git merge -X theirs <branch>` |
| Continue after resolving | `git add <file>` then `git commit` / `git rebase --continue` / `git cherry-pick --continue` |
| Copy a commit to another branch | `git cherry-pick -x <hash>` |
| Merge unrelated histories | `git merge <branch> --allow-unrelated-histories` |
| Replay all local commits on the remote | `git rebase --onto origin/main --root` |
| Add the original repo of a fork | `git remote add upstream <url>` |
| Update a fork from upstream | `git fetch upstream` then `git merge --ff-only upstream/main` |
| Safer force push | `git push --force-with-lease` |

---

## Useful resources

- [Pro Git – Branching and Merging](https://git-scm.com/book/en/v2/Git-Branching-Basic-Branching-and-Merging)
- [Pro Git – Advanced Merging](https://git-scm.com/book/en/v2/Git-Tools-Advanced-Merging)
- [Trunk Based Development](https://trunkbaseddevelopment.com)
- [A successful Git branching model (Git Flow)](https://nvie.com/posts/a-successful-git-branching-model/)
- [GitHub Docs – Working with forks](https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/working-with-forks)
- [GitHub Docs – Resolving a merge conflict](https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/addressing-merge-conflicts)