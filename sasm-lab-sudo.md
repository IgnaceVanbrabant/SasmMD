# SaSM Lab: sudo

## Goal of this lab

Configure `sudo` so the `check` user can run only the required monitoring
commands as root without entering a password.

Required packages:

- `sudo`
- `monitoring-plugins-basic`, which provides
  `/usr/lib/nagios/plugins/check_apt`
- `net-tools`, which provides `/usr/sbin/arp`

Required sudo permissions for user `check`:

- `/usr/lib/nagios/plugins/check_apt`
- `/usr/sbin/arp`

No other commands should be granted to `check` through sudo for this lab.

## Progress log

### Step 1: Start from a clean lab branch

- Created branch `cursor/sudo-lab-6b86`.
- Confirmed the repository was on `main` before branching.
- Confirmed `sudo` and `apt-get` commands are available on the machine.

### Step 2: Install required packages

Ran:

```bash
sudo apt-get update
sudo DEBIAN_FRONTEND=noninteractive apt-get install -y sudo monitoring-plugins-basic net-tools
```

Result:

- `sudo` was already installed.
- `net-tools` was already installed.
- `monitoring-plugins-basic` was installed successfully.
- `/usr/lib/nagios/plugins/check_apt` exists and is executable.
- `/usr/sbin/arp` exists and is executable.

### Step 3: Configure sudo for the `check` user

The `check` user did not exist on this machine, so it was created without sudo
group membership.

Do not type the sudoers rule directly at the shell prompt. This is not a shell
command:

```sudoers
check ALL=(root) NOPASSWD: /usr/lib/nagios/plugins/check_apt, /usr/sbin/arp
```

Put that rule in `/etc/sudoers.d/sasm-lab-check` with:

```bash
printf '%s\n' 'check ALL=(root) NOPASSWD: /usr/lib/nagios/plugins/check_apt, /usr/sbin/arp' | sudo tee /etc/sudoers.d/sasm-lab-check >/dev/null
sudo chmod 0440 /etc/sudoers.d/sasm-lab-check
```

The `printf ... | sudo tee ...` command creates or replaces the sudoers drop-in
file. The file must be set to mode `0440`.

Validation:

```bash
sudo visudo -cf /etc/sudoers.d/sasm-lab-check
sudo visudo -c
```

Both commands reported the sudoers configuration parsed successfully.

### Step 4: Validate sudo behavior as `check`

Checked the effective sudo permissions:

```bash
sudo -u check sudo -n -l
```

Result:

```text
User check may run the following commands on cursor:
    (root) NOPASSWD: /usr/lib/nagios/plugins/check_apt, /usr/sbin/arp
```

Allowed command tests:

```bash
sudo -u check sudo -n /usr/lib/nagios/plugins/check_apt
sudo -u check sudo -n /usr/sbin/arp
sudo -u check sudo -n /usr/sbin/arp -n
```

Results:

- `check_apt` executed without a password and reported pending package updates.
  It returned Nagios status `2` because updates are available, not because sudo
  failed.
- `/usr/sbin/arp` executed without a password.
- `/usr/sbin/arp -n` executed without a password.

Denied command test:

```bash
sudo -u check sudo -n ip neighbor show
```

Result:

```text
sudo: a password is required
```

This confirms `check` does not have passwordless sudo for unrelated commands.
