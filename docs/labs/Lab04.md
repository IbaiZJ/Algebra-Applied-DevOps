# Lab 4: Git workflow basics

Use the results of **Lab 3** as the starting point for this lab.

In [Lab 3](Lab03.md), you created a GitHub Project, issues with a hierarchy, dependencies, and branches linked to issues. You already used a few git commands (`git fetch`, `git switch`, `git push`) without explaining them. In this lab, you will learn how git actually works day to day: how changes move from your working directory to a commit, how to put work aside, how to mark releases, and how your **local repository** stays in sync with the **remote repository** on GitHub.

The results of this lab will be used in the following sessions when creating the CI/CD workflow. Pipelines are triggered by pushes, pull requests and tags, so you need to be confident in how those events are produced.

> **Prerequisites:** git installed (`git --version`), a GitHub account, the repository from Lab 1 with a few issues from Lab 3, and a terminal (Git Bash on Windows, Terminal on macOS/Linux). The commands below use `git switch` and `git restore`, available from git 2.23.

All examples use the made-up **LendIt** project from Lab 3. Replace names and issue numbers with your own.

---

## Concepts: the four places where your changes live

Before running commands, understand where a change can be:

```text
 Working directory  --git add-->  Staging area  --git commit-->  Local repository  --git push-->  Remote repository
   (your files)                    (index)                         (.git folder)                    (GitHub: origin)
        ^                                                                |                               |
        |------------------------------ git switch / restore -----------|<--------- git fetch / pull ----|
```

| Place | Contains | Visible to teammates? |
|-------|----------|-----------------------|
| Working directory | The files you are editing right now | No |
| Staging area (index) | Changes you selected for the next commit | No |
| Local repository | Your commits, branches and tags | No |
| Remote repository (`origin`) | Commits you pushed | Yes |
| Stash | Unfinished work you put aside (local only) | No |

---

## Task 1: Setting up and making commits

### 1.1 One-time configuration

Tell git who you are. This information is stored in every commit you make.

```bash
git config --global user.name "Your Name"
git config --global user.email "you@example.com"   # use the e-mail connected to your GitHub account
git config --global init.defaultBranch main
git config --global core.editor "code --wait"      # optional: use VS Code for commit messages
git config --list                                  # check the result
```

### 1.2 Get the repository

If you already have the repository from earlier labs on your machine, skip this step. Otherwise, clone it (this also sets up the remote called `origin`):

```bash
git clone https://github.com/<user>/<repository>.git
cd <repository>
```

Cloning uses HTTPS or SSH. For HTTPS, GitHub requires a personal access token (or a credential manager) instead of your password. The simplest way to sign in is the GitHub CLI: `gh auth login`.

Starting from scratch instead (a new local repository connected later to a remote one):

```bash
mkdir demo && cd demo
git init
```

### 1.3 The basic cycle: status, add, commit

```bash
git status                       # what changed? what is staged?
git add src/register.py          # stage one file
git add src/ tests/              # stage folders
git add -p                       # stage selected parts of files interactively
git diff                         # changes NOT yet staged
git diff --staged                # changes staged for the next commit
git commit -m "Add registration endpoint (#12)"
git log --oneline                # list of commits
```

Do these steps in order:

1. Create a new file `docs/git-workflow.md` with a title and a sentence.
2. Run `git status` and observe that the file is **untracked**.
3. Stage it with `git add` and run `git status` again. Observe how the state changed.
4. Edit the file once more *after* staging, then run `git diff` and `git diff --staged`. Explain the difference.
git diff show all the diferences a git diff --staged shows the diferences of the files in staging mode
5. Commit the file.

### 1.4 Good commit practice

- One commit = one logical change that can be explained in a sentence.
- Write messages in the imperative mood: `Add login form`, not `Added login form`.
- Reference the issue number: `Fix duplicate e-mail check (#14)`. The commit then appears on the issue's timeline.
- Do not commit generated files, secrets or dependencies. Use a `.gitignore`:

```text
# .gitignore
node_modules/
__pycache__/
*.log
.env
.idea/
.vscode/
```

If a file is already tracked, adding it to `.gitignore` is not enough. Remove it from tracking with `git rm --cached <file>`.

### 1.4 Inspecting history

```bash
git log --oneline --graph --decorate --all   # branches and tags as a graph
git log -p -2                                # last two commits with their changes
git show <commit-hash>                       # one commit in detail
git blame src/register.py                    # who changed which line
```

