# Redis and Rspamd Bayes Storage

Redis stored Rspamd state/Bayes data.

## Test
```bash
redis-cli PING
```
Successful result:
```text
PONG
```

## Historical state
```bash
redis-cli DBSIZE
```
returned:
```text
39446
```

`redis-cli INFO keyspace` showed:
```text
db0:keys=39446,expires=34,avg_ttl=34179823,subexpiry=0
```

## Rspamd connection
The Bayes configuration used:
```text
backend = "redis";
cache {
    backend = "redis";
}
servers = "127.0.0.1";
```

## Verify
```bash
systemctl status redis-server
rspamc stat
```

## Do not destroy production data
Never use `FLUSHALL` or `FLUSHDB` on the production Redis instance unless intentional destruction is required.
