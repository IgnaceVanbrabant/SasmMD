# SaSM Lab: NTP

## Goal of this lab

In this lab you configure an NTP server with `ntpd`. By the end, you should be
able to:

- Explain why NTP is used.
- Set the correct timezone.
- Synchronize your server with `be.pool.ntp.org`.
- Allow NTP time queries only from `yoda.uclllabs.be` and
  `matteo.knops.sasm.uclllabs.be`.
- Block all other IPv4 and IPv6 clients from querying your NTP server.
- Verify synchronization and access control.
- Keep a clear git revision trail with useful commit messages.

## Small theory section

### What is NTP?

NTP means **Network Time Protocol**. It keeps clocks synchronized between
computers over the network.

That matters because many systems depend on accurate time:

- Kerberos and other authentication systems reject requests when clocks differ
  too much.
- Logs are easier to compare during troubleshooting or forensic investigation.
- Firewalls, monitoring tools, and intrusion detection systems need correct
  timestamps.
- Scheduled jobs and certificates can break when the system clock is wrong.

### NTP client and server roles

Your machine has two roles in this lab:

- **NTP client**: it asks internet time servers for the correct time.
- **NTP server**: it gives time to a small list of allowed machines.

For the upstream internet time source, this lab uses:

```text
be.pool.ntp.org
```

### Why access control is needed

An open NTP server can be abused from the internet. Old or badly configured NTP
servers have been used in amplification attacks. That is why this lab requires
you to drop all normal client queries except from the explicitly allowed hosts.

The important rule is:

> Use IP addresses in the NTP access rules. Do not put hostnames in `restrict`
> lines.

You may use DNS tools once to discover the IP addresses, but the final
configuration must contain only IP addresses.

## Step-by-step configuration

The commands below assume Ubuntu and a user with `sudo` rights. The assignment
uses `/etc/ntp.conf` and `systemctl restart ntp`, so this guide uses the classic
`ntp` service name.

Replace these placeholders before editing the config:

- `YODA_IPV4`
- `YODA_IPV6`
- `MATTEO_IPV4`
- `MATTEO_IPV6`

The second allowed server for this guide is:

```text
matteo.knops.sasm.uclllabs.be
```

Resolve its IPv4 and IPv6 addresses on your lab machine and use those IP
addresses in the config.

## Step 1: Go to `/etc` and check git status

The SaSM labs usually track system configuration from `/etc`.

```bash
cd /etc
git status
```

What you should see:

- Git shows the current branch.
- Ideally it says `working tree clean`.
- If files are already modified, do not overwrite them blindly.

Create a small progress file:

```bash
nano ntp-lab-progress.md
```

Add:

```markdown
# NTP lab progress

Started the NTP lab by checking git status before changing configuration.
```

Commit this first step:

```bash
git add ntp-lab-progress.md
git commit -m "Start NTP lab progress notes"
```

Expected result:

```text
[main abc1234] Start NTP lab progress notes
 1 file changed, ...
```

The exact commit hash will be different.

## Step 2: Set the correct timezone

For this lab environment in Belgium, use the Brussels timezone:

```bash
sudo timedatectl set-timezone Europe/Brussels
timedatectl
```

Expected important lines:

```text
Time zone: Europe/Brussels (CEST, +0200)
```

During winter it may show `CET, +0100` instead of `CEST, +0200`. That is fine.

Check the human-readable date:

```bash
date
```

Expected style of output:

```text
Tue Jun 30 16:35:00 CEST 2026
```

The exact date and time must match the current real time.

Commit the timezone change:

```bash
git status
git add timezone localtime
git commit -m "Configure Brussels timezone"
```

Explanation:

- `timezone` records the timezone name.
- `localtime` points the system to the correct timezone data.

## Step 3: Install the needed NTP tools

Install only the NTP daemon and the verification tools used in the assignment:

```bash
sudo apt update
sudo apt install ntp ntpdate ntpstat
```

Expected result:

```text
Setting up ntp ...
Setting up ntpdate ...
Setting up ntpstat ...
```

If a package is already installed, `apt` may say it is already the newest
version. That is also fine.

Check that the service exists:

```bash
systemctl status ntp --no-pager
```

Expected important line:

```text
Active: active (running)
```

If it is not running yet, start it:

```bash
sudo systemctl enable --now ntp
```

Commit the package/service state:

```bash
git status
git add ntp.conf
git commit -m "Install NTP daemon and tools"
```

If `git status` shows no tracked file changes after installation, write it in
your progress file instead:

```bash
nano ntp-lab-progress.md
git add ntp-lab-progress.md
git commit -m "Record NTP package installation"
```

## Step 4: Resolve the allowed hosts to IP addresses

The final NTP config cannot contain hostnames in `restrict` lines. Resolve the
allowed hosts first.

Resolve `yoda.uclllabs.be`:

```bash
getent ahosts yoda.uclllabs.be
```

Expected style of output:

```text
193.191.xxx.xxx STREAM yoda.uclllabs.be
193.191.xxx.xxx DGRAM
193.191.xxx.xxx RAW
```

