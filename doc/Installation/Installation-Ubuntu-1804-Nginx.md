source: Installation/Installation-Ubuntu-1804-Nginx.md
path: blob/master/doc/

> NOTE: These instructions assume you are the **root** user.  If you
> are not, prepend `sudo` to the shell commands (the ones that aren't
> at `mysql>` prompts) or temporarily become a user with root
> privileges with `sudo -s` or `sudo -i`.

**Please note the minimum supported PHP version is 7.2.5**

# Install Required Packages

```bash
apt install software-properties-common
add-apt-repository universe
sudo add-apt-repository ppa:ondrej/php
apt update
apt install php7.4
apt install acl sed curl composer fping git graphviz imagemagick mariadb-client mariadb-server mtr-tiny nginx-full nmap php7.4-cli php7.4-curl php7.4-fpm php7.4-gd php7.4-json php7.4-mbstring php7.4-mysql php7.4-snmp php7.4-xml php7.4-zip python3-memcache python3-mysqldb rrdtool snmp snmpd whois unzip python3-pip
```

# Add librenms user

```bash
useradd librenms -d /opt/librenms -M -r
usermod -a -G librenms www-data
```

# Download LibreNMS

```bash
cd /opt
git clone https://github.com/librenms/librenms.git
```

# Set permissions

```bash
chown -R librenms:librenms /opt/librenms
chmod 770 /opt/librenms
setfacl -d -m g::rwx /opt/librenms/rrd /opt/librenms/logs /opt/librenms/bootstrap/cache/ /opt/librenms/storage/
setfacl -R -m g::rwx /opt/librenms/rrd /opt/librenms/logs /opt/librenms/bootstrap/cache/ /opt/librenms/storage/
```

# Install PHP dependencies

```bash
cd /op  
cd /opt/librenms
./scripts/composer_wrapper.php install --no-dev
exit
```

# DB Server

## Configure MySQL

> NOTE: Please change the `PASSWORD` below to something secure.

```sql
mysql -u root <<EOF
    CREATE DATABASE librenms CHARACTER SET utf8 COLLATE utf8_unicode_ci;
    CREATE USER 'librenms'@'localhost' IDENTIFIED BY 'PASSWORD';
    GRANT ALL PRIVILEGES ON librenms.* TO 'librenms'@'localhost';
    FLUSH PRIVILEGES;
    exit
EOF
```

```bash
nano /etc/mysql/mariadb.conf.d/50-server.cnf
```

Within the `[mysqld]` section please add:

```bash
innodb_file_per_table=1
lower_case_table_names=0
```

```bash
systemctl restart mysql
```

# Web Server

## Configure and Restart PHP-FPM

Ensure `date.timezone` is set in `php.ini` to your preferred time zone.

Valid examples are: `"America/New_York"`, `"Australia/Brisbane"`, `"Europe/Bratislava"`, `"Etc/UTC"`.

>See <http://php.net/manual/en/timezones.php> for a list of supported timezones and change `Europe/Bratislava` in sed command to your timezone.

```bash
echo date.timezone = \"Europe/Bratislava\" >> /etc/php/7.4/fpm/php.ini
echo date.timezone = \"Europe/Bratislava\" >> /etc/php/7.4/cli/php.ini
```

Restart php-fpm

```bash
systemctl restart php7.4-fpm
```

## Configure NGINX

```bash
nano /etc/nginx/conf.d/librenms.conf
```

Add the following config, edit `server_name` as required:

```nginx
server {
    listen      80;
    server_name librenms.example.com;
    root        /opt/librenms/html;
    index       index.php;

    charset utf-8;
    gzip on;
    gzip_types text/css application/javascript text/javascript application/x-javascript image/svg+xml text/plain text/xsd  text/xsl text/xml image/x-icon;
 
    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }
 
    location /api/v0 {
        try_files $uri $uri/ /api_v0.php?$query_string;
    }
 
    location ~ \.php {
        include fastcgi.conf;
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass unix:/var/run/php/php7.2-fpm.sock;
    }
 
    location ~ /\.ht {
        deny all;
    }
}
```

