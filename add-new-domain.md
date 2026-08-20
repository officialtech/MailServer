
---  

# Adding a New Mail Domain – Step‑by‑Step Guide  

> **TL;DR** – The core workflow for adding a new domain is:  
> DNS → DKIM key → DKIM DNS → Rspamd mapping → `virtual_domains` row → `virtual_users` row → (optional) `virtual_aliases` rows → DMARC → Postfix/Dovecot tests → Backup.

---

## 1. What Actually Changes  

When you add a new domain the following items normally **change** (ordered by dependency):  

1. **DNS**  
2. **DKIM key + DNS**  
3. **Rspamd DKIM domain mapping**  
4. `virtual_domains` database row  
5. `virtual_users` database row(s)  
6. `virtual_aliases` database row(s) – *if needed*  
7. **DMARC DNS record**  
8. **Testing**  

The following services **do not change**:  

- Postfix SMTP service definition  
- Dovecot service definition  
- Redis configuration  
- Nginx mail configuration  
- Server hostname / PTR records  

> **Note:** In this architecture you **don’t** modify OpenDKIM because Rspamd is the active signer.

---

## 2. Pre‑flight Checks  

Run these commands to verify the current server health **before** making any changes.

```bash
# Postfix
sudo postfix check

# Dovecot
sudo doveconf -n >/dev/null

# Rspamd
sudo rspamadm configtest

# Service status
systemctl is-active postfix dovecot rspamd redis-server

# Mail queue (should be empty)
postqueue -p
```

> **Preferred state:** The mail queue is empty. If there are existing delivery problems, resolve them first.

---

## 3. Choose the Mail Hostname for the New Domain  

The simplest design is to keep the existing mail hostname:

```
mail.example.com
```

Then point the new domain to that host:

```
newdomain.com.   MX   10   mail.example.com.
```

**Resulting client configuration:**  

- **SMTP:** `mail.example.com`  
- **IMAP:** `mail.example.com`  
- **Email address:** `admin@newdomain.com`

Create a separate `mail.newdomain.com` **only** if you need a distinct hostname/certificate.

---

## 4. DNS – MX Record  

Add the MX record:

```
newdomain.com.   MX   10   mail.example.com.
```

Verify:

```bash
dig +short MX newdomain.com
```

Expected output:

```
10 mail.example.com.
```

---

## 5. DNS – SPF Record  

Publish **exactly one** SPF TXT record for the new domain.

```text
newdomain.com. TXT "v=spf1 mx a ip4:<SERVER_PUBLIC_IP> ~all"
```

Validate:

```bash
dig +short TXT newdomain.com
```

> **Do not** create multiple independent SPF records.

---

## 6. Generate a DKIM Key for the New Domain  

Because Rspamd is the active DKIM signer, generate a key readable by Rspamd.

```bash
# Create directory for the domain
sudo mkdir -p /etc/rspamd/dkim/newdomain.com

# Generate a 2048‑bit key with selector "mail"
sudo rspamadm dkim_keygen \
    -b 2048 \
    -s mail \
    -d newdomain.com \
    -k /etc/rspamd/dkim/newdomain.com/mail.key
```

> *Take the output it gave you and create a TXT record in your DNS settings*
> **NOTE: It did generate the private key just like OpenDKIM, it just conveniently gave you the public DNS record at the same time so you wouldn't have to extract it yourself! So don't worry about private key** 

- If you ever need to see the public key again, you don't need to recreate anything. You can extract and view the public key directly from the private key file at any time.
- Run this command:
```bash
sudo rspamadm dkim_keygen -p -k /etc/rspamd/dkim/newdomain.com/mail.key
# The -p flag tells it to output the public key.
# The -k flag tells it which private key file to read from.

## NOTE: If above not worked, try below one:
openssl rsa -in /etc/rspamd/dkim/newdomain.com/mail.key -pubout
```
- This will print the exact same DNS TXT record to your screen that you saw when you first created the key.


