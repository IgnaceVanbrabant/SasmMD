# SaSM Lab: Change an Existing Apache Virtual Host

## Goal of this guide

This guide teaches you how to safely change an Apache virtual host that already
exists.

Example change:

```text
old hostname: www3.example.com
new hostname: blog.example.com
```

By the end, you should be able to:

- Find the existing vhost configuration file.
- Back up the current vhost before editing it.
- Change `ServerName`, `ServerAlias`, `DocumentRoot`, logs, or directory rules.
- Move or create the matching website files.
- Test the Apache configuration before reloading.
- Verify that Apache serves the changed vhost.
- Roll back to the previous vhost if something goes wrong.

> Important: this guide uses Apache on Ubuntu/Debian. Replace
> `www3.example.com`, `blog.example.com`, and `YOUR_SERVER_IP` with the real
> names and IP address for your lab.

## 1. Understand what you are changing

An Apache vhost usually has two parts:

```text
/etc/apache2/sites-available/www3.example.com.conf
/var/www/www3.example.com/public_html
```

The first path is the Apache configuration file. The second path is the
document root, where the website files live.

Common changes:

| Change | File or directory to edit |
| --- | --- |
| Change the hostname | Vhost config, DNS or `/etc/hosts` |
| Add another hostname | Vhost config, `ServerAlias` |
| Change the website folder | Vhost config, document root directory |
| Change access rules | Vhost config, `<Directory ...>` block |
| Change logs | Vhost config, `ErrorLog` and `CustomLog` |
| Change page content | Files inside the document root |

## 2. Find the existing vhost

List the enabled Apache vhosts:

```bash
sudo apache2ctl -S
```

Look for the vhost you want to change:

```text
port 80 namevhost www3.example.com (/etc/apache2/sites-enabled/www3.example.com.conf:1)
```

The enabled file is usually a symlink to `sites-available`.

Check it:

```bash
ls -l /etc/apache2/sites-enabled/www3.example.com.conf
```

Expected result:

```text
www3.example.com.conf -> ../sites-available/www3.example.com.conf
```

Edit the real file in:

```text
/etc/apache2/sites-available/
```

## 3. Back up the current vhost

Before changing the file, create a backup:

```bash
sudo cp /etc/apache2/sites-available/www3.example.com.conf \
    /etc/apache2/sites-available/www3.example.com.conf.backup
```

If you will change the document root too, back up the website files:

```bash
sudo cp -a /var/www/www3.example.com /var/www/www3.example.com.backup
```

What this does:

- Keeps a copy of the old Apache config.
- Keeps a copy of the old website files.
- Makes rollback easier if the new vhost does not work.

## 4. Make sure the new hostname points to the server

If you are changing the vhost hostname, Apache can only serve the new hostname
when the name resolves to your server IP address.

With real DNS, add or change an `A` record:

```text
blog.example.com.  IN  A  YOUR_SERVER_IP
```

For local testing, edit the client computer's `/etc/hosts` file:

```bash
sudo nano /etc/hosts
```

Add:

```text
YOUR_SERVER_IP blog.example.com
```

Test resolution:

```bash
dig blog.example.com
```

Expected result:

```text
blog.example.com.  ...  IN  A  YOUR_SERVER_IP
```

## 5. Prepare the new document root

If the vhost will serve a different folder, create it:

```bash
sudo mkdir -p /var/www/blog.example.com/public_html
```

Copy the old website files if you want to keep the same content:

```bash
sudo cp -a /var/www/www3.example.com/public_html/. \
    /var/www/blog.example.com/public_html/
```

Or create a new test page:

```bash
sudo nano /var/www/blog.example.com/public_html/index.html
```

Example content:

```html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="utf-8">
    <title>blog.example.com</title>
  </head>
  <body>
    <h1>Changed virtual host works</h1>
    <p>This page is served by blog.example.com.</p>
  </body>
</html>
```

Set safe permissions:

```bash
sudo chown -R root:root /var/www/blog.example.com
sudo chmod -R 755 /var/www/blog.example.com
```

## 6. Edit the existing vhost file

Open the existing vhost file:

```bash
sudo nano /etc/apache2/sites-available/www3.example.com.conf
```

Old example:

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

Changed example:

```apache
<VirtualHost *:80>
    ServerName blog.example.com
    ServerAlias blog

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

Save the file.

What changed:

| Old value | New value |
| --- | --- |
| `ServerName www3.example.com` | `ServerName blog.example.com` |
| `ServerAlias www3` | `ServerAlias blog` |
| `/var/www/www3.example.com/public_html` | `/var/www/blog.example.com/public_html` |
| `www3.example.com-error.log` | `blog.example.com-error.log` |
| `www3.example.com-access.log` | `blog.example.com-access.log` |

> Note: you may keep the old filename, but renaming it to match the new hostname
> makes the configuration easier to understand.

## 7. Optional: rename the vhost config file

If you want the config filename to match the new hostname, disable the old name:

```bash
sudo a2dissite www3.example.com.conf
```

Rename the file:

```bash
sudo mv /etc/apache2/sites-available/www3.example.com.conf \
    /etc/apache2/sites-available/blog.example.com.conf
