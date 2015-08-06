# redis-nginx-ssl

## Disclaimer #1

> [Redis is designed to be accessed by trusted clients inside trusted environments.](http://redis.io/topics/security)

## Disclaimer #2

> It has not been audited nor battle-tested and does not provide high-availability nor durability.

That said, this is a cloud-config script designed to be used as a template to setup nginx as an SSL proxy in front of Redis, allowing access from any client that understands HTTP.

## What it does

### SSH

> root login is forbidden and only the `ubuntu` user may SSH in.

```
sed -i -e '/^PermitRootLogin/s/^.*$/PermitRootLogin no/' /etc/ssh/sshd_config
sed -i -e '$aAllowUsers ubuntu' /etc/ssh/sshd_config
```

### Uncomplicated Firewall

> incoming connections are only allowed on port 22 (SSH) and 443 (HTTPS)

```
ufw default deny incoming
ufw default allow outgoing
ufw allow 22/tcp
ufw allow 443/tcp
```

### SSL

> generate a self-signed certificate amd Diffie-Hellman parameters for forward secrecy (depending on your machine, this key takes a while to generate)

```
openssl req -x509 -nodes -days 365 -newkey rsa:4096 -subj "/C=CA/ST=MyState/L=MyCity/O=MyOrganization /CN=my.example.com" -keyout /etc/nginx/ssl/nginx.key -out /etc/nginx/ssl/nginx.crt
openssl dhparam -out /etc/nginx/ssl/dhparams.pem 4096
```

### Redis

> install, bind to 127.0.0.1 and set a password (it should be *very* long)

```
sed -ie 's/# bind 127.0.0.1/bind 127.0.0.1/g' /etc/redis/6379.conf
sed -ie '/^# requirepass/s/^.*$/requirepass MyVeryComplexPassword/' /etc/redis/6379.conf
```

### nginx

> installed from source, creates a daemon, listens for SSL connections and implements forward secrecy

```nginx
# truncated for brevity
http {
  ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
  ssl_prefer_server_ciphers on;
  ssl_ciphers "EECDH+ECDSA+AESGCM EECDH+aRSA+AESGCM EECDH+ECDSA+SHA384 EECDH+ECDSA+SHA256 EECDH+aRSA+SHA384 EECDH+aRSA+SHA256 EECDH+aRSA+RC4 EECDH EDH+aRSA RC4 !aNULL !eNULL !LOW !3DES !MD5 !EXP !PSK !SRP !DSS !MEDIUM";
  ssl_dhparam /etc/nginx/ssl/dhparams.pem;
  
  add_header Strict-Transport-Security "max-age=31536000; includeSubDomains";
  add_header X-Frame-Options SAMEORIGIN;
  
  server {
    listen 443 ssl;
    ssl_certificate /etc/nginx/ssl/nginx.crt;
    ssl_certificate_key /etc/nginx/ssl/nginx.key;
  }
}
```

As of August 6th 2015, it gets the following score in the [Qualys SSL Server Test](https://www.ssllabs.com/ssltest/)
- Protocol Support 95
- Key Exchange 100
- Cipher Strength 90

## Known issues

1. `redis-server` runs with root privileges`
1. Directories where the SSL certificate and key need stricter permissions
1. Ubuntu hangs when rebooting at the end of the script and requires a power cycle

## Licence

MIT
