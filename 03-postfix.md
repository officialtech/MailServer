# Postfix Configuration

Postfix handled SMTP receive/send, authenticated submission and virtual-mail routing.

## Main values observed
```text
smtpd_relay_restrictions = permit_mynetworks permit_sasl_authenticated defer_unauth_destination
smtpd_sasl_type = dovecot
smtpd_sasl_path = private/auth
smtpd_sasl_auth_enable = yes
smtpd_sasl_security_options = noanonymous
smtpd_sasl_local_domain = $myhostname
smtpd_recipient_restrictions = permit_mynetworks, permit_sasl_authenticated, reject_unauth_destination
recipient_delimiter = +
smtpd_client_connection_count_limit = 20
virtual_mailbox_maps = mysql:/etc/postfix/mysql-virtual-users.cf
```

## Inspect
```bash
postconf -n
postconf -M
postconf -P
```

## Validate
```bash
sudo postfix check
```

## Reload
```bash
sudo postfix reload
```

## Queue
```bash
postqueue -p
mailq
```

The queue was verified empty during the project.

## Logs
```bash
sudo journalctl -u postfix --since "1 hour ago" --no-pager
```
or:
```bash
sudo tail -f /var/log/mail.log
```

## Master services
Submission and SMTPS were configured for authenticated TLS submission; see `10-smtp-submission-sasl.md`.
