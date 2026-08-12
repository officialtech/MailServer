# System and Hostname Configuration

Target:
```text
Hostname: mail.example.com
Mail domain: example.com
```

The original server was Ubuntu 26.04 LTS (`resolute`) on a small VPS.

## Verify
```bash
cat /etc/os-release
uname -a
hostnamectl
```

## Set hostname
```bash
sudo hostnamectl set-hostname mail.example.com
hostname -f
```

## Hosts
Check `/etc/hosts` and ensure the hostname resolves locally as appropriate:
```bash
sudo nano /etc/hosts
```

Example:
```text
127.0.0.1 localhost
127.0.1.1 mail.example.com mail
```

Do not overwrite an already-correct host configuration blindly.

## Services
```bash
systemctl --failed
systemctl status nginx
systemctl status postfix
systemctl status dovecot
systemctl status rspamd
systemctl status redis-server
```