Delete default nginx site.

```bash
rm /etc/nginx/sites-enabled/default
systemctl restart nginx
```

Check nginx configuration and restart nginx service.

```bash
nginx -t
systemctl restart nginx
```

# Configure snmpd

Copy example `snmpd.conf`.

```bash
cp /opt/librenms/snmpd.conf.example /etc/snmp/snmpd.conf
```

Edit the text which says `public` and set your own community string if you need it.

```bash
sed -e 's/RANDOMSTRINGGOESHERE/public/' -i /etc/snmp/snmpd.conf
```

Set up distro and restart `snmpd` service.

```bash
curl -o /usr/bin/distro https://raw.githubusercontent.com/librenms/librenms-agent/master/snmp/distro
chmod +x /usr/bin/distro
systemctl restart snmpd
```

# Cron job

```bash
cp /opt/librenms/librenms.nonroot.cron /etc/cron.d/librenms
```

> NOTE: Keep in mind  that cron, by default, only uses a very limited
> set of environment variables. You may need to configure proxy
> variables for the cron invocation. Alternatively adding the proxy
> settings in config.php is possible too. The config.php file will be
> created in the upcoming steps. Review the following URL after you
> finished librenms install steps:
> <https://docs.librenms.org/Support/Configuration/#proxy-support>

# Copy logrotate config

LibreNMS keeps logs in `/opt/librenms/logs`. Over time these can
become large and be rotated out.  To rotate out the old logs you can
use the provided logrotate config file:

```bash
cp /opt/librenms/misc/librenms.logrotate /etc/logrotate.d/librenms
```

# Set permissions

```bash
chown -R librenms:librenms /opt/librenms
setfacl -d -m g::rwx /opt/librenms/rrd /opt/librenms/logs /opt/librenms/bootstrap/cache/ /opt/librenms/storage/
setfacl -R -m g::rwx /opt/librenms/rrd /opt/librenms/logs /opt/librenms/bootstrap/cache/ /opt/librenms/storage/
```

# Web installer

Now head to the web installer and follow the on-screen instructions.

<http://librenms.example.com/install.php>

The web installer might prompt you to create a `config.php` file in
your librenms install location manually, copying the content displayed
on-screen to the file. If you have to do this, please remember to set
the permissions on config.php after you copied the on-screen contents
to the file. 

```bash
chown librenms:librenms /opt/librenms/config.php
```

# Final steps

That's it!  You now should be able to log in to
<http://librenms.example.com/>.  Please note that we have not covered
HTTPS setup in this example, so your LibreNMS install is not secure by
default.  Please do not expose it to the public Internet unless you
have configured HTTPS and taken appropriate web server hardening
steps.

# Add the first device

We now suggest that you add localhost as your first device from within
the WebUI.

# Troubleshooting

If you ever have issues with your install, run validate.php as root in
the librenms directory:

```bash
cd /opt/librenms
./validate.php
```

There are various options for getting help listed on the LibreNMS web
site: <https://www.librenms.org/#support>

# What next?

Now that you've installed LibreNMS, we'd suggest that you have a read
of a few other docs to get you going:

- [Performance tuning](http://docs.librenms.org/Support/Performance)
- [Alerting](http://docs.librenms.org/Extensions/Alerting/)
- [Device Groups](http://docs.librenms.org/Extensions/Device-Groups/)
- [Auto discovery](http://docs.librenms.org/Extensions/Auto-Discovery/)

# Closing

We hope you enjoy using LibreNMS. If you do, it would be great if you
would consider opting into the stats system we have, please see [this
page](http://docs.librenms.org/General/Callback-Stats-and-Privacy/) on
what it is and how to enable it.

If you would like to help make LibreNMS better there are [many ways to
help](http://docs.librenms.org/Support/FAQ/#what-can-i-do-to-help). You
can also [back LibreNMS on Open
Collective](https://t.libren.ms/donations).
