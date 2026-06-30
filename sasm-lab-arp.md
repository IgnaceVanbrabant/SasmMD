# SaSM Lab: ARP

## Goal of this lab

Protect your machine against IPv4 ARP spoofing by configuring a static ARP
entry for the default gateway. The entry must:

- point to the real MAC address of the default gateway;
- survive a reboot;
- still be applied after `eth0` goes down and comes back up;
- be re-applied when the interface is brought online again;
- allow the `check` user to run only these commands with sudo:
  - `ip link set dev eth0 up`
  - `ip link set dev eth0 down`

For file editing, this guide only uses `nano`.

## Important placeholders

In the commands below, replace these placeholders with your real values:

```text
GATEWAY_IP
GATEWAY_MAC
```

Example values:

```text
GATEWAY_IP=192.168.1.1
GATEWAY_MAC=aa:bb:cc:dd:ee:ff
```

Do not blindly copy the example IP or MAC address. Use the values from your own
machine.

## Step 1: Go to the git repository

Run:

```bash
cd /etc
```

What this command does:

- `cd` means "change directory".
- `/etc` is where Linux system configuration files are usually stored.
- In many SaSM labs, `/etc` is tracked with git so your configuration changes can
  be evaluated.

Check git:

```bash
sudo git status
```

What this command does:

- `sudo` runs the command with administrator rights.
- `git status` shows the current git branch and changed files.

What you should see:

```text
On branch ...
```

If it says there are already changed files, do not remove them. Just continue
carefully.

Commit your starting point if your lecturer expects a visible revision trail:

```bash
sudo git commit --allow-empty -m "Start ARP spoofing mitigation lab"
```

What this command does:

- `git commit` creates a revision in git.
- `--allow-empty` creates a commit even when no files changed yet.
- `-m` gives the commit a descriptive message.

What you should see:

```text
[branch-name abc1234] Start ARP spoofing mitigation lab
```

## Step 2: Find the default gateway IP address

Run:

```bash
ip route show default
```

What this command does:

- `ip route` shows network routes.
- `show default` only shows the default route, which is the route used for
  traffic outside your local network.

What you should see:

```text
default via 192.168.1.1 dev eth0
```

The IP address after `via` is your gateway IP.

In this example:

```text
GATEWAY_IP=192.168.1.1
```

Write down your real gateway IP.

Commit this investigation step:

```bash
sudo git commit --allow-empty -m "Identify default gateway for static ARP configuration"
```

## Step 3: Find the real gateway MAC address

First, send one ping to the gateway:

```bash
ping -c 1 GATEWAY_IP
```

Example:

```bash
ping -c 1 192.168.1.1
```

What this command does:

- `ping` tests whether a host is reachable.
- `-c 1` sends exactly one ping packet.
- This also makes your machine learn the gateway MAC address through ARP.

What you should see:

```text
1 packets transmitted, 1 received
```

Now inspect the neighbor table:

```bash
ip neigh show GATEWAY_IP dev eth0
```

Example:

```bash
ip neigh show 192.168.1.1 dev eth0
```

What this command does:

- `ip neigh` shows the ARP and neighbor table.
- `show GATEWAY_IP` filters for your gateway.
- `dev eth0` filters for the `eth0` interface.

What you should see:

```text
192.168.1.1 dev eth0 lladdr aa:bb:cc:dd:ee:ff REACHABLE
```

The value after `lladdr` is the MAC address.

In this example:

```text
GATEWAY_MAC=aa:bb:cc:dd:ee:ff
```

Important:

- Make sure this is the real MAC address of the gateway.
- If the attacker is already spoofing ARP, this learned MAC address could be
  fake.
- If your lecturer or lab documentation provides the gateway MAC address, use
  that trusted value.

Commit this investigation step:

```bash
sudo git commit --allow-empty -m "Record gateway MAC address for static ARP entry"
```

## Step 4: Create the static ARP script with nano

Create a small directory for this lab:

```bash
sudo mkdir -p /etc/sasm
```

What this command does:

- `mkdir` creates a directory.
- `-p` means it does not complain if the directory already exists.
- `/etc/sasm` keeps the lab script inside `/etc`, so it can be tracked by the
  lab git repository.