### 1.5 Undoing things

Choose the tool by *what you want to undo*:

| Situation | Command |
|-----------|---------|
| Discard changes in a file (not committed) | `git restore <file>` |
| Unstage a file but keep the changes | `git restore --staged <file>` |
| Fix the last commit message or add a forgotten file (**not yet pushed**) | `git commit --amend` |
| Undo a commit that is **already pushed** by creating a new opposite commit | `git revert <commit-hash>` |
| Move the branch back and keep the changes staged / unstaged | `git reset --soft HEAD~1` / `git reset --mixed HEAD~1` |
| Move the branch back and **delete the changes** | `git reset --hard HEAD~1` |

> **Warning:** `git restore <file>` and `git reset --hard` permanently delete uncommitted work. Never rewrite (amend, reset) commits that your teammates may already have pulled. Use `git revert` for those.

### Expected result of Task 1

- Git is configured with your name and e-mail.
- You made at least three commits with clear messages, one referencing an issue number.
- A `.gitignore` is committed.
- You used `git restore --staged` or `git commit --amend` at least once and can explain why.

---

## Task 2: Branches, merging and conflicts

You already created branches from issues in Lab 3. Here you will see what is happening underneath.

### 2.1 Working with branches

```bash
git branch                        # list local branches (* marks the current one)
git branch -a                     # include remote-tracking branches
git switch -c 20-t-2-1-1-item-model   # create a new branch and switch to it
git switch main                   # go back to main
```

### 2.2 Merging

```bash
git switch main
git merge 20-t-2-1-1-item-model   # bring the branch's commits into main
git branch -d 20-t-2-1-1-item-model   # delete the merged branch
```

If `main` has not moved since the branch was created, git performs a **fast-forward** (no new commit). If both branches have new commits, git creates a **merge commit**. Use `git log --oneline --graph --all` after each case to see the difference.

### 2.3 Resolving a merge conflict

A conflict appears when two branches change the same lines. Create one deliberately (alone or in a pair):

1. On `main`, change the first line of `README.md` and commit.
2. Create a branch from the commit *before* that change (`git switch -c conflict-demo HEAD~1`), change the **same line** differently and commit.
3. Switch to `main` and run `git merge conflict-demo`.

Git stops and marks the file:

```text
<<<<<<< HEAD
# LendIt – item lending app
=======
# LendIt – lend things to friends
>>>>>>> conflict-demo
```

To resolve it:

```bash
# 1. Edit the file: keep the desired text and delete the <<<<<<<, =======, >>>>>>> lines
git add README.md
git commit              # completes the merge (the default message is fine)
```

To give up instead: `git merge --abort`.

### 2.4 Rebase (overview)

`git rebase main` (run on your feature branch) replays your commits on top of the latest `main`, producing a linear history instead of a merge commit. You will mostly meet it through `git pull --rebase` (see Task 5). The rule: **never rebase commits that have already been pushed and used by others.**

### Expected result of Task 2

- You can show a fast-forward merge and a merge commit in `git log --graph`.
- You resolved one conflict and can explain what the markers mean.

---

## Task 3: Stashing

The stash is a temporary shelf for unfinished work. A typical moment: you are in the middle of a task when an urgent bug appears, but your changes are not ready to be committed.

### 3.1 Basic stash cycle

```bash
git stash push -m "WIP: item form validation"   # put tracked changes aside, working directory is clean
git status                                      # nothing to commit
# ... switch branches, fix the urgent bug, commit it ...
git stash list                                  # stash@{0}: On 20-...: WIP: item form validation
git stash pop                                   # re-apply the latest stash and remove it from the list
```

Variants worth knowing:

| Command | Effect |
|---------|--------|
| `git stash push -u` | Also stash **untracked** (new) files. Ignored files are stashed only with `-a`. |
| `git stash show -p stash@{0}` | Show the content of a stash |
| `git stash apply stash@{1}` | Re-apply a stash but **keep** it in the list |
| `git stash drop stash@{0}` | Delete a stash |
| `git stash clear` | Delete all stashes |
| `git stash branch <new-branch> stash@{0}` | Create a branch from the commit where the stash was made and apply it there |

### 3.2 Exercise

