# SaSM Lab: DNS

## Goal of this lab

In this lab you configure an authoritative DNS server with BIND9 for:

```text
slimme-rik.sasm.uclllabs.be
```

By the end, you should be able to:

- Explain how DNS lookups work from resolver to authoritative nameserver.
- Explain why the Dan Kaminsky DNS vulnerability was dangerous.
- Install and configure BIND9 as an authoritative-only DNS server.
- Serve the required records for `slimme-rik.sasm.uclllabs.be`.
- Verify the master and slave nameservers with `dig`.
- Automate temporary subzone and record creation with scripts.
- Automatically clean up scripted zones older than 4 hours.

> Important: replace `YOUR_SERVER_IP` everywhere with the IPv4 address of your
> own server. In the examples from the assignment this is often in the
> `193.191.176.0/23` lab range.

## 1. DNS theory you need to know

### What is DNS?

DNS means **Domain Name System**. It translates human-friendly names into IP
addresses and other data.

Examples:

```text
www.example.com       -> 93.184.216.34
example.com mail      -> mail exchanger from MX records
192.0.2.10            -> reverse DNS name
```

DNS is used for much more than websites:

- IPv4 address lookups with `A` records.
- IPv6 address lookups with `AAAA` records.
- Mail delivery with `MX` records.
- Domain ownership checks for TLS certificates.
- Anti-spam policies with `SPF`, `DKIM`, and `DMARC` in `TXT` records.
- Nameserver delegation with `NS` records.

### The DNS lookup chain

When a client wants to resolve a name, several DNS roles can be involved.

1. **Recursive resolver**
   - Receives the client's question.
   - Does the work of finding the answer.
   - Caches answers to speed up later lookups.

2. **Root nameserver**
   - Knows where the Top Level Domain nameservers are.
   - For `.be`, it refers the resolver to the `.be` nameservers.

3. **TLD nameserver**
   - Knows where delegated domains below the TLD are.
   - For `uclllabs.be`, it refers the resolver to the UCLL lab nameservers.

4. **Authoritative nameserver**
   - Holds the actual DNS records for a zone.
   - For this lab, your server is authoritative for
     `slimme-rik.sasm.uclllabs.be`.

The simplified lookup for `test.slimme-rik.sasm.uclllabs.be` is:

```text
client
  -> recursive resolver
  -> root nameserver
  -> .be TLD nameserver
  -> uclllabs.be nameserver
  -> authoritative nameserver for slimme-rik.sasm.uclllabs.be
  -> answer: 193.191.177.254
```

### Recursive, iterative, and non-recursive queries

DNS queries can work in different ways:

- **Recursive query**: the server must return the final answer or an error.
  Recursive resolvers do this for clients.
- **Iterative query**: the server can return a referral to the next nameserver.
  Resolvers use this when walking the DNS tree.
- **Non-recursive query**: the server already knows the answer because it is
  authoritative or has the answer cached.

This lab asks you to configure an **authoritative server**, not an open
recursive resolver.

### Common DNS record types

| Record | Purpose | Example |
| --- | --- | --- |
| `A` | Name to IPv4 address | `www IN A 193.191.176.50` |
| `AAAA` | Name to IPv6 address | `www IN AAAA 2001:db8::10` |
| `CNAME` | Alias to another name | `blog IN CNAME www.example.com.` |
| `MX` | Mail exchanger | `@ IN MX 10 mail.example.com.` |
| `NS` | Authoritative nameserver | `@ IN NS ns.example.com.` |
| `SOA` | Start of Authority | Serial and zone metadata |
| `TXT` | Text data | SPF, DKIM, verification tokens |

### Glue records

A glue record is an address record stored in the parent zone for a nameserver
that lives inside the delegated zone.

Example:

```text
slimme-rik.sasm.uclllabs.be. IN NS ns.slimme-rik.sasm.uclllabs.be.
ns.slimme-rik.sasm.uclllabs.be. IN A YOUR_SERVER_IP
```

Without glue, a resolver might need to resolve
`ns.slimme-rik.sasm.uclllabs.be` before it can contact the nameserver for
`slimme-rik.sasm.uclllabs.be`, which creates a circular dependency. The parent
zone solves this by also publishing the IP address of the in-zone nameserver.

## 2. The Dan Kaminsky DNS vulnerability

### The normal DNS response structure

A DNS answer can contain multiple sections:

```bash
dig @ns.example.com www.example.com
```

Example output:

```text
;; ANSWER SECTION:
www.example.com.    120      IN    A    192.168.1.10

;; AUTHORITY SECTION:
example.com.        86400    IN    NS   ns1.example.com.
example.com.        86400    IN    NS   ns2.example.com.

;; ADDITIONAL SECTION:
ns1.example.com.    604800   IN    A    192.168.2.20
ns2.example.com.    604800   IN    A    192.168.3.30
```

The sections mean:

- **Question section**: the name and type you asked for.
- **Answer section**: the direct answer.
- **Authority section**: nameservers responsible for the zone.
- **Additional section**: extra helpful records, often glue records.

### Bailiwick checking

Bailiwick checking limits which additional records a resolver accepts.

If you ask about `example.com`, an extra record for `ns1.example.com` is in
bailiwick and can be useful. An extra record for `www.bank.com` is outside
bailiwick and should not be trusted.

This prevents many simple cache-poisoning attacks, but it did not fully stop the
Kaminsky attack.

### Why the Kaminsky bug worked

DNS usually uses UDP. The resolver sends a query and accepts the first matching
response with the correct:

- query name,
- query type,
- query ID,
- source/destination information.

Historically, the query ID was only 16 bits. That means only 65,536 possible
values.

The Kaminsky attack repeatedly triggered lookups for random subdomains:

```text
aaa1.example.com
aaa2.example.com
aaa3.example.com
...
```

