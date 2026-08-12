# MailServer
A mail server is a software system that sends, receives, and stores email using protocols like SMTP, IMAP, and POP3. It acts as the backbone of email communication, routing messages between clients such as Outlook, Gmail, or Apple Mail.

---
<img width="991" height="299" alt="image" src="https://github.com/user-attachments/assets/bc9121ec-180e-41ed-98c8-8c7050de8c5b" />
<img width="977" height="207" alt="image" src="https://github.com/user-attachments/assets/be80b8c3-d7b4-4f35-844a-9a43f2a238c4" />


## Mail Server Command Reference 

A practical command reference for building and troubleshooting a Linux mail server using Postfix, Dovecot, MySQL/MariaDB, Rspamd, Redis, Nginx and TLS.

Examples use:

```text
Mail host: mail.example.com
Mail domain: example.com
```

Replace placeholders before running commands.

> **Safety:** Read destructive commands carefully. Commands such as `rm`, `postsuper`, `redis-cli FLUSHALL`, database `DROP`, firewall changes, and service removal can cause data loss or downtime.

---
<img width="512" height="512" alt="image" src="https://github.com/user-attachments/assets/f26704cf-34da-479d-9b52-0e89b37f13a8" />


## 1. System and OS

### Identify OS and kernel

```bash
cat /etc/os-release
# Shows Ubuntu/Debian release information.

uname -a
# Shows kernel, architecture and build information.
```

### Hostname

```bash
hostnamectl
# Shows the configured system hostname.

hostname -f
# Shows the fully qualified hostname.
```

A mail server should normally have a stable FQDN such as:

```text
mail.example.com
```

### CPU, RAM and load

```bash
uptime
# Shows uptime, load averages and logged-in users.

free -h
# Shows RAM and swap usage.

df -h
# Shows filesystem capacity and free space.

top
# Interactive process/resource view.
```

### Find the biggest directories

```bash
sudo du -xhd1 / | sort -h
# Shows which top-level directories use the most disk.

sudo du -xhd1 /var | sort -h
# Useful for finding mail/log/database growth.
```

### Reboot readiness

```bash
systemctl --failed
# Lists failed systemd units.

systemctl is-enabled postfix dovecot rspamd redis-server nginx
# Confirms whether important services will start after reboot.

systemctl is-active postfix dovecot rspamd redis-server nginx
# Confirms whether important services are currently running.
```

---
<img width="512" height="512" alt="image" src="https://github.com/user-attachments/assets/1b1eb2aa-6819-42eb-ab88-dce06f1e3328" />


## 2. DNS

### A record

```bash
dig +short A mail.example.com
# Shows the IPv4 address for the mail host.
```

### MX record

```bash
dig +short MX example.com
# Shows where mail for the domain is delivered.
```

### SPF

```bash
dig +short TXT example.com
# Shows TXT records, including SPF.
```

### DKIM

```bash
dig +short TXT mail._domainkey.example.com
# Queries the public DKIM key for selector "mail".
```

### DMARC

```bash
dig +short TXT _dmarc.example.com
# Shows the DMARC policy.
```

### Reverse DNS / PTR

```bash
dig -x <SERVER_PUBLIC_IP> +short
# Shows the PTR/reverse-DNS hostname for the server IP.
```

### Compare multiple DNS resolvers

```bash
dig TXT example.com @1.1.1.1
# Query Cloudflare DNS.

dig TXT example.com @8.8.8.8
# Query Google DNS.
```

Useful when a DNS change appears correct locally but is not visible everywhere.

---
<img width="512" height="512" alt="image" src="https://github.com/user-attachments/assets/fbbc3ca8-61d2-4855-ad90-fa237470d679" />


## 3. Network Ports

### Listening services

```bash
sudo ss -lntup
# Lists TCP/UDP sockets and the processes listening on them.
```

### Check a specific port

```bash
sudo ss -lntp | grep ':25'
# Shows what is listening on SMTP port 25.

sudo ss -lntp | grep ':587'
# Shows submission service.

sudo ss -lntp | grep ':993'
# Shows IMAPS if enabled.
```

### Check local connectivity

```bash
nc -vz 127.0.0.1 11334
# Tests whether Rspamd controller is reachable locally.

nc -vz 127.0.0.1 3306
# Tests whether MariaDB is listening locally.
```

---
<img width="130" height="91" alt="image" src="https://github.com/user-attachments/assets/b8869aa2-a4df-4c2d-ba23-092b3c3942c8" />


## 4. Postfix

### Show effective configuration

