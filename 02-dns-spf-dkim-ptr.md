# DNS, SPF, DKIM and PTR

Use:
```text
Mail host: mail.example.com
Domain: example.com
```

## A
```text
mail.example.com.  A  <SERVER_PUBLIC_IP>
```

When Cloudflare is used, the mail host should normally be DNS-only for mail protocols.

## MX
```text
example.com.  MX  10  mail.example.com.
```

## SPF
Publish one SPF record:
```text
example.com. TXT "v=spf1 a mx ip4:<SERVER_PUBLIC_IP> ~all"
```
Adapt it to the actual sending infrastructure. Do not publish multiple independent SPF records.

## DKIM
Example:
```text
selector1._domainkey.example.com. TXT "v=DKIM1; k=rsa; p=<PUBLIC_KEY>"
```

## DMARC
A starting monitoring policy:
```text
_dmarc.example.com. TXT "v=DMARC1; p=none; rua=mailto:dmarc@example.com"
```

## PTR
The SMTP server IP must reverse-resolve:
```text
<SERVER_PUBLIC_IP> -> mail.example.com
```
and:
```text
mail.example.com -> <SERVER_PUBLIC_IP>
```

Verify:
```bash
dig +short A mail.example.com
dig +short MX example.com
dig +short TXT example.com
dig +short TXT _dmarc.example.com
dig -x <SERVER_PUBLIC_IP> +short
```

PTR is configured at the VPS/provider level, not as an ordinary Cloudflare DNS record.

## Historical issue
Gmail rejected mail when the sending IP lacked a valid PTR/reverse-DNS relationship. Correct PTR/A alignment was therefore an essential part of getting outbound mail accepted.