1. On a feature branch, edit two files and create one new file. Do **not** commit.
2. Run `git stash push -m "WIP"` and check `git status`. Which files were stashed and which was left behind? Repeat with `-u`.
3. Switch to `main`, make and commit a small fix (e.g. a typo in the README), switch back.
4. Run `git stash pop` and check that your changes returned.
5. Stash again, but this time run `git stash apply` and then `git stash list`. Explain the difference compared with `pop`, and clean up with `git stash drop`.

> The stash is stored **only in your local repository**. It is never pushed, so it does not protect against a lost laptop. If you want to share unfinished work, commit it to a branch and push it.

### Expected result of Task 3

- You stashed and restored changes, including an untracked file.
- You can explain the difference between `pop` and `apply`.

---

## Task 4: Tags and releases

A **tag** is a permanent, human-readable name for a specific commit, most often a released version. Unlike a branch, a tag does not move.

### 4.1 Versioning scheme

Use **Semantic Versioning**: `vMAJOR.MINOR.PATCH`.

- `MAJOR`: incompatible changes
- `MINOR`: new functionality, backwards compatible
- `PATCH`: bug fixes

The first working version of your project could be `v0.1.0`.

### 4.2 Creating tags

```bash
git tag v0.1.0-rc1                              # lightweight tag: just a name for a commit
git tag -a v0.1.0 -m "First working version"   # annotated tag: stores author, date and message (preferred for releases)
git tag -a v0.0.1 <commit-hash> -m "Initial prototype"   # tag an older commit
```

### 4.3 Listing, inspecting and checking out

```bash
git tag                       # list all tags
git tag -l "v0.*"             # filter
git show v0.1.0               # tag details plus the commit
git switch --detach v0.1.0    # look at the code as it was in that release (detached HEAD)
git switch main               # return
```

### 4.4 Sharing tags

Tags are **not pushed automatically** with `git push`.

```bash
git push origin v0.1.0        # push one tag
git push --follow-tags        # push commits together with the annotated tags that point to them
git push origin --tags        # push all local tags (includes lightweight tags)
```

Deleting a tag:

```bash
git tag -d v0.1.0-rc1                  # local
git push origin --delete v0.1.0-rc1    # remote
```

### 4.5 GitHub Release

On GitHub, open **Releases → Draft a new release**, select the existing tag `v0.1.0`, add release notes (you can use **Generate release notes**) and publish. In later labs, a pipeline will be triggered by tags matching a pattern such as `v*`.

### Expected result of Task 4

- An annotated tag `v0.1.0` exists locally and on GitHub.
- A GitHub Release is created from that tag.
- You removed one unneeded lightweight tag, both locally and on the remote.

---

## Task 5: Working with the remote repository

### 5.1 Remotes and tracking branches

```bash
git remote -v                                   # show remotes and their URLs
git remote add origin https://github.com/<user>/<repository>.git   # only if none exists
git push -u origin main                         # first push of a branch; -u sets its upstream
git branch -vv                                  # which local branch tracks which remote branch, ahead/behind counts
```

A **remote-tracking branch** such as `origin/main` is your local, read-only snapshot of what `main` looked like on GitHub the last time you contacted it.

### 5.2 fetch, pull and push

| Command | What it does |
|---------|--------------|
| `git fetch` | Downloads new commits and branches from the remote and updates `origin/*`. **Does not change your files or your branch.** |
| `git pull` | `git fetch` followed by merging `origin/<branch>` into your current branch |
| `git pull --rebase` | `git fetch` followed by replaying your local commits on top of `origin/<branch>` (cleaner history) |
| `git push` | Uploads your local commits of the current branch to the remote |

After a fetch, compare before you integrate:

```bash
git fetch
git log --oneline main..origin/main    # commits on the remote that you do not have
git log --oneline origin/main..main    # your commits that are not pushed yet
```

### 5.3 A rejected push

If a teammate pushed to the same branch before you, your push is rejected:

```text
! [rejected]        main -> main (fetch first)
```

This is not an error in your work. The remote has commits you do not have. Fix it like this:

```bash
git pull --rebase       # or: git pull
# resolve conflicts if any, then: git add <file> && git rebase --continue  (or commit, if you merged)
git push
```

Do not use `git push --force` to "fix" this, as it overwrites your teammates' work. If you have to rewrite your own branch's history, prefer `git push --force-with-lease`, which refuses to overwrite commits you have not seen.

### 5.4 Simulating a team (pair exercise)

Work in pairs, or simulate two people by cloning your repository into two folders (`lendit-anna`, `lendit-marko`).

