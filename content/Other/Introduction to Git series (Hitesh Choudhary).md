---
date: 2024-05-11
tags:
  - git
modified: 2024-06-18T21:15:02+02:00
---

In this tutorial, I will share the insights and tips I gathered from the YouTube playlist "Git for beginner" by "**Hitesh Choudhary**" channel (link here: [video playlist here]( https://www.youtube.com/watch?v=7tOLcNZfPso&list=PLRAV69dS1uWT4v4iK1h6qejyhGObFH9_o&index=3&ab_channel=HiteshChoudhary)). These notes have been organized to follow the video step by step.
# 1. Git init an hidden folder

**Git** is a software, while **GitHub** is a service or a service provider (there are many other service provider). So whatever you do through Git and you want to keep an online version of it, that’s what the GitHub is used for.

What does Git actually do? Think of a videogame where you cannot clear the level on just one go because you need the checkpoint to ensure that, if I restart the game, I can restart from that point. This is what happens in the **Version Control System**: it keeps the track of the file and the changes of that. So, whatever the changes you have made, it marks a checkpoint there and, if anything goes wrong, you can just go back there.

Git also helps in collaborative environment

**Repo**. It’s a short form for repository, or we can informally call that as folder which contains files.

First command:

```bash
git --version
```

Let’s see this diagram:

![Untitled](Other/attachments/Untitled%2089.png)

This is the root folder and in this folder we have multiple folders. If I want some of these folders getting tracked, I need to specify to Git which folder it needs to track and which ones don’t.

So, let’s create a test folder and create three different folders inside. Let’s say I want to track `gitone` and `gittwo` and not `gitthree`. If I use `git status`, it’ll say that there is no a git repository and that’s what we expected. It’s not suggested to initialize Git outside where you don’t want to track things, so let’s go first inside `gitone`. It’s always a good habit to type `git status`: 

![Untitled](Other/attachments/Untitled%201%201.png)

Then, let’s initialize the Git software with:

```bash
git init 
```

If I’ll type again git status, I’ll have a different message than before:

![Untitled](Other/attachments/Untitled%202%201.png)

The folder `gitone` is now being tracked by the Git.

Note that you always run `git init` only one time in a project because, once the initialization happens, you don’t do it again.

What `git init` does is it creates a `.git` hidden folder that keeps track of all files and sub-folders. Let’s go inside this folder to look at the files it contains:

![Untitled](Other/attachments/Untitled%203%201.png)

All these folders are going to grow up as we bring more files into these folders, but you never ever go into `.git` folder and do any changes manually.

Before visualizing the entire workflow of the Git, let’s highlight two points:

- Commit statement: it’s like checkpoints in a video game.
- The whole idea behind the workflow of the Git comprises three sub-ideas:
    1. write something into a directory
    2. we add that
    3. we commit that

Let’s finally visualize the entire workflow of the Git schematically:

![Untitled](Other/attachments/Untitled%204%201.png)

- **Working directory**: we have to make a working directory, meaning that we need to initialize Git in a specific directory.
- `git add`: this is not a complete command because you have to mention the file name/names you want to track with Git.
- **Staging Area**: after adding files, they’re not committed yet (the checkpoint is not made yet), but files in this area are ready to be committed.
- `git commit`: we are now ready to make the checkpoint.
- **Repo**: now the repo is onto a checking stage.
- `git push`: once the repo is created, it’s kind of very common to push the entire thing onto some of the Cloud Git provider (like GitHub).

---

# 2. Git commits and logs

We’re going to understand this part:

![Untitled](Other/attachments/Untitled%205%201.png)

As we already mentioned, `git status` is the first command you should always run in every folder to check out whether that folder is already initalized with the Git or we need to do some initialization.

Let’s see again what git status returns in an already-initialized folder:

![Untitled](Other/attachments/Untitled%206%201.png)

Note that in an initialized folder you always be on a branch (like `master`, `main`, `checkout`, `feature`, etc.). Right now we are in the master `branch`.

Let’s create a bunch of folders and file in `gitone` folder:

```bash
touch testone.txt testtwo.txt
```

Now, let’s check `git status` again:

![Untitled](Other/attachments/Untitled%207%201.png)

It returns:

- some untracked files
- nothing to be commited

This means: Git is initialized in this repository, but there are some of the files that are not in tracking zone yet, so, if you make any changes or anything else in these files, Git will not be aware of those and it doesn’t keep track them.

Considering the schema behind the whole idea of Git (Write→Add→Commit), we have just **wrote** some code, and, considering the entire workflow, we have only the **Working Directory** and nothing else. Now, according to the workflow, the next step is to use the command `git add` that it just keeps files under the tracked zone by adding them in the Staging Area. Suppose that you are only ready to track `testone.txt` because you are still working on `testtwo.txt`:

```bash
git add testone.txt
```

The result of `git status` is:

![Untitled](Other/attachments/Untitled%208%201.png)

- `testone.txt` is being tracked and it’s ready to be committed
- there is still one untracked file (`testtwo.txt`)

This means that now `testone.txt` is in the **Staging Area**, that is an intermediate zone before you make any commit.

Now we’re ready to commit tracked files. `git commit` command is a little bit tricky to understand. This command needs a message every time you need to commit something explaining why you are committing those files:

```bash
git commit -m "add file one"
```

It returns this message:

![Untitled](Other/attachments/Untitled%209%201.png)

Looking at the entire workflow, now we are in the **Repo** stage:

![Untitled](Other/attachments/Untitled%2010%201.png)

Let’s see what `git status` returns:

![Untitled](Other/attachments/Untitled%2011%201.png)

Now Git is only worried about `testtwo.txt` because `testone.txt` is already gone in the Repo zone, meaning that is is tracked and everything in this file is updated (nothing is changed from the last checkpoint).

If you open gitone folder with VSCode and look inside the Git Lens extension, you’ll see:

![Untitled](Other/attachments/Untitled%2012%201.png)

Now, let’s do the same we already done with `testtwo.txt`:

```bash
git add testtwo.txt
git commit
```

Note that I didn’t insert the message, which is mandatory, as we already said. What happens is:

![Untitled](Other/attachments/Untitled%2013%201.png)

Press `i` to insert the message, then `ESC`, then type `:wq` (`w` to write, `q` to quit), then hit `ENTER`.

Now, in Git Lens there are two commits:

![Untitled](Other/attachments/Untitled%2014%201.png)

Note that if you hit:

```bash
git add .
```

it will add in the staging area everything is in that folder.

Let’s now use another command:

```bash
git log
```

it returns:

![Untitled](Other/attachments/Untitled%2015%201.png)

Note that each commit that we made has a commit id, which is also visible in VSCode, but here’s is shorter version of the original one:

![Untitled](Other/attachments/Untitled%2016%201.png)

If we’ll type instead:

```bash
git log --oneline
```

It returns the same shorter commit id version as VSCode:

![Untitled](Other/attachments/Untitled%2017%201.png)

Some notes about **commit message**:

- On internet there is a big debate about it and everybody has their own opinion. The opinion of the official Git founders and the official Git website is the following: Git actually follows a principle of **Atomic Commits**, meaning that one commit does one job. It’s not like you do thousands of different things and then make a commit, but instead you do often commits and you do one task at a time with the corresponding commit message.
    
    So, the general idea behind Atomic Commits principle can be summed up as: keep commits centric to one feature, one component or one fix.
    
- Another debate is about using **Present** or **Past** to commit message. The official recommendation is simply use the **Present tense, but Imperative**, meaning like give order to your codebase (i.e. “hey codebase, add this file; hey codebase, add a function to connect to database”, etc.)

But how does Git know about the Author name and email? That’s because there a **config file** (also known as configuration file) in which all these information needs to be saved first so that we can make all these commits and messages. We’ll see in the following chapter.

---

# 3. Git internal working and configs

## 3.1. `git config` command

How does Git know about the **Author** name and email?

`git config` command allows us to configure the configuration file. There are two ways to do that:

1. Globally, meaning that it’s systemwide. Anytime you make any git repository or something else, this is going to be interacted with the global file.
2. Locally, meaning that you can keep the configuration file local to a particular repository.

Usually, anybody who is working with Git like to keep everything as Global, especially name and email.

If you look at the documentation there are so many parameters you can specify in `git config` command, but majorly you’ll be setting two things: `user.name` and `user.email`:

```bash
git config --global user.name "simone d'angelo"
git config --global user.email "sim.dangelo.95@gmail.com"
```

I don’t run these commands because they’re already correctly set.

Another aspect to set with `git config` command is the Code Editor. Since Git is Linux friendly, the default code editor is Vim, which is not so cool for beginners.

Open VSCode and type `command+shift+p`, then type code and click on “Shell Command: Install ‘code’ command in PATH”. This is going to make sure that your computer is aware about a command which is in the path and installs the `code` there. Without this, the whole command that we’re going about to give to the Git is not going to work. Now, to make VSCode the default editor instead of Vim let’s run:

```bash
git config --global code.editor "code --wait"
```

Note that the file will be modified with `git config` command is the hidden file `.gitconfig` at the path `/Users/simonedangelo/`. To find it just type `cd` in the terminal that redirect the terminal to the home directory. Then type:

```bash
cat .gitconfig
```

This what it returns:

![Untitled](Other/attachments/Untitled%2018%201.png)

You can open this file and modify the content.

## 3.2. `.gitignore` file

`.gitignore` is a special file which is read by the Git itself and we have to create this file. Let’s create it with:

```bash
touch .gitignore
```

and it is initially empty. Whenever you are building any big project there will be a file known as environmental variables and this file holds all the sensitive information like API keys, AWS keys. So, there are sensitive information which we want to keep separate and safe.

Let’s create a file called .env and we simulate this file as the one which contains sensitive information. In this file we add:

```bash
MONGODBURI = testurlpath
```

Now, if we look at `git status`, we get:

![Untitled](Other/attachments/Untitled%2019%201.png)

If you have lots of files to add to the staging area, but you also have the file containing sensitive information, you cannot use `git add .` safely. So, open the `.gitignore` file and list all the files which you don’t want to keep a track of them:

![Untitled](Other/attachments/Untitled%2020%201.png)

Now, let’s check again `git status`:

![Untitled](Other/attachments/Untitled%2021%201.png)

Git is no longer keeping a track of `.env` file. Now we can use `git add .` and let’s see again `git status`:

![Untitled](Other/attachments/Untitled%2022%201.png)

Now Git is tracking `.gitignore` file because we add it into the staging area. Let’s create some more files just to look better at the situation with `touch testthree.txt testfour.txt`. Then use `git add .` and commit them.

How will I know in thousands and thousands of files in my project which files should I write in `.gitignore` file? Choose a website like [https://www.toptal.com/developers/gitignore](https://www.toptal.com/developers/gitignore). If you are working on a Node.js project, write Node and then generate the file. It comes out with a pre-generated template which simply gives you commonly ignored files. Copy and paste them into `.gitgnore` file.

## 3.3. How `commit` works behind the scene

Let’s look again at the commits made so far:

![Untitled](Other/attachments/Untitled%2023%201.png)

The long unique commit id that each commit has is known as Hash and it’s generated not randomly, but following some algorithms. In the Git ecosystem every commit is dependent on the previous commit except the first commit and this phenomenon is not shown in the `git log --oneline`.

![Untitled](Other/attachments/Untitled%2024%201.png)

Behind the scenes, besides the Hash code, each commit has information about the **parent**, that represents the previous commit (of course the first commit has no parents). The hash of the second commit is generated based on the hash and the information of the previous parent commit.

Let’s go inside the `.git` hidden folder (that is visible in the terminal only if you type `ls -la`) and let’s run `ls -la` and let’s see the result:

![Untitled](Other/attachments/Untitled%2025%201.png)

Let’s look at those files and folder directly in VSCode. To make it visible in VSCode, type `command+,` and then click on the x on `**/.git`

![Untitled](Other/attachments/Untitled%2026%201.png)

Let’s look at some of the files in the `.git` folder.

- `COMMIT_EDITSMG` file, that is the file that opens up when you hit `git commit` without the message:
    
    ![Untitled](Other/attachments/Untitled%2027%201.png)
    
- `HEAD` file, which is pointing towards the **master**. As we already mentioned, in Git we always are on some branch and now we are on master branch:
    
    ![Untitled](Other/attachments/Untitled%2028%201.png)
    
- `hooks` folder, that is one of the most advanced folder in Git. If you need to execute some code just before the commit or just after the commit message or you want to control how the commit message actually goes (maybe you want to add the ticket IDs and you want to validate that whether whoever is committing the message is providing the ticket ID for solving this bug), you can actually do all of this through here. There are some samples that you can change:
    
    ![Untitled](Other/attachments/Untitled%2029%201.png)
    

---

# 4. Git merge and git conflicts

## 4.1. Create new branches

Let’s open `gittwo` folder, which is not initialized yet. Let’s initialize with `git init` and then hit `git status`. It says “On branch master”. What is a **Branch** and what is **master**?

Let’s create a file by typing `touch index.html`, then add it into the staging area with `git add index.html`, then commit with `git commit -m "add index file"`.

Now, let’s type:

```bash
git branch
```

It returns an asterisk pointer which is pointing towards the master:

![Untitled](Other/attachments/Untitled%2030%201.png)

**Branch**: think of branch as an alternative timeline and every single coder who is contributing to the same code base can be on its own timeline without affecting each other’s timeline.

![Untitled](Other/attachments/Untitled%2031%201.png)

There is always one branch on which you will be. By default, Git always by default creates a **master** as a branch (which is the green one in the image above), and all those nodes are the commits (checkpoints) that we have made. Nowadays nobody likes to call it as master, so it’s pretty obvious that you will be asked to rename your branch as **main**.

Let’s type `!+tab` in the index.html file to create an html template, then let’s create a checkpoint of this file with a commit:

```bash
git commit -m "update code for index file"
```

Of course the branch we are pointing is still `master`.

There are a bunch of ways to create new branches and let’s start with the basic one:

```bash
git branch nav-bar
```

and look again at `git branch`:

![Untitled](Other/attachments/Untitled%2032%201.png)

The pointer is still pointing towards the `master` branch, but there is the new branch.

We move to a different branch with:

```bash
git checkout nav-bar
```

Now, the pointer is pointing to `nav-bar` branch:

![Untitled](Other/attachments/Untitled%2033%201.png)

If we look at the `HEADS` file, we can see that the it’s pointing to the `nav-bar` branch:

![Untitled](Other/attachments/Untitled%2034%201.png)

If we look at the `.git/refs/heads/` folder we have a new file called `nav-bar` as well as `master` that contain a string representing the commit id each branch is pointing to:

![Untitled](Other/attachments/Untitled%2035%201.png)

They contains the same commit id because the parent commit is the same. This is better visible on Git Lens where `master` and `nav-bar` branches are on the same line pointing to the same location:

![Untitled](Other/attachments/Untitled%2036%201.png)

Now that we are in the nav-bar branch, let’s create a new file called `nav-bar.html` and put something inside. Let’s add the new file in staging area, then commit it with `git commit -m "add navbar to code base"`.

From Git Lens we can see that the `nav-bar` branch is moved a little bit ahead and `master` is still at the same position (note the blue circle close to the branch name indicates the branch you currently are on):

![Untitled](Other/attachments/Untitled%2037%201.png)

If we change again the branch with `git checkout master`, we’ll see that the `nav-bar.html` file is no longer in the project folder because you can suppose it’s a file added by another coder who is working on its own timeline.

Let’s create a new file called `hero-section.html` and put something inside. Then add it to the staging area and commit it.

The situation we are currently at is the one highlighted by the red laser:

![Untitled](Other/attachments/Untitled%2038%201.png)

Suppose the green timeline as the `master` branch and the blue one as the `nav-bar` branch. They’re completely different timeline, different code bases, different files.

## 4.2. Head

Let’s talk about the **Head**. Head always points to where the branch is currently at. It’s not always necessary that the Head is pointing towards the end often the branch. Usually it does, but you can actually allow your Head to point a commit back as well because it’s a checkpoint:

![Untitled](Other/attachments/Untitled%2039%201.png)

The idea behind Head concept came from the image of cassettes where you have a head which plays onto that and if, you flip these cassettes, this head is still pointing towards wherever you left it last time (*I don’t understand this sentence*). Now that I am on the master `branch`, the `HEAD` file in the `.git` folder shows that the head is pointing to the latest commit of the master `branch`:

![Untitled](Other/attachments/Untitled%2040%201.png)

How can I know that is this the latest commit? Let’s look a the `master` file into the `.git/refs/heads/` folder and then let’s compare it to what we’ll see after running `git log --online`:

![Untitled](Other/attachments/Untitled%2041%201.png)

The commit id is exactly the same. We can also forcefully ask Git to point to another commit (like the `34f80d7` one) and we’ll see how to do that.

## 4.3. Shortcut

You can write two separate commands like:

```bash
git branch dark-mode
git checkout dark-mode
```

into a unique command:

```bash
git checkout -b dark-mode
```

An alternative command to `checkout` is `switch`:

```bash
git branch dark-mode
git switch master
```

or into a unique command:

```bash
git switch -c dark-mode
```

## 4.4. Merge branches

There are two types of Merging Branches:

- **fast-forwards merging**, in which the main branch doesn’t do much because you just work entirely on to a separate brunch and then you merge that branch into the `master` one (suppose the green branch as the `master`):
    
    ![Untitled](Other/attachments/Untitled%2042%201.png)
    
    This is the easiest merging type.
    
- **not fast-forwards merging**, in which both master and alternative branches are working and eventually at some point of time you want to pack into a unique branch:
    
    ![Untitled](Other/attachments/Untitled%2043%201.png)
    
    In order to make this merge, it’s ideal to be on the `master` branch and bring the alternative timeline into that.
    

Let’s make it practical. Firstly ensure to be on the `master` branch. Note that we are in a special situation where **nothing is conflicting each other between the two branches**, meaning that whatever the branch I created that branch didn’t work on the same that other branches worked on. In our case, `master` branch worked on `hero-section` file, while `nav-bar` branch worked on `nav-bar.html` file, so there are no conflicts.

To merge the two branches, run:

```bash
git branch nav-bar
```

What you’ll see in the terminal is:

![Untitled](Other/attachments/Untitled%2044%201.png)

VSCode automatically opens a specific file and the terminal is waiting because I haven’t passed any message for this operation. The default message it gives to me is:

![Untitled](Other/attachments/Untitled%2045%201.png)

Merge requires a message because it’s creating a new node in the timeline:

![Untitled](Other/attachments/Untitled%2046%201.png)

So let’s save the message and, as soon as you close `MERGE_MSG` file, let’s look at the terminal output:

![Untitled](Other/attachments/Untitled%2047%201.png)

The output is pretty clear: 1 file is changed (nav-bar.html file is added) and the modifications are only insertions (this is what the `++++++` symbols mean), no deletions.

Let’s look at the `git status` output:

![Untitled](Other/attachments/Untitled%2048%201.png)

From this output we can see that also **all the commits that were made on the `nav-bar` branch (in our case only one) are also part of the `master` branch**.

This is better visible from Git Lens:

![Untitled](Other/attachments/Untitled%2049%201.png)

After merging the two branches, we have all the files coming from the two branches into a unique branch:

![Untitled](Other/attachments/Untitled%2050%201.png)

## 4.5. Delete branches

Even tough it’s pretty dangerous to do that, many companies ask to writing one more additional command:

```bash
git branch -d nav-bar
```

which **deletes** the `nav-bar` branch because that branches it’s no longer useful for us. This command doesn’t delete the history of itself indeed the Git Lens show the same graph as before, but the `git log --oneline`  output is a little bit different:

![Untitled](Other/attachments/Untitled%2051%201.png)

## 4.6. Exercise: fast-forward merge

Let’s create a new branch called `footer` and move inside it, then create a new file called `footer.html` and put something inside, then add it and commit it. Now switch to the `master` branch. Now let’s merge `master` and `footer` branches:

```bash
git merge footer
```

That’s what we see:

![Untitled](Other/attachments/Untitled%2052%201.png)

## 4.7. Conflicts

In the `master` branch let’s modify the `index.html` by writing something inside it, then add it in the staging area, then commit it.

Now checkout to the `footer` branch, then modify the `index.html` file by writing something else as you just did in the other branch, then add it, then commit it.

In this situation the same file `index.html` is different between `master` and footer `branches`. This is a situation of **conflict**!

Whenever such conflicts happen, we have to deal with a specific **diagram** like this one:

![Untitled](Other/attachments/Untitled%2053%201.png)

The green line (or the HEAD) is actually going to be pointing towards the branch your are currently are on (`master` in our case). Everything above those equal signs is part of the `master` branch, while anything below is some code coming from another branch (`footer` in our case). Here you have to decide how to solve this conflict: you can keep the first code, you can keep the second one, or you can keep both.

Let’s do that. Checkout to the master branch, then merge the two branches. What the terminal returns is:

![Untitled](Other/attachments/Untitled%2054%201.png)

Automatically, VSCode opens this window:

![Untitled](Other/attachments/Untitled%2055%201.png)

Then manually do your choice by keeping the code you want, like:

![Untitled](Other/attachments/Untitled%2056%201.png)

Then save the file, then go back to the terminal, then hit `git add .`, then `git commit -m "merged footer branch"`, and now everything works fine.

# 5. git diff and stashing

`git diff` command shows the differences of the same file in two different version, not differences between file1 and file2. Say you are working on file1 and you staged some changes, now you want to compare how the file looked like when the things were not staged and now they are in the stage.

How to read diff?

- a→file1 & b→file2 (same file over time, not different files);
- `---` symbol: file1;
- `+++` symbol: file2.

## 5.1. View changes in the staging area

Let’s move to the `master` branch and add something to `index.html` file. Then add it in the staging area. This is the place where you can run the command `git diff --staged` to check the difference between the staging area (so you have no committed it yet, only staged) and the last commit:

```bash
git diff --staged
```

What we’ll see is:

![Untitled](Other/attachments/Untitled%2057%201.png)

- `diff --git a/index.html b/index.html`: it is saying that it’s checking between a file `a` and a file `b`, where both are `index.html` file.
- `--- a/index.html` and  `+++ b/index.html`: this doesn’t mean that code is being removed or being added. Those symbols are just used to represent the two file versions.
- The code in red and green represents the code present in the staging area (the one with `+`), and the code present in the last commit (the one with `-`).

Let’ change now the `footer.html` file, then add it, then look again at the difference:

![Untitled](Other/attachments/Untitled%2058%201.png)

It’s the same as before, but now we are comparing two files, not only one. Then commit the modifications.

## 5.2. **View changes between specific commits**

Let’s see the logs:

![Untitled](Other/attachments/Untitled%2059%201.png)

So, you can see difference between two specific commits:

```bash
git diff 1e768f0 b957965
```

alternatively

```bash
git diff 1e768f0..b957965
```

## 5.3. git stash

- Create a repo, work on it, then commit on `main`;
- switch to another branch (i.e. `bugfix` branch) and work on it (in our code change `footer.html` file);
- I will not be able to move to `main` branch without commit those changes.

![Untitled](Other/attachments/Untitled%2060%201.png)

How to go back into the `main` branch without committing changes into `bugfix` branch? Commands that we’ll use are:

- `git stash`
- `git stash pop`

Let’s use first:

```bash
git stash
```

Here’s the terminal output:

![Untitled](Other/attachments/Untitled%2061%201.png)

Now, I’m able to switch branch:

![Untitled](Other/attachments/Untitled%2062%201.png)

But If now I’ll try to go back to `bugfix` branch, we’ll see that the modifications I did before are not there. That’s because I have stashed those changes, meaning that it’s like a temporary shelf where you can keep your code and move around and then go back, but now you have to brings things back from that shelf. So bring our modification back:

```bash
git stash pop
```

The result:

![Untitled](Other/attachments/Untitled%2063%201.png)

If we now look at the code, we can see that the modifications are back.

If I stash some modifications in a particular branch, is it possible to pop those stash onto another branch?

Let’s modify again the `footer.html` file and then use the `git stash` command. Now move to the `master` branch and hit `git stash pop`. It works and here’s the result:

![Untitled](Other/attachments/Untitled%2064%201.png)

So, stash is not limited to a specific branch, but stash can be moved between different branches.

Let’s stash again modification you have just popped and then hit `git stash list`:

![Untitled](Other/attachments/Untitled%2065%201.png)

In this case you can specify the stash you want to pop with:

```bash
git stash apply stash@{0}
```

![Untitled](Other/attachments/Untitled%2066%201.png)

Stash should be used only temporarily and it must be used very carefully when you work in collaboration with other developer.

## 5.4. More commands

- `git checkout <hash>`:
    
    Run `git log --online`, then choose the log id corresponding to the commit you want to restore and run:
    
    ```bash
    git checkout d651a83
    ```
    
    Here’s the output:
    
    ![Untitled](Other/attachments/Untitled%2067%201.png)
    
    Look at the last line: that’s what we said before, that is `HEAD` file is now pointing to this commit id:
    
    ![Untitled](Other/attachments/Untitled%2068%201.png)
    
    Furthermore if you type `git branch`:
    
    ![Untitled](Other/attachments/Untitled%2069%201.png)
    
    How can I go back now? If I run `git log --oneline`, I cannot see the commit id I was before:
    
    ![Untitled](Other/attachments/Untitled%2070%201.png)
    
    To go back where you were, just run:
    
    ```bash
    git checkout master
    ```
    
    (I asked ChatGPT and you also can use this command:
    
    ```bash
    git checkout -
    ```
    
    I tried and it works.)
    
    In case you forgot where you were you can use a command which is rarely used, that is:
    
    ```bash
    git reflog
    ```
    
    This command just moves your HEAD where you were.
    

---

# 6. git rebase

`git rebase` command always rewrite the history, then it must be used carefully.

This command can be used:

- as an alternative way of merging
- as a clean-up tool (clean up commits)

**Note**: whenever you are on the master or main branch, you must never run `git rebase` command; use it only in side branches.

Let’s stay into the `master` branch and do some changes into `index.html` file. Then add it and commit it. Now move to the `bugfix` branch and do some changes into `nav-bar.html` file. Then add it and commit it. Then I want to work again on the `master` branch, so move into it and do some changes to `index.html` file. Then add and commit it. Now move to the `bugfix` branch again. It would be nice if could bring here the whole code base from the `master` branch and we’ll do it with:

```bash
git merge master
```

Then the VSCode opens, save it and close, and the merge is done. In this `bugfix` branch, do some changes into `nav-bar.html` file, then add it and commit it.

Then move to the `master` branch and do some changes into index.html file. Then add it and commit it.

If we look at the log of both `master` and `bugfix` branches, we can notice that a lot of commits are just merging commits and they are not putting any word, they’re just merging and they’re not really necessary and many people don’t like it.

The current timeline situation is:

![Untitled](Other/attachments/Untitled%2071%201.png)

Be sure not to be on `master` branch and move to `bugfix` branch and then run:

```bash
git rebase master
```

meaning that you’re rebasing the `bugfix` branch with the `master` one.

The timeline is rewritten:

![Untitled](Other/attachments/Untitled%2072%201.png)

(to be finished)

# 7. Insight of pushing code to Github

## 7.1. Push a local repo to a remote repo

**GitHub** is one of the many online services which helps you to keep all of your code base along wit the git maintenance and the git timeline onto a cloud environment, so everything is safe.

You cannot work with the terminal using your email and password because GitHub works with SSH keys. So you have to generate some SSH key and have to save that same key in the GitHub settings page so that GitHub knows that you are the same person who is communicating. And this communication in GitHub happens on the basis of SSH, not on the basis of email and password.

Since we want to connect our system with GitHub we want to have an SSH key on our system and an SSH key on the GitHub settings page. So the two links you have to open to do that are the following ones:

![Untitled](Other/attachments/Untitled%2073%201.png)

Let’s move to `gitthree` folder and let’s initialize a git repository. Then create a new file called `index.html` and put something inside, then add it to the staging area and then commit it.

Go in your GitHub account and create a new repository that we call `learn-git`. After creating you’ll see:

![Untitled](Other/attachments/Untitled%2074%201.png)

- `echo "# learn-git"  >> README.md`: appends the text `# learn-git` to the end of the file named README.md in the current directory. If the file doesn't exist, the **`>>`** operator will create it.
- `git init`: it initialize a new Git repository.
- `git commit -m "first commit"`: we already know.
- `git branch -M main`: rename the actual branch.
- `git remote add origin git@github.com:simdangelo/learn-git.git`: to explain this command let’s explore some other things before. So far, we worked on a **local repository**, while now we have to deal with a **remote repository**.
    - `git remote add <name> <url>`: is used to add a new remote repository to your Git project.
        - `<name>`: is a short name used to refer to the remote repository. It's common to use `origin` as the name, but you can choose any name you like;
        - `<url>`: is the URL of the remote repository.
- `git push -u origin main`: the syntax of this command is `git push <remote> <branch>`. It means that our command pushes local `main` branch to the `main` branch on the `origin` remote repository. If you run `git push -u origin dev`, it pushes our `dev` branch to the `dev` branch on the `origin` remote repository.
    
    `-u`: this flag stands for “upstream”. It tells Git to set up a tracking relationship between the current branch in your local repository and the branch on the remote repository. This means that in the future, when you use `git push` without specifying a remote branch, Git will know which branch to push to by default. If I run `git push origin main`, this track doesn’t happen and each time I need to push I would write `git push origin main` again.
    

## 7.2. README.md file

Let’s create a `README.md` file in the project directory, then write something inside, add it, commit it, and push it. If you open the repository on GitHub, you’ll notice that GitHub picks up that file and shows it on the homepage of the repository:

![Untitled](Other/attachments/Untitled%2075%201.png)

## 7.3. Other commands

`git pull <remote repository url>`: it brings the code from a specific remote repository onto your system. To use it, go into a public repository on GitHub like the following one:

![Untitled](Other/attachments/Untitled%2076%201.png)

then copy the HTTPS url (or the SSH url), then go to the terminal into a specific directory where you want to clone that project, then run:

```bash
git clone https://github.com/simdangelo/apache-spark-blog-tutorial.git
```

It could happen that you are working on some remote repository with other people and suppose you are working on `index.html` file and a colleague is working on `footer.html`. Before doing my push, I want to grab the things which is updated on the GitHub so that I can make a fresh push onto the GitHub repo.

In this context `git pull` and `git fetch` both actually bring the code from the connected remote repository to your codebase.

Look at this diagram:

![Untitled](Other/attachments/Untitled%2077%201.png)

- `git fetch`: brings new updates from Remote Repository into your Local Repository, but it doesn’t bring into your working area yet.
- `git pull`: brings new updates from Remote Repository into your Working Area. Think of it as `git pull = git fetch + git merge`.