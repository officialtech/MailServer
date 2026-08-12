# OpenDKIM Configuration

OpenDKIM signed outgoing mail with DKIM.

Flow:
```text
Postfix -> OpenDKIM -> DKIM-Signature -> recipient
```

## Test key
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