Open the script file:

```bash
sudo nano /etc/sasm/static-arp-gateway
```

What this command does:

- `sudo` opens the file as root.
- `nano` opens the nano text editor.
- `/etc/sasm/static-arp-gateway` is the script that will install the
  static ARP entry.

Paste this content into nano:

```sh
#!/bin/sh
set -eu

IFACE="eth0"
GATEWAY_IP="192.168.1.1"
GATEWAY_MAC="aa:bb:cc:dd:ee:ff"

ip neigh replace "$GATEWAY_IP" lladdr "$GATEWAY_MAC" dev "$IFACE" nud permanent
```

Replace:

```text
192.168.1.1
```

with your real gateway IP.

Replace:

```text
aa:bb:cc:dd:ee:ff
```

with your real gateway MAC address.

Save and exit nano:

```text
CTRL + O
Enter
CTRL + X
```

What the script does:

- `#!/bin/sh` tells Linux to run the script with `/bin/sh`.
- `set -eu` makes the script stop on errors and missing variables.
- `IFACE="eth0"` protects the `eth0` interface.
- `GATEWAY_IP` stores the gateway IP.
- `GATEWAY_MAC` stores the real gateway MAC.
- `ip neigh replace ... nud permanent` creates or replaces the ARP entry as a
  permanent static entry.

Make the script executable:

```bash
sudo chmod 755 /etc/sasm/static-arp-gateway
```

What this command does:

- `chmod` changes file permissions.
- `755` means root can read/write/execute, and others can read/execute.

What you should see:

```text
```

No output means the command succeeded.

Test the script:

```bash
sudo /etc/sasm/static-arp-gateway
```

What this command does:

- Runs the script immediately.
- Adds the static ARP entry now.

What you should see:

```text
```

No output means the script succeeded.

Check the ARP entry:

```bash
ip neigh show GATEWAY_IP dev eth0
```

Example:

```bash
ip neigh show 192.168.1.1 dev eth0
```

What you should see:

```text
192.168.1.1 dev eth0 lladdr aa:bb:cc:dd:ee:ff PERMANENT
```

The important word is:

```text
PERMANENT
```

Commit this step:

```bash
sudo git add sasm/static-arp-gateway
sudo git commit -m "Add script for permanent gateway ARP entry"
```

What these commands do:

- `git add` stages the new script.
- `git commit` saves the script in git with a descriptive message.

## Step 5: Create a systemd service with nano

Open the service file:

```bash
sudo nano /etc/systemd/system/sasm-static-arp-gateway.service
```

Paste this content:

```ini
[Unit]
Description=Install static ARP entry for the default gateway
After=network-pre.target
Wants=network-pre.target

[Service]
Type=oneshot
ExecStart=/etc/sasm/static-arp-gateway

[Install]
WantedBy=multi-user.target
```

Save and exit nano:

```text
CTRL + O
Enter
CTRL + X
```

What this file does:

- `[Unit]` describes when the service should run.
- `After=network-pre.target` runs it early in the networking process.
- `[Service]` says this is a one-time command.
- `ExecStart` runs your static ARP script.
- The service does not stay active forever, so udev can start it again later.
- `[Install]` allows the service to be enabled at boot.

Reload systemd:

```bash
sudo systemctl daemon-reload
```

What this command does:

- Tells systemd to re-read service files.

Enable the service:

```bash
sudo systemctl enable sasm-static-arp-gateway.service
```

What this command does:

- Makes the service run automatically after reboot.

What you should see:

```text
Created symlink ...
```

Start the service now:

```bash
sudo systemctl start sasm-static-arp-gateway.service
```

What this command does:

- Runs the service immediately without waiting for a reboot.

Check the service:

```bash
systemctl status sasm-static-arp-gateway.service
```

What you should see:

```text
Active: inactive (dead)
```

For a `Type=oneshot` service, `inactive (dead)` is normal after it successfully
finished. You should also see that the command exited successfully.

Press `q` to leave the status screen if a pager opens.

Check the ARP entry again:

```bash
ip neigh show GATEWAY_IP dev eth0
```

What you should see:

```text
GATEWAY_IP dev eth0 lladdr GATEWAY_MAC PERMANENT
```

Commit this step:

```bash
sudo git add /etc/systemd/system/sasm-static-arp-gateway.service
sudo git commit -m "Run static ARP configuration at boot"
```

## Step 6: Re-apply the ARP entry when eth0 changes state

A boot service alone is not enough for this assignment, because the static ARP
entry must also be applied when the interface is brought online again.

Create a udev rule:

```bash
sudo nano /etc/udev/rules.d/90-sasm-static-arp-gateway.rules
```

Paste this content:

```udev
SUBSYSTEM=="net", ACTION=="add|change", KERNEL=="eth0", TAG+="systemd", ENV{SYSTEMD_WANTS}+="sasm-static-arp-gateway.service"
```

Save and exit nano:

```text
CTRL + O
Enter
CTRL + X
```

What this rule does:

- `SUBSYSTEM=="net"` only matches network devices.
- `ACTION=="add|change"` reacts when the interface appears or changes state.
- `KERNEL=="eth0"` only applies to `eth0`.
- `TAG+="systemd"` allows udev to ask systemd to start something.
- `ENV{SYSTEMD_WANTS}+=...` starts your static ARP service again.

Reload udev rules:

```bash
sudo udevadm control --reload-rules
```

What this command does:

- Tells udev to re-read the rules in `/etc/udev/rules.d`.

Trigger the rule once:

```bash
sudo udevadm trigger --subsystem-match=net --action=change
```

What this command does:

- Simulates a network-device change event.
- This should cause the service to run again for `eth0`.

Check the ARP entry:

```bash
ip neigh show GATEWAY_IP dev eth0
```

What you should see:

```text
GATEWAY_IP dev eth0 lladdr GATEWAY_MAC PERMANENT
```

Commit this step:

```bash
sudo git add /etc/udev/rules.d/90-sasm-static-arp-gateway.rules
sudo git commit -m "Reapply static ARP entry when eth0 changes state"
```

## Step 7: Allow the check user to toggle eth0 only

Find the full path of the `ip` command:

```bash
command -v ip
```

What this command does:

- Shows the exact path of the `ip` program.

What you should usually see:

```text
/usr/sbin/ip
```

Open a sudoers drop-in safely with nano through `visudo`:

```bash
sudo EDITOR=nano visudo -f /etc/sudoers.d/sasm-arp-check
```

What this command does:

- `visudo` edits sudoers files safely.
- `-f /etc/sudoers.d/sasm-arp-check` edits a separate sudoers drop-in file.
- `EDITOR=nano` makes sure nano is used.

Add this exact rule if `command -v ip` showed `/usr/sbin/ip`:

```sudoers
check ALL=(root) NOPASSWD: /usr/sbin/ip link set dev eth0 up, /usr/sbin/ip link set dev eth0 down
```

If your `ip` path was different, replace `/usr/sbin/ip` with the path shown by
`command -v ip`.

Save and exit nano:

```text
CTRL + O
Enter
CTRL + X
```

Set safe permissions:

```bash
sudo chmod 440 /etc/sudoers.d/sasm-arp-check
```

What this command does:

- Makes the sudoers file readable by root and group only.
- Sudoers drop-in files should not be world-writable.

Validate the sudoers file:

```bash
sudo visudo -cf /etc/sudoers.d/sasm-arp-check
```

What this command does:

- Checks only this sudoers drop-in file for syntax errors.

What you should see:

```text
/etc/sudoers.d/sasm-arp-check: parsed OK
```

Validate all sudoers configuration:

```bash
sudo visudo -c
```

What you should see:

```text
/etc/sudoers: parsed OK
...
```

Check what `check` may run:

```bash
sudo -l -U check
```

What you should see:

```text
User check may run the following commands on ...
    (root) NOPASSWD: /usr/sbin/ip link set dev eth0 up, /usr/sbin/ip link set dev eth0 down
```

Important:

- The `check` user must not receive full access to `/usr/sbin/ip`.
- The rule must allow only the two exact `ip link set dev eth0 ...` commands.

Commit this step:

```bash
sudo git add /etc/sudoers.d/sasm-arp-check
sudo git commit -m "Allow check user to verify eth0 link recovery"
```

## Step 8: Test without disconnecting yourself accidentally

