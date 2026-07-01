# SaSM Lab: Apache Virtual Hosts

## Goal of this guide

This guide teaches you how to create an Apache virtual host for any site name,
for example:

```text
www3.example.com
```

By the end, you should be able to:

- Explain what a virtual host is.
- Point a hostname such as `www3.example.com` to your server.
- Create a document root for a website.
- Create an Apache vhost configuration file.
- Enable, test, reload, and troubleshoot the vhost.
- Repeat the same steps for other names such as `www4`, `blog`, or `test`.

> Important: this guide uses Apache on Ubuntu/Debian. Replace
> `www3.example.com` with the real hostname you want to configure, and replace
> `YOUR_SERVER_IP` with your server's IPv4 address.
>
> If you need to change a vhost that already exists, use
> [Change an Existing Apache Virtual Host](sasm-lab-vhost-change.md).

## 1. What is a virtual host?

A virtual host, often shortened to **vhost**, lets one webserver host multiple
websites.

Example:

```text
www1.example.com -> /var/www/www1.example.com
www2.example.com -> /var/www/www2.example.com
www3.example.com -> /var/www/www3.example.com
```

All three names can point to the same server IP address. Apache decides which
website to show by looking at the HTTP `Host` header.

Example request:

```http
GET / HTTP/1.1
Host: www3.example.com
```

When Apache receives this request, it searches its enabled vhost files for:

```apache
ServerName www3.example.com
```

If it finds a match, that vhost handles the request.

## 2. Choose the hostname

Decide which name you want to create.

Examples:

```text
www3.example.com
blog.example.com
test.example.com
shop.example.com
```

In the commands below, this guide uses:

```text
www3.example.com
```

If you want another name, replace `www3.example.com` everywhere.

## 3. Make sure the hostname points to your server

Apache can only receive traffic for the hostname if the name resolves to your
server IP address.

There are two common ways to do this.

### Option A: Use real DNS

If you control the DNS zone, add an `A` record:

```text
www3.example.com.  IN  A  YOUR_SERVER_IP
```

Example:

```text
www3.example.com.  IN  A  203.0.113.10
```

Then test it:

```bash
dig www3.example.com
```

Expected result:

```text
www3.example.com.  ...  IN  A  203.0.113.10
```

### Option B: Use `/etc/hosts` for local testing

If you do not have real DNS yet, add the hostname to the client computer's
`/etc/hosts` file.

Open the file:

```bash
sudo nano /etc/hosts
```

Add a line:

```text
YOUR_SERVER_IP www3.example.com
```

Example:

```text
203.0.113.10 www3.example.com
```

Save the file, then test:

```bash
ping -c 3 www3.example.com
```

The name should resolve to your server IP address.

> Note: editing `/etc/hosts` only affects the computer where you changed the
> file. Other computers still need DNS or their own `/etc/hosts` entry.

## 4. Install Apache

Update package information:

```bash
sudo apt update
```

Install Apache:

```bash
sudo apt install apache2
```

Check that Apache is running:

```bash
systemctl status apache2
```

You should see:

```text
active (running)
```

If Apache is not running, start it:

```bash
sudo systemctl start apache2
```

Enable Apache at boot:

```bash
sudo systemctl enable apache2
```

## 5. Create the document root

The **document root** is the directory where the website files live.

Create a directory for the site:

```bash
sudo mkdir -p /var/www/www3.example.com/public_html
```

Give your user ownership while you build the site:

```bash
sudo chown -R "$USER":"$USER" /var/www/www3.example.com
```

Set safe directory permissions:

```bash
sudo chmod -R 755 /var/www/www3.example.com
```

Create a test page:

```bash
nano /var/www/www3.example.com/public_html/index.html
```

Add this content:

```html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="utf-8">
    <title>www3.example.com</title>
  </head>
  <body>
    <h1>Virtual host works</h1>
    <p>This page is served by www3.example.com.</p>
  </body>
</html>
```

Save the file.

## 6. Create the Apache vhost file

Apache vhost files are usually stored in:

```text
/etc/apache2/sites-available/
```

Create a new config file:

```bash
sudo nano /etc/apache2/sites-available/www3.example.com.conf
```

Add this configuration:

```apache
<VirtualHost *:80>
    ServerName www3.example.com
    ServerAlias www3

    DocumentRoot /var/www/www3.example.com/public_html

    ErrorLog ${APACHE_LOG_DIR}/www3.example.com-error.log
    CustomLog ${APACHE_LOG_DIR}/www3.example.com-access.log combined

    <Directory /var/www/www3.example.com/public_html>
        Options -Indexes +FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>
</VirtualHost>
```

What the important lines mean:

| Line | Meaning |
| --- | --- |
| `<VirtualHost *:80>` | Listen for HTTP traffic on port 80. |
| `ServerName www3.example.com` | The main hostname for this vhost. |
| `ServerAlias www3` | Optional extra hostname that also matches. |
| `DocumentRoot ...` | Directory Apache serves for this hostname. |
| `ErrorLog ...` | Error log for this vhost. |
| `CustomLog ... combined` | Access log for this vhost. |
| `Options -Indexes` | Prevents visitors from seeing directory listings. |
| `AllowOverride All` | Allows `.htaccess` files if you need them. |
| `Require all granted` | Allows visitors to access the directory. |

## 7. Enable the vhost

Enable the new site:

```bash
sudo a2ensite www3.example.com.conf
```

If you do not want Apache's default site to answer unknown requests, disable it:

```bash
sudo a2dissite 000-default.conf
```

Check the Apache configuration:

```bash
sudo apache2ctl configtest
```

Expected result:

```text
Syntax OK
```

Reload Apache:

```bash
sudo systemctl reload apache2
```

Use `reload` when possible because it applies configuration changes without
fully stopping Apache.

## 8. Test the vhost

Test from the server itself:

```bash
curl -H "Host: www3.example.com" http://127.0.0.1/
```

Expected result:

```html
<h1>Virtual host works</h1>
```

Test by hostname:

```bash
curl http://www3.example.com/
```

Open it in a browser:

```text
http://www3.example.com/
```

You should see the test page.

## 9. Check which vhosts Apache knows

List enabled vhosts:

```bash
sudo apache2ctl -S
```

Look for a line like:

```text
port 80 namevhost www3.example.com (/etc/apache2/sites-enabled/www3.example.com.conf:1)
```

This confirms Apache loaded the vhost.

## 10. Read the logs

Watch the access log:

```bash
sudo tail -f /var/log/apache2/www3.example.com-access.log
```

In another terminal, request the site:

```bash
curl http://www3.example.com/
```

You should see a new access log line.

Watch the error log:

```bash
sudo tail -f /var/log/apache2/www3.example.com-error.log
```

Use the error log when a page gives an error or Apache cannot read a file.

## 11. Make another vhost

To make another vhost, repeat the same pattern with a new name.

Example for:

```text
blog.example.com
```

Create the document root:

```bash
sudo mkdir -p /var/www/blog.example.com/public_html
sudo chown -R "$USER":"$USER" /var/www/blog.example.com
sudo chmod -R 755 /var/www/blog.example.com
```

Create the test page:

```bash
nano /var/www/blog.example.com/public_html/index.html
```

Create the vhost file:

```bash
sudo nano /etc/apache2/sites-available/blog.example.com.conf
```

Use this config:

```apache
<VirtualHost *:80>
    ServerName blog.example.com

    DocumentRoot /var/www/blog.example.com/public_html

    ErrorLog ${APACHE_LOG_DIR}/blog.example.com-error.log
    CustomLog ${APACHE_LOG_DIR}/blog.example.com-access.log combined

    <Directory /var/www/blog.example.com/public_html>
        Options -Indexes +FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>
</VirtualHost>
```

Enable and reload:

```bash
sudo a2ensite blog.example.com.conf
sudo apache2ctl configtest
sudo systemctl reload apache2
```

Make sure DNS or `/etc/hosts` also points `blog.example.com` to your server.

## 12. Useful template

Copy this template whenever you need a new Apache vhost:

```apache
<VirtualHost *:80>
    ServerName SITE_NAME_HERE
    ServerAlias OPTIONAL_ALIAS_HERE

    DocumentRoot /var/www/SITE_NAME_HERE/public_html

    ErrorLog ${APACHE_LOG_DIR}/SITE_NAME_HERE-error.log
    CustomLog ${APACHE_LOG_DIR}/SITE_NAME_HERE-access.log combined

    <Directory /var/www/SITE_NAME_HERE/public_html>
        Options -Indexes +FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>
</VirtualHost>
```

Replace:

```text
SITE_NAME_HERE -> www3.example.com
OPTIONAL_ALIAS_HERE -> www3
```

If you do not need an alias, remove the `ServerAlias` line.

## 13. Common problems and fixes

### Problem: The browser shows the wrong website

Possible causes:

- The hostname does not resolve to your server.
- `ServerName` is misspelled.
- The vhost is not enabled.
- Apache was not reloaded after the change.

Check:

```bash
dig www3.example.com
sudo apache2ctl -S
sudo systemctl reload apache2
```

### Problem: `apache2ctl configtest` does not say `Syntax OK`

Apache found a mistake in a config file.

Check the line number shown in the error, then edit the file:

```bash
sudo nano /etc/apache2/sites-available/www3.example.com.conf
```

Run the test again:

```bash
sudo apache2ctl configtest
```

Only reload Apache after the syntax is fixed.

### Problem: You get `403 Forbidden`

Apache can see the vhost, but it cannot serve the directory.

Check:

```bash
ls -ld /var/www/www3.example.com
ls -ld /var/www/www3.example.com/public_html
ls -l /var/www/www3.example.com/public_html
```

Fix common permission issues:

```bash
sudo chmod -R 755 /var/www/www3.example.com
```

Also make sure the vhost has:

```apache
Require all granted
```

### Problem: You get the Apache default page

Apache is probably using the default vhost instead of your new one.

Check:

```bash
sudo apache2ctl -S
```

Make sure your file appears under `sites-enabled`.

Enable it if needed:

```bash
sudo a2ensite www3.example.com.conf
sudo systemctl reload apache2
```

### Problem: `curl http://www3.example.com/` cannot connect

Check that Apache is running:

```bash
systemctl status apache2
```

Check that port 80 is listening:

```bash
sudo ss -tulpn | grep ':80'
```

If a firewall is enabled, allow HTTP:

```bash
sudo ufw allow 80/tcp
```

## 14. Remove a vhost

Disable the site:

```bash
sudo a2dissite www3.example.com.conf
```

Reload Apache:

```bash
sudo systemctl reload apache2
```

Remove the config file if you no longer need it:

```bash
sudo rm /etc/apache2/sites-available/www3.example.com.conf
```

Remove the website files if you no longer need them:

```bash
sudo rm -rf /var/www/www3.example.com
```

Remove the DNS record or `/etc/hosts` line so the hostname no longer points to
the server.

## 15. Quick checklist

Use this checklist every time you create a vhost:

1. Choose the hostname, for example `www3.example.com`.
2. Point the hostname to your server with DNS or `/etc/hosts`.
3. Create `/var/www/www3.example.com/public_html`.
4. Add an `index.html` test page.
5. Create `/etc/apache2/sites-available/www3.example.com.conf`.
6. Set `ServerName www3.example.com`.
7. Set `DocumentRoot /var/www/www3.example.com/public_html`.
8. Run `sudo a2ensite www3.example.com.conf`.
9. Run `sudo apache2ctl configtest`.
10. Reload Apache with `sudo systemctl reload apache2`.
11. Test with `curl http://www3.example.com/`.
12. Check logs if something does not work.

## 16. One-command test summary

After setup, these commands should all work:

```bash
dig www3.example.com
sudo apache2ctl -S
sudo apache2ctl configtest
curl -I http://www3.example.com/
curl http://www3.example.com/
```

Expected signs of success:

- DNS returns your server IP.
- `apache2ctl -S` lists `www3.example.com`.
- `configtest` says `Syntax OK`.
- `curl -I` returns `HTTP/1.1 200 OK`.
- `curl` shows your test page.