For IPv6, use:

```bash
getent ahostsv6 yoda.uclllabs.be
```

Expected style of output:

```text
2a02:xxxx:xxxx::xxxx STREAM yoda.uclllabs.be
```

Now resolve the second allowed server,
`matteo.knops.sasm.uclllabs.be`:

```bash
getent ahosts matteo.knops.sasm.uclllabs.be
getent ahostsv6 matteo.knops.sasm.uclllabs.be
```

Write down one IPv4 address and one IPv6 address for each allowed server:

```text
YODA_IPV4=...
YODA_IPV6=...
MATTEO_IPV4=...
MATTEO_IPV6=...
```

If the `matteo` hostname does not resolve, check that you typed it correctly and
that you are using the lab DNS/network. Do not replace it with
`leia.uclllabs.be`, because the assignment explicitly excludes Leia.

Update your progress file:

```bash
nano ntp-lab-progress.md
```

Add the IP addresses you found, for example:

```markdown
Resolved yoda.uclllabs.be and matteo.knops.sasm.uclllabs.be to IPv4 and IPv6
addresses for the NTP restrict rules.
```

Commit:

```bash
git add ntp-lab-progress.md
git commit -m "Record allowed NTP client addresses"
```

## Step 5: Configure `/etc/ntp.conf`

Open the NTP config:

```bash
sudo nano /etc/ntp.conf
```

Keep or add this upstream pool:

```text
pool be.pool.ntp.org iburst
```

Replace the existing default `restrict` lines with the access-control lines
below. Replace every placeholder with a real IP address:

```text
# Drop all normal clients by default.
restrict -4 default ignore
restrict -6 default ignore

# Allow localhost administration.
restrict 127.0.0.1
restrict ::1

# Allow replies from upstream time sources selected from be.pool.ntp.org.
restrict source nomodify notrap noquery

# Allow only the approved clients to get time from this server.
restrict YODA_IPV4 nomodify notrap nopeer noquery
restrict -6 YODA_IPV6 nomodify notrap nopeer noquery
restrict MATTEO_IPV4 nomodify notrap nopeer noquery
restrict -6 MATTEO_IPV6 nomodify notrap nopeer noquery

# Workaround for strict external checks that may trigger rate limiting.
discard minimum 1
```

What these lines mean:

- `restrict -4 default ignore`: drop all IPv4 NTP packets by default.
- `restrict -6 default ignore`: drop all IPv6 NTP packets by default.
- `restrict 127.0.0.1` and `restrict ::1`: keep local administration working.
- `restrict source ...`: allow your server to receive replies from the internet
  time servers it uses.
- `nomodify`: clients may not change your NTP server settings.
- `notrap`: clients may not use old trap features.
- `nopeer`: clients may not create peer associations.
- `noquery`: clients may not run NTP control queries such as `ntpq` against your
  server, but normal time synchronization still works.
- `discard minimum 1`: avoids overly aggressive rate limiting during checks.

Important:

- Do not write `restrict yoda.uclllabs.be ...`.
- Do not write `restrict matteo.knops.sasm.uclllabs.be ...`.
- The `pool be.pool.ntp.org iburst` line may use a hostname because it is an
  upstream server line, not an access-control `restrict` line.
- Do not leave an old permissive `restrict default ...` rule in the file.

Save and exit nano:

1. Press `Ctrl+O`.
2. Press `Enter`.
3. Press `Ctrl+X`.

Commit the config change:

```bash
git diff ntp.conf
git add ntp.conf
git commit -m "Configure NTP upstream and client restrictions"
```

## Step 6: Restart NTP and check synchronization

Restart the daemon:

```bash
sudo systemctl restart ntp
systemctl status ntp --no-pager
```

Expected important line:

```text
Active: active (running)
```

Check the date:

```bash
date
```

Expected style:

```text
Tue Jun 30 16:35:00 CEST 2026
```

Check synchronization status:

```bash
ntpstat
```

Good output looks like:

```text
synchronised to NTP server (...) at stratum ...
   time correct to within ... ms
   polling server every ... s
```

Right after restarting NTP, you may temporarily see:

```text
unsynchronised
```

Wait a few minutes and run `ntpstat` again.

Check the NTP peers:

```bash
ntpq -pn
```

Good output looks like:

```text
     remote           refid      st t when poll reach   delay   offset  jitter
==============================================================================
 be.pool.ntp.org .POOL.          16 p    -   64    0    0.000    0.000   0.000
*193.190.xxx.xxx 193.79.xxx.xxx   2 u   42   64  377    2.100   -0.500   0.300
```

How to read the first character:

- `*`: the server currently used for synchronization.
- `+`: a good candidate server.
- `-`: a usable but less preferred server.
- blank: not currently selected.

Update your progress file:

```bash
nano ntp-lab-progress.md
```

Add a short note with the result, for example:

```markdown
Restarted ntp and verified synchronization with ntpstat and ntpq -pn.
```

Commit:

```bash
git add ntp-lab-progress.md
git commit -m "Verify NTP synchronization"
```

## Step 7: Verify access control from allowed and blocked clients

You cannot fully test this from only the NTP server itself. Test from other
machines.

From `yoda.uclllabs.be` or `matteo.knops.sasm.uclllabs.be`, run:

```bash
ntpdate -q your-ntp-server-name-or-ip
```

Expected good output:

```text
server 193.191.xxx.xxx, stratum 3, offset -0.000268, delay 0.02588
30 Jun 16:35:00 ntpdate[12345]: adjust time server 193.191.xxx.xxx offset -0.000268 sec
```

This means the allowed client can query your NTP server.

From a server that is not allowed, run the same command:

```bash
ntpdate -q your-ntp-server-name-or-ip
```

Expected blocked output:

```text
server 193.191.xxx.xxx, stratum 0, offset 0.000000, delay 0.00000
30 Jun 16:36:00 ntpdate[12345]: no server suitable for synchronization found
```

This means the client did not receive usable time from your server.

Repeat the same idea over IPv6:

```bash
ntpdate -q your-ntp-server-ipv6-address
```

Expected result:

- Allowed IPv6 client: receives a valid NTP response.
- Blocked IPv6 client: gets `no server suitable for synchronization found`.

Update your progress file:

```bash
nano ntp-lab-progress.md
```

Add which machines worked and which machines were blocked:

```markdown
Verified NTP access control:
- yoda IPv4: allowed
- yoda IPv6: allowed
- matteo IPv4: allowed
- matteo IPv6: allowed
- unrelated client IPv4: blocked
- unrelated client IPv6: blocked
```

Commit:

```bash
git add ntp-lab-progress.md
git commit -m "Verify NTP client access restrictions"
```

## Step 8: Final git check

Check the revision trail:

```bash
git log --oneline --decorate -n 8
```

Expected style:

```text
abc1234 (HEAD -> main) Verify NTP client access restrictions
def5678 Verify NTP synchronization
123abcd Configure NTP upstream and client restrictions
...
```

Check that nothing is forgotten:

```bash
git status
```

Expected result:

```text
nothing to commit, working tree clean
```

If something is still modified, inspect it:

```bash
git diff
```

Then commit it with a clear message if it belongs to the lab.

## Troubleshooting

### `ntpstat` says `unsynchronised`

This can happen immediately after installation or restart. Check peers:

```bash
ntpq -pn
```

If all `reach` values are `0`, wait a few minutes. If they stay `0`, check that:

- `pool be.pool.ntp.org iburst` exists in `/etc/ntp.conf`.
- The server has internet access.
- `restrict source nomodify notrap noquery` exists.
- UDP port `123` is not blocked for outgoing NTP traffic.

### Allowed clients are blocked

Check for typing mistakes in `/etc/ntp.conf`:

```bash
sudo nano /etc/ntp.conf
```

Common mistakes:

- Using a hostname in a `restrict` line.
- Adding the wrong IP address.
- Forgetting the IPv6 `restrict -6 ...` line.
- Accidentally using `leia.uclllabs.be` instead of
  `matteo.knops.sasm.uclllabs.be`.

After fixing the file:

```bash
sudo systemctl restart ntp
```

Then test again with `ntpdate -q`.

### Everyone can still query your server

Make sure the default rules are strict:

```text
restrict -4 default ignore
restrict -6 default ignore
```

Then restart:

```bash
sudo systemctl restart ntp
```

## Quick checklist

- [ ] Timezone is `Europe/Brussels`.
- [ ] `ntp`, `ntpdate`, and `ntpstat` are installed.
- [ ] `/etc/ntp.conf` uses `pool be.pool.ntp.org iburst`.
- [ ] Default IPv4 clients are ignored.
- [ ] Default IPv6 clients are ignored.
- [ ] `restrict source nomodify notrap noquery` is present.
- [ ] `yoda.uclllabs.be` IPv4 address is allowed.
- [ ] `yoda.uclllabs.be` IPv6 address is allowed.
- [ ] `matteo.knops.sasm.uclllabs.be` IPv4 address is allowed.
- [ ] `matteo.knops.sasm.uclllabs.be` IPv6 address is allowed.
- [ ] No hostnames are used in `restrict` lines.
- [ ] `ntpstat` shows synchronized.
- [ ] `ntpq -pn` shows a selected peer with `*`.
- [ ] Allowed clients succeed with `ntpdate -q`.
- [ ] Blocked clients fail with `no server suitable for synchronization found`.
- [ ] Each important step has a descriptive git commit.

## Small summary

This lab teaches you how to run a secure NTP server. Your server synchronizes
its own clock with `be.pool.ntp.org`, then provides time only to approved
clients. The key security part is the access control: all IPv4 and IPv6 clients
are ignored by default, and only the IP addresses of `yoda.uclllabs.be` and
`matteo.knops.sasm.uclllabs.be` are permitted. You also verify the setup with
`date`, `ntpstat`, `ntpq -pn`, and `ntpdate -q`, while documenting every
important change in git.
