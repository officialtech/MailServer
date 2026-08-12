# Email Aliases and Applying Changes

Aliases were stored in the MySQL virtual-alias database and looked up through:
```text
/etc/postfix/mysql-virtual-aliases.cf
```

## After adding an alias

If the alias is a direct MySQL record, the new record is available to a new SQL lookup immediately.

Verify:
```bash
postmap -q alias@example.com mysql:/etc/postfix/mysql-virtual-aliases.cf
```

If the destination is returned, Postfix can resolve it.

Then reload Postfix:
```bash
sudo postfix reload
```

Test delivery and watch:
```bash
sudo journalctl -u postfix -f
```
or:
```bash
sudo tail -f /var/log/mail.log
```

## Troubleshooting
```bash
postmap -q alias@example.com mysql:/etc/postfix/mysql-virtual-aliases.cf
postconf -n | grep virtual
sudo postfix check
sudo postfix reload
```

## `newaliases`
Do not use `newaliases` for MySQL-backed virtual aliases. That command applies to the traditional local `/etc/aliases` database.

## Mailbox/user changes
Verify:
```bash
postmap -q user@example.com mysql:/etc/postfix/mysql-virtual-users.cf
```
Then verify Dovecot authentication and mailbox storage using the live configuration.