```bash
postconf -n
# Shows non-default Postfix configuration.
```

### Search important settings

```bash
postconf -n | grep -iE 'sasl|tls|relay|recipient|virtual|milter|rate'
# Quickly reviews authentication, TLS, relay, virtual-mailbox, milter and rate settings.
```

### Validate

```bash
sudo postfix check
# Validates Postfix configuration, permissions and required files.
```

### Reload

```bash
sudo postfix reload
# Reloads configuration without a full service restart.
```

### Status

```bash
systemctl status postfix --no-pager
# Shows service state and recent logs.
```

### Master services

```bash
postconf -M
# Lists Postfix master.cf services.

postconf -P
# Shows service-specific master.cf parameter overrides.
```

### Full configuration

```bash
postconf
# Shows all Postfix parameters, including defaults.
```

### Queue

```bash
postqueue -p
# Shows the current Postfix queue.

mailq
# Equivalent queue inspection command.
```

### Flush/retry queue

```bash
sudo postqueue -f
# Requests delivery retries for queued mail.
```

Do not use destructive queue deletion commands unless you have identified the affected messages.

### Inspect a queue message

```bash
sudo postcat -q <QUEUE_ID>
# Displays the contents/metadata of a queued message.
```

If the queue ID no longer exists, the message has already left the queue.

### Show only queued message IDs

```bash
postqueue -p | grep -E '^[A-F0-9]{10,}'
# Extracts queue IDs from the queue listing.
```

---
<img width="512" height="512" alt="image" src="https://github.com/user-attachments/assets/203c71bd-43b9-4f8c-892f-e6903fc4cda0" />


## 5. Postfix MySQL Maps

### Test a virtual user lookup

```bash
postmap -q user@example.com mysql:/path/to/mysql-virtual-users.cf
# Asks Postfix's MySQL map whether the mailbox exists.
```

### Test a virtual alias

```bash
postmap -q alias@example.com mysql:/path/to/mysql-virtual-aliases.cf
# Shows the destination returned for the alias.
```

### Test a virtual domain

```bash
postmap -q example.com mysql:/path/to/mysql-virtual-domains.cf
# Shows whether the domain lookup returns a valid result.
```

These are among the most useful commands when Postfix says a recipient/domain cannot be found.

---
<img width="512" height="512" alt="image" src="https://github.com/user-attachments/assets/131a9f71-4d0d-4113-b831-0bb681c15ba0" />


## 6. Postfix Logs

### Recent logs

```bash
sudo journalctl -u postfix --since "1 hour ago" --no-pager
# Shows recent Postfix systemd logs.
```

### Follow logs live

```bash
sudo journalctl -u postfix -f
# Follows Postfix events as they happen.
```

### Traditional mail log

```bash
sudo tail -f /var/log/mail.log
# Useful on systems that write mail events to mail.log.
```

### Search for a message ID

```bash
sudo grep 'QUEUE_ID' /var/log/mail.log
# Tracks one message through Postfix using its queue ID.
```

### Search for delivery failures

```bash
sudo grep -iE 'deferred|bounced|reject|warning|error' /var/log/mail.log | tail -100
# Quickly finds common delivery/failure events.
```

---
<img width="398" height="200" alt="image" src="https://github.com/user-attachments/assets/99e27b04-6780-461c-b33a-6d436e7ed42c" />


## 7. Dovecot

### Effective configuration

```bash
sudo doveconf -n
# Shows the effective Dovecot configuration.
```

### Specific settings

```bash
doveconf -n | grep auth_username_format
# Shows username transformation rules.

doveconf -n | grep recipient_delimiter
# Shows address-extension delimiter settings.
```

### User lookup

```bash
doveadm user user@example.com
# Queries userdb for a mailbox.

doveadm user user@example.com -x protocol=imap
# Tests the userdb lookup in IMAP context.

doveadm user user@example.com -x protocol=lmtp
# Tests the userdb lookup in LMTP context.
```

This comparison is extremely useful when IMAP works but LMTP says:

```text
User doesn't exist
```

### Authentication

```bash
doveadm auth test user@example.com
# Tests authentication. Enter the password only when prompted.
```

### List users

```bash
doveadm user '*'
# Lists users if the configured userdb supports iteration.
```

### Mailbox path

```bash
doveadm user -f home user@example.com
# Shows the resolved home/mailbox path.
```

### Mailbox status

```bash
doveadm mailbox status -u user@example.com INBOX
# Shows INBOX status information.
```

### Search mail

