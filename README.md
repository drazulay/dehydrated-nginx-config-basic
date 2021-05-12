# Configuration for Dehydrated
Disclaimer: This guide was written off the top of my head so there may be some mistakes or inconsistencies. If you bork your machine, that's on you.
## Prerequisites:
- [nginx](https://www.nginx.com/)
- [dehydrated](https://github.com/dehydrated-io/dehydrated)

This guide assumes you are running debian/ubuntu, you have a user `deploy` and nginx is configured to run as `www-data`

## Usage
- Place the configuration files under `src` in their respective locations in your root filesystem, make sure their permissions are correct (e.g. `root:root`) and include `/etc/nginx/snippets/dehydrated` in the server block for `/etc/nginx/sites-available/default`:

```
server {
        listen 80 default_server;
        listen [::]:80 default_server;
        server_name _;

	    include /etc/nginx/snippets/dehydrated;

        return 301 https://$host$request_uri;
}
```

- Create a directory `/var/www/dehydrated/acme`.

- Add a file `/var/www/dehydrated/domains.txt` and add your domains, one per line:

```
myfirstdomain.eu
myseconddomain.co.uk
```

- Set the correct permissions for `/var/www/dehydrated`:
`sudo chown -R deploy:www-data /var/www/dehydrated`

- Run dehydrated to fetch your keypairs:
`sudo dehydrated -4 -a prime256v1 -t http-01 -c --accept-terms`

**Note:** you only need to do this once in order to get the needed files for your nginx configuration, from now on cron should do this twice daily.

- Now edit your nginx configuration to include the relevant keypairs:

```
        ssl_certificate     /var/www/dehydrated/certs/myfirstdomain.eu/fullchain.pem;
        ssl_certificate_key /var/www/dehydrated/certs/myfirstdomain.eu/privkey.pem;
```

- Restart nginx:
`sudo service nginx restart`