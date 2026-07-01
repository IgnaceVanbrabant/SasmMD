# SaSM Lab: HTTP

## Goal of this lab

In this lab you configure an Apache HTTP webserver for:

```text
ignace-vanbrabant.sasm.uclllabs.be
```

By the end, you should be able to:

- Explain the basic HTTP request and response flow.
- Use tools like `curl`, `tshark`, and Wireshark to inspect HTTP traffic.
- Install Apache and PHP support.
- Configure a default virtual host and named virtual hosts.
- Configure vhost-specific logging.
- Protect a private directory with basic authentication through `.htaccess`.
- Disable directory listing.
- Automate temporary vhost creation with `http_add_vhost`.
- Automatically clean up scripted vhosts older than 4 hours.

> Important: this guide uses Apache on Ubuntu/Debian. Replace
> `ignace-vanbrabant` with your own machine name if your lab machine has a
> different name.

## 1. HTTP theory you need to know

### What HTTP does

HTTP is the protocol used by web browsers and webservers. The client sends a
request, and the server sends a response.

Simple HTTP request:

```http
GET /toupper.php?code=AbCdEfGh123 HTTP/1.1
Host: www2.ignace-vanbrabant.sasm.uclllabs.be
```

Important parts:

- `GET` is the request method.
- `/toupper.php?code=AbCdEfGh123` is the path and query string.
- `Host` tells Apache which virtual host should handle the request.

Simple HTTP response:

```http
HTTP/1.1 200 OK
Content-Type: text/html; charset=UTF-8

ABCDEFGH123
```

Important parts:

- `200 OK` means the request succeeded.
- The headers describe the response.
- The blank line separates headers from the response body.

### Why the Host header matters

Many websites can share one IP address. Apache chooses the correct virtual host
by looking at the `Host` header.

Example:

```text
Host: www1.ignace-vanbrabant.sasm.uclllabs.be -> www1 vhost
Host: www2.ignace-vanbrabant.sasm.uclllabs.be -> www2 vhost
Host: does-not-exist.example           -> default vhost
No Host header, HTTP/1.0 request       -> default vhost
```

The first enabled Apache vhost for an IP and port is the fallback vhost. That is
why the default vhost must be configured carefully.

## 2. Install the required packages

Update package metadata:

```bash
sudo apt-get update
```

What this does:

- Downloads the latest package lists from configured repositories.
- Does not upgrade packages by itself.

Expected result:

```text
Reading package lists... Done
```

Install Apache, PHP, htpasswd tools, DNS tools, and tshark:

```bash
sudo DEBIAN_FRONTEND=noninteractive apt-get install -y apache2 apache2-utils php libapache2-mod-php dnsutils tshark
```

What this does:

- Installs `apache2`, the webserver.
- Installs `apache2-utils`, which provides `htpasswd`.
- Installs `php` and `libapache2-mod-php`, so Apache can run PHP pages.
- Installs `dnsutils`, which provides `dig`.
- Installs `tshark`, the terminal version of Wireshark.

Expected result:

```text
Setting up apache2 ...
Setting up apache2-utils ...
Setting up php ...
Setting up libapache2-mod-php ...
Setting up dnsutils ...
Setting up tshark ...
```

Enable Apache at boot and start it now:

```bash
sudo systemctl enable --now apache2
```

What this does:

- `enable` starts Apache automatically after reboot.
- `--now` starts Apache immediately.

Expected result:

```text
Created symlink /etc/systemd/system/multi-user.target.wants/apache2.service ...
```

Check the service:

```bash
systemctl status apache2 --no-pager
```

What this does:

- Shows whether Apache is running.

Expected result:

```text
Active: active (running)
```


## 3. Check that DNS points to your server

Check the main names:

```bash
dig +short www.ignace-vanbrabant.sasm.uclllabs.be
dig +short www1.ignace-vanbrabant.sasm.uclllabs.be
dig +short www2.ignace-vanbrabant.sasm.uclllabs.be
```