```bash
doveadm search -u user@example.com mailbox INBOX ALL
# Lists matching message identifiers in the mailbox.
```

### Logs

```bash
sudo journalctl -u dovecot --since "1 hour ago" --no-pager
# Shows recent Dovecot logs.
```

```bash
sudo journalctl -u dovecot -f
# Follows Dovecot logs live.
```

---
<img width="512" height="512" alt="image" src="https://github.com/user-attachments/assets/12e34e93-08c6-45bf-aa3c-2692a56e2b54" />


## 8. LMTP Troubleshooting

### Verify Postfix transport

```bash
postconf virtual_transport
# Shows the virtual delivery transport.

postconf | grep dovecot-lmtp
# Searches for Dovecot LMTP-related settings.
```

A typical setup uses:

```text
virtual_transport = lmtp:unix:private/lmtp
```

### Verify socket

```bash
sudo ls -l /var/spool/postfix/private/
# Look for the LMTP/auth sockets used by the installation.
```

### Direct Dovecot LMTP test

```bash
doveadm exec lmtp
# Starts a local LMTP test session.
```

### Common failure

If Postfix reports:

```text
550 5.1.1 User doesn't exist
```

compare:

```bash
doveadm user user@example.com
doveadm user user@example.com -x protocol=imap
doveadm user user@example.com -x protocol=lmtp
```

If only LMTP fails, check protocol-specific username formatting.

---
<img width="549" height="390" alt="image" src="https://github.com/user-attachments/assets/2e2f4455-ac0a-4a28-93e3-8b85f5e41840" />


## 9. MySQL/MariaDB

### Service

```bash
systemctl status mariadb --no-pager
# Shows database service state.
```

### Login

```bash
sudo mariadb
# Opens a local MariaDB administrative shell when socket authentication is configured.
```

Or:

```bash
mariadb -u <DB_USER> -p
# Prompts for a database password.
```

### List databases

```sql
SHOW DATABASES;
```

### Select the mail database

```sql
USE mailserver;
```

### Tables

```sql
SHOW TABLES;
```

### Inspect table structure

```sql
DESCRIBE virtual_users;
DESCRIBE virtual_aliases;
DESCRIBE virtual_domains;
```

### Count mailboxes

```sql
SELECT COUNT(*) FROM virtual_users;
```

### Count aliases

```sql
SELECT COUNT(*) FROM virtual_aliases;
```

### Check user

```sql
SELECT email, active
FROM virtual_users
WHERE email = 'user@example.com';
```

### Check alias

```sql
SELECT source, destination, active
FROM virtual_aliases
WHERE source = 'alias@example.com';
```

### Privileges

```sql
SHOW GRANTS FOR 'mailuser'@'localhost';
```

Use a least-privileged application account. Do not use database root credentials in Postfix/Dovecot.

---
<img width="549" height="351" alt="image" src="https://github.com/user-attachments/assets/fb7f541e-9910-4f06-8315-530cc99751e7" />


## 10. DKIM / OpenDKIM

### Service

```bash
systemctl status opendkim --no-pager
# Shows OpenDKIM service state.
```

### Logs

```bash
sudo journalctl -u opendkim --since "1 hour ago" --no-pager
# Shows DKIM signing/verification events.
```

### Test DNS/key

```bash
sudo opendkim-testkey -d example.com -s mail -vvv
# Verifies selector/DNS/key configuration for the selected domain.
```

### Find signing configuration

```bash
sudo grep -RniE 'KeyTable|SigningTable|Selector|Domain' /etc/opendkim 2>/dev/null
# Finds common OpenDKIM configuration references.
```

### Socket

```bash
sudo ss -lxnp | grep opendkim
# Shows whether an OpenDKIM UNIX socket is listening.
```

### Postfix milter

```bash
postconf -n | grep -i milter
# Shows the active milter integration.
```

---
<img width="772" height="484" alt="image" src="https://github.com/user-attachments/assets/32d49451-c7a2-4c1d-b2af-6587679384fc" />


## 11. Rspamd

### Version

```bash
rspamadm --version
# Shows installed Rspamd version.
```

### Validate configuration

```bash
sudo rspamadm configtest
# Checks configuration syntax.
```

### Effective classifier

```bash
rspamadm configdump classifier
# Shows Bayes/classifier settings.
```

### Effective DKIM signing

```bash
rspamadm configdump dkim_signing
# Shows DKIM signing configuration.
```

### Effective ratelimit

```bash
rspamadm configdump ratelimit
# Shows current Rspamd rate-limit configuration.
```

