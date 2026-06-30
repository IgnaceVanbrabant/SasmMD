# SaSM Lab: SSH

## Goal of this lab

In this lab you configure and secure an SSH server on Ubuntu 24.04. By the end,
you should be able to:

- Use the SSH client for normal logins and basic troubleshooting.
- Configure the SSH daemon to listen on TCP port `22345`.
- Disable SSH reverse DNS lookups.
- Add the required `check` user with the provided public key.
- Keep lecturer root access intact.
- Detect and mitigate SSH brute-force attacks with `sshguard`.
- Configure and explain the common SSH tunnel types.
- Keep a clear git revision trail with descriptive commits.

> Important: This lab is a prerequisite for later exercises. Do not remove
> existing lecturer root access while completing it.

## Basic theory you need to know

### SSH client and server

SSH has two main sides:

- **SSH client**: the command you run from your laptop, usually `ssh`.
- **SSH server / daemon**: the service running on the remote machine, usually
  `sshd`.

The client connects to the server over TCP. The default port is `22`, but this
lab requires the server to listen on port `22345`.

Basic client command:

```bash
ssh username@server.example
```

With a custom port:

```bash
ssh -p 22345 username@server.example
```

### SSH host keys

The SSH server has host keys that identify the server. On first connection, your
client stores the host key in:

```text
~/.ssh/known_hosts
```

If the host key changes unexpectedly, SSH warns you because it could indicate a
reinstalled server or a man-in-the-middle attack.

### SSH user keys

Users can authenticate with public/private key pairs:

- The **private key** stays on the client and must remain secret.
- The **public key** is placed on the server in the user's
  `~/.ssh/authorized_keys` file.

Correct permissions matter:

```text
~/.ssh                 700
~/.ssh/authorized_keys 600
```

### SSH daemon configuration

The main OpenSSH server configuration file is:

```text
/etc/ssh/sshd_config
```

On modern Ubuntu systems, additional configuration can also be placed in:

```text
/etc/ssh/sshd_config.d/*.conf
```

Always validate the configuration before restarting SSH:

```bash
sudo sshd -t
```

### Ubuntu 24.04 socket activation

Ubuntu 24.04 uses systemd socket activation for SSH by default. This means
`ssh.socket` may listen for incoming connections and start `ssh.service` when
needed.

Because of this, changing `Port` in `sshd_config` is not always enough if you
only restart `ssh.service`. In this guide we use the Ubuntu 24.04-friendly
method: keep socket activation and reload systemd so the SSH socket generator
uses the configured SSH port.

### Reverse DNS lookups

Reverse DNS lookup means SSH tries to resolve the connecting IP address back to
a hostname. This can slow logins and is not required for this lab.

Disable it with:

```text
UseDNS no
```

### Brute-force protection

A brute-force attack repeatedly tries usernames and passwords. Even if password
authentication is not your main login method, the server should detect repeated
failed attempts and temporarily block the attacking IP address.

This lab uses `sshguard` to block IPs that try more than five SSH username and
password combinations in two minutes. The block should last five minutes.

### SSH tunnels

SSH tunnels forward network traffic through an encrypted SSH connection.

Common types:

- **Local tunnel (`-L`)**: access a remote service through a local port.
- **Remote tunnel (`-R`)**: expose a local service through a port on the remote
  SSH server.
- **Dynamic tunnel (`-D`)**: create a SOCKS proxy through SSH.

## Step-by-step configuration

The commands below assume Ubuntu 24.04 and a user with `sudo` rights.

Keep your current SSH session open while changing SSH settings. Open a second
terminal for testing the new connection before closing the original session.

## Step 1: Start a git revision trail

The assignment requires progress to be documented in git after every meaningful
step. If your lab notes are stored in a repository, update them and commit after
each section.

Example:

```bash
git status
printf '%s\n' '# SSH lab progress' > ssh-lab-progress.md
git add ssh-lab-progress.md
git commit -m "Start SSH lab progress notes"
```

