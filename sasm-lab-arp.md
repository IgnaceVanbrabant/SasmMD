# SaSM Lab: ARP

Minimal setup for the ARP lab.

Goal:

- make the default gateway ARP entry static;
- keep it after reboot;
- re-apply it when `eth0` comes back online;
- allow user `check` to run only the two required `ip link` commands.

Use `nano` for every file edit.

## 1. Start in `/etc` and commit the start

```bash
cd /etc
sudo git status
sudo git commit --allow-empty -m "Start ARP lab"
```

You should see a new git commit.

## 2. Get the gateway IP and MAC

Find the default gateway:

```bash
ip route show default
```

You should see something like:

```text
default via 192.168.1.1 dev eth0
```

The IP after `via` is your gateway IP.

Ping it once:

```bash
ping -c 1 192.168.1.1
```

Replace `192.168.1.1` with your gateway IP.

Get the MAC:

```bash
ip neigh show 192.168.1.1 dev eth0
```

You should see something like:

```text
192.168.1.1 dev eth0 lladdr aa:bb:cc:dd:ee:ff REACHABLE
```

The value after `lladdr` is the gateway MAC.

Commit this step:

```bash
sudo git commit --allow-empty -m "Find gateway address for static ARP"
```

## 3. Create the static ARP script

Create a lab directory:

```bash
sudo mkdir -p /etc/sasm
```

Open the script:

```bash
sudo nano /etc/sasm/static-arp-gateway
```

Put this in the file:

```sh
#!/bin/sh
set -eu

GATEWAY_IP="192.168.1.1"
GATEWAY_MAC="aa:bb:cc:dd:ee:ff"

ip neigh replace "$GATEWAY_IP" lladdr "$GATEWAY_MAC" dev eth0 nud permanent
```

Replace:

- `192.168.1.1` with your gateway IP;
- `aa:bb:cc:dd:ee:ff` with your gateway MAC.

Save in nano:

```text
CTRL+O, Enter, CTRL+X
```

Make it executable and run it:

```bash
sudo chmod 755 /etc/sasm/static-arp-gateway
sudo /etc/sasm/static-arp-gateway
```

Check it:

```bash
ip neigh show 192.168.1.1 dev eth0
```

You should see:

```text
192.168.1.1 dev eth0 lladdr aa:bb:cc:dd:ee:ff PERMANENT
```

Commit it:

```bash
sudo git add sasm/static-arp-gateway
sudo git commit -m "Add static ARP script for gateway"
```

## 4. Run the script at boot

Remove the old udev rule if it exists:

```bash
sudo rm -f /etc/udev/rules.d/90-sasm-static-arp-gateway.rules
```

Create the systemd service:

```bash
sudo nano /etc/systemd/system/sasm-static-arp-gateway.service
```

Put this in the file:

```ini
[Unit]
Description=Set static ARP entry for default gateway
Wants=network-online.target
After=network-online.target

[Service]
Type=oneshot
ExecStart=/etc/sasm/static-arp-gateway

[Install]
WantedBy=multi-user.target
```

Save:

```text
CTRL+O, Enter, CTRL+X
```

Enable it:

```bash
sudo systemctl daemon-reload
sudo systemctl reset-failed sasm-static-arp-gateway.service
sudo systemctl enable sasm-static-arp-gateway.service
sudo systemctl start sasm-static-arp-gateway.service
```

Check again:

```bash
ip neigh show 192.168.1.1 dev eth0
```

You should still see `PERMANENT`.

Commit it:

```bash
sudo git add systemd/system/sasm-static-arp-gateway.service
sudo git rm -f udev/rules.d/90-sasm-static-arp-gateway.rules 2>/dev/null || true
sudo git commit -m "Apply static ARP entry at boot"
```

Install it if needed and create the script directory:

```bash
sudo apt update
sudo apt install -y networkd-dispatcher
sudo mkdir -p /etc/networkd-dispatcher/routable.d
```

Create a dispatcher script:

```bash
sudo nano /etc/networkd-dispatcher/routable.d/50-static-arp-gateway
```

Put this in the file:

```sh
#!/bin/sh
[ "$IFACE" = "eth0" ] || exit 0
/etc/sasm/static-arp-gateway
```

Save:

```text
CTRL+O, Enter, CTRL+X
```

Enable it:

```bash
sudo chmod 755 /etc/networkd-dispatcher/routable.d/50-static-arp-gateway
sudo test -x /etc/networkd-dispatcher/routable.d/50-static-arp-gateway
sudo systemctl enable --now networkd-dispatcher.service
```

Commit it:

```bash
sudo git add networkd-dispatcher/routable.d/50-static-arp-gateway
sudo git commit -m "Reapply static ARP when networkd marks eth0 routable"
```

## 5. Allow `check` to run only the required commands

Check the path of `ip`:

```bash
command -v ip
```

Usually this shows:

```text
/usr/sbin/ip
```

Open the sudoers file:

```bash
sudo EDITOR=nano visudo -f /etc/sudoers.d/sasm-arp-check
```

Put this in the file:

```sudoers
check ALL=(root) NOPASSWD: /usr/sbin/ip link set dev eth0 up, /usr/sbin/ip link set dev eth0 down
```

Save:

```text
CTRL+O, Enter, CTRL+X
```

Set permissions and validate:

```bash
sudo chmod 440 /etc/sudoers.d/sasm-arp-check
sudo visudo -cf /etc/sudoers.d/sasm-arp-check
sudo -l -U check
```

You should see that `check` may run only:

```text
/usr/sbin/ip link set dev eth0 up
/usr/sbin/ip link set dev eth0 down
```

Commit it:

```bash
sudo git add sudoers.d/sasm-arp-check
sudo git commit -m "Allow check user to toggle eth0 for ARP test"
```

## 6. Final test

Only do this from the console if SSH uses `eth0`, because it can disconnect you.

```bash
sudo ip link set dev eth0 down
sudo ip link set dev eth0 up
sleep 2
ip neigh show 192.168.1.1 dev eth0
```

You should see:

```text
192.168.1.1 dev eth0 lladdr aa:bb:cc:dd:ee:ff PERMANENT
```

Commit the test:

```bash
sudo git commit --allow-empty -m "Validate static ARP after eth0 restart"
```

After reboot, check again:

```bash
ip neigh show 192.168.1.1 dev eth0
```

It should still show `PERMANENT`.