1. Both clone the repository.
2. Person A creates a branch, commits a change to `docs/git-workflow.md` and pushes it (`git push -u origin <branch>`).
3. Person B runs `git fetch`, finds the branch in `git branch -a` and switches to it with `git switch <branch>`.
4. Person B adds a commit and pushes. Person A pulls (`git pull`) and sees it.
5. Now both edit the **same line** on the same branch without pulling first. The first one pushes. The second one gets a rejected push, runs `git pull`, resolves the conflict and pushes.

### 5.5 Keeping branches tidy

```bash
git fetch --prune                       # remove remote-tracking branches that were deleted on GitHub
git branch -d <branch>                  # delete a merged local branch
git push origin --delete <branch>       # delete a remote branch
```

GitHub can delete branches automatically after a pull request is merged (repository **Settings → General → Automatically delete head branches**).

### 5.6 Protecting `main` (recommended)

In the repository, go to **Settings → Branches** (or **Rules → Rulesets**) and add a rule for `main` that requires a pull request before merging. This enforces the workflow from Lab 3, and in later labs you will also require the build and tests to pass.

### Expected result of Task 5

- You can explain `fetch` vs `pull` and show the ahead/behind state with `git branch -vv`.
- You experienced and fixed a rejected push without forcing.
- A branch protection rule requires pull requests for `main`.

---

## Task 6: Putting it all together: one task from issue to release

Take one real Task from your GitHub Project and carry it through the complete cycle. Document each step.

1. **Issue.** Move the Task to `In Progress` on the Kanban board.
2. **Branch.** Create the branch from the issue (Development → Create a branch, or `gh issue develop <n> --checkout`).
3. **Commit.** Make at least two commits, each referencing the issue (`#<n>`).
4. **Interrupt.** Stash your work in progress, perform a small unrelated fix on its own branch, return and `stash pop`.
5. **Push.** `git push -u origin <branch>`.
6. **Pull request.** Open a PR to `main` with `Closes #<n>` in the description, and have a teammate review it.
7. **Merge.** Merge the PR. Verify that the issue is closed and the card moved to `Done`.
8. **Sync.** `git switch main && git pull`, then `git branch -d <branch>` and `git fetch --prune`.
9. **Release.** Tag `v0.1.0` (annotated), push the tag and publish a GitHub Release.

Finish with `git log --oneline --graph --decorate --all` and save its output.

---

## Deliverables

Add a short document `docs/git-workflow.md` to the repository with:

- the branch naming and commit message convention your team uses (from Lab 3 and this lab);
- the numbered steps of your team's workflow from issue to release;
- the output of `git log --oneline --graph --decorate --all` from Task 6;
- a link to the GitHub Release `v0.1.0`.

Final checklist:

- [ ] Git configured; `.gitignore` committed
- [ ] Multiple commits with clear messages referencing issues
- [ ] One conflict resolved
- [ ] Stash used (with an untracked file) and restored
- [ ] Annotated tag `v0.1.0` pushed and a GitHub Release published
- [ ] Rejected push fixed without `--force`
- [ ] `main` protected: changes only through pull requests
- [ ] `docs/git-workflow.md` added

---

## Command cheat sheet

| Goal | Command |
|------|---------|
| See the state | `git status`, `git diff`, `git diff --staged` |
| Stage / commit | `git add <path>`, `git commit -m "msg"` |
| History | `git log --oneline --graph --decorate --all` |
| Undo working changes / unstage | `git restore <file>` / `git restore --staged <file>` |
| Undo a pushed commit | `git revert <hash>` |
| Branches | `git switch -c <name>`, `git switch <name>`, `git branch -d <name>` |
| Merge | `git merge <branch>`, `git merge --abort` |
| Stash | `git stash push -m "msg"`, `git stash list`, `git stash pop` |
| Tags | `git tag -a v1.0.0 -m "msg"`, `git push origin v1.0.0` |
| Remote | `git remote -v`, `git fetch`, `git pull --rebase`, `git push -u origin <branch>` |
| Clean up | `git fetch --prune`, `git push origin --delete <branch>` |

---

## Useful resources

- [Pro Git book (free)](https://git-scm.com/book)
- [Git reference documentation](https://git-scm.com/docs)
- [GitHub Docs – Git basics](https://docs.github.com/en/get-started/using-git)
- [Semantic Versioning](https://semver.org)