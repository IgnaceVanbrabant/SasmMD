# SaSM Lab Git Guide

This guide explains what you need to do to complete the SaSM Git lab. Follow the steps in order and keep your commits clean, small, and descriptive.

> Important: your GitHub repository for this lab must be **private**. A public repository can cost you points.

## 1. What this lab is about

In this lab you must:

- Use Git to track configuration files and scripts.
- Keep a clear revision history for every exercise.
- Document your progress with descriptive commit messages.
- Push your local work to a private GitHub repository.
- Select exactly five good commits for each evaluation moment.

The main repository is created directly in `/etc`, so you manage configuration files in their real location. Do **not** copy files from `/etc` into another folder just to commit them.

## 2. Commit rules you must follow

There are three important commit types:

1. **Initial commit**
   - The first commit after creating the repository.
   - Usually contains your `.gitignore` and the first files you decide to track.

2. **Software installation or update commit**
   - Use this after installing or updating software.
   - This commit must only contain changes caused by the install or update.
   - Do not include your own configuration edits in this commit.

3. **Configuration or script change commit**
   - Use this after you manually change configuration files or scripts.
   - This commit must only contain your own changes for one logical step.
   - Do not mix typo fixes, package installation changes, and new configuration for the next step in one commit.

Good commit messages are short but descriptive:

```bash
git commit -m "Track initial SSH configuration"
git commit -m "Record nginx package installation"
git commit -m "Configure static address for server interface"
git commit -m "Fix typo in backup script path"
```

Avoid vague messages:

```bash
git commit -m "changes"
git commit -m "fix"
git commit -m "stuff"
```

## 3. Install Git

Run these commands as root on your server:

```bash
apt update
apt install git
```

Check the installed version:

```bash
git --version
```

You can also inspect the package version available from your repositories:

```bash
apt-cache policy git
```

## 4. Configure your Git identity

Set your name and email address. Replace the example values with your own details.

```bash
git config --global user.name "Your Name"
git config --global user.email "your.name@example.com"
```

Check the result:

```bash
git config --global --list
```

## 5. Create the Git repository in `/etc`

Go to `/etc` and initialize the repository:

```bash
cd /etc
git init
```

Check the repository status:

```bash
git status
```

At this point Git can see many files in `/etc`. You should not blindly add everything.

## 6. Create a safe `.gitignore`

A common approach is to ignore everything by default and then explicitly allow only the files and directories you need to track.

Create `/etc/.gitignore`:

```bash
nano /etc/.gitignore
```

Example starting point:

```gitignore
/*
!.gitignore
!aliases
!apt/
!cron.d/
!default/
!group
!passwd
!hosts
!iproute2/
!resolv.conf
!scripts/
!ssh/
!systemd/
!systemd/system/
!udev/
!vim/
!extra_exercises/
```

Important notes:

- Do not use the example blindly. Adapt it to the files and directories needed for your assignments.
- If a later exercise creates a new file that must be tracked, update `.gitignore`.
- If a directory is ignored, Git cannot track files inside it unless the directory path is also unignored.

For example, to track `/etc/systemd/system/ladvd.service`, you need entries like:

```gitignore
!systemd/
!systemd/system/
!systemd/system/ladvd.service
```

## 7. Make the initial commit

After creating `.gitignore`, check what Git sees:

```bash
cd /etc
git status
```

Add the initial files you want to track:

```bash
git add .gitignore
git add hosts
```

Inspect what will be committed:

```bash
git status
git diff --cached
```

Commit:

```bash
git commit -m "Initial configuration tracking setup"
```

## 8. Create a private GitHub repository

On GitHub:

1. Create a new repository.
2. Set visibility to **Private**.
3. Do not add a README, `.gitignore`, or license from GitHub if your local repository already exists.
4. Configure SSH key authentication.

Check your SSH access:

```bash
ssh -T git@github.com
```

Add the private GitHub repository as the remote. Replace the URL with your real private repository URL:

```bash
cd /etc
git remote add origin git@github.com:your-user/your-private-repo.git
```

Verify the remote:

```bash
git remote -v
```

Push the repository:

```bash
git branch -M main
git push -u origin main
```

If your lecturer expects the branch name `master`, keep using `master` instead of renaming it to `main`.

## 9. Daily workflow for every exercise

Use this workflow for every assignment step.

### Step 1: Start clean

Before changing anything:

```bash
cd /etc
git status
```

Good status:

```text
nothing to commit, working tree clean
```

If Git shows untracked or modified files, deal with them before starting the next task.

### Step 2: Install or update software if needed

If the exercise requires installing a package:

```bash
apt install package-name
```

Then immediately check Git:

```bash
cd /etc
git status
```

Review the changes:

```bash
git diff
```

Add only the relevant installation-created files:

```bash
git add path/to/file
```

Commit the installation result separately:

```bash
git commit -m "Record package-name installation"
```

Push:

```bash
git push
```

Only after this should you start editing configuration files.

### Step 3: Make your configuration change

Edit the required configuration file:

```bash
nano /etc/path/to/config-file
```

Check what changed:

```bash
cd /etc
git status
git diff
```

Add only the files for this specific change:

```bash
git add path/to/config-file
```

Commit:

```bash
git commit -m "Configure specific feature or service"
```

Push:

```bash
git push
```

### Step 4: Fix mistakes in separate commits

If you later notice a typo or configuration mistake, fix it in its own commit:

```bash
nano /etc/path/to/config-file
git diff
git add path/to/config-file
git commit -m "Fix typo in specific configuration"
git push
```

Do not hide unrelated fixes inside another commit.

## 10. Handling exercises without `/etc` changes

Some exercises may ask questions or require written answers instead of configuration changes.

