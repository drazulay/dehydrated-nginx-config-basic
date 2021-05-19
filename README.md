# Configuration for Dehydrated
Disclaimer: This guide was written off the top of my head so there may be some mistakes or inconsistencies. If you bork your machine, that's on you.

## Prerequisites:
- [nginx](https://www.nginx.com/)
- [dehydrated](https://github.com/dehydrated-io/dehydrated)

This guide assumes you are running debian/ubuntu, you have a user `dehydrated` and nginx is configured to run as `www-data`

## Usage
- Place the configuration files under `src` in their respective locations in your root filesystem, make sure their permissions are correct (e.g. `root:root`) and include `/etc/nginx/snippets/dehydrated` in the server block for `/etc/nginx/sites-available/default`:

```
server {
        listen 80 default_server;
        listen [::]:80 default_server;
        server_name _;

	location ^~ /.well-known/acme-challenge {
		alias /var/www/dehydrated;
		include /etc/nginx/snippets/dehydrated;
	}
	
	location ^~ / {
		return 301 https://$host$request_uri;
	}
}
```

- Create a directory: `mkdir -p /var/www/dehydrated/acme`.

- Add a file `/var/www/dehydrated/domains.txt` and add your domains, one per line:

```
mydomain.eu
www.mydomain.eu
```

- Set the correct permissions for `/var/www/dehydrated`:
`sudo chown -R dehydrated:www-data /var/www/dehydrated`

- Run dehydrated to fetch your keypairs:
`sudo dehydrated -4 -a secp384r1 -t http-01 -c --accept-terms`

**Note:** you only need to run dehydrated manually once, in order make sure it works and to get the initial key/cert pairs for your nginx configuration. From now on cron should do this twice daily, if you correctly copied over the files at the beginning.

- As user `dehydrated`, run..
```
cd /var/www/dehydrated/certs
openssl dhparam -out dhparam.pem 4096
```
..to get your dhparam.pem file for use in nginx's ssl configs. This will take a while as it needs to farm some entropy.

- Now edit your nginx configuration to enable ssl and include the relevant keypairs. Something like this:

```
	listen 443 ssl;
        listen [::]:443 ssl;
	server_name mydomain.eu www.mydomain.eu
	
        ssl_certificate /var/www/dehydrated/certs/mydomain.eu/fullchain.pem;
        ssl_certificate_key /var/www/dehydrated/certs/mydomain.eu/privkey.pem;
	ssl_dhparam /var/www/dehydrated/certs/dhparam.pem;
	
	ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
        ssl_ciphers "EECDH+AESGCM:EDH+AESGCM:ECDHE-RSA-AES128-GCM-SHA256:AES256+EECDH:DHE-RSA-AES128-GCM-SHA256:AES256+EDH:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-SHA384:ECDHE-RSA-AES128-SHA256:ECDHE-RSA-AES256-SHA:ECDHE-RSA-AES128-SHA:DHE-RSA-AES256-SHA256:DHE-RSA-AES128-SHA256:DHE-RSA-AES256-SHA:DHE-RSA-AES128-SHA:ECDHE-RSA-DES-CBC3-SHA:EDH-RSA-DES-CBC3-SHA:AES256-GCM-SHA384:AES128-GCM-SHA256:AES256-SHA256:AES128-SHA256:AES256-SHA:AES128-SHA:DES-CBC3-SHA:HIGH:!aNULL:!eNULL:!EXPORT:!DES:!MD5:!PSK:!RC4";
	
        ssl_prefer_server_ciphers on;
        ssl_session_cache shared:SSL:10m;
```

- Restart nginx:
`sudo service nginx restart`

Neat!