Warning:

- If you are connected over SSH through `eth0`, bringing `eth0` down can
  disconnect you.
- Use the server console if possible.

Check the ARP entry before the link test:

```bash
ip neigh show GATEWAY_IP dev eth0
```

What you should see:

```text
GATEWAY_IP dev eth0 lladdr GATEWAY_MAC PERMANENT
```

Bring the interface down:

```bash
sudo ip link set dev eth0 down
```

What this command does:

- Disables the `eth0` network link.

What you should see:

```text
```

No output usually means success.

Bring the interface up:

```bash
sudo ip link set dev eth0 up
```

What this command does:

- Enables the `eth0` network link again.

What you should see:

```text
```

No output usually means success.

Give udev and systemd a moment to re-run the service:

```bash
sleep 2
```

What this command does:

- Waits two seconds before checking the ARP entry again.

Check the ARP entry again:

```bash
ip neigh show GATEWAY_IP dev eth0
```

What you should see:

```text
GATEWAY_IP dev eth0 lladdr GATEWAY_MAC PERMANENT
```

Commit the successful validation:

```bash
sudo git commit --allow-empty -m "Validate static ARP entry after eth0 link cycle"
```

## Step 9: Test reboot persistence

Reboot:

```bash
sudo reboot
```

What this command does:

- Restarts the machine.

After the machine comes back, log in again and run:

```bash
ip neigh show GATEWAY_IP dev eth0
```

What you should see:

```text
GATEWAY_IP dev eth0 lladdr GATEWAY_MAC PERMANENT
```

Check the service:

```bash
systemctl status sasm-static-arp-gateway.service
```

What you should see:

```text
Active: inactive (dead)
```

For this oneshot service, `inactive (dead)` is normal after the command has
finished successfully.

Press `q` to exit if a pager opens.

Commit the reboot validation:

```bash
cd /etc
sudo git commit --allow-empty -m "Validate static ARP entry after reboot"
```

## Step 10: Final git check

Run:

```bash
cd /etc
sudo git log --oneline -10
```

What this command does:

- Shows the last ten commits.

What you should see:

```text
abc1234 Validate static ARP entry after reboot
def5678 Validate static ARP entry after eth0 link cycle
...
```

Run:

```bash
sudo git status
```

What you should see:

```text
nothing to commit, working tree clean
```

If the output says there are uncommitted changes, inspect them with:

```bash
sudo git status
```

Then add and commit only the files that belong to this ARP lab.

## Quick troubleshooting

### The ARP entry does not show `PERMANENT`

Run:

```bash
sudo /etc/sasm/static-arp-gateway
ip neigh show GATEWAY_IP dev eth0
```

If it still does not show `PERMANENT`, reopen the script:

```bash
sudo nano /etc/sasm/static-arp-gateway
```

Check for typing mistakes in:

```text
GATEWAY_IP
GATEWAY_MAC
eth0
```

### The service does not start

Run:

```bash
sudo systemctl status sasm-static-arp-gateway.service
```

What to check:

- The script path must be `/etc/sasm/static-arp-gateway`.
- The script must be executable.
- The gateway IP and MAC must be valid.

Open the service file with nano if needed:

```bash
sudo nano /etc/systemd/system/sasm-static-arp-gateway.service
```

After fixing it, run:

```bash
sudo systemctl daemon-reload
sudo systemctl restart sasm-static-arp-gateway.service
```

### The sudoers rule fails validation

Reopen it safely:

```bash
sudo EDITOR=nano visudo -f /etc/sudoers.d/sasm-arp-check
```

Make sure the rule is one single line:

```sudoers
check ALL=(root) NOPASSWD: /usr/sbin/ip link set dev eth0 up, /usr/sbin/ip link set dev eth0 down
```

Then validate again:

```bash
sudo visudo -cf /etc/sudoers.d/sasm-arp-check
```

## Final expected result

At the end of the lab:

- `ip neigh show GATEWAY_IP dev eth0` shows `PERMANENT`.
- `sasm-static-arp-gateway.service` is enabled.
- The udev rule re-runs the service when `eth0` changes state.
- The `check` user can run only the two required link commands with sudo.
- Git contains a clear revision trail with descriptive commit messages.