What this does:

- Asks DNS for the IPv4 address of each required hostname.

Expected result:

```text
YOUR_SERVER_IP
YOUR_SERVER_IP
YOUR_SERVER_IP
```

If a hostname gives no output, fix the DNS lab first. The HTTP lab depends on
working DNS.

## 4. Create the required document roots

Create directories:

```bash
sudo mkdir -p /var/www/html/default
sudo mkdir -p /var/www/html/www1.ignace-vanbrabant.sasm.uclllabs.be/private
sudo mkdir -p /var/www/html/www2.ignace-vanbrabant.sasm.uclllabs.be
```

What this does:

- Creates a document root for the default website.
- Creates a document root for `www1`.
- Creates a `private` directory under `www1`.
- Creates a document root for `www2`.

Expected result:

```text
```

No output means the directories were created successfully.

Create the default page:

```bash
echo 'welcome default' | sudo tee /var/www/html/default/index.html
```

What this does:

- Writes an index page for the default vhost.
- The page contains the required word `welcome`.

Expected result:

```text
welcome default
```

Create the `www1` page:

```bash
echo 'www1' | sudo tee /var/www/html/www1.ignace-vanbrabant.sasm.uclllabs.be/index.html
```

What this does:

- Writes the default page for `www1`.
- It contains `www1`.
- It does not contain `welcome`.

Expected result:

```text
www1
```

Create a private test page for `www1`:

```bash
echo 'private www1 page' | sudo tee /var/www/html/www1.ignace-vanbrabant.sasm.uclllabs.be/private/index.html
```

What this does:

- Creates a page that should only be accessible after basic authentication.

Expected result:

```text
private www1 page
```

Create the PHP page for `www2`:

```bash
sudo tee /var/www/html/www2.ignace-vanbrabant.sasm.uclllabs.be/toupper.php >/dev/null <<'PHP'
<?php
$code = $_GET['code'] ?? '';
echo htmlspecialchars(strtoupper($code), ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8');
PHP
```

What this does:

- Creates `/toupper.php`.
- Reads the query parameter named `code`.
- Converts it to uppercase with `strtoupper`.
- Escapes the output safely with `htmlspecialchars`.

Expected result:

```text
```

No output means the file was written successfully.

Set safe ownership and permissions:

```bash
sudo chown -R root:root /var/www/html/default /var/www/html/www1.ignace-vanbrabant.sasm.uclllabs.be /var/www/html/www2.ignace-vanbrabant.sasm.uclllabs.be
sudo find /var/www/html/default /var/www/html/www1.ignace-vanbrabant.sasm.uclllabs.be /var/www/html/www2.ignace-vanbrabant.sasm.uclllabs.be -type d -exec chmod 0755 {} \;
sudo find /var/www/html/default /var/www/html/www1.ignace-vanbrabant.sasm.uclllabs.be /var/www/html/www2.ignace-vanbrabant.sasm.uclllabs.be -type f -exec chmod 0644 {} \;
```

What this does:

- Makes root the owner of the web content.
- Makes directories readable and enterable.
- Makes files readable by Apache.

Expected result:

```text
```

No output means the permissions were applied.


## 5. Configure basic authentication for `www1/private`

Create the password file:

```bash
sudo htpasswd -bc /etc/apache2/.htpasswd-www1-private check ch3ck
```

What this does:

- `-b` reads the password from the command line.
- `-c` creates the password file.
- Creates user `check` with password `ch3ck`.

Expected result:

```text
Adding password for user check
```

Protect the password file:

```bash
sudo chown root:www-data /etc/apache2/.htpasswd-www1-private
sudo chmod 0640 /etc/apache2/.htpasswd-www1-private
```

What this does:

- Keeps the file owned by root.
- Allows Apache, running as group `www-data`, to read it.

Expected result:

```text
```

No output means the permissions were applied.

Create the `.htaccess` file:

```bash
sudo tee /var/www/html/www1.ignace-vanbrabant.sasm.uclllabs.be/private/.htaccess >/dev/null <<'EOF'
AuthType Basic
AuthName "www1 private"
AuthUserFile /etc/apache2/.htpasswd-www1-private
Require user check
EOF
```

What this does:

- Enables HTTP Basic Authentication for the private directory.
- Uses the password file created above.
- Allows only the user `check`.

Expected result:

```text
```

No output means the file was written successfully.


## 6. Configure Apache virtual hosts

Disable the packaged default vhost:

```bash
sudo a2dissite 000-default.conf
```

What this does:

- Disables Apache's default example site.
- Prevents it from catching requests before your lab default vhost.

Expected result:

```text
Site 000-default disabled.
To activate the new configuration, you need to run:
  systemctl reload apache2
```

Create the lab default vhost:

```bash
sudo tee /etc/apache2/sites-available/000-sasm-default.conf >/dev/null <<'EOF'
<VirtualHost *:80>
    ServerName www.ignace-vanbrabant.sasm.uclllabs.be
    ServerAlias *
    DocumentRoot /var/www/html/default

    ErrorLog ${APACHE_LOG_DIR}/ignace-vanbrabant-default-error.log
    CustomLog ${APACHE_LOG_DIR}/ignace-vanbrabant-default-access.log combined

    <Directory /var/www/html/default>
        Options -Indexes
        AllowOverride None
        Require all granted
    </Directory>
</VirtualHost>
EOF
```

What this does:

- Creates the fallback vhost.
- Serves `/var/www/html/default`.
- Logs this vhost to its own access and error logs.
- Disables directory listing with `Options -Indexes`.

Expected result:

```text
```

No output means the file was written successfully.

Create the `www1` vhost:

```bash
sudo tee /etc/apache2/sites-available/www1.ignace-vanbrabant.sasm.uclllabs.be.conf >/dev/null <<'EOF'
<VirtualHost *:80>
    ServerName www1.ignace-vanbrabant.sasm.uclllabs.be
    DocumentRoot /var/www/html/www1.ignace-vanbrabant.sasm.uclllabs.be

    ErrorLog ${APACHE_LOG_DIR}/www1-ignace-vanbrabant-error.log
    CustomLog ${APACHE_LOG_DIR}/www1-ignace-vanbrabant-access.log combined

    <Directory /var/www/html/www1.ignace-vanbrabant.sasm.uclllabs.be>
        Options -Indexes
        AllowOverride AuthConfig
        Require all granted
    </Directory>
</VirtualHost>
EOF
```

What this does:

- Creates a vhost for `www1.ignace-vanbrabant.sasm.uclllabs.be`.
- Allows `.htaccess` authentication rules.
- Keeps directory listing disabled.
- Uses vhost-specific logs.

Expected result:

```text
```

Create the `www2` vhost:

```bash
sudo tee /etc/apache2/sites-available/www2.ignace-vanbrabant.sasm.uclllabs.be.conf >/dev/null <<'EOF'
<VirtualHost *:80>
    ServerName www2.ignace-vanbrabant.sasm.uclllabs.be
    DocumentRoot /var/www/html/www2.ignace-vanbrabant.sasm.uclllabs.be

    ErrorLog ${APACHE_LOG_DIR}/www2-ignace-vanbrabant-error.log
    CustomLog ${APACHE_LOG_DIR}/www2-ignace-vanbrabant-access.log combined

    <Directory /var/www/html/www2.ignace-vanbrabant.sasm.uclllabs.be>
        Options -Indexes
        AllowOverride None
        Require all granted
    </Directory>
</VirtualHost>
EOF
```

What this does:

- Creates a vhost for `www2.ignace-vanbrabant.sasm.uclllabs.be`.
- Serves the PHP `toupper.php` page.
- Disables directory listing.
- Uses vhost-specific logs.

Expected result:

```text
```

Enable the vhosts:

```bash
sudo a2ensite 000-sasm-default.conf
sudo a2ensite www1.ignace-vanbrabant.sasm.uclllabs.be.conf
sudo a2ensite www2.ignace-vanbrabant.sasm.uclllabs.be.conf
```

What this does:

- Creates symlinks from `sites-enabled` to `sites-available`.
- Makes Apache load these vhosts.

Expected result:

```text
Enabling site 000-sasm-default.
Enabling site www1.ignace-vanbrabant.sasm.uclllabs.be.
Enabling site www2.ignace-vanbrabant.sasm.uclllabs.be.
To activate the new configuration, you need to run:
  systemctl reload apache2
```

Check Apache syntax:

```bash
sudo apache2ctl configtest
```

What this does:

- Validates Apache configuration before you reload.

Expected result:

```text
Syntax OK
```

Reload Apache:

```bash
sudo systemctl reload apache2
```

What this does:

- Applies the new configuration without fully stopping Apache.

Expected result:

```text
```

No output means the reload succeeded.


## 7. Verify the required webpages

### Default page

Test the configured default name:

```bash
curl -s -H 'Host: www.ignace-vanbrabant.sasm.uclllabs.be' http://127.0.0.1/
```

What this does:

- Sends a request to local Apache.
- Sets the HTTP `Host` header to the main website name.

Expected result:

```text
welcome default
```

Test a non-existing host header:

```bash
curl -s -H 'Host: not-existing.ignace-vanbrabant.sasm.uclllabs.be' http://127.0.0.1/
```

What this does:

- Confirms unknown names fall back to the default vhost.

Expected result:

```text
welcome default
```

Test an HTTP/1.0 request without a real Host header:

```bash
curl -s --http1.0 -H 'Host:' http://127.0.0.1/
```

What this does:

- Simulates an old HTTP/1.0 client where the Host header is missing.

Expected result:

```text
welcome default
```

### `www1` page

Test `www1`:

```bash
curl -s -H 'Host: www1.ignace-vanbrabant.sasm.uclllabs.be' http://127.0.0.1/
```

What this does:

- Requests the root page of the `www1` vhost.

Expected result:

```text
www1
```

Check that the page does not contain `welcome`:

```bash
curl -s -H 'Host: www1.ignace-vanbrabant.sasm.uclllabs.be' http://127.0.0.1/ | grep -i welcome
```

What this does:

- Searches the `www1` output for the forbidden string.

Expected result:

```text
```

No output means `welcome` was not found.

### `www1/private` authentication

Try without credentials:

```bash
curl -i -H 'Host: www1.ignace-vanbrabant.sasm.uclllabs.be' http://127.0.0.1/private/
```

What this does:

- Requests a protected page without username and password.

Expected result:

```text
HTTP/1.1 401 Unauthorized
WWW-Authenticate: Basic realm="www1 private"
```

Try with the required credentials:

```bash
curl -s -u check:ch3ck -H 'Host: www1.ignace-vanbrabant.sasm.uclllabs.be' http://127.0.0.1/private/
```

What this does:

- Sends HTTP Basic Authentication as user `check`.

Expected result:

```text
private www1 page
```

### `www2` PHP uppercase page

Test the required PHP page:

```bash
curl -s -H 'Host: www2.ignace-vanbrabant.sasm.uclllabs.be' 'http://127.0.0.1/toupper.php?code=AbCdEfGh123'
```

What this does:

- Requests the `toupper.php` page.
- Sends the query parameter `code=AbCdEfGh123`.

Expected result:

```text
ABCDEFGH123
```

Test directory listing:

```bash
curl -i -H 'Host: www2.ignace-vanbrabant.sasm.uclllabs.be' http://127.0.0.1/
```

What this does:

- Requests a directory that has no `index.html`.
- Confirms Apache does not show a file listing.

Expected result:

```text
HTTP/1.1 403 Forbidden
```


## 8. Inspect HTTP traffic with tshark

Open one terminal and start a short capture:

```bash
sudo tshark -i any -f 'tcp port 80' -Y http -T fields -e ip.src -e ip.dst -e http.request.method -e http.host -e http.request.uri
```

What this does:

- Captures packets on all interfaces with `-i any`.
- Captures only TCP port 80 traffic with `-f 'tcp port 80'`.
- Displays only HTTP packets with `-Y http`.
- Prints selected fields instead of full packet details.

Expected result at first:

```text
Capturing on 'any'
```

In another terminal, make a request:

```bash
curl -s -H 'Host: www2.ignace-vanbrabant.sasm.uclllabs.be' 'http://127.0.0.1/toupper.php?code=AbCdEfGh123'
```

Expected `curl` result:

```text
ABCDEFGH123
```

Expected `tshark` line:

```text
127.0.0.1    127.0.0.1    GET    www2.ignace-vanbrabant.sasm.uclllabs.be    /toupper.php?code=AbCdEfGh123
```

Stop `tshark` with `Ctrl+C`.


## 9. Prepare `/etc/scripts` and sudo access

Create the script directory:

```bash
sudo mkdir -p /etc/scripts
```

What this does:

- Creates the required directory for lab scripts.

Expected result:

```text
```

Make sure only root can modify the directory:

```bash
sudo chown root:root /etc/scripts
sudo chmod 0755 /etc/scripts
```

What this does:

- Root owns the directory.
- Other users can enter and run scripts, but cannot write there.

Expected result:

```text
```

Add `/etc/scripts` to the secure sudo path for user `check` and allow only the
HTTP script:

```bash
sudo tee /etc/sudoers.d/sasm-http-check >/dev/null <<'EOF'
Defaults:check secure_path="/etc/scripts:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
check ALL=(root) NOPASSWD: /etc/scripts/http_add_vhost
EOF
sudo chmod 0440 /etc/sudoers.d/sasm-http-check
sudo visudo -cf /etc/sudoers.d/sasm-http-check
```

What this does:

- Allows `check` to run `sudo http_add_vhost`.
- Does not allow `check` to run arbitrary commands as root.
- Validates the sudoers file before relying on it.

Expected result:

```text
/etc/sudoers.d/sasm-http-check: parsed OK
```

## 10. Create `http_add_vhost`

Create the script:

```bash
sudo tee /etc/scripts/http_add_vhost >/dev/null <<'EOF'
#!/usr/bin/env bash
set -euo pipefail

BASE_DOMAIN="ignace-vanbrabant.sasm.uclllabs.be"
SITES_AVAILABLE="/etc/apache2/sites-available"
DOCROOT_BASE="/var/www/html"
LOG_DIR="/var/log/apache2"
AUTO_PREFIX="auto-http"

fail() {
    echo "ERROR: $*" >&2
    exit 1
}

cleanup_old_vhosts() {
    local changed=0
    while IFS= read -r -d '' conf; do
        local file fqdn
        file="$(basename "$conf")"
        fqdn="${file%.conf}"
        fqdn="${fqdn#${AUTO_PREFIX}-}"

        a2dissite "$file" >/dev/null 2>&1 || true
        rm -f "/etc/apache2/sites-enabled/$file"
        rm -f "$conf"
        rm -rf "$DOCROOT_BASE/$fqdn"
        rm -f "$LOG_DIR/${AUTO_PREFIX}-${fqdn}-access.log"
        rm -f "$LOG_DIR/${AUTO_PREFIX}-${fqdn}-error.log"
        changed=1
    done < <(find "$SITES_AVAILABLE" -maxdepth 1 -type f -name "${AUTO_PREFIX}-*.conf" -mmin +240 -print0)

    if [ "$changed" -eq 1 ]; then
        systemctl reload apache2
    fi
}

[ "${EUID}" -eq 0 ] || fail "run this script with sudo"
[ "${SUDO_USER:-}" = "check" ] || fail "only user check may run this script through sudo"
[ "$#" -eq 1 ] || fail "usage: sudo http_add_vhost fqdn"

FQDN="$1"

[[ "$FQDN" =~ ^[a-z0-9]([a-z0-9-]{0,61}[a-z0-9])?(\.[a-z0-9]([a-z0-9-]{0,61}[a-z0-9])?)*$ ]] \
    || fail "invalid fqdn: $FQDN"
[[ "$FQDN" == *".${BASE_DOMAIN}" ]] \
    || fail "fqdn must be below ${BASE_DOMAIN}"

if ! getent ahosts "$FQDN" >/dev/null; then
    fail "domain does not exist in DNS: $FQDN"
fi

cleanup_old_vhosts

CONF_FILE="$SITES_AVAILABLE/${AUTO_PREFIX}-${FQDN}.conf"
DOCROOT="$DOCROOT_BASE/$FQDN"

[ ! -e "$CONF_FILE" ] || fail "vhost already exists: $FQDN"
[ ! -e "$DOCROOT" ] || fail "document root already exists: $DOCROOT"

mkdir -p "$DOCROOT"
echo "welcome $FQDN" > "$DOCROOT/index.html"
chown -R root:root "$DOCROOT"
chmod 0755 "$DOCROOT"
chmod 0644 "$DOCROOT/index.html"

cat > "$CONF_FILE" <<APACHE
<VirtualHost *:80>
    ServerName $FQDN
    DocumentRoot $DOCROOT

    ErrorLog \${APACHE_LOG_DIR}/${AUTO_PREFIX}-${FQDN}-error.log
    CustomLog \${APACHE_LOG_DIR}/${AUTO_PREFIX}-${FQDN}-access.log combined

    <Directory $DOCROOT>
        Options -Indexes
        AllowOverride None
        Require all granted
    </Directory>
</VirtualHost>
APACHE

chmod 0644 "$CONF_FILE"
a2ensite "$(basename "$CONF_FILE")" >/dev/null
apache2ctl configtest >/dev/null
systemctl reload apache2

echo "created vhost $FQDN"
echo "document root: $DOCROOT"
echo "config: $CONF_FILE"
EOF
```

What this does:

- Creates `/etc/scripts/http_add_vhost`.
- Requires the script to run as root through `sudo`.
- Allows only `SUDO_USER=check`.
- Accepts exactly one argument: the FQDN.
- Rejects invalid names.
- Rejects names outside `ignace-vanbrabant.sasm.uclllabs.be`.
- Rejects names that do not exist in DNS.
- Creates the document root.
- Creates `index.html` containing exactly `welcome vhost_name`.
- Creates vhost-specific logging.
- Enables the vhost.
- Reloads Apache.
- Cleans old generated vhosts before adding a new one.

Expected result:

```text
```

Make the script executable but editable only by root:

```bash
sudo chown root:root /etc/scripts/http_add_vhost
sudo chmod 0755 /etc/scripts/http_add_vhost
```

What this does:

- Root owns the script.
- Everyone can read and execute it.
- Only root can edit it.

Expected result:

```text
```

Check the script syntax:

```bash
sudo bash -n /etc/scripts/http_add_vhost
```

What this does:

- Checks for Bash syntax errors without running the script.

Expected result:

```text
```

No output means the syntax is valid.


## 11. Verify `http_add_vhost`

First make sure the test name exists in DNS. Replace `subdomain` with a
subdomain that your DNS script created:

```bash
dig +short subdomain.ignace-vanbrabant.sasm.uclllabs.be
```

Expected result:

```text
YOUR_SERVER_IP
```

Switch to the `check` user:

```bash
su - check
```

What this does:

- Opens a shell as the `check` user.

Expected result:

```text
check@your-server:~$
```

Run the script exactly as required:

```bash
sudo http_add_vhost subdomain.ignace-vanbrabant.sasm.uclllabs.be
```

What this does:

- Runs `/etc/scripts/http_add_vhost` as root through sudo.
- Creates and enables the vhost.

Expected result:

```text
created vhost subdomain.ignace-vanbrabant.sasm.uclllabs.be
document root: /var/www/html/subdomain.ignace-vanbrabant.sasm.uclllabs.be
config: /etc/apache2/sites-available/auto-http-subdomain.ignace-vanbrabant.sasm.uclllabs.be.conf
```

Test the new vhost:

```bash
curl -s -H 'Host: subdomain.ignace-vanbrabant.sasm.uclllabs.be' http://127.0.0.1/
```

Expected result:

```text
welcome subdomain.ignace-vanbrabant.sasm.uclllabs.be
```

Try a name that does not exist in DNS:

```bash
sudo http_add_vhost does-not-exist.ignace-vanbrabant.sasm.uclllabs.be
```

Expected result:

```text
ERROR: domain does not exist in DNS: does-not-exist.ignace-vanbrabant.sasm.uclllabs.be
```

Leave the `check` shell:

```bash
exit
```


## 12. Add automatic cleanup

The script above already calls `cleanup_old_vhosts` each time it runs. Add a
cron job too, so cleanup happens even if Yoda stops calling the script.

Create a cleanup script:

```bash
sudo tee /etc/scripts/http_cleanup_vhosts >/dev/null <<'EOF'
#!/usr/bin/env bash
set -euo pipefail

SITES_AVAILABLE="/etc/apache2/sites-available"
DOCROOT_BASE="/var/www/html"
LOG_DIR="/var/log/apache2"
AUTO_PREFIX="auto-http"
changed=0

while IFS= read -r -d '' conf; do
    file="$(basename "$conf")"
    fqdn="${file%.conf}"
    fqdn="${fqdn#${AUTO_PREFIX}-}"

    a2dissite "$file" >/dev/null 2>&1 || true
    rm -f "/etc/apache2/sites-enabled/$file"
    rm -f "$conf"
    rm -rf "$DOCROOT_BASE/$fqdn"
    rm -f "$LOG_DIR/${AUTO_PREFIX}-${fqdn}-access.log"
    rm -f "$LOG_DIR/${AUTO_PREFIX}-${fqdn}-error.log"
    changed=1
done < <(find "$SITES_AVAILABLE" -maxdepth 1 -type f -name "${AUTO_PREFIX}-*.conf" -mmin +240 -print0)

if [ "$changed" -eq 1 ]; then
    systemctl reload apache2
fi
EOF
```

What this does:

- Finds generated vhost configs older than 240 minutes, which is 4 hours.
- Disables each old site.
- Removes the config file.
- Removes the matching document root.
- Removes the matching access and error logs.
- Reloads Apache only if something changed.

Expected result:

```text
```

Set permissions:

```bash
sudo chown root:root /etc/scripts/http_cleanup_vhosts
sudo chmod 0755 /etc/scripts/http_cleanup_vhosts
sudo bash -n /etc/scripts/http_cleanup_vhosts
```

Expected result:

```text
```

Create the cron job:

```bash
sudo tee /etc/cron.d/sasm-http-vhost-cleanup >/dev/null <<'EOF'
*/15 * * * * root /etc/scripts/http_cleanup_vhosts
EOF
sudo chmod 0644 /etc/cron.d/sasm-http-vhost-cleanup
```

What this does:

- Runs cleanup as root every 15 minutes.
- Keeps generated vhosts from staying around longer than required.

Expected result:

```text
```


## 13. Final verification checklist

Run these commands before marking the lab done.

Apache config:

```bash
sudo apache2ctl configtest
```

Expected result:

```text
Syntax OK
```

Default vhost:

```bash
curl -s -H 'Host: www.ignace-vanbrabant.sasm.uclllabs.be' http://127.0.0.1/
curl -s -H 'Host: fake.example' http://127.0.0.1/
curl -s --http1.0 -H 'Host:' http://127.0.0.1/
```

Expected result:

```text
welcome default
welcome default
welcome default
```

`www1`:

```bash
curl -s -H 'Host: www1.ignace-vanbrabant.sasm.uclllabs.be' http://127.0.0.1/
curl -i -H 'Host: www1.ignace-vanbrabant.sasm.uclllabs.be' http://127.0.0.1/private/
curl -s -u check:ch3ck -H 'Host: www1.ignace-vanbrabant.sasm.uclllabs.be' http://127.0.0.1/private/
```

Expected result:

```text
www1
HTTP/1.1 401 Unauthorized
private www1 page
```

`www2`:

```bash
curl -s -H 'Host: www2.ignace-vanbrabant.sasm.uclllabs.be' 'http://127.0.0.1/toupper.php?code=AbCdEfGh123'
curl -i -H 'Host: www2.ignace-vanbrabant.sasm.uclllabs.be' http://127.0.0.1/
```

Expected result:

```text
ABCDEFGH123
HTTP/1.1 403 Forbidden
```

Script permission check:

```bash
sudo -u check sudo -n -l
```

Expected result:

```text
User check may run the following commands on your-server:
    (root) NOPASSWD: /etc/scripts/http_add_vhost
```

Script execution check:

```bash
su - check
sudo http_add_vhost subdomain.ignace-vanbrabant.sasm.uclllabs.be
curl -s -H 'Host: subdomain.ignace-vanbrabant.sasm.uclllabs.be' http://127.0.0.1/
exit
```

Expected result:

```text
created vhost subdomain.ignace-vanbrabant.sasm.uclllabs.be
welcome subdomain.ignace-vanbrabant.sasm.uclllabs.be
```

Old generated vhost cleanup check:

```bash
sudo touch -d '5 hours ago' /etc/apache2/sites-available/auto-http-subdomain.ignace-vanbrabant.sasm.uclllabs.be.conf
sudo /etc/scripts/http_cleanup_vhosts
test ! -e /etc/apache2/sites-available/auto-http-subdomain.ignace-vanbrabant.sasm.uclllabs.be.conf && echo removed
```

Expected result:

```text
removed
```

## 14. Troubleshooting

### Apache reload fails

Run:

```bash
sudo apache2ctl configtest
```

Expected bad result:

```text
Syntax error on line ...
```

Fix the mentioned file and line, then run:

```bash
sudo systemctl reload apache2
```

### Wrong vhost is shown

Check enabled sites:

```bash
ls -l /etc/apache2/sites-enabled/
```

What to look for:

- `000-sasm-default.conf` should be enabled.
- `000-default.conf` should be disabled.
- The default vhost should alphabetically load before the other lab vhosts.

### PHP code is downloaded instead of executed

Run:

```bash
PHP_MODULE="php$(php -r 'echo PHP_MAJOR_VERSION.".".PHP_MINOR_VERSION;')"
sudo a2enmod "$PHP_MODULE"
sudo systemctl restart apache2
```

Expected result:

```text
Considering dependency ...
Module php8.3 already enabled
```

If this does not work, reinstall PHP support:

```bash
sudo apt-get install -y php libapache2-mod-php
sudo systemctl restart apache2
```

### `.htaccess` is ignored

Check the `www1` vhost has:

```apache
AllowOverride AuthConfig
```

Then reload:

```bash
sudo systemctl reload apache2
```

### `sudo http_add_vhost` says command not found

Check the sudoers secure path:

```bash
sudo visudo -cf /etc/sudoers.d/sasm-http-check
sudo -u check sudo -n env | grep '^PATH='
```

Expected result:

```text
/etc/sudoers.d/sasm-http-check: parsed OK
PATH=/etc/scripts:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
```

### Script refuses a valid vhost

Check DNS:

```bash
getent ahosts subdomain.ignace-vanbrabant.sasm.uclllabs.be
dig +short subdomain.ignace-vanbrabant.sasm.uclllabs.be
```

Expected result:

```text
YOUR_SERVER_IP ...
YOUR_SERVER_IP
```

If there is no output, create the DNS record first with the DNS lab script.