### Service

```bash
systemctl status rspamd --no-pager
# Shows Rspamd service state.
```

### Logs

```bash
sudo journalctl -u rspamd --since "1 hour ago" --no-pager
# Shows Rspamd logs.
```

### Follow logs

```bash
sudo journalctl -u rspamd -f
# Follows Rspamd logs live.
```

### Statistics

```bash
rspamc stat
# Shows scan, spam/ham and Bayes learning statistics.
```

---
<img width="322" height="213" alt="image" src="https://github.com/user-attachments/assets/1737678d-3773-4621-8736-6fc41ed80ead" />


## 12. Rspamd Controller / WebUI

### Check listening socket

```bash
sudo ss -ltnp | grep 11334
# Confirms the controller is listening; ideally it should be localhost-only.
```

### Direct GET test

```bash
curl -s http://127.0.0.1:11334/ | head
# Tests whether the controller serves the WebUI locally.
```

Do not use `curl -I` as the sole test for Rspamd controller resources because the controller may respond differently to HEAD requests.

### Verify public static file

```bash
curl -s -o /dev/null -w '%{http_code}\n' \
  https://mail.example.com/rspamd/js/main.js
# Checks whether a reverse-proxy static asset is reachable.
```

### API authorization

Unauthenticated API requests may return:

```text
401 Unauthorized
```

That is expected for protected endpoints.

Keep controller ports private and expose the UI through authenticated HTTPS if needed.

---
<img width="321" height="94" alt="image" src="https://github.com/user-attachments/assets/8039f691-d5a5-459a-ad90-5c0801c61f62" />


## 13. Redis

### Ping

```bash
redis-cli PING
# Basic connectivity test.
```

Expected:

```text
PONG
```

### Key count

```bash
redis-cli DBSIZE
# Shows the number of keys in the current Redis database.
```

### Keyspace

```bash
redis-cli INFO keyspace
# Shows keys, expirations and TTL information.
```

### Scan keys safely

```bash
redis-cli --scan | head -50
# Iterates keys without blocking Redis like KEYS * can.
```

Do not use destructive commands on production Redis unless you are intentionally deleting the stored state.

---
<img width="512" height="512" alt="image" src="https://github.com/user-attachments/assets/fcfc9979-e39f-4fff-b0ef-2075c0348b29" />


## 14. Nginx

### Validate

```bash
sudo nginx -t
# Validates syntax before reloading.
```

### Full effective configuration

```bash
sudo nginx -T
# Prints the complete merged Nginx configuration, including included files.
```

This is useful when a configuration appears not to be in `/etc/nginx/nginx.conf` because it may be included from `sites-enabled`.

### Search configuration

```bash
sudo grep -Rni 'server_name mail.example.com' /etc/nginx 2>/dev/null
# Locates the server block.
```

### Reload

```bash
sudo systemctl reload nginx
# Reloads Nginx without dropping established connections.
```

### Status

```bash
systemctl status nginx --no-pager
# Shows Nginx service state.
```

### Logs

```bash
sudo tail -n 100 /var/log/nginx/error.log
# Shows recent errors.

sudo tail -n 100 /var/log/nginx/access.log
# Shows recent requests.
```

---
<img width="597" height="399" alt="image" src="https://github.com/user-attachments/assets/43de5edd-089c-4feb-b3b4-5dde7f5d2ba3" />


## 15. TLS

### SMTP submission 587

```bash
openssl s_client -connect mail.example.com:587 -starttls smtp
# Tests STARTTLS and shows certificate details.
```

### SMTPS 465

```bash
openssl s_client -connect mail.example.com:465
# Tests implicit TLS.
```

### IMAPS 993

```bash
openssl s_client -connect mail.example.com:993
# Tests IMAP over TLS.
```

### Verify certificate dates

```bash
echo | openssl s_client -connect mail.example.com:443 2>/dev/null \
  | openssl x509 -noout -dates -subject -issuer
# Shows certificate validity and identity.
```

---
<img width="135" height="130" alt="image" src="https://github.com/user-attachments/assets/7e8a11d7-117e-4a4a-a94d-0313a13d4fe2" />


## 16. Fail2ban

### Status

```bash
sudo fail2ban-client status
# Lists active jails.
```

### Jail details

```bash
sudo fail2ban-client status sshd
# Shows status, failed attempts and banned IPs for the SSH jail.
```

### Unban a specific IP

```bash
sudo fail2ban-client set sshd unbanip <IP>
# Removes an IP from the specified jail.
```

