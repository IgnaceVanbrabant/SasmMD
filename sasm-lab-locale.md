# SaSM Lab: Locale

## Goal of this lab

In this lab you configure the server locale so commands no longer warn about
missing locale settings. By the end, you should be able to:

- Check the current locale configuration.
- Generate the required `en_US.UTF-8` and `nl_BE.UTF-8` locales.
- Configure `/etc/default/locale` with the required settings.
- Verify that `perl -e exit` no longer shows locale warnings.
- Keep a clear git revision trail with descriptive commits.

## Small theory section

### What is a locale?

A locale tells Linux which language, country, character encoding, and formatting
rules to use. Examples:

- `en_US.UTF-8`: English language, United States conventions, UTF-8 encoding.
- `nl_BE.UTF-8`: Dutch language, Belgium conventions, UTF-8 encoding.

### Why the warning happens

Programs such as Perl read locale environment variables like `LANG`,
`LANGUAGE`, and `LC_TIME`. If one of those variables points to a locale that is
not generated on the server, Perl prints a warning like:

```text
perl: warning: Setting locale failed.
```

The fix is to make sure the locale exists and then configure the system to use
the correct values.

### Important locale variables

- `LANG`: the default locale for the system.
- `LANGUAGE`: the preferred message language.
- `LC_TIME`: date and time formatting.
- `LC_MONETARY`: money formatting.
- `LC_NUMERIC`: number formatting.
- `LC_ALL`: overrides all other locale variables when set. In this lab it should
  stay empty.

## Step-by-step configuration

The commands below assume Ubuntu and a user with `sudo` rights. File edits are
done with `nano`, as requested.

## Step 1: Go to the git repository and check the starting state

If your lab repository is in `/etc`, go there first:

```bash
cd /etc
git status
```

What you should see:

- Git should show the current branch.
- Ideally it should say `working tree clean` before you start.
- If files are already modified, read them carefully before continuing.

Record the start of the lab in your progress notes:

```bash
nano locale-lab-progress.md
```

Add something like:

```markdown
# Locale lab progress

Started by checking the current git status before changing locale settings.
```

Save and exit nano:

1. Press `Ctrl+O`.
2. Press `Enter`.
3. Press `Ctrl+X`.

Commit this first step:

```bash
git add locale-lab-progress.md
git commit -m "Start locale lab progress notes"
```

What you should see:

- Git creates a commit.
- `git status` should be clean again after the commit.

## Step 2: Recreate and inspect the locale problem

Check the current locale values:

```bash
locale
```

Then recreate the warning from the assignment:

```bash
perl -e exit
```

What you may see before fixing the server:

```text
perl: warning: Setting locale failed.
perl: warning: Please check that your locale settings:
```

Also check which locales are currently generated:

```bash
locale -a
```

What you are looking for:

- `en_US.utf8` should exist.
- `nl_BE.utf8` should exist.

If one or both are missing, continue with the next step.

Update your progress notes:

```bash
nano locale-lab-progress.md
```

Add what you found, for example:

```markdown
Checked the current locale output and confirmed which locales were generated.
```

Commit this step:

```bash
git add locale-lab-progress.md
git commit -m "Record initial locale state"
```

## Step 3: Enable the required locales

Open the locale generation file:

```bash
sudo nano /etc/locale.gen
```

Find these lines:

```text
# en_US.UTF-8 UTF-8
# nl_BE.UTF-8 UTF-8
```

Remove the `#` at the start of both lines so they become:

```text
en_US.UTF-8 UTF-8
nl_BE.UTF-8 UTF-8
```

Save and exit nano:

1. Press `Ctrl+O`.
2. Press `Enter`.
3. Press `Ctrl+X`.

Generate the enabled locales:

```bash
sudo locale-gen
```

What you should see:

```text
Generating locales
  en_US.UTF-8... done
  nl_BE.UTF-8... done
Generation complete.
```

Check that both locales now exist:

```bash
locale -a
```

Expected relevant lines:

```text
en_US.utf8
nl_BE.utf8
```

