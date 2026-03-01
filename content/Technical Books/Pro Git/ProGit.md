---
date: 2026-03-01
---
# Git Basics

## Getting a Git Repository
There are two primary ways to obtain a Git repository:
- The first is to **initialize a new one from an existing local directory**.
- The second is to **clone an existing remote repository**.

Both approaches result in a fully functional Git repository on your local machine.
### 1. Initializing a Repository in an Existing Directory
To place an existing project under Git version control, navigate to the project's root directory and run:
```bash
$ git init
```

This creates a hidden `.git` subdirectory containing all the internal repository files — essentially a **repository skeleton**. At this point no files are tracked yet. To begin tracking files and record an initial snapshot, use `git add` to stage the desired files followed by `git commit`:
```bash
$ git add *.c
$ git add LICENSE
$ git commit -m 'Initial project version'
```
### 2. Cloning an Existing Repository
To obtain a copy of an existing repository — for example, to contribute to a project — use `git clone <url>`. Unlike systems such as Subversion where `checkout` retrieves only a working copy, **Git's `clone` downloads the full history of every version of every file** in the repository. This means that if a server's disk becomes corrupted, almost any client clone can be used to restore it.
```bash
$ git clone https://github.com/libgit2/libgit2
```

This command creates a directory named `libgit2`, initializes a `.git` directory inside it, pulls down all repository data, and checks out a working copy of the latest version. To clone into a directory with a different name, pass the desired name as an additional argument:
```bash
$ git clone https://github.com/libgit2/libgit2 mylibgit
```

Git supports several transfer protocols, including `https://`, `git://`, and SSH (`user@server:path/to/repo.git`).
## Recording Changes to the Repository
Every file in a Git working directory is in one of two states: **tracked** or **untracked**:
- **Tracked files** are those that existed in the last snapshot or have been staged; they may be **unmodified**, **modified**, or **staged**.
- **Untracked files** are everything else — files Git has never seen.

When you first clone a repository, all files are tracked and unmodified. The typical cycle is: edit files → stage changes → commit snapshot → repeat.

![[Pasted image 20260301105110.png]]
### Checking the Status of Your Files
The **`git status`** command is the primary tool for inspecting which files are in which state:
```bash
$ git status
On branch master
nothing to commit, working tree clean
```

A clean working directory means no tracked files have been modified and no untracked files are present. If you add a new file, it appears under "Untracked files". Git will not include it in any commit until you explicitly tell it to.
### Tracking New Files
To begin tracking a new file, use **`git add <file>`**:
```bash
$ git add README
```

The file now appears under "Changes to be committed," meaning it is **staged**. If passed a directory path, `git add` recursively stages all files within it.
### Staging Modified Files
When a previously tracked file is modified, it appears under "Changes not staged for commit". Running `git add` on it moves it to the staging area. **It is important to understand that `git add` stages the file exactly as it is at the moment the command is run.** If you modify the file again after staging it, the file will appear in both "Changes to be committed" (the previously staged version) and "Changes not staged for commit" (the new edits). You must run `git add` again to stage the latest version.

A helpful mental model: think of `git add` not as "add this file to the project" but as **"add precisely this content to the next commit."**
### Short Status
For a more compact view, use the **`-s`** or **`--short`** flag:
```bash
$ git status -s
 M README
MM Rakefile
A  lib/git.rb
M  lib/simplegit.rb
?? LICENSE.txt
```

The output has two columns: the **left column** reflects the staging area status and the **right column** reflects the working tree status. The status codes are:
- `??` — untracked file
- `A` — newly staged file
- `M` — modified file

For example, `MM` on `Rakefile` means it was modified, staged, and then modified again, so it has both staged and unstaged changes.
## Ignoring Files
To prevent Git from tracking or reporting certain files (such as build artifacts, log files, or compiled binaries), create a **`.gitignore`** file in the project root listing the patterns to exclude:
```
*.[oa]
*~
```

The first line ignores all `.o` and `.a` files; the second ignores files ending with `~` (common editor temporaries). The rules for `.gitignore` patterns are:
- Blank lines and lines starting with `#` are ignored (treated as comments).
- **Standard glob patterns** work and are applied recursively throughout the working tree.
- A leading `/` restricts the pattern to the current directory only (no recursion).
- A trailing `/` specifies a directory.
- A leading `!` negates a pattern, re-including files previously excluded.
- `**` matches nested directories (e.g., `a/**/z` matches `a/z`, `a/b/z`, `a/b/c/z`).

A more illustrative example:
```
# ignore all .a files
*.a

# but do track lib.a, even though you're ignoring .a files above
!lib.a

# only ignore the TODO file in the current directory, not subdir/TODO
/TODO

# ignore all files in any directory named build
build/

# ignore doc/notes.txt, but not doc/server/arch.txt
doc/*.txt

# ignore all .pdf files in the doc/ directory and any of its subdirectories
doc/**/*.pdf
```

