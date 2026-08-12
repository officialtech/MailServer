# Rspamd Configuration

Rspamd provided spam filtering, scoring, Bayes classification and the WebUI.

Installed version at the pause point:
```text
rspamd 3.8.1-1.2ubuntu3
Ubuntu 26.04 LTS / resolute
```

## Bayes
The active classifier configuration was:
```text
classifier {
    bayes {
        backend = "redis";
        cache {
            backend = "redis";
        }
        tokenizer {
            name = "osb";
        }
        store_tokens = false;
        signatures = false;
        min_tokens = 11;
        min_learns = 200;
        learn_condition = "return require(\"lua_bayes_learn\").can_learn";
        autolearn = true;
        new_schema = true;
        servers = "127.0.0.1";

        statfile {
            spam = false;
            symbol = "BAYES_HAM";
        }

        statfile {
            spam = true;
            symbol = "BAYES_SPAM";
        }
    }
}
```

## Validate
```bash
rspamadm configdump classifier
sudo rspamadm configtest
```
Expected:
```text
syntax OK
```

The packaged file was:
```text
/etc/rspamd/modules.d/ratelimit.conf
```
It instructed administrators to use:
```text
/etc/rspamd/local.d/ratelimit.conf
/etc/rspamd/override.d/ratelimit.conf
```
At the time checked, no custom local/override ratelimit file existed.

## WebUI

The Rspamd WebUI can be exposed behind an HTTPS reverse proxy rather than exposing the controller port directly.

The controller should remain bound to localhost, for example:

```text
localhost:11334
```

A reverse proxy can publish it at:

```text
https://mail.example.com/rspamd/
```

Keep the controller password protected and do not expose port 11334 directly to the Internet.


