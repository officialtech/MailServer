# SMTP Submission, TLS and SASL

Postfix provided authenticated SMTP submission on ports 587 and 465.

## Port 587 overrides
Observed:
```text
submission/inet/smtpd_sasl_auth_enable = yes
submission/inet/smtpd_tls_auth_only = yes
submission/inet/smtpd_tls_security_level = encrypt
submission/inet/smtpd_recipient_restrictions = permit_sasl_authenticated,reject
submission/inet/milter_macro_daemon_name = ORIGINATING
```

## Port 465 overrides
Observed:
```text
submissions/inet/smtpd_sasl_auth_enable = yes
submissions/inet/smtpd_tls_wrappermode = yes
submissions/inet/smtpd_recipient_restrictions = permit_sasl_authenticated,reject
submissions/inet/milter_macro_daemon_name = ORIGINATING
```

## Inspect
```bash
postconf -M
postconf -P
```

## TLS tests
587:
```bash
openssl s_client -connect mail.example.com:587 -starttls smtp
```

465:
```bash
openssl s_client -connect mail.example.com:465
```

## Authentication architecture
```text
Email client
  -> Postfix :587/:465
  -> TLS + SMTP AUTH
  -> Dovecot SASL
  -> virtual-user authentication
```

Authenticated submission was restricted with:
```text
permit_sasl_authenticated,reject
```