Only unban an address you recognize.

---
<img width="395" height="390" alt="image" src="https://github.com/user-attachments/assets/fcdd64a0-0951-4ec4-b36b-5f7c088c800a" />


## 17. Firewall

### UFW status

```bash
sudo ufw status verbose
# Shows firewall policy and allowed ports.
```

### Listening ports

```bash
sudo ss -lntup
# Cross-checks actual listeners against firewall rules.
```

Do not open administrative services to the Internet unless needed.

---
<img width="347" height="360" alt="image" src="https://github.com/user-attachments/assets/05be3ec2-cd28-4094-8340-bbc267ade592" />


## 18. Service Restart vs Reload

### Postfix

```bash
sudo postfix reload
# Preferred after configuration-only changes.
```

### Dovecot

```bash
sudo systemctl reload dovecot
# Reloads Dovecot configuration.
```

### Rspamd

```bash
sudo systemctl reload rspamd
# Reloads Rspamd configuration.
```

### Nginx

```bash
sudo nginx -t && sudo systemctl reload nginx
# Validates before reloading.
```

Use full restarts when a reload is insufficient or when the service documentation explicitly requires one.

---
<img width="405" height="401" alt="image" src="https://github.com/user-attachments/assets/68f57035-6c7b-48f4-924c-b8e249a54a27" />


## 19. Backup Validation

### Check compressed SQL

```bash
gzip -t mailserver.sql.gz
# Verifies the gzip archive is readable.
```

### SHA-256 verification

```bash
sha256sum -c SHA256SUMS
# Verifies backup-file integrity.
```

Do not include the checksum file itself in `SHA256SUMS`.

### Inspect archive

```bash
tar --zstd -tf mail-vhosts.tar.zst | head -50
# Lists files in a zstd-compressed tar archive.
```

### Backup service

```bash
systemctl status example-mail-backup.service --no-pager
# Shows an example systemd backup service.

systemctl list-timers --all | grep backup
# Shows scheduled backup timers.
```

---
<img width="491" height="366" alt="image" src="https://github.com/user-attachments/assets/2eee921b-30b9-4273-bb7f-cabf018e2ab5" />


## 20. Useful One-Line Health Check

```bash
systemctl is-active postfix dovecot rspamd redis-server nginx
# Quickly checks whether the main services are active.
```

```bash
postqueue -p
# Check mail queue.
```

```bash
rspamc stat
# Check Rspamd statistics.
```

```bash
redis-cli PING
# Check Redis connectivity.
```

```bash
sudo postfix check && sudo rspamadm configtest && sudo nginx -t
# Validate several major configuration layers at once.
```

---
<img width="559" height="302" alt="image" src="https://github.com/user-attachments/assets/7a9af5c8-dd32-4aa2-8b99-69e793baaa23" />


## 21. Message-Tracking Workflow

When a message fails, do not change configuration immediately.

### Step 1 — Find the event

```bash
sudo tail -f /var/log/mail.log
```

Send a test message.

### Step 2 — Record the queue ID

Example:

```text
2A3BC4D5EF
```

### Step 3 — Search logs

```bash
sudo grep '2A3BC4D5EF' /var/log/mail.log
```

### Step 4 — Identify where it failed

Typical stages:

```text
SMTP reception
  -> cleanup
  -> milter/filter
  -> queue manager
  -> transport
  -> remote SMTP server
  -> delivery response
```

### Step 5 — Use the error itself

Examples:

```text
550 5.1.1
```

Often means recipient/user does not exist.

```text
550 5.7.26
```

Often points toward authentication/policy rejection.

```text
451 / 421
```

Often indicates temporary failure/defer/retry behavior.

Do not infer the exact cause from a status code alone; read the complete server response.

---
<img width="589" height="283" alt="image" src="https://github.com/user-attachments/assets/c71b574a-17a9-40ad-b568-3b921ee323f3" />


## 22. Recommended Debugging Order

For almost every mail problem, use this order:

```text
1. DNS
2. Network/port
3. TLS
4. SMTP authentication
5. Postfix configuration
6. SQL lookup
7. Dovecot userdb/authentication
8. LMTP
9. Rspamd/OpenDKIM
10. Remote-provider rejection
```

This prevents changing several unrelated services at once.


<img width="664" height="222" alt="image" src="https://github.com/user-attachments/assets/9afb6aac-1839-41a3-a440-9fd5b130f287" />


## Golden rule

Change **one layer at a time**, validate it, then move to the next layer.

Always keep a known-good backup before major configuration changes.