Commit this configuration change:

```bash
git status
git add locale.gen
git commit -m "Enable required UTF-8 locales"
```

## Step 4: Configure the system locale values

Open the system locale file:

```bash
sudo nano /etc/default/locale
```

Set the file to these values:

```text
LANG=en_US.UTF-8
LANGUAGE=en_US
LC_NUMERIC=nl_BE.UTF-8
LC_TIME=nl_BE.UTF-8
LC_MONETARY=nl_BE.UTF-8
LC_PAPER=nl_BE.UTF-8
LC_NAME=nl_BE.UTF-8
LC_ADDRESS=nl_BE.UTF-8
LC_TELEPHONE=nl_BE.UTF-8
LC_MEASUREMENT=nl_BE.UTF-8
LC_IDENTIFICATION=nl_BE.UTF-8
```

Do not add `LC_ALL`. The assignment example shows `LC_ALL=` empty, so leaving it
unset is correct.

Save and exit nano:

1. Press `Ctrl+O`.
2. Press `Enter`.
3. Press `Ctrl+X`.

Commit this configuration change:

```bash
git status
git add default/locale
git commit -m "Configure system locale defaults"
```

## Step 5: Load the new locale settings

Log out and log back in, or start a new login shell:

```bash
su - $USER
```

If you are working as root, use:

```bash
su -
```

What this does:

- It starts a new login shell.
- The new shell reads the updated locale defaults.

## Step 6: Verify the final locale output

Run:

```bash
locale
```

Expected result:

```text
LANG=en_US.UTF-8
LANGUAGE=en_US
LC_CTYPE="en_US.UTF-8"
LC_NUMERIC=nl_BE.UTF-8
LC_TIME=nl_BE.UTF-8
LC_COLLATE="en_US.UTF-8"
LC_MONETARY=nl_BE.UTF-8
LC_MESSAGES="en_US.UTF-8"
LC_PAPER=nl_BE.UTF-8
LC_NAME=nl_BE.UTF-8
LC_ADDRESS=nl_BE.UTF-8
LC_TELEPHONE=nl_BE.UTF-8
LC_MEASUREMENT=nl_BE.UTF-8
LC_IDENTIFICATION=nl_BE.UTF-8
LC_ALL=
```

Small differences in quotes are normal. The important part is that the values
match.

Now run the Perl test again:

```bash
perl -e exit
```

Expected result:

- No output.
- No locale warning.
- You simply get a new shell prompt.

Update your progress notes:

```bash
nano locale-lab-progress.md
```

Add the verification result, for example:

```markdown
Verified the final locale output and confirmed that perl -e exit shows no
locale warning.
```

Commit the verification notes:

```bash
git add locale-lab-progress.md
git commit -m "Verify locale configuration"
```

## Step 7: Check the revision trail

Show the commits you made for this lab:

```bash
git log --oneline
```

What you should see:

- A commit for starting the locale lab notes.
- A commit for recording the initial locale state.
- A commit for enabling the required locales.
- A commit for configuring the system locale defaults.
- A commit for verifying the final locale configuration.

Final check:

```bash
git status
```

Expected result:

```text
nothing to commit, working tree clean
```

## Troubleshooting

### `locale -a` does not show `nl_BE.utf8`

Open `/etc/locale.gen` again:

```bash
sudo nano /etc/locale.gen
```

Make sure this line is present and not commented:

```text
nl_BE.UTF-8 UTF-8
```

Save the file and run:

```bash
sudo locale-gen
```

### `perl -e exit` still shows a warning

Check for spelling mistakes:

```bash
locale
locale -a
```

Then reopen the locale file:

```bash
sudo nano /etc/default/locale
```

Make sure every configured locale exists in `locale -a`. After saving, log out
and log back in before testing again.

## What to remember

- Generate a locale before using it.
- Use `LANG=en_US.UTF-8` for the default language in this lab.
- Use the `nl_BE.UTF-8` `LC_*` values requested by the assignment.
- Keep `LC_ALL` empty.
- Commit after each meaningful step with a descriptive message.