```

Enable the renamed file:

```bash
sudo a2ensite blog.example.com.conf
```

What this does:

- Removes the old symlink from `sites-enabled`.
- Renames the config in `sites-available`.
- Creates a new symlink in `sites-enabled`.

## 8. Test before reloading Apache

Always test the Apache syntax before applying changes:

```bash
sudo apache2ctl configtest
```

Expected result:

```text
Syntax OK
```

If you see an error, do not reload Apache yet. Edit the file shown in the error
message and run the config test again.

## 9. Reload Apache

Apply the changed vhost:

```bash
sudo systemctl reload apache2
```

Use `reload` instead of `restart` when possible. It applies the new
configuration without fully stopping Apache.

## 10. Verify the changed vhost

Check which vhosts Apache loaded:

```bash
sudo apache2ctl -S
```

Look for:

```text
port 80 namevhost blog.example.com (/etc/apache2/sites-enabled/blog.example.com.conf:1)
```

Test with a Host header:

```bash
curl -H "Host: blog.example.com" http://127.0.0.1/
```

Expected result:

```html
<h1>Changed virtual host works</h1>
```

Test by hostname:

```bash
curl http://blog.example.com/
```

Open it in a browser:

```text
http://blog.example.com/
```

## 11. Check the logs

Watch the new access log:

```bash
sudo tail -f /var/log/apache2/blog.example.com-access.log
```

In another terminal, request the site:

```bash
curl http://blog.example.com/
```

You should see a new access log line.

If the page does not work, check the error log:

```bash
sudo tail -f /var/log/apache2/blog.example.com-error.log
```

## 12. Roll back if needed

If the changed vhost does not work and you need to restore the old one, disable
the changed config:

```bash
sudo a2dissite blog.example.com.conf
```

Restore the backup:

```bash
sudo cp /etc/apache2/sites-available/www3.example.com.conf.backup \
    /etc/apache2/sites-available/www3.example.com.conf
```

Enable the old config:

```bash
sudo a2ensite www3.example.com.conf
```

Restore the old website files if needed:

```bash
sudo rm -rf /var/www/www3.example.com
sudo cp -a /var/www/www3.example.com.backup /var/www/www3.example.com
```

Test and reload:

```bash
sudo apache2ctl configtest
sudo systemctl reload apache2
```

## 13. Common problems and fixes

### Problem: Apache still serves the old hostname

Check:

```bash
sudo apache2ctl -S
```

Possible causes:

- The old vhost file is still enabled.
- The changed file was saved in `sites-available` but not enabled.
- Apache was not reloaded after the change.
- DNS or `/etc/hosts` still points users to a different server.

### Problem: You get the Apache default page

Apache did not match the new `ServerName`.

Check:

```bash
dig blog.example.com
sudo apache2ctl -S
```

Make sure the vhost has:

```apache
ServerName blog.example.com
```

Then reload:

```bash
sudo systemctl reload apache2
```

### Problem: You get `403 Forbidden`

Apache can match the vhost, but it cannot serve the document root.

Check:

```bash
ls -ld /var/www/blog.example.com
ls -ld /var/www/blog.example.com/public_html
ls -l /var/www/blog.example.com/public_html
```

Fix common permission issues:

```bash
sudo chmod -R 755 /var/www/blog.example.com
```

Also check that the vhost has:

```apache
Require all granted
```

### Problem: `apache2ctl configtest` fails

Apache found a syntax error.

Run:

```bash
sudo apache2ctl configtest
```

Read the file and line number in the error, edit the vhost file, then run the
test again.

## 14. Quick checklist

Use this checklist whenever you change an existing vhost:

1. Run `sudo apache2ctl -S` to find the current vhost.
2. Back up the vhost config file.
3. Back up the document root if you will change website files.
4. Update DNS or `/etc/hosts` if the hostname changes.
5. Create or move the new document root.
6. Edit `ServerName`, `ServerAlias`, `DocumentRoot`, logs, and `<Directory>`.
7. Optionally rename the config file to match the new hostname.
8. Run `sudo apache2ctl configtest`.
9. Reload Apache with `sudo systemctl reload apache2`.
10. Test with `curl -H "Host: blog.example.com" http://127.0.0.1/`.
11. Check `sudo apache2ctl -S` and the Apache logs if something fails.