It is best practice to set up `.gitignore` before making your first commit. GitHub maintains a comprehensive collection of `.gitignore` templates at [github.com/github/gitignore](https://github.com/github/gitignore). It is also possible to have nested `.gitignore` files in subdirectories; the rules in each apply only to files under that directory.
## Viewing Your Staged and Unstaged Changes
While `git status` tells you _which_ files changed, **`git diff`** tells you _what_ changed — the exact lines added and removed:
- **`git diff`** (no arguments) — compares the working directory to the staging area, showing changes that are **unstaged**.
- **`git diff --staged`** (or equivalently `--cached`) — compares the staging area to the last commit, showing changes **staged for the next commit**.

**Important:** `git diff` alone does not show all changes since the last commit — only those that remain unstaged. If all changes have been staged, `git diff` produces no output.

For those who prefer a graphical interface, **`git difftool`** can display diffs in external tools such as `vimdiff` or `emerge`. Run `git difftool --tool-help` to see what is available on your system.
## Committing Your Changes
Only staged changes are included in a commit. Any modified but unstaged files remain untouched on disk. The simplest way to commit is:
```bash
$ git commit
```

This opens your configured editor (set via `git config --global core.editor`) with a default message that includes the output of `git status` as comments. Alternatively, supply the message inline with the **`-m`** flag:
```bash
$ git commit -m "Story 182: fix benchmarks for speed"
[master 463dc4f] Story 182: fix benchmarks for speed
 2 files changed, 2 insertions(+)
 create mode 100644 README
```

The output confirms the branch (`master`), the SHA-1 checksum (`463dc4f`), and the number of files and lines changed. Using the **`-v`** option additionally includes the diff in the editor, giving a more explicit view of what is being committed.
### Skipping the Staging Area
If you want to skip `git add` entirely and commit all changes to already-tracked files at once, use the **`-a`** flag:
```bash
$ git commit -a -m 'Add new benchmarks'
```

**Be careful:** this flag automatically stages every tracked modified file, which may include unintended changes.
## Removing Files
To remove a file from Git, you must remove it from the staging area and then commit. The **`git rm`** command handles both the staging of the removal and the deletion of the file from the working directory:
```bash
$ git rm PROJECTS.md
```

If the file has been modified or already staged, you must force the removal with **`-f`** to prevent accidental loss of unrecorded data.

To **stop tracking a file without deleting it from disk** (e.g., a file accidentally staged that should have been in `.gitignore`), use the **`--cached`** option:
```bash
$ git rm --cached README
```

The `git rm` command also accepts glob patterns, with the caveat that the `*` must be escaped with a backslash so Git performs its own expansion in addition to the shell's:
```bash
$ git rm log/\*.log
$ git rm \*~
```
## Moving Files
Git does not explicitly track file renames at the metadata level, but it is smart enough to infer them. The convenience command for renaming is:
```bash
$ git mv file_from file_to
```

This is exactly equivalent to running:
```bash
$ mv README.md README
$ git rm README.md
$ git add README
```

**`git mv` is simply a convenience wrapper** — three operations condensed into one. You may rename files with any tool you like, as long as you stage the removal and addition before committing.
## Viewing the Commit History
After several commits, or after cloning a repository, use **`git log`** to inspect the history. By default, it lists commits in **reverse chronological order**, each showing its SHA-1 checksum, author name and email, date, and commit message:
```bash
$ git log
```
### Useful `git log` Options
**`-p` or `--patch`** shows the diff introduced by each commit. Combined with **`-<n>`** (e.g., `-2`), it limits output to the last _n_ commits:
```bash
$ git log -p -2
```

**`--stat`** prints a summary of modified files, the number of files changed, and lines added/removed per commit.

**`--pretty`** changes the output format. Built-in values include `oneline`, `short`, `full`, and `fuller`. The most powerful value is `format`, which lets you define a custom template:
```bash
$ git log --pretty=format:"%h - %an, %ar : %s"
```

Useful format specifiers include:

| Specifier | Description              |
| --------- | ------------------------ |
| `%H`      | Commit hash              |
| `%h`      | Abbreviated commit hash  |
| `%an`     | Author name              |
| `%ae`     | Author email             |
| `%ar`     | Author date, relative    |
| `%cn`     | Committer name           |
| `%cr`     | Committer date, relative |
| `%s`      | Subject (commit message) |

Note the distinction between **author** (the person who originally wrote the work) and **committer** (the person who last applied the work to the repository).

**`--graph`** adds an ASCII branch-and-merge diagram, especially useful when combined with `--pretty=format`:
```bash
$ git log --pretty=format:"%h %s" --graph
```

A summary of common `git log` options:

|Option|Description|
|---|---|
|`-p`|Show patch diff for each commit|
|`--stat`|Show file modification statistics|
|`--shortstat`|Show only the changed/insertions/deletions line|
|`--name-only`|Show list of modified files|
|`--abbrev-commit`|Show abbreviated SHA-1|
|`--relative-date`|Show relative dates (e.g., "2 weeks ago")|
|`--graph`|ASCII branch/merge graph|
|`--pretty`|Alternate output format|
|`--oneline`|Shorthand for `--pretty=oneline --abbrev-commit`|
### Limiting Log Output
In addition to formatting options, `git log` supports several filtering options:
- **`-<n>`** — show only the last _n_ commits.
- **`--since` / `--after`** and **`--until` / `--before`** — filter by date (supports formats like `"2008-01-15"` or `"2 weeks ago"`).
- **`--author`** — filter by author name.
- **`--grep`** — filter by keyword in commit messages.
- **`-S <string>`** — the **"pickaxe" option**, which shows only commits that changed the number of occurrences of a given string. Useful for finding when a function was introduced or removed.
- **`-- <path>`** — limit output to commits that affected specific files or directories (always placed last, preceded by `--`).

To combine multiple `--grep` patterns as an AND rather than OR, add **`--all-match`**. To suppress merge commits from the output, add **`--no-merges`**.

A practical example — finding commits by a specific author in a date range on a specific path:
```bash
$ git log --pretty="%h - %s" --author='Junio C Hamano' --since="2008-10-01" \
  --before="2008-11-01" --no-merges -- t/
```
## Undoing Things
Git provides several tools for undoing changes. **Be aware that some of these operations are irreversible.**
### Amending the Last Commit
If you committed too early — forgot to stage a file, or want to correct the commit message — use **`git commit --amend`**:
```bash
$ git commit -m 'Initial commit'
$ git add forgotten_file
$ git commit --amend
```

**The amended commit entirely replaces the previous one.** It is as if the original commit never happened. This is appropriate only for **local, unpushed commits** — amending a commit that has already been pushed will cause problems for collaborators.
### Unstaging a Staged File
To remove a file from the staging area while keeping its working directory changes intact, use **`git reset HEAD <file>`**:
```bash
$ git reset HEAD CONTRIBUTING.md
```

In newer Git versions (2.23.0+), the preferred alternative is:
```bash
$ git restore --staged CONTRIBUTING.md
```

Both commands leave the file's working directory content untouched; they only unstage it.
### Unmodifying a Modified File
To discard working directory changes and revert a file to its last committed (or staged) state, use **`git checkout -- <file>`**:
```bash
$ git checkout -- CONTRIBUTING.md
```

Or, in Git 2.23.0+:
```bash
$ git restore CONTRIBUTING.md
```

**This is a dangerous operation.** Any local changes to that file are permanently lost — Git replaces the file with the last committed version. Only use these commands when you are absolutely certain you want to discard those changes. Anything that has been committed in Git can almost always be recovered; **anything that was never committed and is discarded this way is likely gone forever.**
## Working with Remotes
**Remote repositories** are versions of your project hosted on a network or the Internet (or even on the same machine). Collaborating requires knowing how to add, inspect, fetch from, push to, and remove remotes.
### Showing Your Remotes
**`git remote`** lists the shortnames of configured remotes. After cloning, you will see at least `origin` — the default name Git assigns to the server you cloned from:
```bash
$ git remote
origin
```

Adding **`-v`** shows the associated URLs for both fetch and push operations:
```bash
$ git remote -v
origin  https://github.com/schacon/ticgit (fetch)
origin  https://github.com/schacon/ticgit (push)
```
### Adding Remote Repositories
To add a remote explicitly, use **`git remote add <shortname> <url>`**:
```bash
$ git remote add pb https://github.com/paulboone/ticgit
```

You can now use `pb` as a shorthand on the command line. For example, `git fetch pb` downloads all data from that remote that you don't yet have locally.
### Fetching and Pulling from Your Remotes
**`git fetch <remote>`** downloads all new data from the remote but **does not automatically merge** it into your working branch. You must merge manually when ready.

**`git pull`** fetches and then automatically merges the tracked remote branch into the current branch. By default, after cloning, your local `master` tracks the remote's default branch. From Git 2.27 onward, if `pull.rebase` is not configured, Git will issue a warning. You can suppress it by setting:
```bash
$ git config --global pull.rebase "false"   # merge on pull (default behavior)
$ git config --global pull.rebase "true"    # rebase on pull
```
### Pushing to Your Remotes
To share your work, use **`git push <remote> <branch>`**:
```bash
$ git push origin master
```

This only succeeds if you have write access and no one has pushed conflicting changes in the meantime. If the remote has newer commits, you must fetch and integrate them first before your push will be accepted.
### Inspecting a Remote
**`git remote show <remote>`** provides detailed information about a remote, including its fetch/push URLs, which remote branches are tracked, and how local branches are configured for `git pull` and `git push`:
```bash
$ git remote show origin
```

This is especially useful in complex projects where branches may be stale, new, or automatically merging.
### Renaming and Removing Remotes
To rename a remote's shortname, use **`git remote rename`**:
```bash
$ git remote rename pb paul
```

This also renames all associated remote-tracking branch references (e.g., `pb/master` becomes `paul/master`).

To remove a remote, use **`git remote remove`** (or `git remote rm`):
```bash
$ git remote remove paul
```

Removing a remote also deletes all associated remote-tracking branches and configuration settings
## Tagging
Git allows you to mark specific commits as important using **tags**, most commonly to denote release points such as `v1.0` or `v2.0`.
### Listing Tags
**`git tag`** (optionally with `-l` or `--list`) lists all tags in alphabetical order:
```bash
$ git tag
v1.0
v2.0
```

To search for tags matching a pattern, use a wildcard with `-l` (the `-l` flag is **mandatory** when using wildcards):
```bash
$ git tag -l "v1.8.5*"
```
### Creating Tags
Git supports two tag types: **lightweight** and **annotated**.

A **lightweight tag** is simply a pointer to a specific commit — a commit checksum stored in a file with no additional metadata:
```bash
$ git tag v1.4-lw
```

An **annotated tag** is stored as a full object in the Git database. It includes the tagger's name, email, date, a tagging message, and can be signed with GPG. **Annotated tags are generally recommended** because they carry this extra context:

```bash
$ git tag -a v1.4 -m "my version 1.4"
```

Use **`git show <tagname>`** to view the tag data along with the commit it points to. For annotated tags, this displays tagger information and the annotation message; for lightweight tags, it shows only the commit details.
### Tagging Later
You can tag a commit after the fact by appending its checksum (or a unique prefix of it) to the `git tag` command:
```bash
$ git tag -a v1.2 9fceb02
```
### Sharing Tags
**Tags are not pushed to remotes by default.** To push a specific tag:
```bash
$ git push origin v1.5
```

To push all local tags not yet on the remote:
```bash
$ git push origin --tags
```

Note that `--tags` pushes both lightweight and annotated tags. If you want to push **only annotated tags**, use `--follow-tags` instead.
### Deleting Tags
To delete a tag locally:
```bash
$ git tag -d v1.4-lw
```

This does not affect the remote. To delete a tag from a remote, use either:
```bash
$ git push origin :refs/tags/v1.4-lw
```

or the more intuitive:
```bash
$ git push origin --delete v1.4-lw
```
### Checking Out Tags
You can check out the state of the repository at a tag with `git checkout <tagname>`, but this places the repository in **detached HEAD state** — any commits you make will not belong to any branch and will be unreachable by branch name. If you need to make changes based on a tag (e.g., fixing a bug in an old release), the correct approach is to create a new branch from the tag:
```bash
$ git checkout -b version2 v2.0.0
```
## Git Aliases
Git allows you to create **aliases** for commands using `git config`, making your workflow faster and more intuitive:
```bash
$ git config --global alias.co checkout
$ git config --global alias.br branch
$ git config --global alias.ci commit
$ git config --global alias.st status
```

Aliases can also encode entire command sequences. For example, creating a custom `unstage` alias:
```bash
$ git config --global alias.unstage 'reset HEAD --'
```

This makes `git unstage fileA` equivalent to `git reset HEAD -- fileA`. Another useful example is a `last` alias to quickly review the most recent commit:
```bash
$ git config --global alias.last 'log -1 HEAD'
```

If you want an alias to execute an **external command** (rather than a Git subcommand), prefix it with `!`:
```bash
$ git config --global alias.visual '!gitk'
```

# Git Branching
## Branches in a Nutshell
Git's branching model is widely considered its "killer feature", setting it apart from virtually every other version control system. To understand why, it helps to first understand how Git stores data internally.

As established earlier, Git does not store data as a series of changesets or diffs — it stores **snapshots**. When you make a commit, Git creates a **commit object** containing a pointer to the staged snapshot, along with the author's name and email, the commit message, and pointers to the commit's parent(s): zero parents for the initial commit, one for a normal commit, and multiple parents for a merge commit.

To make this concrete: suppose you stage three files and commit. Git computes a SHA-1 checksum for each file and stores each file's content as a **blob** object. It then stores a **tree object** that maps filenames to blobs. Finally, it creates the **commit object** pointing to that root tree. Your repository now contains five objects: three blobs, one tree, and one commit.

![[Pasted image 20260301153105.png]]

Each subsequent commit stores a pointer to its predecessor, forming a chain.

![[Pasted image 20260301153131.png]]

**A branch in Git is simply a lightweight, movable pointer to one of these commit objects.** The default branch created by `git init` is called `master` — but this name carries no special significance; it is just a convention. Every time you make a commit on a branch, that branch pointer moves forward automatically to the new commit.

![[Pasted image 20260301153216.png]]
### The HEAD Pointer
Git tracks which **branch you are currently on** using a special pointer called **HEAD**. Unlike the concept of HEAD in systems like Subversion, in Git HEAD always points to the local branch you currently have checked out. When you are on `master`, HEAD points to `master`; when you switch branches, HEAD moves accordingly.

You can visualize branch pointer positions using:
```bash
$ git log --oneline --decorate
f30ab (HEAD -> master, testing) Add feature #32
34ac2 Fix bug #1328
98ca9 Initial commit
```
## Creating and Switching Branches
To create a new branch, use **`git branch <branchname>`**:
```bash
$ git branch testing
```

This creates a new pointer to the current commit but **does not switch to it** — HEAD remains on the current branch:
![[Pasted image 20260301155548.png]]

Creating a branch in Git is essentially instantaneous because it only writes a 41-byte file (a 40-character SHA-1 checksum and a newline).

To switch to an existing branch, use **`git checkout <branchname>`**:
```bash
$ git checkout testing
```

This moves HEAD to point to `testing` and updates the working directory to match the snapshot that branch points to:
![[Pasted image 20260301155658.png]]

Once you make a new commit on `testing`, that branch pointer advances while `master` remains where it was, causing the two branches to **diverge**:
![[Pasted image 20260301155722.png]]

Switching back to `master` with `git checkout master` does two things: it moves HEAD back to `master`, and it **reverts the working directory files** to the snapshot that `master` points to:
![[Pasted image 20260301155750.png]]

This is important to understand — switching branches physically changes the files in your working directory. If Git cannot cleanly perform that transition (e.g., due to uncommitted conflicts), it will not allow the switch.

If you make a new commit, not your project history has diverged:
![[Pasted image 20260301160116.png]]

**Important:** By default, `git log` only shows history reachable from the currently checked-out branch. To see a specific branch's history, run `git log <branchname>`. To see all branches, add `--all`. To visualize divergent history, combine options:
```bash
$ git log --oneline --decorate --graph --all
* c2b9e (HEAD -> master) Make other changes
| * 87ab2 (testing) Make a change
|/
* f30ab Add feature #32
```
### Shortcut Commands
You can create and switch to a new branch in a single step:
```bash
$ git checkout -b <newbranchname>
```

From Git 2.23 onward, **`git switch`** is the modern, more explicit alternative to `git checkout` for branch operations:
- Switch to an existing branch: `git switch testing-branch`
- Create and switch to a new branch: `git switch -c new-branch`
- Return to the previously checked-out branch: `git switch -`
## Basic Branching and Merging
A real-world branching workflow typically looks like this: you are developing a feature in a topic branch when an urgent hotfix is needed on the production `master` branch. Git handles this context-switching cleanly as long as your working directory is in a clean state (all changes committed or stashed) before you switch.
### A Practical Workflow Example
First, let’s say you’re working on your project and have a couple of commits already on the master:
![[Pasted image 20260301160952.png]]

You’ve decided that you’re going to work on issue #53 in whatever issue-tracking system your
company uses:
```bash
# Start working on a new feature
$ git checkout -b iss53
```
![[Pasted image 20260301161104.png]]

You work on your website and do some commits:
```
$ vim index.html
$ git commit -a -m 'Create new footer [issue 53]'
```

The iss53 branch has moved forward with your work:
![[Pasted image 20260301161249.png]]

Now you get the call that there is an issue with the website, and you need to fix it immediately:
```
$ git checkout master
$ git checkout -b hotfix
```

At this point, your project working directory is exactly the way it was before you started working
on issue #53, and you can concentrate on your hotfix in this new branch:
```
$ vim index.html
$ git commit -a -m 'Fix broken email address'
```

![[Pasted image 20260301161537.png]]

You can run your tests, make sure the hotfix is what you want, and finally merge the hotfix branch
back into your master branch to deploy to production. You do this with the `git merge` command:
```
$ git checkout master
$ git merge hotfix
Updating f42c576..3a0874c
Fast-forward
	index.html | 2 ++
	1 file changed, 2 insertions(+)
```

When the `hotfix` branch is a **direct ancestor** of `master` (no divergent work has occurred), Git performs a **fast-forward merge** — it simply advances the `master` pointer forward to the `hotfix` commit. No new merge commit is created:
![[Pasted image 20260301161744.png]]

After merging, the hotfix branch is no longer needed and can be deleted:
```bash
$ git branch -d hotfix
```

You then return to the feature branch iss53 and continue work:
```bash
$ git checkout iss53
Switched to branch "iss53"
$ vim index.html
$ git commit -a -m 'Finish the new footer [issue 53]'
[iss53 ad82d7a] Finish the new footer [issue 53]
1 file changed, 1 insertion(+)
```

![[Pasted image 20260301161926.png]]

Note that **the hotfix changes are not automatically present in `iss53`** — to incorporate them, you can either `git merge master` into `iss53`, or wait until you merge `iss53` back into `master` later.
### Three-Way Merge
When the branch being merged is **not a direct ancestor** of the target branch — that is, *the two branches have diverged* — Git cannot simply move a pointer forward. Instead, it performs a **three-way merge** using three snapshots: the **two branch tips** and their **common ancestor**:
![[Pasted image 20260301162921.png]]

Git automatically creates a new **merge commit** from this three-way merge. A merge commit is distinguished by having **more than one parent**.
```bash
$ git checkout master
$ git merge iss53
Merge made by the 'recursive' strategy.
index.html |
1 +
1 file changed, 1 insertion(+)
```

![[Pasted image 20260301163119.png]]
### Resolving Merge Conflicts
If the same portion of the same file has been modified differently in two branches, Git cannot merge them automatically. It pauses the merge and marks the conflicted files:
```
$ git merge iss53
Auto-merging index.html
CONFLICT (content): Merge conflict in index.html
Automatic merge failed; fix conflicts and then commit the result.
```

Running `git status` shows which files are unmerged. Git inserts **conflict-resolution markers** into those files:
```
Merge branch 'iss53'

Conflicts:
	index.html
#
# It looks like you may be committing a merge.
# If this is not correct, please remove the file
# .git/MERGE_HEAD
# and try again.


# Please enter the commit message for your changes. Lines starting
# with '#' will be ignored, and an empty message aborts the commit.
# On branch master
# All conflicts fixed but you are still merging.
#
# Changes to be committed:
# modified: index.html
#
```

Everything above `=======` is the HEAD (current branch) version; everything below is the incoming branch version. You must manually resolve each conflict by editing the file to the desired final state, removing all conflict markers. Once resolved, stage each file with `git add` to mark it as resolved, then finalize the merge with `git commit`. commit to finalize the merge commit. The commit message by default looks something like this:
```
<<<<<<< HEAD:index.html
<div id="footer">contact : email.support@github.com</div>
=======
<div id="footer">
please contact us at support@github.com
</div>
>>>>>>> iss53:index.html
```

For a graphical resolution experience, run **`git mergetool`**, which launches a supported visual merge tool (such as `vimdiff`, `meld`, or `opendiff`).
## Branch Management
Running **`git branch`** with no arguments lists all local branches, with an asterisk (`*`) marking the currently checked-out branch (i.e., where HEAD points):
```bash
$ git branch
  iss53
* master
  testing
```

Useful options include:
- **`git branch -v`** — shows the last commit on each branch.
- **`git branch --merged`** — lists branches already merged into the current branch; these are generally safe to delete with `git branch -d`.
- **`git branch --no-merged`** — lists branches containing unmerged work. Attempting to delete these with `-d` will fail; use **`-D`** to force deletion and accept the loss of unmerged commits.
- Both `--merged` and `--no-merged` accept an optional branch name argument to check merge status relative to a branch other than the current one: `git branch --no-merged master`.
### Renaming Branches
To rename a branch locally, use **`git branch --move`**:
```bash
$ git branch --move bad-branch-name corrected-branch-name
```

To propagate the rename to the remote, push the corrected branch and set its upstream:
```bash
$ git push --set-upstream origin corrected-branch-name
```

Then delete the old remote branch:
```bash
$ git push origin --delete bad-branch-name
```

**Renaming foundational branches like `master` or `main` requires extra care.** Such a rename can break integrations, CI/CD pipelines, build scripts, and collaborators' workflows. Before doing so, consult all collaborators and update all references to the old branch name in code, documentation, configuration files, pull requests, and repository host settings.
## Branching Workflows
We’ll cover some common workflows that this lightweight branching makes possible.
### Long-Running Branches
Because Git's three-way merge is straightforward, many teams maintain multiple **long-running branches** at different levels of stability. A common pattern is:
- `master` — contains only fully stable, production-ready code.
- `develop` or `next` — an integration branch where features are assembled and tested before being merged into `master`. It isn't necessarily always stable, but whenever it gets to a stable state, it can be merged into `master`. It’s used to pull in **topic branches** (short-lived branches, like your earlier `iss53` branch) when they’re ready, to make sure they pass all the tests and don’t introduce bugs.
- Optionally, a `proposed` or `pu` (proposed updates) branch for experimental integration.

Think of these as **stability silos** — commits "graduate" upward from less stable to more stable branches as they are tested and verified.
![[Pasted image 20260301170930.png]]
### Topic Branches
A **topic branch** is a short-lived branch created for a single feature, bug fix, or piece of related work. Because branching is instantaneous in Git, it is normal to create, work on, merge, and delete topic branches several times a day. This practice enables clean context-switching: all changes related to one topic are isolated together, making code review clearer and allowing features to be merged independently of one another, regardless of the order they were developed.

All branching and merging described so far is entirely **local** — no communication with any server occurs until you explicitly push.
## Remote Branches
**Remote-tracking branches** are local, read-only references that record the state of branches on a remote repository as of the last time you communicated with it. Git moves them automatically during network operations. They follow the naming convention **`<remote>/<branch>`** — for example, `origin/master` or `origin/iss53`.

When you clone a repository, Git automatically creates `origin/master` (pointing to the remote's master) and a local `master` branch that starts at the same commit:
![[Pasted image 20260301175110.png]]

If you and others both make commits, your local and remote-tracking branches diverge:
![[Pasted image 20260301175234.png]]

To synchronize your work with a given remote, you run a `git fetch <remote>` command (in our case, `git fetch origin`). This command looks up which server “`origin`” is (in this case, it’s `git.ourcompany.com`), fetches any data from it that you don’t yet have, and updates your local database, moving your origin/master pointer to its new, more up-to-date position:
![[Pasted image 20260301175301.png]]

The names "origin" and "master" have no special meaning in Git — they are simply the defaults assigned by `git clone` and `git init` respectively. If you clone with `git clone -o booyah`, your default remote tracking branch will be `booyah/master`.

To demonstrate having multiple remote servers and what remote branches for those remote projects look like, let’s assume you have another internal Git server that is used only for development by one of your sprint teams. This server is at `git.team1.ourcompany.com`. You can add it as a new remote reference to the project you’re currently working on by running the git remote add command as we covered in Git Basics. Name this remote `teamone`, which will be your shortname for that whole URL:
![[Pasted image 20260301182234.png]]

Now, you can run git fetch `teamone` to fetch everything the remote teamone server has that you don’t have yet. Because that server has a subset of the data your origin `server` has right now, Git fetches no data but sets a remote-tracking branch called `teamone/master` to point to the commit that teamone has as its master branch:
![[Pasted image 20260301182350.png]]
### Pushing Branches
Local branches are **not automatically synchronized** to remotes. To share a branch, you must push it explicitly:
```bash
$ git push origin serverfix
Counting objects: 24, done.
Delta compression using up to 8 threads.
Compressing objects: 100% (15/15), done.
Writing objects: 100% (24/24), 1.91 KiB | 0 bytes/s, done.
Total 24 (delta 2), reused 0 (delta 0)
To https://github.com/schacon/simplegit
* [new branch]
serverfix -> serverfix
```

Git expands this to `refs/heads/serverfix:refs/heads/serverfix`, which means, “Take my `serverfix` local branch and push it to update the remote’s `serverfix` branch”. You can also push to a differently-named remote branch:
```bash
$ git push origin serverfix:awesomebranch
```

The next time one of your collaborators fetches from the server, they will get a reference to where
the server’s version of `serverfix` is under the remote branch `origin/serverfix:`
```bash
$ git fetch origin
remote: Counting objects: 7, done.
remote: Compressing objects: 100% (2/2), done.
remote: Total 3 (delta 0), reused 3 (delta 0)
Unpacking objects: 100% (3/3), done.
From https://github.com/schacon/simplegit
* [new branch]
serverfix
-> origin/serverfix
```

It’s important to note that when you do a fetch that brings down new remote-tracking branches, you don’t automatically have local, editable copies of them. In other words, in this case, you don’t have a new `serverfix` branch — you have only an `origin/serverfix` pointer that you can’t modify. If you want your own `serverfix` branch that you can work on, you can base it off your remote-tracking branch:
```bash
$ git checkout -b serverfix origin/serverfix
```

To avoid typing credentials on every HTTPS push, configure the credential cache:
```bash
$ git config --global credential.helper cache
```
### Tracking Branches
A **tracking branch** is a local branch with a configured relationship to a remote branch (its **upstream branch**). When on a tracking branch, `git pull` automatically knows which server to fetch from and which remote branch to merge. Cloning sets up `master` to track `origin/master` automatically.

To create a tracking branch explicitly:
```bash
$ git checkout -b serverfix origin/serverfix
# Shorthand:
$ git checkout --track origin/serverfix
# Even shorter (when branch name is unambiguous):
$ git checkout serverfix
```

To track a remote branch under a different local name:
```bash
$ git checkout -b sf origin/serverfix
```

To **set or change the upstream for an existing local branch**:
```bash
$ git branch -u origin/serverfix
```

You can reference a tracking branch's upstream with the shorthand `@{u}` or `@{upstream}` (e.g., `git merge @{u}`).

To **inspect all tracking relationships** and their **ahead/behind status**:
```bash
$ git branch -vv
  iss53    7e424c3 [origin/iss53: ahead 2] Add forgotten brackets
  master   1ae2a45 [origin/master] Deploy index fix
* serverfix f8674d9 [teamone/server-fix-good: ahead 3, behind 1] This should do it
  testing  5ea463a Try something new
```

**Ahead** means local commits not yet pushed; **behind** means remote commits not yet merged. These numbers reflect only what has been cached since the last fetch — to get fully current numbers, run `git fetch --all` first.
### Pulling
**`git pull`** is essentially `git fetch` followed immediately by `git merge` — it fetches data from the tracked remote and merges it into the current branch automatically. It is functionally convenient, but some developers prefer the explicit two-step approach for more control.
### Deleting Remote Branches
Once a remote branch is no longer needed (e.g., after a feature has been merged), delete it with:
```bash
$ git push origin --delete serverfix
```

The data is not immediately removed from the server — Git typically retains it until garbage collection runs, making accidental deletions recoverable for a time.
## Rebasing
Besides merging, Git offers a second way to integrate changes from one branch into another: **rebasing**. Where a merge creates a new merge commit that ties together two divergent histories, a rebase **replays the commits from one branch on top of another**, producing a linear history.
### The Basic Rebase
Let's consider a scenario where `experiment` and `master` have diverged history, but they share a common ancestor:
![[Pasted image 20260301190331.png]]

The easiest way to integrate the branches, as we’ve already covered, is the `merge` command. It performs a three-way merge between the two latest branch snapshots (C3 and C4) and the most recent common ancestor of the two (C2), creating a new snapshot (and commit):
![[Pasted image 20260301190406.png]]

However, there is another way: you can take the patch of the change that was introduced in C4 and reapply it on top of C3. In Git, this is called **rebasing**. With the rebase command, you can **take all the changes that were committed on one branch and replay them on a different branch**. For this example, you would check out the experiment branch, and then rebase it onto the master branch as follows:
```bash
$ git checkout experiment
$ git rebase master
First, rewinding head to replay your work on top of it...
Applying: added staged command
```

Git finds the common ancestor of `experiment` and `master` (in our case, C2), extracts the diffs introduced by each commit on the branch you are on (in our case, only C4), saves them to temporary files, resets the current branch to point to the tip of `master` (so that they are aligned), and then **reapplies each diff in sequence** on top (in our case, only C4). The result is a new series of commits (in our case, C4') with new SHA-1 hashes that appear to follow directly from `master`:
![[Pasted image 20260301191750.png]]

At this point, you can go back to the `master` branch and do a fast-forward merge:
```bash
$ git checkout master
$ git merge experiment
```

![[Pasted image 20260301192146.png]]

**The end snapshot is identical to what a three-way merge would produce — only the history is different.** Rebasing produces a cleaner, linear history that is easier to read, while merging preserves the full record of when branches actually diverged and converged.
### Advanced Rebasing: `--onto`
Consider this scenario where you have a topic branch (`client`) based on another topic branch (`server`):
![[Pasted image 20260301193206.png]]

Nou want to rebase only `client`'s changes (those not shared with `server`, that are C8 and C9) onto `master`, use the **`--onto`** option:
```bash
$ git rebase --onto master server client
```

This says: "Take the commits on `client` that are not on `server`, and replay them onto `master`":
![[Pasted image 20260301193405.png]]

Now you can fast-forward your `master` branch:
![[Pasted image 20260301193456.png]]

Let’s say you decide to pull in your `server` branch as well. You can rebase the server branch onto the master branch without having to check it out first by running `git rebase <basebranch> <topicbranch>` — which checks out the topic branch (in this case, `server`) for you and replays it onto the base branch (`master`):
```bash
$ git rebase master server
```

Notice that `git rebase <basebranch> <topicbranch>` is equivalent to:
```bash
$ git checkout server
$ git rebase master
```

This replays your `server` work on top of your `master` work:
![[Pasted image 20260301193729.png]]

Then, you can fast-forward the base branch (master) and remove `client` and `server` branches:
```bash
$ git checkout master
$ git merge server
$ git branch -d client
$ git branch -d server
```

The result is a perfectly linear history with no merge commits:
![[Pasted image 20260301194440.png]]
## The Perils of Rebasing
Before diving into the mechanics of what can go wrong, the single most important rule of rebasing must be stated clearly:

> **Do not rebase commits that exist outside your local repository and that other people may have based their work on.**

Breaking this rule does not just create minor inconveniences — it can cause serious confusion, duplicate commits, and a corrupted shared history that is painful to untangle. To understand _why_, you need to follow a concrete scenario step by step.
### What Rebasing Actually Does to Commits
The root cause of all rebase-related trouble lies in one fundamental fact: **when you rebase, Git does not move commits — it creates brand new ones.** The new commits contain the same changes (patches) as the originals, but they have different parent pointers and therefore different SHA-1 hashes. From Git's perspective, the original commits and the rebased commits are entirely different objects.

When those original commits were only on your local machine, this is harmless. Nobody else knew about them. But when those commits were already pushed to a shared remote — and worse, when a collaborator has pulled them down and built new work on top of them — abandoning them and replacing them with new commits breaks that collaborator's history in a very concrete way.
### A Step-by-Step Disaster Scenario
#### Step 1 — The starting point
You clone a repository from a central server. At this point, your local history and the server's history are identical. You then do some work locally and add commits `C2` and `C3`:
![[Pasted image 20260301210804.png]]
#### Step 2 — A collaborator pushes a merge commit
Meanwhile, your collaborator has also been working. They made commits on a separate branch, then merged it back into `master`, producing a **merge commit `C6`** (which has two parents: `C4` and `C5`). They push all of this to the central server:
![[Pasted image 20260301210952.png]]

You run `git fetch` and then `git merge origin/master` to merge the new remote branch into your work to incorporate their work, creating merge commit `C7` (merge commit that combines your `C3` with the collaborator's `C6`). Your history now looks like this:
![[Pasted image 20260301211130.png]]

The history has diverged and been rejoined.
#### Step 3 — The collaborator rewrites history with a force push
Here is where the trouble begins. The collaborator decides that the merge commit `C6` makes the history look messy. They go back, **rebase** their original work, and produce a clean linear history, creating a new commit `C4'`. Note that `C4'` introduces the _same change_ as `C6`, but it is a **different commit object** with a different SHA-1 hash and a different parent. They then run `git push --force` to overwrite the server's history with the new, cleaner version:
![[Pasted image 20260301211856.png]]

Commits `C4` and `C6` have been **abandoned** on the server. They no longer appear in any branch's history there. But here is the problem: **you still have them locally.** Your local `C7` merge commit has `C6` as one of its parents, and `C6` has `C4` as one of its parents.
#### Step 4 — You fetch and pull, making things worse
You run `git fetch` and see the new commits from the server. Your local history now contains _both_ the old line of history (with `C4` and `C5`) and the new one (with `C4'`):
![[Pasted image 20260301212153.png]]

You run `git pull`, which performs a `git merge origin/master`. Git sees two divergent histories and creates **yet another merge commit** — let's call it `C8` — to tie them together:
![[Pasted image 20260301212308.png]]

Your history now contains **both `C6` and `C4'`**, which introduce essentially the same change. Running `git log` shows two commits from the same author, at the same time, with the same message, doing the same thing. If you then push this back to the server, you reintroduce the abandoned commits (`C4`, `C6`) to the central repository — the exact opposite of what your collaborator was trying to achieve by rebasing in the first place.

This is the mess the Golden Rule is designed to prevent.
### How Git Can Help You Recover: Rebase When You Rebase
If you do find yourself in this situation — someone has force-pushed rebased commits that overwrite history you have based work on — Git provides a recovery mechanism, though it is not always perfect.

In addition to the standard SHA-1 commit hash, Git computes a second checksum called a **patch-id**, which is based solely on the _content of the change_ introduced by a commit, not on its metadata or parent. This means that even though `C6` and `C4'` have different SHA-1 hashes (different parents, different timestamps), if they introduce the **same diff**, they will have the same patch-id. Git can use this to identify commits that are essentially duplicates.
### What to do instead of `git pull`
Instead of running a plain `git pull` (which causes the double-merge disaster described above), you run:
```bash
$ git pull --rebase
```

Or manually:
```bash
$ git fetch origin
$ git rebase origin/master
```

When you rebase instead of merge in this scenario, Git goes through the following process:
1. It identifies all commits that are **unique to your local branch** — in our example, `C2`, `C3`, `C4`, `C6`, `C7`.m
2. Determine which are **not merge commits** (`C2`, `C3`, `C4`).
3. Determine which have not been rewritten into the target branch (just `C2` and `C3`, since `C4` is the same patch as `C4'`).
4. Apply those commits to the top of `teamone/master`.

The result is a clean, non-duplicated history:
![[Pasted image 20260301215057.png]]

This is far cleaner than the double-history mess produced by a plain `git pull`. **However, this mechanism only works reliably when `C4` and `C4'` are nearly identical patches.** If the collaborator significantly altered the commit during their rebase, the patch-ids will differ, Git will not recognize them as duplicates, and you may end up with genuine conflicts or duplicate changes.
### Making `--rebase` the default for `git pull`
To always use rebase instead of merge when pulling, configure it globally:
```bash
$ git config --global pull.rebase true
```
### Summary of the Danger Zones
It helps to think about rebase safety in three tiers:
- **Safe:** Rebasing commits that exist **only on your local machine** and have never been pushed anywhere. Nobody else has seen them, so rewriting them has no consequences.
- **Generally safe:** Rebasing commits that have been pushed to a remote, **but where you are certain nobody else has fetched and based work on them** — for example, a personal feature branch on a fork that only you use.
- **Dangerous:** Rebasing commits that have been pushed to a **shared remote** and that collaborators may have already pulled and built upon. This is the scenario described above, and it should be avoided.
### Rebase vs. Merge: Which Should You Use?
This is ultimately a question about what your commit history is _for_, and there are two coherent philosophies.

The first treats history as a **faithful record of what actually happened.** Merge commits, however numerous and tangled, are honest — they reflect the real sequence of parallel work and integration points. Rewriting history, from this point of view, is a form of dishonesty: you are presenting a fiction that the work was always sequential and clean, when in reality it was not.

The second treats history as a **curated narrative of how the project was built.** A messy sequence of micro-commits, wrong turns, and merge commits may be an accurate record, but it is not a useful one for future developers trying to understand the codebase. Just as you would not publish a first draft, you should not expose every intermediate step. Rebasing lets you tell a cleaner, more comprehensible story.

**Neither philosophy is wrong.** The right approach depends on your team's culture and the nature of your project. What matters is that everyone on the team agrees on the workflow, because the perils of rebasing arise precisely when team members have different expectations about which commits are considered stable and shareable.

A practical rule that works well for most teams: **rebase your local, unpushed commits to clean up your own work before sharing it, but treat any commit that has been pushed to a shared remote as immutable.** This gives you the benefits of a clean history without the risks of rewriting shared history
## Summary of Key Commands

|Command|Description|
|---|---|
|`git branch <name>`|Create a new branch|
|`git checkout <branch>` / `git switch <branch>`|Switch to a branch|
|`git checkout -b <branch>` / `git switch -c <branch>`|Create and switch to a new branch|
|`git merge <branch>`|Merge a branch into the current branch|
|`git branch -d <branch>`|Delete a merged branch|
|`git branch -D <branch>`|Force-delete an unmerged branch|
|`git branch -v`|Show last commit on each branch|
|`git branch --merged` / `--no-merged`|Filter branches by merge status|
|`git branch --move <old> <new>`|Rename a branch locally|
|`git push origin --delete <branch>`|Delete a remote branch|
|`git fetch <remote>`|Download remote data without merging|
|`git pull`|Fetch and merge remote tracking branch|
|`git push <remote> <branch>`|Push a branch to a remote|
|`git branch -u <remote>/<branch>`|Set upstream tracking branch|
|`git branch -vv`|Show tracking info and ahead/behind status|
|`git rebase <branch>`|Rebase current branch onto another|
|`git rebase --onto <base> <upstream> <branch>`|Rebase a subset of commits onto a base|
|`git pull --rebase`|Fetch and rebase instead of merging|