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