Because each name was random, the resolver could not answer from cache and had
to ask the authoritative server again. While the resolver waited for the real
answer, the attacker flooded forged replies that tried to guess the query ID.

The forged replies tried to replace the delegation:

```text
example.com. IN NS ns.attacker.example.
ns.attacker.example. IN A 6.6.6.6
```

If one forged answer matched the query ID, the resolver cached the malicious NS
record. After that, users of the resolver could be sent to attacker-controlled
servers.

### The fixes

Modern resolvers defend against this with:

- **Source port randomization**: the attacker must guess both the 16-bit query
  ID and the UDP source port.
- **Better bailiwick rules**: resolvers are more careful with additional data.
- **DNSSEC**: signed DNS records allow validation of authenticity and integrity.

## 3. Required DNS result

Your final authoritative setup must provide:

| Name | Record | Value |
| --- | --- | --- |
| `slimme-rik.sasm.uclllabs.be` | `SOA` | primary server `ns.slimme-rik.sasm.uclllabs.be` |
| `slimme-rik.sasm.uclllabs.be` | `NS` | `ns.slimme-rik.sasm.uclllabs.be` |
| `slimme-rik.sasm.uclllabs.be` | `NS` | `ns1.uclllabs.be` |
| `slimme-rik.sasm.uclllabs.be` | `NS` | `ns2.uclllabs.be` |
| `ns.slimme-rik.sasm.uclllabs.be` | `A` | `YOUR_SERVER_IP` |
| `www.slimme-rik.sasm.uclllabs.be` | `A` | `YOUR_SERVER_IP` |
| `test.slimme-rik.sasm.uclllabs.be` | `A` | `193.191.177.254` |

Later, only after the first part works, you can add:

```text
ns.otherstudent.sasm.uclllabs.be
```

as an extra slave nameserver.

## 4. Step-by-step BIND9 configuration

The commands below assume Ubuntu and a user with `sudo` rights.

### Step 1: Discover your server IP address

Run:

```bash
ip -4 addr show
```

What the command does:

- `ip` manages network interfaces.
- `-4` shows only IPv4 information.
- `addr show` prints interface addresses.

Example result:

```text
2: ens18: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500
    inet 193.191.176.50/23 brd 193.191.177.255 scope global ens18
```

In this example:

```text
YOUR_SERVER_IP=193.191.176.50
```

Use your own address in the rest of the guide.

### Step 2: Install BIND9

```bash
sudo apt update
sudo apt install bind9 bind9utils bind9-doc
```

What the commands do:

- `apt update` refreshes the package index.
- `bind9` installs the `named` DNS daemon.
- `bind9utils` installs tools such as `named-checkconf`, `named-checkzone`,
  `rndc`, and `nsupdate`.
- `bind9-doc` installs documentation.

Expected result:

```text
Setting up bind9 ...
Created symlink /etc/systemd/system/multi-user.target.wants/named.service ...
```

Check the service:

```bash
systemctl status bind9 --no-pager
```

Expected important lines:

```text
Active: active (running)
```

### Step 3: Configure BIND as authoritative-only

Open the options file:

```bash
sudo nano /etc/bind/named.conf.options
```

Set or add these options inside the `options { ... };` block:

```text
options {
    directory "/var/cache/bind";

    recursion no;
    allow-transfer { none; };

    dnssec-validation auto;

    listen-on-v6 { any; };
};
```

What these options do:

- `directory "/var/cache/bind";` sets BIND's working directory.
- `recursion no;` prevents your server from acting as a public recursive
  resolver.
- `allow-transfer { none; };` blocks zone transfers by default.
- `dnssec-validation auto;` keeps default DNSSEC validation behavior.
- `listen-on-v6 { any; };` allows BIND to listen on IPv6 too.

Validate the config:

```bash
sudo named-checkconf
```

Expected result:

```text
```

No output means the syntax is valid.

### Step 4: Create a writable zone directory

Use `/var/lib/bind` for zone files that BIND may need to update later. This
matches Ubuntu's default AppArmor profile.

```bash
sudo mkdir -p /var/lib/bind/zones
sudo chown bind:bind /var/lib/bind/zones
sudo chmod 0775 /var/lib/bind/zones
```

What the commands do:

- `mkdir -p` creates the directory if it does not exist.
- `chown bind:bind` makes the BIND user and group own it.
- `chmod 0775` allows owner and group writes.

Check it:

```bash
ls -ld /var/lib/bind/zones
```

Expected result:

```text
drwxrwxr-x 2 bind bind 4096 ... /var/lib/bind/zones
```

### Step 5: Define the zone in `named.conf.local`

Open:

```bash
sudo nano /etc/bind/named.conf.local
```

Add:

```text
zone "slimme-rik.sasm.uclllabs.be" {
    type master;
    file "/var/lib/bind/zones/db.slimme-rik.sasm.uclllabs.be";
    allow-transfer {
        127.0.0.1;
        ::1;
        193.191.177.20;
        193.191.177.21;
        2001:6a8:2880:a021::20;
        2001:6a8:2880:a021::21;
    };
    notify yes;
    also-notify {
        193.191.177.20;
        193.191.177.21;
    };
};
```

What this configuration means:

- `zone "slimme-rik.sasm.uclllabs.be"` declares your DNS zone.
- `type master;` makes this server the primary source for the zone.
- `file` points to the zone database file.
- `allow-transfer` allows only localhost and the UCLL slave nameservers to copy
  the zone.
- `notify yes;` tells BIND to notify slaves when the zone changes.
- `also-notify` explicitly notifies `ns1.uclllabs.be` and `ns2.uclllabs.be`.

Validate:

```bash
sudo named-checkconf
```

Expected result:

```text
```

No output means the syntax is valid.

### Step 6: Create the zone file

Create:

```bash
sudo nano /var/lib/bind/zones/db.slimme-rik.sasm.uclllabs.be
```

Add this content and replace `YOUR_SERVER_IP`:

```zone
$TTL 3600
@   IN  SOA ns.slimme-rik.sasm.uclllabs.be. admin.slimme-rik.sasm.uclllabs.be. (
        2026063001 ; serial, format YYYYMMDDnn
        3600       ; refresh
        900        ; retry
        1209600    ; expire
        900 )      ; negative cache TTL

; Authoritative nameservers
@       IN  NS  ns.slimme-rik.sasm.uclllabs.be.
@       IN  NS  ns1.uclllabs.be.
@       IN  NS  ns2.uclllabs.be.

; Required A records
ns      IN  A   YOUR_SERVER_IP
www     IN  A   YOUR_SERVER_IP
test    IN  A   193.191.177.254
```

Important details:

- The SOA serial must increase every time you change the zone.
- `admin.slimme-rik.sasm.uclllabs.be.` represents the email address
  `admin@slimme-rik.sasm.uclllabs.be`.
- The final dot on fully qualified names is important.
- `@` means the zone itself: `slimme-rik.sasm.uclllabs.be`.

Set permissions:

```bash
sudo chown bind:bind /var/lib/bind/zones/db.slimme-rik.sasm.uclllabs.be
sudo chmod 0644 /var/lib/bind/zones/db.slimme-rik.sasm.uclllabs.be
```

Check the zone file:

```bash
sudo named-checkzone slimme-rik.sasm.uclllabs.be /var/lib/bind/zones/db.slimme-rik.sasm.uclllabs.be
```

Expected result:

```text
zone slimme-rik.sasm.uclllabs.be/IN: loaded serial 2026063001
OK
```

### Step 7: Reload BIND and inspect logs

Use one terminal for logs:

```bash
sudo journalctl -fu named
```

What the command does:

- `journalctl` reads systemd logs.
- `-u named` selects the BIND service.
- `-f` follows new log lines live.

Use another terminal to reload:

```bash
sudo rndc reload
```

If `rndc` fails, restart the service:

```bash
sudo systemctl restart bind9
```

Expected log lines:

```text
zone slimme-rik.sasm.uclllabs.be/IN: loaded serial 2026063001
all zones loaded
running
zone slimme-rik.sasm.uclllabs.be/IN: sending notifies
```

If you see permission errors such as:

```text
/var/lib/bind/zones/db.slimme-rik.sasm.uclllabs.be.jnl: create: permission denied
```

fix ownership:

```bash
sudo chown -R bind:bind /var/lib/bind/zones
sudo rndc reload
```

## 5. Verify the authoritative DNS setup

### Check the SOA record directly on your server

```bash
dig -t SOA slimme-rik.sasm.uclllabs.be @localhost
```

What the command does:

- `dig` sends a DNS query.
- `-t SOA` asks for the Start of Authority record.
- `@localhost` asks your local BIND server.

Expected important output:

```text
;; ->>HEADER<<- opcode: QUERY, status: NOERROR
;; flags: qr aa

;; ANSWER SECTION:
slimme-rik.sasm.uclllabs.be. 3600 IN SOA ns.slimme-rik.sasm.uclllabs.be. admin.slimme-rik.sasm.uclllabs.be. 2026063001 3600 900 1209600 900
```

The `aa` flag means **authoritative answer**.

### Check the nameservers

```bash
dig +short -t NS slimme-rik.sasm.uclllabs.be @localhost
```

Expected result:

```text
ns.slimme-rik.sasm.uclllabs.be.
ns1.uclllabs.be.
ns2.uclllabs.be.
```

### Check the required A records

```bash
dig +short ns.slimme-rik.sasm.uclllabs.be @localhost
dig +short www.slimme-rik.sasm.uclllabs.be @localhost
dig +short test.slimme-rik.sasm.uclllabs.be @localhost
```

Expected result:

```text
YOUR_SERVER_IP
YOUR_SERVER_IP
193.191.177.254
```

### Check that recursion is disabled

Ask your server about an unrelated external domain:

```bash
dig www.google.com @localhost
```

Expected important output:

```text
;; ->>HEADER<<- opcode: QUERY, status: REFUSED
```

`REFUSED` is correct for an authoritative-only server.

### Check from the delegated public path

After the UCLL parent zone delegates your zone, test through a recursive
resolver:

```bash
dig +short test.slimme-rik.sasm.uclllabs.be @recursive-dns.uclllabs.be
```

Expected result:

```text
193.191.177.254
```

Trace the delegation:

```bash
dig +trace +nodnssec test.slimme-rik.sasm.uclllabs.be @8.8.8.8
```

What the command does:

- `+trace` walks the DNS tree step by step.
- `+nodnssec` hides DNSSEC records to keep output readable.
- `@8.8.8.8` asks Google DNS for the initial root hints.

Expected final line:

```text
test.slimme-rik.sasm.uclllabs.be. 3600 IN A 193.191.177.254
```

### Compare SOA records on all authoritative nameservers

Run:

```bash
NAME=slimme-rik
dig +short -t ns $NAME.sasm.uclllabs.be | while read DNS_SERVER; do
    printf "%-40s" "$DNS_SERVER"
    echo SOA: $(dig +short -t soa $NAME.sasm.uclllabs.be @$DNS_SERVER 2>&1)
done | sort
```

Expected result:

```text
ns.slimme-rik.sasm.uclllabs.be.         SOA: ns.slimme-rik.sasm.uclllabs.be. admin.slimme-rik.sasm.uclllabs.be. 2026063001 3600 900 1209600 900
ns1.uclllabs.be.                        SOA: ns.slimme-rik.sasm.uclllabs.be. admin.slimme-rik.sasm.uclllabs.be. 2026063001 3600 900 1209600 900
ns2.uclllabs.be.                        SOA: ns.slimme-rik.sasm.uclllabs.be. admin.slimme-rik.sasm.uclllabs.be. 2026063001 3600 900 1209600 900
```

All SOA serials must match. If they do not:

1. Increase the SOA serial on the master.
2. Reload BIND.
3. Check logs for denied AXFR or NOTIFY errors.
4. Verify `allow-transfer` includes the slave IPs.

### Compare full zone transfers with hashes

Run:

```bash
NAME=slimme-rik
dig +short -t ns $NAME.sasm.uclllabs.be | while read DNS_SERVER; do
    dig axfr $NAME.sasm.uclllabs.be @$DNS_SERVER | grep -v ';' | sort | sha256sum
done
```

Expected result:

```text
e07a92df6e2e89e5b8973637d13af9cd54f4d6d7cec9bb560217d7abced7c67e  -
e07a92df6e2e89e5b8973637d13af9cd54f4d6d7cec9bb560217d7abced7c67e  -
e07a92df6e2e89e5b8973637d13af9cd54f4d6d7cec9bb560217d7abced7c67e  -
```

The exact hash will be different, but all hash values should be the same.

> Warning: if the SOA records or hashes differ, later labs that depend on DNS
> may fail. Fix synchronization before continuing.

## 6. Prepare Dynamic DNS updates

The scripting part is easier and safer when scripts use authenticated dynamic
DNS updates instead of editing active zone files directly.

### Step 1: Create a TSIG key

```bash
sudo tsig-keygen ddnskey | sudo tee /etc/bind/ddns.key
sudo chown root:bind /etc/bind/ddns.key
sudo chmod 0640 /etc/bind/ddns.key
```

What the commands do:

- `tsig-keygen ddnskey` creates a shared secret key named `ddnskey`.
- `tee` writes the generated key to a file.
- `chown root:bind` lets root own the file while BIND can read it as group.
- `chmod 0640` prevents normal users from reading the secret.

Expected file content format:

```text
key "ddnskey" {
    algorithm hmac-sha256;
    secret "...";
};
```

Do not share the secret value.

### Step 2: Include the key and allow updates

Edit:

```bash
sudo nano /etc/bind/named.conf.local
```

Add this near the top:

```text
include "/etc/bind/ddns.key";
```

Update the main zone stanza:

```text
zone "slimme-rik.sasm.uclllabs.be" {
    type master;
    file "/var/lib/bind/zones/db.slimme-rik.sasm.uclllabs.be";
    allow-transfer {
        127.0.0.1;
        ::1;
        193.191.177.20;
        193.191.177.21;
        2001:6a8:2880:a021::20;
        2001:6a8:2880:a021::21;
    };
    allow-update { key ddnskey; };
    notify yes;
    also-notify {
        193.191.177.20;
        193.191.177.21;
    };
};
```

Validate and reload:

```bash
sudo named-checkconf
sudo rndc reload
```

### Step 3: Test `nsupdate`

Add a temporary record:

```bash
sudo nsupdate -k /etc/bind/ddns.key
```

Type:

```text
server 127.0.0.1
zone slimme-rik.sasm.uclllabs.be
update add ddnstest.slimme-rik.sasm.uclllabs.be. 60 IN A 192.0.2.123
send
quit
```

Verify:

```bash
dig +short ddnstest.slimme-rik.sasm.uclllabs.be @localhost
```

Expected result:

```text
192.0.2.123
```

Delete it again:

```bash
sudo nsupdate -k /etc/bind/ddns.key
```

Type:

```text
server 127.0.0.1
zone slimme-rik.sasm.uclllabs.be
update delete ddnstest.slimme-rik.sasm.uclllabs.be. A
send
quit
```

## 7. Prepare files for the scripting part

### Step 1: Create a separate auto-generated BIND include

Create an empty file:

```bash
sudo touch /etc/bind/named.conf.yoda-zones
sudo chown root:bind /etc/bind/named.conf.yoda-zones
sudo chmod 0644 /etc/bind/named.conf.yoda-zones
```

Include it from `/etc/bind/named.conf.local`:

```text
include "/etc/bind/named.conf.yoda-zones";
```

What this does:

- BIND reads `named.conf.local`.
- When it sees the `include`, it also reads the generated Yoda zone file.
- Automatically created zones stay separate from your hand-written config.

### Step 2: Create the script and zone directories

```bash
sudo mkdir -p /etc/scripts
sudo mkdir -p /var/lib/bind/yoda
sudo chown root:root /etc/scripts
sudo chmod 0755 /etc/scripts
sudo chown bind:bind /var/lib/bind/yoda
sudo chmod 0770 /var/lib/bind/yoda
```

What the commands do:

- `/etc/scripts` stores your scripts.
- `/var/lib/bind/yoda` stores generated subzone files.
- Only root edits the scripts.
- BIND can read and write generated zone files.

### Step 3: Create command symlinks

The grader calls commands such as:

```bash
sudo dns_add_zone foobar
```

So create scripts in `/etc/scripts` and symlinks in `/usr/local/sbin`:

```bash
sudo touch /etc/scripts/dns_add_zone.py
sudo touch /etc/scripts/dns_add_record.py
sudo touch /etc/scripts/dns_cleanup.py

sudo chmod 0750 /etc/scripts/dns_add_zone.py
sudo chmod 0750 /etc/scripts/dns_add_record.py
sudo chmod 0750 /etc/scripts/dns_cleanup.py
sudo chown root:root /etc/scripts/dns_add_zone.py /etc/scripts/dns_add_record.py /etc/scripts/dns_cleanup.py

sudo ln -sf /etc/scripts/dns_add_zone.py /usr/local/sbin/dns_add_zone
sudo ln -sf /etc/scripts/dns_add_record.py /usr/local/sbin/dns_add_record
sudo ln -sf /etc/scripts/dns_cleanup.py /usr/local/sbin/dns_cleanup
```

Check:

```bash
ls -l /usr/local/sbin/dns_*
```

Expected result:

```text
/usr/local/sbin/dns_add_zone -> /etc/scripts/dns_add_zone.py
/usr/local/sbin/dns_add_record -> /etc/scripts/dns_add_record.py
/usr/local/sbin/dns_cleanup -> /etc/scripts/dns_cleanup.py
```

### Step 4: Configure sudo for user `check`

Create a sudoers drop-in:

```bash
sudo visudo -f /etc/sudoers.d/dns-lab-check
```

Add:

```text
check ALL=(root) NOPASSWD: /usr/local/sbin/dns_add_zone, /usr/local/sbin/dns_add_record, /usr/local/sbin/dns_cleanup
```

Validate:

```bash
sudo visudo -cf /etc/sudoers.d/dns-lab-check
```

Expected result:

```text
/etc/sudoers.d/dns-lab-check: parsed OK
```

## 8. Script: `dns_add_zone`

### Required behavior

The command:

```bash
check$ sudo dns_add_zone foobar
```

must create:

```text
foobar.slimme-rik.sasm.uclllabs.be
```

It must:

1. Only run as root via `sudo` from user `check`.
2. Create a zone file in `/var/lib/bind/yoda`.
3. Register the zone in `/etc/bind/named.conf.yoda-zones`.
4. Add an `NS` delegation in the parent zone using `nsupdate`.
5. Reconfigure or reload BIND.

### Script content

Edit:

```bash
sudo nano /etc/scripts/dns_add_zone.py
```

Add:

```python
#!/usr/bin/env python3

import grp
import os
import pwd
import re
import subprocess
import sys
from datetime import datetime
from pathlib import Path

FQDN = "slimme-rik.sasm.uclllabs.be"
ZONE_DIR = Path("/var/lib/bind/yoda")
BIND_CONFIG = Path("/etc/bind/named.conf.yoda-zones")
TSIG_KEY = "/etc/bind/ddns.key"
NS_HOST = f"ns.{FQDN}."


def fail(message):
    print(f"Error: {message}", file=sys.stderr)
    raise SystemExit(1)


def require_check_via_sudo():
    if os.geteuid() != 0:
        fail("must be run with sudo as user check")
    if os.environ.get("SUDO_USER") != "check":
        fail("run exactly as: check$ sudo dns_add_zone <subzone>")


def validate_label(label):
    if not re.fullmatch(r"[a-z0-9]([a-z0-9-]{0,61}[a-z0-9])?", label):
        fail("subzone must be one DNS label with lowercase letters, digits, and hyphens")


def run(command, **kwargs):
    try:
        return subprocess.run(command, text=True, capture_output=True, check=True, **kwargs)
    except subprocess.CalledProcessError as error:
        if error.stdout:
            print(error.stdout)
        if error.stderr:
            print(error.stderr, file=sys.stderr)
        fail(f"command failed: {' '.join(command)}")


def create_zone_file(zone_name):
    zone_file = ZONE_DIR / f"db.{zone_name}"
    if zone_file.exists():
        fail(f"zone file already exists: {zone_file}")

    serial = datetime.now().strftime("%Y%m%d01")
    zone_file.write_text(
        f"""$TTL 3600
@   IN  SOA {NS_HOST} admin.{FQDN}. (
        {serial} ; serial
        3600     ; refresh
        900      ; retry
        1209600  ; expire
        900 )    ; negative cache TTL

@   IN  NS  {NS_HOST}
""",
        encoding="utf-8",
    )

    bind_user = pwd.getpwnam("bind")
    bind_group = grp.getgrnam("bind")
    os.chown(zone_file, bind_user.pw_uid, bind_group.gr_gid)
    zone_file.chmod(0o640)
    run(["named-checkzone", zone_name, str(zone_file)])
    return zone_file


def append_zone_config(zone_name, zone_file):
    stanza = f'''
zone "{zone_name}" {{
    type master;
    file "{zone_file}";
    allow-update {{ key ddnskey; }};
}};
'''
    current = BIND_CONFIG.read_text(encoding="utf-8") if BIND_CONFIG.exists() else ""
    if f'zone "{zone_name}"' in current:
        fail(f"zone already exists in {BIND_CONFIG}")
    with BIND_CONFIG.open("a", encoding="utf-8") as handle:
        handle.write(stanza)
    run(["named-checkconf"])


def add_parent_delegation(zone_name):
    nsupdate_input = f"""server 127.0.0.1
zone {FQDN}
update add {zone_name}. 3600 IN NS {NS_HOST}
send
"""
    run(["nsupdate", "-k", TSIG_KEY], input=nsupdate_input)


def reload_bind():
    run(["rndc", "reconfig"])


def main():
    require_check_via_sudo()
    if len(sys.argv) != 2:
        fail("usage: sudo dns_add_zone <subzone>")

    subzone = sys.argv[1]
    validate_label(subzone)
    zone_name = f"{subzone}.{FQDN}"

    zone_file = create_zone_file(zone_name)
    append_zone_config(zone_name, zone_file)
    reload_bind()
    add_parent_delegation(zone_name)

    print(f"Created zone {zone_name}")


if __name__ == "__main__":
    main()
```

Make it executable:

```bash
sudo chmod 0750 /etc/scripts/dns_add_zone.py
```

### Test `dns_add_zone`

Switch to user `check`:

```bash
su - check
```

Run:

```bash
sudo dns_add_zone foobar
```

Expected result:

```text
zone foobar.slimme-rik.sasm.uclllabs.be/IN: loaded serial 2026063001
OK
Created zone foobar.slimme-rik.sasm.uclllabs.be
```

Verify:

```bash
dig +short -t NS foobar.slimme-rik.sasm.uclllabs.be @localhost
```

Expected result:

```text
ns.slimme-rik.sasm.uclllabs.be.
```

Check parent delegation:

```bash
dig +short -t NS foobar.slimme-rik.sasm.uclllabs.be @localhost
```

Expected result:

```text
ns.slimme-rik.sasm.uclllabs.be.
```

