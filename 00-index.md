# Mail Server Documentation — mail.example.com

This documentation describes a virtual-mail deployment using `mail.example.com` and `example.com`.

## Documents

1. `01-system-hostname.md` — OS, hostname and server layout
2. `02-dns-spf-dkim-ptr.md` — DNS, SPF, DKIM and PTR
3. `03-postfix.md` — Postfix
4. `04-dovecot.md` — Dovecot
5. `05-mysql-virtual-mailboxes.md` — MySQL virtual domains, users and aliases
6. `06-opendkim.md` — OpenDKIM
7. `07-rspamd.md` — Rspamd
8. `08-redis-bayes.md` — Redis and Bayes
9. `09-nginx.md` — Nginx and HTTPS
10. `10-smtp-submission-sasl.md` — SMTP submission, TLS and SASL
11. `11-aliases-and-reloads.md` — Aliases and applying changes
12. `12-backup-maintenance.md` — Backups and operations
13. `13-troubleshooting-history.md` — Troubleshooting guide

## Architecture

```text
Internet
  |
  +-- Nginx :443 --> HTTPS / web services
  |
  +-- Postfix :25 --> SMTP receive/send
  |       |
  |       +--> DKIM signing / filtering
  |       +--> Dovecot mailbox delivery
  |
  +-- Postfix :587/:465 --> TLS + SASL --> Dovecot
  |
  +-- MySQL --> virtual domains/users/aliases
  |
  +-- Redis --> Rspamd Bayes/state
```

## Conventions

Replace placeholders such as:

```text
<MAIL_DOMAIN>
<MAIL_HOST>
<SERVER_PUBLIC_IP>
```

with values for your deployment.

Never publish:

- mailbox passwords
- database passwords
- DKIM private keys
- TLS private keys
- Rspamd controller passwords
- SSH private keys