Do not commit passwords, private keys, or other secrets.

## Step 2: Inspect the current SSH state

Check the current SSH service and socket:

```bash
sudo systemctl status ssh
sudo systemctl status ssh.socket
```

Check which ports are listening:

```bash
sudo ss -tlnp | grep ssh
```

Check the effective SSH daemon configuration:

```bash
sudo sshd -T | less
```

Create a backup before making changes:

```bash
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak.$(date +%F-%H%M)
```

Commit your notes:

```bash
git add ssh-lab-progress.md
git commit -m "Record initial SSH service state"
```

## Step 3: Preserve lecturer root access

The student servers are configured so lecturers can log in as root. Before
editing SSH keys, verify the existing root authorized keys:

```bash
sudo ls -l /root/.ssh
sudo less /root/.ssh/authorized_keys
```

These lecturer keys must remain present:

```text
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAINU1W8ZTn6J5ywfb+CJpr5BPivbFbg45mwQEFefpJ9E5 pieter@cipki
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAINBAvHyUh/yza5LOzvOmVEo6IoNlVEo7vptZc5u03hwB swenr@zbook
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAICYiZXupz62zX493Y5EfJalzaU5SjgUl69K13olMzixw hans@late7420
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQDZfJMfDvHkb/KMkhHk5PI3VEpzj75WZGPOL2GqCwgwtBdztl2oLuuD7S9MykzsG4J5z2D645ZoqgNbTlly2IEN796PvtrH5OlmQNrotq+wcrQNNMBmRHvgkLCgTgOqVz5s85498Kyw04m1VFT0vTM8YRxVw0MquzJzZfvIkngELYYGCoJTD4pSS4/mtYAOmGtv3yve/kcysn668GzSUmXE5mM0ohWfbMPUqLIYHsMNaqRqX567XmWI6yt1fo3vgSHCEXomE4XkJvjjDrpDKIrE/WAP4AvTS19tuhhC/g6yeKIrm01xlrvjXvrisydLHx7MpWsygoCII+2Je2TPwLjQHVwC7rTA4rzeaM3iJOs9s0FVGj/fl/rDqTKqNmidJbC20R+TvCy1JzsmBxfGiU2Xq06S4MGyaqec3zPG44qhqd3LMALOAWlV0sDrF/C65sSuEmJEPg764P0/nwpKzqp4kQaTw5XvMse0tHsrQhSqfiYFNCuklhV/3G4tCxjxuqJ+VTd/6FJFXFczrNxieYFEzFbjLPAiDXMGQntg5aOV5mTHxPUYkSLrUM1Ign3t65kkLxYqAGMD27cbc/h9N11dmF+YA5O0xLMigFePxVR913efJRWF9b0x6+JTgV1e8XHv/T2auQvohivdmI9Ex3o4qPId98qCYBYj9KTL+6xd4w== dietrichd@VITRIOL.local
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAID/4c8GIeZhHp+06rmukfpO+f7/alpG6+VVcFPNv+aF5 ucll\u0162359@LT2250486
```

If you accidentally remove one, restore it immediately.

Commit your notes:

```bash
git add ssh-lab-progress.md
git commit -m "Verify lecturer root SSH keys"
```

## Step 4: Create the `check` user

Create the user:

```bash
sudo adduser --gecos "" check
```

Set a strong password when prompted. The assignment says password
authentication must not be completely disabled for user `check`, because the
brute-force test needs to attempt password logins.

Create the SSH directory:

```bash
sudo install -d -m 700 -o check -g check /home/check/.ssh
```

Install the required public key:

```bash
printf '%s\n' 'ssh-rsa AAAAB3NzaC1yc2EAAAABIwAAAQEAw2YPreIBDz/BbRF8ftteme4wyV8T6aNc9TLNY4Xk7K2ta9pPWux7g5fnwnMv/WVBMLbYPh3ECX8G95OeUDGk5UgefjZiBqyAqUFmekzQnOcfhy6aiSc1xe8r0dEMF10Fj3Duvy18Vc0yPQMQCQPkBgr/7n4dxfBdXJsp/GF2p4bLVzKSNoRho0msZEaX/QuCcOgntRzLBtr7+HpVxoCOsTQ9njeC4FBEmKx4soxQG7u4EZI2ZZAVRGYXVANiYodXjgGwQTMTO2pKJzu8s0SK6JcRouGMVdPORf9VoFq2V8YjbhAtrrbkGCemJtXltsxiUe7w5V+8GGGQvigCOo6Gmw== Patience-you-must-have-my-young-Padawan@yoda' | sudo tee /home/check/.ssh/authorized_keys >/dev/null
sudo chown check:check /home/check/.ssh/authorized_keys
sudo chmod 600 /home/check/.ssh/authorized_keys
```

Verify:

```bash
sudo ls -ld /home/check/.ssh
sudo ls -l /home/check/.ssh/authorized_keys
sudo passwd -S check
```

Commit your notes:

```bash
git add ssh-lab-progress.md
git commit -m "Create check user for SSH lab"
```

## Step 5: Configure the SSH daemon

Create a lab-specific SSH config file:

```bash
sudo nano /etc/ssh/sshd_config.d/10-sasm-lab.conf
add this text:
Port 22345
UseDNS no
PubkeyAuthentication yes
PasswordAuthentication yes
```

Notes:

- `Port 22345` makes SSH listen on the required lab port.
- `UseDNS no` disables reverse DNS lookups.
- `PasswordAuthentication yes` keeps password attempts possible for the
  brute-force detection test.
- You may keep port `22` open locally if you add another `Port 22` line, but
  the assignment states that TCP port `22` is not accessible from the internet.

Validate the SSH configuration:

```bash
sudo sshd -t
```

If the command prints nothing and exits successfully, the syntax is valid.

Commit your notes:

```bash
git add ssh-lab-progress.md
git commit -m "Configure SSH daemon for lab requirements"
```

## Step 6: Apply the SSH port change on Ubuntu 24.04

Use one method only. This guide keeps socket activation enabled and lets the
systemd SSH socket generator read the SSH daemon config.

Reload systemd and restart the SSH socket:

```bash
sudo systemctl daemon-reload
sudo systemctl restart ssh.socket
sudo systemctl restart ssh
```

Verify the generated socket configuration:

```bash
sudo systemctl cat ssh.socket
```

You should see a generated section containing port `22345`, similar to:

```text
ListenStream=0.0.0.0:22345
ListenStream=[::]:22345
```

Verify the listening port:

```bash
sudo ss -tlnp | grep 22345
```

If `ufw` is active, allow the new SSH port:

```bash
sudo ufw status
sudo ufw allow 22345/tcp
sudo ufw reload
```

Commit your notes:

```bash
git add ssh-lab-progress.md
git commit -m "Activate SSH service on port 22345"
```

## Step 7: Test SSH login

From your laptop or another machine, test the new port:

```bash
ssh -p 22345 check@your-server-name-or-ip
```

If login fails, inspect logs on the server:

```bash
sudo tail -f /var/log/auth.log
```

Or with systemd:

```bash
sudo journalctl -fu ssh
```

Useful debug command from the client:

```bash
ssh -vvv -p 22345 check@your-server-name-or-ip
```

Commit your notes:

```bash
git add ssh-lab-progress.md
git commit -m "Test SSH login for check user"
```

## Step 8: Install sshguard

Install the package:

```bash
sudo apt update
sudo apt install -y sshguard
```

Check the service:

```bash
sudo systemctl status sshguard
```

Find the active sshguard configuration file:

```bash
sudo ls -l /etc/sshguard
sudo grep -R "THRESHOLD\|BLOCK_TIME\|DETECTION_TIME\|WHITELIST" /etc/sshguard
```

Commit your notes:

```bash
git add ssh-lab-progress.md
git commit -m "Install sshguard for SSH brute-force protection"
```

## Step 9: Whitelist trusted IP addresses

The assignment requires these addresses in the whitelist:

- Your laptop IP on eduroam.
- Your laptop IP at home.
- The IPv4 address of `yoda.uclllabs.be`.
- The IPv6 address of `yoda.uclllabs.be`.

Do not put hostnames in the whitelist. Resolve them first.

Find your public IPv4 and IPv6 addresses:

```bash
curl -4 https://ifconfig.me
curl -6 https://ifconfig.me
```

Resolve yoda's addresses:

```bash
dig +short A yoda.uclllabs.be
dig +short AAAA yoda.uclllabs.be
```

If `dig` is not installed:

```bash
sudo apt install -y dnsutils
```

Create or edit the whitelist:

```bash
sudoedit /etc/sshguard/whitelist
```

Example format:

```text
# Laptop on eduroam
203.0.113.10
2001:db8:10::10

# Laptop at home
198.51.100.25
2001:db8:20::25

# yoda.uclllabs.be, resolved manually with dig
203.0.113.50
2001:db8:50::50
```

Do not whitelist `leia`, because that machine is used for the brute-force test.

Commit your notes:

```bash
git add ssh-lab-progress.md
git commit -m "Configure sshguard whitelist"
```

## Step 10: Configure sshguard blocking behavior

Edit the sshguard configuration:

```bash
sudoedit /etc/sshguard/sshguard.conf
```

Set or adjust these values:

```text
THRESHOLD=60
BLOCK_TIME=300
DETECTION_TIME=120
WHITELIST_FILE=/etc/sshguard/whitelist
```

Meaning:

- `THRESHOLD=60`: blocks after roughly six failed SSH authentication events.
  sshguard scores attacks internally; on most installations one failed SSH login
  counts as about 10 points.
- `BLOCK_TIME=300`: blocks the attacker for 300 seconds, or five minutes.
- `DETECTION_TIME=120`: counts failures within a 120 second, or two minute,
  window.
- `WHITELIST_FILE`: points to the trusted IP list.

Use the backend already selected by your package unless your lecturer instructs
otherwise. On Ubuntu 24.04 this is commonly an nftables-based backend.

Confirm sshguard is reading SSH authentication logs:

```bash
sudo systemctl cat sshguard
```

The Ubuntu package commonly reads from the systemd journal. That is acceptable
for this lab. If your configuration exposes a log reader or file option, make
sure it reads either the systemd journal or `/var/log/auth.log`.

Restart sshguard:

```bash
sudo systemctl restart sshguard
sudo systemctl status sshguard
```

Check logs:

```bash
sudo journalctl -u sshguard -n 100 --no-pager
```

Commit your notes:

```bash
git add ssh-lab-progress.md
git commit -m "Tune sshguard brute-force blocking rules"
```

## Step 11: Verify brute-force mitigation

Watch SSH and sshguard logs:

```bash
sudo journalctl -fu ssh
```

In another terminal:

```bash
sudo journalctl -fu sshguard
```

Ask the test system to perform the brute-force test, or perform a controlled
test from a non-whitelisted machine. After more than five failed attempts in two
minutes, sshguard should block the source IP for about five minutes.

Check firewall state. The exact command depends on the backend:

```bash
sudo nft list ruleset
```

or:

```bash
sudo iptables -S
```

Commit your notes:

```bash
git add ssh-lab-progress.md
git commit -m "Verify sshguard blocks SSH brute-force attempts"
```

## Step 12: Configure useful SSH client shortcuts

On your laptop, edit:

```bash
nano ~/.ssh/config
```

Example:

```sshconfig
Host sasm-server
    HostName your-server-name-or-ip
    User check
    Port 22345
    IdentityFile ~/.ssh/id_ed25519
    IdentitiesOnly yes
```

Then connect with:

```bash
ssh sasm-server
```

## Step 13: Local SSH tunnel

Use a local tunnel when you want to access a service on the server through a
local port on your laptop.

Example: forward local port `8080` to the server's local port `80`:

```bash
ssh -N -L 127.0.0.1:8080:127.0.0.1:80 -p 22345 your-user@your-server-name-or-ip
```

Then open this on your laptop:

```text
http://127.0.0.1:8080
```

Explanation:

```text
127.0.0.1:8080       local listening address and port
127.0.0.1:80         destination from the server's point of view
-N                   do not start a remote shell
```

## Step 14: Remote SSH tunnel

Use a remote tunnel when you want the server to expose a port that forwards back
to your laptop.

Example: expose your laptop's local port `3000` on the server's loopback port
`9090`:

```bash
ssh -N -R 127.0.0.1:9090:127.0.0.1:3000 -p 22345 your-user@your-server-name-or-ip
```

On the server, test:

```bash
curl http://127.0.0.1:9090
```

By default this binds to the server's loopback interface only. Exposing it to
other machines requires extra SSH server settings such as `GatewayPorts`, and
should only be done when required.

## Step 15: Dynamic SSH tunnel

Use a dynamic tunnel to create a SOCKS proxy.

Example:

```bash
ssh -N -D 127.0.0.1:1080 -p 22345 your-user@your-server-name-or-ip
```

Configure your browser or tool to use SOCKS5 proxy:

```text
Host: 127.0.0.1
Port: 1080
```

This sends selected application traffic through the SSH connection.

## Step 16: Final validation checklist

Before submitting, verify:

- [ ] SSH listens on TCP port `22345`.
- [ ] You can log in as `check` with the provided public key.
- [ ] `UseDNS no` is active.
- [ ] Lecturer root keys are still present.
- [ ] Password authentication is not completely disabled for user `check`.
- [ ] `sshguard` is installed and running.
- [ ] sshguard blocks more than five failed SSH attempts within two minutes.
- [ ] sshguard blocks attackers for five minutes.
- [ ] Your laptop IPs and yoda's IPv4 and IPv6 addresses are whitelisted.
- [ ] `leia` is not whitelisted.
- [ ] You can explain local, remote, and dynamic SSH tunnels.
- [ ] Your git history contains descriptive commits for each meaningful step.

Final git check:

```bash
git status
git log --oneline --decorate --max-count=10
```

## Troubleshooting

### SSH does not start

Validate syntax:

```bash
sudo sshd -t
```

Read logs:

```bash
sudo journalctl -u ssh -n 100 --no-pager
```

### SSH still listens on port 22

Check whether socket activation is active:

```bash
sudo systemctl status ssh.socket
sudo systemctl cat ssh.socket
```

Reload systemd and restart the socket:

```bash
sudo systemctl daemon-reload
sudo systemctl restart ssh.socket
```

### Public key login fails

Check ownership and permissions:

```bash
sudo ls -ld /home/check /home/check/.ssh
sudo ls -l /home/check/.ssh/authorized_keys
```

Expected:

```text
/home/check/.ssh owned by check:check with mode 700
authorized_keys owned by check:check with mode 600
```

Check logs:

```bash
sudo journalctl -fu ssh
```

### You locked yourself out

Use the still-open SSH session or the VM console. Restore the backup:

```bash
sudo cp /etc/ssh/sshd_config.bak.YYYY-MM-DD-HHMM /etc/ssh/sshd_config
sudo rm -f /etc/ssh/sshd_config.d/10-sasm-lab.conf
sudo sshd -t
sudo systemctl daemon-reload
sudo systemctl restart ssh.socket
sudo systemctl restart ssh
```

Then carefully re-apply the lab configuration.