## 9. Script: `dns_add_record`

### Required behavior

The command supports `A`, `CNAME`, and `MX`.

Default `A` record:

```bash
check$ sudo dns_add_record test 12.34.56.78 foobar.slimme-rik.sasm.uclllabs.be
```

Explicit `A` record:

```bash
check$ sudo dns_add_record -t A test 12.34.56.78 foobar.slimme-rik.sasm.uclllabs.be
```

`CNAME` record:

```bash
check$ sudo dns_add_record -t CNAME wwwwww www.slimme-rik.sasm.uclllabs.be
```

`MX` record:

```bash
check$ sudo dns_add_record -t MX mail YOUR_SERVER_IP slimme-rik.sasm.uclllabs.be
```

For `MX`, the script must create two records:

```text
slimme-rik.sasm.uclllabs.be. IN MX 10 mail.slimme-rik.sasm.uclllabs.be.
mail.slimme-rik.sasm.uclllabs.be. IN A YOUR_SERVER_IP
```

### Script content

Edit:

```bash
sudo nano /etc/scripts/dns_add_record.py
```

Add:

```python
#!/usr/bin/env python3

import argparse
import ipaddress
import os
import re
import subprocess
import sys

FQDN = "slimme-rik.sasm.uclllabs.be"
TSIG_KEY = "/etc/bind/ddns.key"
TTL = 3600


def fail(message):
    print(f"Error: {message}", file=sys.stderr)
    raise SystemExit(1)


def require_check_via_sudo():
    if os.geteuid() != 0:
        fail("must be run with sudo as user check")
    if os.environ.get("SUDO_USER") != "check":
        fail("run exactly as: check$ sudo dns_add_record [-t TYPE] <name> <value> [zone]")


def validate_label(label):
    if not re.fullmatch(r"[a-z0-9]([a-z0-9-]{0,61}[a-z0-9])?", label):
        fail("record name must be one DNS label with lowercase letters, digits, and hyphens")


def validate_zone(zone):
    if not zone.endswith(FQDN):
        fail(f"zone must be {FQDN} or a subzone below it")
    if zone.endswith("."):
        fail("zone must not end with a dot")


def fqdn(name, zone):
    return f"{name}.{zone}."


def ensure_abs_name(name):
    return name if name.endswith(".") else f"{name}."


def run_nsupdate(zone, updates):
    body = "\n".join(f"update add {record}" for record in updates)
    nsupdate_input = f"""server 127.0.0.1
zone {zone}
{body}
send
"""
    try:
        subprocess.run(
            ["nsupdate", "-k", TSIG_KEY],
            input=nsupdate_input,
            text=True,
            capture_output=True,
            check=True,
        )
    except subprocess.CalledProcessError as error:
        if error.stdout:
            print(error.stdout)
        if error.stderr:
            print(error.stderr, file=sys.stderr)
        fail("nsupdate failed")


def parse_args():
    parser = argparse.ArgumentParser(
        prog="dns_add_record",
        description="Add A, CNAME, or MX records to a slimme-rik DNS zone.",
    )
    parser.add_argument("-t", "--type", default="A", choices=["A", "CNAME", "MX"])
    parser.add_argument("name")
    parser.add_argument("value")
    parser.add_argument("zone", nargs="?")
    return parser.parse_args()


def main():
    require_check_via_sudo()
    args = parse_args()
    validate_label(args.name)

    record_type = args.type.upper()
    if args.zone is None:
        if record_type != "CNAME":
            fail("zone is required for A and MX records")
        args.zone = FQDN
    validate_zone(args.zone)

    records = []

    if record_type == "A":
        ipaddress.IPv4Address(args.value)
        records.append(f"{fqdn(args.name, args.zone)} {TTL} IN A {args.value}")

    elif record_type == "CNAME":
        records.append(f"{fqdn(args.name, args.zone)} {TTL} IN CNAME {ensure_abs_name(args.value)}")

    elif record_type == "MX":
        ipaddress.IPv4Address(args.value)
        mail_name = fqdn(args.name, args.zone)
        records.append(f"{args.zone}. {TTL} IN MX 10 {mail_name}")
        records.append(f"{mail_name} {TTL} IN A {args.value}")

    run_nsupdate(args.zone, records)
    print(f"Added {record_type} record data to {args.zone}:")
    for record in records:
        print(record)


if __name__ == "__main__":
    main()
```

Make it executable:

```bash
sudo chmod 0750 /etc/scripts/dns_add_record.py
```

### Test `dns_add_record`

As user `check`, run:

```bash
sudo dns_add_record -t A test 12.34.56.78 foobar.slimme-rik.sasm.uclllabs.be
```

Expected result:

```text
Added A record data to foobar.slimme-rik.sasm.uclllabs.be:
test.foobar.slimme-rik.sasm.uclllabs.be. 3600 IN A 12.34.56.78
```

Verify:

```bash
dig +short test.foobar.slimme-rik.sasm.uclllabs.be @localhost
```

Expected result:

```text
12.34.56.78
```

Test a CNAME:

```bash
sudo dns_add_record -t CNAME wwwwww www.slimme-rik.sasm.uclllabs.be
dig +short wwwwww.slimme-rik.sasm.uclllabs.be @localhost
```

Expected result:

```text
www.slimme-rik.sasm.uclllabs.be.
```

Test an MX record:

```bash
sudo dns_add_record -t MX mail YOUR_SERVER_IP slimme-rik.sasm.uclllabs.be
dig +short -t MX slimme-rik.sasm.uclllabs.be @localhost
dig +short mail.slimme-rik.sasm.uclllabs.be @localhost
```

Expected result:

```text
10 mail.slimme-rik.sasm.uclllabs.be.
YOUR_SERVER_IP
```

## 10. Script: `dns_cleanup`

### Required behavior

Yoda will create temporary zones by calling `dns_add_zone`. Your server must
remove zones older than 4 hours.

