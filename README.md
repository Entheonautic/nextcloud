#### Download latest version:
https://download.nextcloud.com/server/releases/

#### Add some required packages and enable (not start):
```
pkg_add php php-curl php-gd php-intl php-pdo_pgsql php-zip postgresql-server gnupg unzip
rcctl enable postgresql httpd phpXX_fpm
```

#### Configure PHP and start:
```
ln -s /usr/local/bin/php /usr/bin/php # or change path in php code.
cp /etc/php-X.X.sample/* /etc/php-X.X
rcctl start phpXX_fpm
```

#### Initialize and start postgresql:
```
su - _postgresql
mkdir /var/postgresql/data
initdb -D /var/postgresql/data -U postgres -A scram-sha-256 -E UTF8 -W
logout
rcctl start postgresql
```

#### Set some values:
```
sysctl -w kern.seminfo.semmni=60
sysctl -w kern.seminfo.semmns=1024
echo kern.seminfo.semmni=60 >> /etc/sysctl.conf
echo kern.seminfo.semmns=1024 >> /etc/sysctl.conf
```

#### Configure DNS in the chroot:
```
mkdir -p /var/www/etc/ssl
grep ^nameserver /etc/resolv.conf > /var/www/etc/resolv.conf
touch /var/www/etc/hosts # optional
cp -a /etc/ssl/{cert.pem, openssl.cnf} /var/www/etc/ssl/
chown -R www:www /var/www/etc
```

#### Using Let's Encrypt for SSL certification 
#### edit /etc/acme-client.conf:
```
authority letsencrypt {
	api url "https://acme-v02.api.letsencrypt.org/directory"
	account key "/etc/acme/letsencrypt-privkey.pem"
}

authority letsencrypt-staging {
	api url "https://acme-staging.api.letsencrypt.org/directory"
	account key "/etc/acme/letsencrypt-staging-privkey.pem"
}

domain domain.tld {
	domain key "/etc/ssl/private/domain.tld.key"
	domain certificate "/etc/ssl/domain.tld.crt"
	domain full chain certificate "/etc/ssl/domain.tld_fullchain.pem"
	sign with letsencrypt
}
```

#### Edit: /etc/httpd.conf (verify config with httpd -nv):
```
server "domain.tld" {
        listen on * port 80
        location "/.well-known/acme-challenge/*" {
                root { "/acme" }
                request strip 2
        }
}
```

#### Start httpd:
```
rcctl start httpd
```

#### Fetch the certificates:
```
acme-client -v domain.tld
```

#### Grab the OCSP stapling file:
```
ocspcheck -N -o /etc/ssl/domain.tld.ocsp.pem /etc/ssl/domain.tld_fullchain.pem
```

#### Edit root's crontab to renew certs and obtain stapling file:
```
doas crontab -e
0 0 * * * acme-client domain.tld && rcctl reload httpd
0 * * * * ocspcheck -N -o /etc/ssl/domain.tld.ocsp.pem /etc/ssl/domain.tld_fullchain.pem && rcctl reload httpd
```

#### Change /etc/httpd.conf to have:
```
ext_addr="*"
domain="domain.tld"

server $domain {
        listen on $ext_addr port 80
        block return 301 "https://$SERVER_NAME$REQUEST_URI"
}

server $domain {
        listen on $ext_addr tls port 443

        tls {
          # tt-rss, nextcloud, etc apps require the chain:
                certificate "/etc/ssl/domain.tld_fullchain.crt"
                key "/etc/ssl/private/domain.tld.key"
                ocsp "/etc/ssl/domain.tld.ocsp.pem"
        }

        directory {
                index "index.php"
        }

        location "*.php*" {
                fastcgi socket "/run/php-fpm.sock"
        }

        location "/nextcloud" {
                root "/htdocs/nextcloud"
        }
}

# Include MIME types instead of the built-in ones
types {
        include "/usr/share/misc/mime.types"
}
```

#### Restart httpd:
```
rcctl restart httpd
```

#### Edit: /etc/php-X.X.ini:
```
memory_limit = 256M
max_input_time = 180
upload_max_filesize = 512M
post_max_size = 32M
opcache.enable=1
opcache.enable_cli=1
opcache.memory_consumption=128
opcache.interned_strings_buffer=8
opcache.max_accelerated_files=10000
opcache.revalidate_freq=1
opcache.save_comments = 1
```

#### Download / install nextcloud:
```
ftp https://download.nextcloud.com/server/releases/nextcloud-XX.X.X.zip{,.asc}
gpg2 --fetch-keys https://nextcloud.com/nextcloud.asc
gpg2 --verify nextcloud-*.zip.asc
unzip -d /var/www/htdocs nextcloud-*.zip
chown -R www:www /var/www/htdocs/nextcloud
```

#### Create database and permissions:
```
doas su - _postgresql
psql -d template1 -U postgres
  CREATE USER nextcloud WITH PASSWORD 'secret-password';
  CREATE DATABASE nextcloud;
  GRANT ALL PRIVILEGES ON DATABASE nextcloud to nextcloud;
  \q
```

#### Connect to browser and fill in the fields:
```
Username: Choose a username
Password: Enter a strong password
Datadirectory: `nextcloud/data`
Database type: PostgreSQL
Database user: nextcloud
Database password: 'secret-password' from before
Database name: nextcloud
Database host: localhost
```

#### Housekeeping and indexing. By default, it runs one task with each page load. The preferred way here is to set this via a cronjob under the www user (doas -u www crontab -e):
```
*/15  *  *  *  * php -f /var/www/nextcloud/cron.php
```

#### re-start services:
```
rcctl restart httpd phpXX_fpm
```

#### Connect to your site and fill in form which creates: /var/www/htdocs/nextcloud/config/config.php
```
https://domain.tld/nextcloud/
```
