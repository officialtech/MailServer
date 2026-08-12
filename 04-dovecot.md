# Dovecot Configuration

Dovecot provided mailbox access and the SASL authentication backend for Postfix.

Postfix used:
```text
smtpd_sasl_type = dovecot
smtpd_sasl_path = private/auth
```

Architecture:
```text
Postfix :587/:465
       |
       v
Dovecot SASL
       |
       v
Virtual user authentication
```

## Verify
```bash
sudo systemctl status dovecot
sudo doveconf -n
```

Authentication test, using the syntax supported by the installed Dovecot version:
```bash
sudo doveadm auth test user@example.com
```

Do not put real passwords into shell history.

## Reload
```bash
sudo systemctl reload dovecot
```

## Logs
```bash
sudo journalctl -u dovecot --since "1 hour ago" --no-pager
```

The exact final mailbox-storage and password-hash settings were not preserved completely in the available history, so they are intentionally not invented here.