Cleanup must remove:

1. The child zone file in `/var/lib/bind/yoda`.
2. The child zone journal file, if it exists.
3. The `zone { ... };` stanza in `/etc/bind/named.conf.yoda-zones`.
4. The parent `NS` delegation record.
5. The active BIND configuration for that child zone.

### Script content

Edit:

```bash
sudo nano /etc/scripts/dns_cleanup.py
```

Add:

```python
#!/usr/bin/env python3

import os
import re
import subprocess
import sys
import time
from pathlib import Path

FQDN = "slimme-rik.sasm.uclllabs.be"
ZONE_DIR = Path("/var/lib/bind/yoda")
BIND_CONFIG = Path("/etc/bind/named.conf.yoda-zones")
TSIG_KEY = "/etc/bind/ddns.key"
NS_HOST = f"ns.{FQDN}."
MAX_AGE_SECONDS = 4 * 60 * 60


def fail(message):
    print(f"Error: {message}", file=sys.stderr)
    raise SystemExit(1)


def require_root():
    if os.geteuid() != 0:
        fail("must be run as root")


def run(command, **kwargs):
    try:
        return subprocess.run(command, text=True, capture_output=True, check=True, **kwargs)
    except subprocess.CalledProcessError as error:
        if error.stdout:
            print(error.stdout)
        if error.stderr:
            print(error.stderr, file=sys.stderr)
        fail(f"command failed: {' '.join(command)}")


def zone_name_from_file(path):
    name = path.name
    if not name.startswith("db."):
        return None
    return name[3:]


def expired_zone_files():
    now = time.time()
    for zone_file in ZONE_DIR.glob(f"db.*.{FQDN}"):
        if zone_file.suffix == ".jnl":
            continue
        if now - zone_file.stat().st_mtime > MAX_AGE_SECONDS:
            zone_name = zone_name_from_file(zone_file)
            if zone_name:
                yield zone_name, zone_file


def remove_zone_stanza(zone_name):
    if not BIND_CONFIG.exists():
        return

    lines = BIND_CONFIG.read_text(encoding="utf-8").splitlines(keepends=True)
    output = []
    skipping = False
    depth = 0
    zone_header = f'zone "{zone_name}"'

    for line in lines:
        if not skipping and zone_header in line:
            skipping = True
            depth = line.count("{") - line.count("}")
            continue

        if skipping:
            depth += line.count("{") - line.count("}")
            if depth <= 0 and "};" in line:
                skipping = False
            continue

        output.append(line)

    BIND_CONFIG.write_text("".join(output), encoding="utf-8")


def delete_parent_delegation(zone_name):
    nsupdate_input = f"""server 127.0.0.1
zone {FQDN}
update delete {zone_name}. IN NS {NS_HOST}
send
"""
    run(["nsupdate", "-k", TSIG_KEY], input=nsupdate_input)


def delete_zone_files(zone_file):
    journal = Path(f"{zone_file}.jnl")
    zone_file.unlink(missing_ok=True)
    journal.unlink(missing_ok=True)


def cleanup_zone(zone_name, zone_file):
    if not re.fullmatch(r"[a-z0-9-]+\." + re.escape(FQDN), zone_name):
        fail(f"refusing suspicious zone name: {zone_name}")

    print(f"Cleaning up {zone_name}")
    delete_parent_delegation(zone_name)
    remove_zone_stanza(zone_name)
    delete_zone_files(zone_file)


def main():
    require_root()
    zones = list(expired_zone_files())

    if not zones:
        print("No generated zones older than 4 hours found.")
        return

    for zone_name, zone_file in zones:
        cleanup_zone(zone_name, zone_file)

    run(["rndc", "reconfig"])
    print(f"Cleaned up {len(zones)} expired zone(s).")


if __name__ == "__main__":
    main()
```

Make it executable:

```bash
sudo chmod 0750 /etc/scripts/dns_cleanup.py
```

### Test `dns_cleanup`

Create a test zone:

```bash
su - check
sudo dns_add_zone oldtest
exit
```

Force its modification time to older than 4 hours:

```bash
sudo touch -d '5 hours ago' /var/lib/bind/yoda/db.oldtest.slimme-rik.sasm.uclllabs.be
```

Run cleanup:

```bash
sudo dns_cleanup
```

Expected result:

```text
Cleaning up oldtest.slimme-rik.sasm.uclllabs.be
Cleaned up 1 expired zone(s).
```

Verify the zone file is gone:

```bash
ls /var/lib/bind/yoda/db.oldtest.slimme-rik.sasm.uclllabs.be
```

Expected result:

```text
ls: cannot access '/var/lib/bind/yoda/db.oldtest.slimme-rik.sasm.uclllabs.be': No such file or directory
```

Verify the delegation is gone:

```bash
dig +short -t NS oldtest.slimme-rik.sasm.uclllabs.be @localhost
```

Expected result:

```text
```

No output means there is no longer an NS delegation.

### Schedule cleanup with cron

Edit root's crontab:

```bash
sudo crontab -e
```

Add:

```cron
30 * * * * /usr/local/sbin/dns_cleanup
```

What this means:

- Minute `30`.
- Every hour.
- Every day, month, and weekday.
- Run `/usr/local/sbin/dns_cleanup`.

Show the installed crontab:

```bash
sudo crontab -l
```

Expected result:

```text
30 * * * * /usr/local/sbin/dns_cleanup
```

## 11. Add another student's server as slave

Do this only after the master, `ns1.uclllabs.be`, and `ns2.uclllabs.be` all work.

Assume the other student owns:

```text
ns.otherstudent.sasm.uclllabs.be
```

and their server IP is:

```text
OTHER_STUDENT_IP
```

### On your server

Allow zone transfers to the other student in `/etc/bind/named.conf.local`:

```text
allow-transfer {
    127.0.0.1;
    ::1;
    193.191.177.20;
    193.191.177.21;
    2001:6a8:2880:a021::20;
    2001:6a8:2880:a021::21;
    OTHER_STUDENT_IP;
};
```

Add an NS record:

```bash
sudo nsupdate -k /etc/bind/ddns.key
```

Type:

```text
server 127.0.0.1
zone slimme-rik.sasm.uclllabs.be
update add slimme-rik.sasm.uclllabs.be. 3600 IN NS ns.otherstudent.sasm.uclllabs.be.
send
quit
```

Reload:

```bash
sudo rndc reload
```

### On the other student's server

They add:

```text
zone "slimme-rik.sasm.uclllabs.be" {
    type slave;
    masters { YOUR_SERVER_IP; };
    file "/var/cache/bind/db.slimme-rik.sasm.uclllabs.be";
};
```

They reload BIND:

```bash
sudo named-checkconf
sudo rndc reconfig
```

### Verify the extra slave

```bash
dig +short -t SOA slimme-rik.sasm.uclllabs.be @ns.otherstudent.sasm.uclllabs.be
```

Expected result:

```text
ns.slimme-rik.sasm.uclllabs.be. admin.slimme-rik.sasm.uclllabs.be. 2026063001 3600 900 1209600 900
```

## 12. Troubleshooting checklist

### BIND syntax errors

Run:

```bash
sudo named-checkconf
```

If there is an error, BIND prints the file and line number:

```text
/etc/bind/named.conf.local:12: missing ';' before '}'
```

Fix the line and run `named-checkconf` again.

### Zone file errors

Run:

```bash
sudo named-checkzone slimme-rik.sasm.uclllabs.be /var/lib/bind/zones/db.slimme-rik.sasm.uclllabs.be
```

Common errors:

```text
has no NS records
not at top of zone
bad dotted quad
```

Fix the zone file and increase the SOA serial when needed.

### Logs

Useful log commands:

```bash
sudo journalctl -u named -n 20
sudo journalctl -fu named
sudo journalctl -u named --since '2 hours ago'
sudo journalctl -u named -p 4
```

What they do:

- `-n 20` shows the last 20 lines.
- `-f` follows new lines.
- `--since` filters by time.
- `-p 4` shows warnings and errors.

### Zone transfers denied

If logs show:

```text
zone transfer 'slimme-rik.sasm.uclllabs.be/AXFR/IN' denied
```

then the requesting slave IP is missing from `allow-transfer`.

Find slave IPs:

```bash
dig +short ns1.uclllabs.be
dig +short ns2.uclllabs.be
```

Expected result:

```text
193.191.177.20
193.191.177.21
```

Add the missing IP and reload.

### Slave data is old

Symptoms:

- `@localhost` has the correct record.
- `@ns1.uclllabs.be` or `@ns2.uclllabs.be` still shows old data.

Fix:

1. Increase the SOA serial in the master zone.
2. Reload the master:

   ```bash
   sudo rndc reload slimme-rik.sasm.uclllabs.be
   ```

3. Watch logs:

   ```bash
   sudo journalctl -fu named
   ```

4. Recheck SOA values:

   ```bash
   NAME=slimme-rik
   for NS in ns1 ns2 ns.$NAME.sasm; do
       echo "$(dig +short -t soa $NAME.sasm.uclllabs.be @$NS.uclllabs.be) ($NS.uclllabs.be)"
   done
   ```

### AppArmor write problems

Ubuntu allows BIND to write under:

```text
/var/lib/bind
/var/cache/bind
```

Use these directories for dynamic or slave zones. Avoid putting dynamic zone
files under `/etc/bind` unless you also understand and update the AppArmor
profile.

## 13. Final verification commands

Run these before marking the lab finished.

```bash
sudo named-checkconf
sudo named-checkzone slimme-rik.sasm.uclllabs.be /var/lib/bind/zones/db.slimme-rik.sasm.uclllabs.be
```

Expected result:

```text
zone slimme-rik.sasm.uclllabs.be/IN: loaded serial 2026063001
OK
```

Check local records:

```bash
dig +short -t NS slimme-rik.sasm.uclllabs.be @localhost
dig +short ns.slimme-rik.sasm.uclllabs.be @localhost
dig +short www.slimme-rik.sasm.uclllabs.be @localhost
dig +short test.slimme-rik.sasm.uclllabs.be @localhost
```

Expected result:

```text
ns.slimme-rik.sasm.uclllabs.be.
ns1.uclllabs.be.
ns2.uclllabs.be.
YOUR_SERVER_IP
YOUR_SERVER_IP
193.191.177.254
```

Check SOA synchronization:

```bash
NAME=slimme-rik
dig +short -t ns $NAME.sasm.uclllabs.be | while read DNS_SERVER; do
    printf "%-40s" "$DNS_SERVER"
    echo SOA: $(dig +short -t soa $NAME.sasm.uclllabs.be @$DNS_SERVER 2>&1)
done | sort
```

Expected result:

```text
ns.slimme-rik.sasm.uclllabs.be.         SOA: ns.slimme-rik.sasm.uclllabs.be. admin.slimme-rik.sasm.uclllabs.be. 2026063001 3600 900 1209600 900
ns1.uclllabs.be.                        SOA: ns.slimme-rik.sasm.uclllabs.be. admin.slimme-rik.sasm.uclllabs.be. 2026063001 3600 900 1209600 900
ns2.uclllabs.be.                        SOA: ns.slimme-rik.sasm.uclllabs.be. admin.slimme-rik.sasm.uclllabs.be. 2026063001 3600 900 1209600 900
```

Check scripts:

```bash
su - check
sudo dns_add_zone finaltest
sudo dns_add_record -t A host 12.34.56.78 finaltest.slimme-rik.sasm.uclllabs.be
dig +short host.finaltest.slimme-rik.sasm.uclllabs.be @localhost
exit
```

Expected result:

```text
12.34.56.78
```
