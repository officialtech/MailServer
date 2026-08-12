# Troubleshooting Guide

## Gmail reverse-DNS rejection

### Symptom

Gmail may reject outbound mail when the sending IP lacks a valid PTR/reverse-DNS relationship.

### Required relationship

```text
<SERVER_PUBLIC_IP> -> mail.example.com
mail.example.com   -> <SERVER_PUBLIC_IP>
```

### Verify

```bash
dig +short A mail.example.com
dig -x <SERVER_PUBLIC_IP> +short
```

The PTR is normally configured with the VPS/provider.

---

## OpenDKIM socket permissions

### Symptom

Postfix may report permission errors when connecting to:

```text
/run/opendkim/opendkim.sock
```

### Checks

```bash
sudo systemctl status opendkim
sudo ls -l /run/opendkim/opendkim.sock
postconf -n | grep -i milter
```

The Postfix service user must be able to access the milter socket.

---

## Postfix queue problems

Check:

```bash
postqueue -p
mailq
```

Then inspect:

```bash
sudo journalctl -u postfix --since "1 hour ago" --no-pager
```

For a deferred queue, identify the queue ID and inspect the delivery error before deleting anything.

Avoid destructive commands such as:

```bash
postsuper -d ALL
```

unless you explicitly intend to delete all queued mail.

---

## Dovecot userdb/LMTP "User doesn't exist"

When IMAP authentication succeeds but Postfix LMTP rejects a valid recipient, compare Dovecot lookups with and without the LMTP protocol context:

```bash
doveadm user user@example.com
doveadm user user@example.com -x protocol=imap
doveadm user user@example.com -x protocol=lmtp
```

A protocol-specific username transformation can cause:

```text
user@example.com
```

to become:

```text
user
```

before a SQL lookup.

For virtual users stored by full email address, preserve the full address during the LMTP userdb lookup.

Always confirm the effective configuration with:

```bash
doveconf -n
```

---

## Rspamd WebUI reverse-proxy issues

If the initial Rspamd WebUI loads but navigation or API calls fail:

1. Verify the controller is listening on localhost:

```bash
ss -ltnp | grep 11334
```

2. Test the controller directly with GET requests:

```bash
curl -s http://127.0.0.1:11334/
```

3. Verify static files:

```bash
curl -s -o /dev/null -w '%{http_code}
'   https://mail.example.com/rspamd/js/main.js
```

4. Check browser Developer Tools → Network for API requests such as:

```text
/rspamd/stat
/rspamd/auth
/rspamd/neighbours
```

A `401 Unauthorized` from an unauthenticated API request is expected.

5. Verify Nginx:

```bash
sudo nginx -t
sudo nginx -T
```

Keep the Rspamd controller bound to localhost and protect it with authentication.

---

## SPF/DKIM/DMARC troubleshooting

When Gmail reports unauthenticated mail, inspect all three independently:

```text
SPF
DKIM
DMARC
```

Also verify:

```text
PTR
HELO/EHLO hostname
A record
MX record
```

Do not assume DMARC is correct simply because the domain has a DMARC TXT record; SPF/DKIM must align with the visible From domain.

---

## Configuration validation

Postfix:

```bash
sudo postfix check
```

Dovecot:

```bash
sudo doveconf -n
```

Rspamd:

```bash
sudo rspamadm configtest
```

Nginx:

```bash
sudo nginx -t
```

These checks should be part of every significant configuration change.