You have two options:

### Option A: Use `/etc/extra_exercises`

Create a directory inside the `/etc` repository:

```bash
mkdir -p /etc/extra_exercises
```

Make sure `.gitignore` allows it:

```gitignore
!extra_exercises/
!extra_exercises/**
```

Create an answer file:

```bash
nano /etc/extra_exercises/exercise-name.txt
```

Commit it:

```bash
cd /etc
git add .gitignore extra_exercises/exercise-name.txt
git commit -m "Add answer for exercise name"
git push
```

### Option B: Use a separate repository

You may also create a separate Git repository for written answers. If you do this, keep the same commit discipline:

- Private remote repository.
- Small commits.
- Descriptive commit messages.
- Clear revision history.

## 11. How to check whether your repository is clean

Run:

```bash
cd /etc
git status
```

Good:

```text
On branch main
nothing to commit, working tree clean
```

Not good:

```text
Untracked files:
  default/irqbalance
```

This means a new file exists but is not committed.

Also not good:

```text
Changes to be committed:
  modified: systemd/network/team0.11.network
```

This means something is staged but not committed.

Before starting a new exercise, either commit the correct files or decide intentionally that the files should not be tracked.

## 12. Useful review commands

Show current status:

```bash
git status
```

Show unstaged changes:

```bash
git diff
```

Show staged changes:

```bash
git diff --cached
```

Show commit history:

```bash
git log --oneline
```

Show full commit IDs:

```bash
git log --format=fuller
```

Show files changed in recent commits:

```bash
git log --stat
```

Show exactly what changed in one commit:

```bash
git show full-commit-id
```

## 13. Evaluation: choose exactly five commits

For each evaluation moment, choose exactly five good commits that follow the lab rules.

Evaluation file names:

- Evaluation 1: `/root/commits1`
- Evaluation 2: `/root/commits2`
- Evaluation 3: `/root/commits3`

Each file must contain exactly five different full 40-character commit IDs.

Example:

```text
6da02046d1ac6b6079db4afa1539d2005a226f7d
dfcb41d409457d9214d49376a4ee3fe93b658f57
98eb3d30220344f25e7eaaeec9c0c61c95db6150
019246f791e2f02d8f31f688bca759b91f64e47d
10eff2275f44cdb61c197d13a8360898e0b26079
```

To get full commit IDs:

```bash
cd /etc
git log --format="%H %s"
```

Create the evaluation file:

```bash
nano /root/commits1
```

Check that the file contains exactly five valid commit IDs:

```bash
grep -P '^\w{40}$' /root/commits1 | wc -l
```

The output must be:

```text
5
```

Check that they are five unique commits:

```bash
grep -P '^\w{40}$' /root/commits1 | sort -u | wc -l
```

The output must also be:

```text
5
```

View the selected commits in the Git log:

```bash
grep -A5 -f /root/commits1 <(git -C /etc log)
```

For evaluation 2 or 3, replace `commits1` with `commits2` or `commits3`.

## 14. What makes a good evaluation commit?

Choose commits that are easy to defend and understand.

Good examples:

- A commit that only records a package installation.
- A commit that only changes one service configuration.
- A commit that only adds one script for an exercise.
- A commit that only fixes one specific mistake.
- A commit with a clear message explaining the purpose.

Bad examples:

- A commit that mixes package installation and manual configuration.
- A commit that adds large unrelated files.
- A commit that changes many services without a clear reason.
- A commit named `fix`, `update`, or `final`.
- A commit that contains copied files instead of files managed in their real location.

## 15. If you make a serious mistake

If your Git repository becomes messy and you cannot repair it cleanly:

1. Delete the Git repository.
2. Undo the configuration changes you made.
3. Purge relevant software if needed:

   ```bash
   apt purge package-name
   ```

4. Manually delete files you created if they remain.
5. Start the lab again from the beginning.

Only do this if necessary. For small mistakes, a separate fix commit is usually better.

## 16. Quick checklist

Before every exercise:

- [ ] `git status` is clean.
- [ ] `.gitignore` allows the files you need to track.
- [ ] You know whether the next step is an install/update commit or a configuration commit.

After installing software:

- [ ] Check `git status`.
- [ ] Review changes with `git diff`.
- [ ] Commit only installation-created changes.
- [ ] Push to the private remote.

After changing configuration:

- [ ] Check `git diff`.
- [ ] Add only the relevant files.
- [ ] Commit with a descriptive message.
- [ ] Push to the private remote.

Before evaluation:

- [ ] Pick exactly five good commits.
- [ ] Use full 40-character commit IDs.
- [ ] Put them in `/root/commits1`, `/root/commits2`, or `/root/commits3`.
- [ ] Verify the file contains exactly five valid IDs.
- [ ] Verify the five IDs are unique.

## 17. Small theory section

Git is a version control system. It records snapshots of files over time, so you can see what changed, when it changed, and why it changed.

Important Git concepts:

- **Repository**: the folder tracked by Git. In this lab, the main repository is `/etc`.
- **Working tree**: the files you are currently editing.
- **Staging area**: the list of changes that will go into the next commit.
- **Commit**: a saved snapshot with an ID, author, date, and message.
- **Remote**: a repository stored somewhere else, such as GitHub.
- **Push**: sending your local commits to the remote repository.
- **`.gitignore`**: a file that tells Git which files to ignore.

Why this matters for system administration:

- You can prove what configuration changed.
- You can recover older versions of configuration files.
- You can explain your work through commit messages.
- You reduce the risk of losing important scripts or settings.
- You create a revision trail that can be evaluated later.

The most important habit in this lab is simple: check `git status`, make one logical change, commit it with a clear message, and push it.
