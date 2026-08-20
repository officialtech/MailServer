# OpenDKIM Configuration
---

## Install OpenDKIM
```
sudo apt update
sudo apt install opendkim opendkim-tools -y
```

- Verify
```
systemctl status opendkim
```

## Create DKIM directories
```
sudo mkdir -p /etc/opendkim/keys/example.com
```
- Generate key
```
sudo opendkim-genkey \
-s mail \
-d example.com \
-D /etc/opendkim/keys/example.com
```

- This creates
```
mail.private
mail.txt
```

## Fix permissions
```
sudo chown -R opendkim:opendkim /etc/opendkim/keys
sudo chmod 600 /etc/opendkim/keys/example.com/mail.private
```

## Configure OpenDKIM
- Create/edit:
```
/etc/opendkim.conf
```

- Replace it with this minimal, clean configuration:
```
Syslog                  yes
UMask                   002
Canonicalization        relaxed/simple
Mode                    sv
SubDomains              no
OversignHeaders         From


KeyTable                /etc/opendkim/key.table
SigningTable            refile:/etc/opendkim/signing.table
ExternalIgnoreList      /etc/opendkim/trusted.hosts
InternalHosts           /etc/opendkim/trusted.hosts


Socket                  local:/run/opendkim/opendkim.sock
```

#### KeyTable

- Create
```
sudo nano /etc/opendkim/key.table
```

- Contents
```
mail._domainkey.example.com example.com:mail:/etc/opendkim/keys/example.com/mail.private
```

#### SigningTable
```
sudo nano /etc/opendkim/signing.table
```

- Contents (-_- Exactly same just replace example.com with your domain)
```
*@example.com mail._domainkey.example.com
```

#### Trusted Hosts
```
sudo nano /etc/opendkim/trusted.hosts
```

- Contents (ˆ_ˆ don't confuse with star just change domain name)
```
127.0.0.1
localhost
mail.example.com
*.example.com
```

---
## Connect Postfix to OpenDKIM

- Edit
```
sudo nano /etc/postfix/main.cf
```

- Add these lines at the end:
```
milter_default_action = accept
milter_protocol = 6


smtpd_milters = unix:/run/opendkim/opendkim.sock
non_smtpd_milters = $smtpd_milters
```

- Restart
```
sudo systemctl restart opendkim
sudo systemctl restart postfix
```




---
OpenDKIM signed outgoing mail with DKIM.

Flow:
```text
Postfix -> OpenDKIM -> DKIM-Signature -> recipient
```

## Test key (Verify DKIM)
```bash
sudo opendkim-testkey -d example.com -s selector1 -vvv
```

A `key not secure` message can relate to DNSSEC status and does not by itself prove the DKIM key is unusable.

## Service
```bash
sudo systemctl status opendkim
sudo journalctl -u opendkim --since "1 hour ago" --no-pager
```

## Socket issue encountered
Postfix initially had permission trouble connecting to:
```text
/run/opendkim/opendkim.sock
```
The socket/service permissions were corrected. Afterward Postfix logs showed DKIM-Signature being added.

## DNS
Publish the corresponding public key:
```text
selector1._domainkey.example.com
```
Keep the private key only on the server.

## Verify Postfix milter configuration
```bash
postconf -n | grep -i milter
```
Use the actual live socket/path rather than inventing it here.
