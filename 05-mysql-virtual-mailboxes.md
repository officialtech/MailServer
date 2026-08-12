# MySQL Virtual Domains, Users and Aliases

MySQL was used for Postfix virtual-domain, mailbox and alias lookups.

Observed files:
```text
/etc/postfix/mysql-virtual-domains.cf
/etc/postfix/mysql-virtual-users.cf
/etc/postfix/mysql-virtual-aliases.cf
```

## Virtual users
The observed `mysql-virtual-users.cf` included:
```text
user = mailuser
query = SELECT email FROM virtual_users WHERE email='%s' AND active=1;
```

Other connection fields should be taken from the live file because they contain deployment-specific database credentials.

## Lookup tests
```bash
postmap -q user@example.com mysql:/etc/postfix/mysql-virtual-users.cf
postmap -q alias@example.com mysql:/etc/postfix/mysql-virtual-aliases.cf
postmap -q example.com mysql:/etc/postfix/mysql-virtual-domains.cf
```

## Permissions
Protect database configuration files:
```bash
sudo ls -la /etc/postfix/mysql-*.cf
```
Only change ownership/permissions after checking the current Postfix group and deployment.

## Important
If aliases are stored directly in MySQL, no `newaliases` command is needed. The SQL lookup is performed for new mail transactions.
