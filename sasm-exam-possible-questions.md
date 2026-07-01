# SaSM Exam: Possible Questions

This file reconstructs possible exam questions from the provided answers. The
exact original questions may have been worded differently, but these prompts fit
the commands, configuration, and tests shown in the answer notes.

## Question 1: IPv6 ICMP traffic analysis

You are connected to a server network on interface `eth0`.

Capture ICMPv6 traffic for 60 seconds and find the unique ICMPv6 packets where
the IPv6 payload length is exactly `18` bytes.

Your answer should:

- Use `tshark` on interface `eth0`.
- Capture only ICMPv6 traffic.
- Display only packets matching `ipv6.plen == 18`.
- Print these fields:
  - ICMPv6 type
  - ICMPv6 code
  - ICMPv6 checksum
  - ICMPv6 data
- Sort the unique results in reverse order.
- Report the observed output.

Possible command expected:

```bash
tshark -a duration:60 -i eth0 -f "icmp6" -Y "ipv6.plen == 18" -T fields -e icmpv6.type -e icmpv6.code -e icmpv6.checksum -e icmpv6.data | sort -u -r
```

## Question 2: DNS, Apache virtual host, HTTPS, and logging

Configure a new exam website for:

```text
exam.2025-2026.matteo-knops.sasm.uclllabs.be
```

The website must resolve to:

```text
193.191.176.50
```

Your answer should include the following tasks.

### 2.1 DNS

Add the required DNS `A` record dynamically with `nsupdate`.

Requirements:

- Use the TSIG key at `/etc/bind/ddns.key`.
- Send the update to DNS server `127.0.0.1`.
- Update the zone `matteo-knops.sasm.uclllabs.be.`.
- Add the record with TTL `3600`.

### 2.2 Web content

Create a document root for the exam site.

Requirements:

- Directory: `/var/www/html/exam2025-2026`
- Homepage: `/var/www/html/exam2025-2026/index.html`
- Homepage content:

```text
exam.2025-2026
```

### 2.3 Apache virtual host

Create an Apache virtual host configuration for the new exam domain.

Requirements for both HTTP and HTTPS:

- `ServerName` must be `exam.2025-2026.matteo-knops.sasm.uclllabs.be`.
- `DocumentRoot` must point to `/var/www/html/exam2025-2026`.
- Every response must include this header:

```http
X-Favorite-Course: SaSM
```

- Use a dedicated error log:

```text
/var/log/apache2/exam.2025-2026-error.log
```

- Use a custom access log:

```text
/var/log/apache2/exam.2025-2026.log
```

- The custom log format must include:
  - request time
  - incoming `X-Trace-ID` header
  - virtual host name
  - client IP address
  - HTTP method
  - user agent

Additional HTTP requirement:

- Redirect every HTTP request permanently to the HTTPS version of the same path.

Additional HTTPS requirement:

- Enable SSL.
- Use the existing Let's Encrypt certificate files:

```text
/etc/letsencrypt/live/matteo-knops.sasm.uclllabs.be/fullchain.pem
/etc/letsencrypt/live/matteo-knops.sasm.uclllabs.be/privkey.pem
```

### 2.4 Enable and test

Enable the Apache site and reload Apache.

Then test the website with `curl` by sending this request header:

```http
X-Trace-ID: testing123
```

The test should request:

```text
http://exam.2025-2026.matteo-knops.sasm.uclllabs.be
```

and follow redirects.

### 2.5 Certificate renewal

Request or renew a Let's Encrypt certificate using the RFC2136 DNS plugin in the
staging environment.

Requirements:

- Use credentials file `/etc/letsencrypt/rfc2136.ini`.
- Include these names on the certificate:
  - `matteo-knops.sasm.uclllabs.be`
  - `secure.matteo-knops.sasm.uclllabs.be`
  - `supersecure.matteo-knops.sasm.uclllabs.be`
  - `exam.2025-2026.matteo-knops.sasm.uclllabs.be`
- Use the Let's Encrypt staging environment.

## Exercise 3: DNS record creation

The answer notes only show the start of this exercise:

```bash
# Add dns record:
```

A likely exam question could be:

Add the required DNS record for a new service or subdomain using the DNS update
method from the DNS lab.

Your answer should probably include:

- The fully qualified domain name that must be created.
- The record type, for example `A`, `AAAA`, `CNAME`, or `TXT`.
- The TTL.
- The target value, such as an IP address or canonical name.
- The command or zone-file change used to add the record.
- A verification command such as `dig`.

Example wording:

> Add a DNS record for the given exam hostname in your personal SaSM zone.
> Use the correct update method and verify that the record resolves correctly.