> **Docs:** <https://docs.rspamd.com/modules/dkim_signing/>  
> **Tutorial:** <https://docs.rspamd.com/tutorials/dkim_signing_guide/>

Secure the private key:

```bash
sudo chown _rspamd:_rspamd /etc/rspamd/dkim/newdomain.com/mail.key
sudo chmod 600 /etc/rspamd/dkim/newdomain.com/mail.key
```

> **Never** make a DKIM private key world‑readable.

---

## 7. DNS – DKIM TXT Record  

The selector used above is `mail`, so publish the following DNS record:

```
mail._domainkey.newdomain.com. TXT "v=DKIM1; k=rsa; p=<PUBLIC_KEY>"
```

Replace `<PUBLIC_KEY>` with the *public* part of the key you just generated.

Verify:

```bash
dig +short TXT mail._domainkey.newdomain.com
```

> **Never** publish the private key.

---

## 8. Add the Domain to Rspamd  

Edit (or create) `dkim_signing.conf` – the *domain‑specific* section should look like this:

```conf
domain {
    example.com {
        selector = "mail";
        path = "/etc/opendkim/keys/example.com/mail.private";
    }

    newdomain.com {
        selector = "mail";
        path = "/etc/rspamd/dkim/newdomain.com/mail.key";
    }
}

use_domain = "header";
sign_authenticated = true;
sign_local = true;
```

- Keep the existing path for `example.com` unchanged.  
- Rspamd will use the domain‑specific entry before falling back to any global configuration.

> **Docs:** <https://docs.rspamd.com/modules/dkim_signing/>

If you have keys under `/etc/opendkim/keys` that are *only* used by Rspamd, that’s fine – the location does **not** tie the key to OpenDKIM.

---

## 9. Do **Not** Touch OpenDKIM (If Disabled)  

If the current architecture is:

```
Postfix → Rspamd milter → DKIM signing
```

and OpenDKIM is **disabled** (`systemctl is-enabled opendkim` → *disabled*), you **must not** edit:

- `/etc/opendkim/trusted.hosts`  
- `/etc/opendkim/signing.table`  
- `/etc/opendkim/key.table`

Those files only matter when OpenDKIM is part of the signing path.

---

## 10. Test Rspamd Configuration  

After editing `/etc/rspamd/local.d/dkim_signing.conf`:

```bash
sudo rspamadm configtest          # should output “syntax OK”
sudo rspamadm configdump dkim_signing   # inspect the config dump
sudo systemctl reload rspamd
systemctl is-active rspamd        # should be “active”
```

---

## 11. Add the New Domain to MySQL/MariaDB  

### 11.1 `virtual_domains`  

```sql
INSERT INTO virtual_domains (name, active)
VALUES ('newdomain.com', 1);
```

> *Adjust column names if your schema differs.*

Verify:

```bash
postmap -q newdomain.com mysql:/etc/postfix/mysql-virtual-domains.cf
```

The returned value must match your existing SQL query expectations.

### 11.2 `virtual_users` (Mailbox)  

```sql
INSERT INTO virtual_users (email, password_hash, domain_id, active)
VALUES ('admin@newdomain.com', '<HASH>', <DOMAIN_ID>, 1);
```

> *Reuse the same hashing method you already use.*
```bash
doveadm pw -s ARGON2ID
```
> *It will ask you the password, you can choose whatever password you want.*
> - *Example Output*
> - *Enter new password:*
> - *Retype new password:*
> - *{ARGON2ID}$argon2id$v=19$m=65536,t=4,p=1$...*
> - *Copy the output and paste it in the database query.*

Verify:

```bash
postmap -q admin@newdomain.com mysql:/etc/postfix/mysql-virtual-users.cf
```

Expected output:

```
admin@newdomain.com
```

### 11.3 `virtual_aliases` (Optional)  

```sql
INSERT INTO virtual_aliases (source, destination, active)
VALUES ('support@newdomain.com', 'admin@newdomain.com', 1);
```

Verify:

```bash
postmap -q support@newdomain.com mysql:/etc/postfix/mysql-virtual-aliases.cf
```

