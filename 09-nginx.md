# Nginx and HTTPS Configuration

Nginx provided HTTPS for `mail.example.com` and reverse-proxied the Rspamd WebUI.

## Rspamd proxy pattern
The intended location was:
```nginx
location /rspamd/ {
    proxy_pass http://127.0.0.1:11334/;

    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For "";
}
```

The trailing slash in `proxy_pass` was important during troubleshooting. Removing it caused frontend problems.

## Verification
```bash
sudo nginx -t
sudo nginx -T
```

Reload:
```bash
sudo systemctl reload nginx
```

## Static files checked
These returned HTTP 200 during troubleshooting:
```text
/rspamd/js/main.js
/rspamd/js/lib/require.min.js
/rspamd/js/app/rspamd.js
/rspamd/css/rspamd.css
```

## Logs
```bash
sudo tail -f /var/log/nginx/access.log
sudo tail -f /var/log/nginx/error.log
```

The exact certificate issuance/renewal mechanism was not preserved in the available history, so this tutorial does not invent a Certbot/ACME configuration.
