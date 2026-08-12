# Backup and Maintenance

```bash
sudo tar -czf /root/rspamd-backup-$(date +%F-%H%M%S).tar.gz   /etc/rspamd   /var/lib/rspamd
```

## Validation
Rspamd:
```bash
sudo rspamadm configtest
```
Postfix:
```bash
sudo postfix check
```
Dovecot:
```bash
sudo doveconf -n
```
Nginx:
```bash
sudo nginx -t
```

## Queue
```bash
postqueue -p
mailq
```

## Rspamd
```bash
rspamc stat
```

## Redis
```bash
redis-cli PING
redis-cli DBSIZE
```

## Service status
```bash
systemctl status postfix
systemctl status dovecot
systemctl status rspamd
systemctl status redis-server
systemctl status nginx
```

## Logs
```bash
sudo journalctl -u postfix --since "1 hour ago" --no-pager
sudo journalctl -u dovecot --since "1 hour ago" --no-pager
sudo journalctl -u rspamd --since "1 hour ago" --no-pager
sudo journalctl -u redis-server --since "1 hour ago" --no-pager
sudo tail -n 100 /var/log/nginx/error.log
```

Never put passwords, DKIM private keys, MySQL passwords, Rspamd controller passwords or TLS private keys in these tutorials or a public repository.