Expected output:

```
admin@newdomain.com
```

---

## 12. Verify Dovecot  

```bash
doveadm user admin@newdomain.com
```

Dovecot should return the user’s home/mail location as defined in your existing `userdb`.  
*Do not* manually create Maildir folders before confirming Dovecot’s dynamic mapping works.

---

## 13. Add Role‑Based Addresses (Optional)  

Typical role mailboxes/aliases:

- `postmaster@newdomain.com`  
- `abuse@newdomain.com`  
- `security@newdomain.com`  
- `hostmaster@newdomain.com`  
- `webmaster@newdomain.com`  
- `noc@newdomain.com`

You can create them as **real** mailboxes or as **aliases** to an administrative mailbox.

---

## 14. Publish a DMARC Reporting Address  

Create mailbox/alias `dmarc@newdomain.com` and publish the TXT record:

```
_dmarc.newdomain.com. TXT "v=DMARC1; p=none; rua=mailto:dmarc@newdomain.com"
```

Verify:

```bash
dig +short TXT _dmarc.newdomain.com
```

Start with `p=none`, then tighten the policy after confirming alignment.

---

## 15. Service‑Reload Checklist  

| Change | Reload/Restart Required? |
|--------|---------------------------|
| Database rows (MySQL/MariaDB) | **No** (Postfix reads live) |
| Rspamd DKIM config | `sudo systemctl reload rspamd` |
| Postfix SQL map changes | `sudo postfix reload` **only** if you edited the map files |
| Dovecot config changes | `sudo systemctl reload dovecot` (usually not needed) |
| Nginx (mail‑related) | **No** (unless you expose webmail) |
| Redis | **No** |
| TLS certs for new domain | **No** (mail clients use `mail.example.com`) |

---

## 16. Full Configuration Verification  

```bash
# DNS checks
dig +short MX newdomain.com
dig +short TXT newdomain.com
dig +short TXT mail._domainkey.newdomain.com
dig +short TXT _dmarc.newdomain.com

# Postfix lookups
postmap -q newdomain.com mysql:/etc/postfix/mysql-virtual-domains.cf
postmap -q admin@newdomain.com mysql:/etc/postfix/mysql-virtual-users.cf

# Dovecot lookup
doveadm user admin@newdomain.com

# Alias lookup
postmap -q support@newdomain.com mysql:/etc/postfix/mysql-virtual-aliases.cf

# Rspamd config test
sudo rspamadm configtest
rspamadm configdump dkim_signing
```

All commands should return **expected** results (no errors, correct values).

---

## 17. Test Inbound Mail  

Send a message from an external provider (Gmail/Outlook/Proton) to `admin@newdomain.com`.

```text
Incoming flow:
External → mail.example.com → Postfix → Dovecot
```

Monitor logs:

```bash
sudo journalctl -u postfix -f
sudo journalctl -u dovecot -f
```

Confirm the message lands in the new mailbox.

---

## 18. Test Outbound Mail  

Send mail **from** `admin@newdomain.com` using your client:

- **SMTP**: `mail.example.com:587` (STARTTLS) – auth required  
- **IMAP**: `mail.example.com:993` (SSL/TLS)

Check the headers of the sent mail (e.g., via Gmail “Show original”) – you should see:

- `SPF: PASS`  
- `DKIM: PASS` (`d=newdomain.com`, `s=mail`)  
- `DMARC: PASS`

---

## 19. Test an Email Client  

1. **Configure** the client with the settings above.  
2. **Send** a test email to an external address.  
3. **Receive** a test email sent to `admin@newdomain.com`.  

Both inbound and outbound flows must work without errors.

---

## 20. Backup Checklist  

The new domain adds:

- Database rows (`virtual_domains`, `virtual_users`, optional `virtual_aliases`)  
- Mailbox directories & contents  
- DKIM private key (`/etc/rspamd/dkim/newdomain.com/mail.key`)  

Ensure your regular mail backup includes **all** of these items and that the resulting checksum matches the source.

---
