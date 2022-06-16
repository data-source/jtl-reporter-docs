---
title: Configuring SSL for JTL-Reporter on an AWS EC2 instance
---
NB: This guide assumes an Ubuntu VM

### Prerequisites
* AWS EC2 instance up and running with JTL Docker images at publicipv4:2020 
([See guide here](https://jtlreporter.site/docs/guides/aws-ec2-installation))
* [Elastic IP assigned to VM host](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/elastic-ip-addresses-eip.html) 
* Certificate, key, CA 

### Preparation

* Connect to your instance via SSH 
* Install apache2 and the required modules 

```
sudo apt install apache2 
sudo a2enmod ssl
sudo a2enmod proxy
sudo a2enmod proxy_balancer
sudo a2enmod proxy_http
sudo a2enmod rewrite
sudo a2enmod headers
```

### Configuration 
Prepare files for your site and keep a backup
```
sudo cp /etc/apache2/sites-available/default-ssl.conf /etc/apache2/sites-available/default-ssl.conf.bak
sudo cp /etc/apache2/sites-available/000-default.conf /etc/apache2/sites-available/<yourfqdn>.conf
```

Move certificate, CA and key to their respective folders and apply ownership
```
sudo mv /tmp/DigiCertCA.crt /tmp/certificate.crt /etc/ssl/certs 
sudo mv /tmp/key.key /etc/ssl/private 

sudo chmod 644 /etc/ssl/certs/DigiCertCA.crt 
sudo chmod 644 /etc/ssl/certs/certificate.crt
sudo chmod 600 /etc/ssl/private/key.key

sudo chown root /etc/ssl/private/key.key
sudo chown root /etc/ssl/certs/DigiCertCA.crt
sudo chown root /etc/ssl/certs/certificate.crt
```

Modify your **/etc/hosts** file with your [Elastic IP to your EC2 instance](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/elastic-ip-addresses-eip.html)
```
127.0.0.1 localhost
<elasticip> <fqdn>
0.0.0.0 jtl-reporter-main_fe_1
```

Edit the **/etc/apache2/sites-available/<yourfqdn>.conf** file:
```
<VirtualHost *:80>
    ServerName         <yourfqdn>
    ProxyRequests Off
    ProxyVia Off
    Redirect permanent / https://<yourfqdn>
</VirtualHost>

<VirtualHost *:443>
   <Proxy *>
    Require all granted
   </Proxy>

    ServerName <yourfqdn>
    ProxyPreserveHost On

    ProxyPass / http://jtl-reporter-main_fe_1:2020
    ProxyPassReverse / http://jtl-reporter-main_fe_1:2020

    <Location />
    ProxyPass          http://jtl-reporter-main_fe_1:2020/
    ProxyPassReverse   http://jtl-reporter-main_fe_1:2020/
    SetEnv             proxy-sendchunks 1
    </Location>

    SSLEngine On
    SSLCertificateFile /etc/ssl/certs/certificate.crt
    SSLCertificateKeyFile /etc/ssl/private/key.key
    SSLCertificateChainFile /etc/ssl/certs/DigiCertCA.crt
    Header always set Strict-Transport-Security "max-age=15768000"
</VirtualHost>
# vim: syntax=apache ts=4 sw=4 sts=4 sr noet
```

Edit the **/etc/apache2/sites-available/default-ssl.conf** file: 
```
<IfModule mod_ssl.c>
            <VirtualHost *:443>

                ServerName <yourfqdn>

            <Directory >

                Options Indexes FollowSymLinks MultiViews
                AllowOverride All
                Order allow,deny
                allow from all

            </Directory>

                SSLEngine on
                SSLProxyEngine on
                ProxyPass / http://jtl-reporter-main_fe_1:2020
                ProxyPassReverse / http://jtl-reporter-main_fe_1:2020
                SSLCipherSuite EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH
                SSLProtocol all -SSLv3
                SSLHonorCipherOrder On

                SSLCertificateFile /etc/ssl/certs/certificate.crt
                SSLCertificateKeyFile /etc/ssl/private/key.key
                SSLCertificateChainFile /etc/ssl/certs/DigiCertCA.crt

                ErrorLog ${APACHE_LOG_DIR}/error.log
                CustomLog ${APACHE_LOG_DIR}/access.log combined

            </VirtualHost>
</IfModule>
# vim: syntax=apache ts=4 sw=4 sts=4 sr noet
```

Edit the **/etc/apache2/apache2.conf** file, so that Server Root and directories are commented, here is an example: 
```
    # This is the main Apache server configuration file.  It contains the 
# configuration directives that give the server its instructions.
# See http://httpd.apache.org/docs/2.4/ for detailed information about
# the directives and /usr/share/doc/apache2/README.Debian about Debian specific
# hints.
#
#
# Summary of how the Apache 2 configuration works in Debian:
# The Apache 2 web server configuration in Debian is quite different to
# upstream's suggested way to configure the web server. This is because Debian's
# default Apache2 installation attempts to make adding and removing modules,
# virtual hosts, and extra configuration directives as flexible as possible, in
# order to make automating the changes and administering the server as easy as
# possible.

# It is split into several files forming the configuration hierarchy outlined
# below, all located in the /etc/apache2/ directory:
#
#       /etc/apache2/
#       |-- apache2.conf
#       |       `--  ports.conf
#       |-- mods-enabled
#       |       |-- *.load
#       |       `-- *.conf
#       |-- conf-enabled
#       |       `-- *.conf
#       `-- sites-enabled
#               `-- *.conf
#
#
# * apache2.conf is the main configuration file (this file). It puts the pieces
#   together by including all remaining configuration files when starting up the
#   web server.
#
# * ports.conf is always included from the main configuration file. It is
#   supposed to determine listening ports for incoming connections which can be
#   customized anytime.
#
# * Configuration files in the mods-enabled/, conf-enabled/ and sites-enabled/
#   directories contain particular configuration snippets which manage modules,
#   global configuration fragments, or virtual host configurations,
#   respectively.
#
#   They are activated by symlinking available configuration files from their
#   respective *-available/ counterparts. These should be managed by using our
#   helpers a2enmod/a2dismod, a2ensite/a2dissite and a2enconf/a2disconf. See
#   their respective man pages for detailed information.
#
# * The binary is called apache2. Due to the use of environment variables, in
#   the default configuration, apache2 needs to be started/stopped with
#   /etc/init.d/apache2 or apache2ctl. Calling /usr/bin/apache2 directly will not
#   work with the default configuration.


# Global configuration
#

#
# ServerRoot: The top of the directory tree under which the server's
# configuration, error, and log files are kept.
#
# NOTE!  If you intend to place this on an NFS (or otherwise network)
# mounted filesystem then please read the Mutex documentation (available
# at <URL:http://httpd.apache.org/docs/2.4/mod/core.html#mutex>);
# you will save yourself a lot of trouble.
#
# Do NOT add a slash at the end of the directory path.
#
#ServerRoot "/etc/apache2"

#
# The accept serialization lock file MUST BE STORED ON A LOCAL DISK.
#
#Mutex file:${APACHE_LOCK_DIR} default

#
# The directory where shm and other runtime files will be stored.
#

DefaultRuntimeDir ${APACHE_RUN_DIR}

#
# PidFile: The file in which the server should record its process
# identification number when it starts.
# This needs to be set in /etc/apache2/envvars
#
PidFile ${APACHE_PID_FILE}
#
# Timeout: The number of seconds before receives and sends time out.
#
Timeout 300

#
# KeepAlive: Whether or not to allow persistent connections (more than
# one request per connection). Set to "Off" to deactivate.
#
KeepAlive On

#
# MaxKeepAliveRequests: The maximum number of requests to allow
# during a persistent connection. Set to 0 to allow an unlimited amount.
# We recommend you leave this number high, for maximum performance.
#
MaxKeepAliveRequests 100

#
# KeepAliveTimeout: Number of seconds to wait for the next request from the
# same client on the same connection.
#
KeepAliveTimeout 5


# These need to be set in /etc/apache2/envvars
User ${APACHE_RUN_USER}
Group ${APACHE_RUN_GROUP}

#
# HostnameLookups: Log the names of clients or just their IP addresses
# e.g., www.apache.org (on) or 204.62.129.132 (off).
# The default is off because it'd be overall better for the net if people
# had to knowingly turn this feature on, since enabling it means that
# each client request will result in AT LEAST one lookup request to the
# nameserver.
#
HostnameLookups Off

# ErrorLog: The location of the error log file.
# If you do not specify an ErrorLog directive within a <VirtualHost>
# container, error messages relating to that virtual host will be
# logged here.  If you *do* define an error logfile for a <VirtualHost>
# container, that host's errors will be logged there and not here.
#
ErrorLog ${APACHE_LOG_DIR}/error.log

#
# LogLevel: Control the severity of messages logged to the error_log.
# Available values: trace8, ..., trace1, debug, info, notice, warn,
# error, crit, alert, emerg.
# It is also possible to configure the log level for particular modules, e.g.
# "LogLevel info ssl:warn"
#
LogLevel warn

# Include module configuration:
IncludeOptional mods-enabled/*.load
IncludeOptional mods-enabled/*.conf

# Include list of ports to listen on
Include ports.conf


# Sets the default security model of the Apache2 HTTPD server. It does
# not allow access to the root filesystem outside of /usr/share and /var/www.
# The former is used by web applications packaged in Debian,
# the latter may be used for local directories served by the web server. If
# your system is serving content from a sub-directory in /srv you must allow
# access here, or in any related virtual host.
#<Directory />
#       Options FollowSymLinks
#       AllowOverride None
#       Require all denied
#</Directory>

#<Directory /usr/share>
#       AllowOverride None
#       Require all granted
#</Directory>

#<Directory /var/www/>
#       Options Indexes FollowSymLinks
#       AllowOverride None
#       Require all granted
#</Directory>

#<Directory /srv/>
#       Options Indexes FollowSymLinks
#       AllowOverride None
#       Require all granted
#</Directory>




# AccessFileName: The name of the file to look for in each directory
# for additional configuration directives.  See also the AllowOverride
# directive.
#
AccessFileName .htaccess

#
# The following lines prevent .htaccess and .htpasswd files from being
# viewed by Web clients.
#
<FilesMatch "^\.ht">
        Require all denied
</FilesMatch>


#
# The following directives define some format nicknames for use with
# a CustomLog directive.
#
# These deviate from the Common Log Format definitions in that they use %O
# (the actual bytes sent including headers) instead of %b (the size of the
# requested file), because the latter makes it impossible to detect partial
# requests.
#
# Note that the use of %{X-Forwarded-For}i instead of %h is not recommended.
# Use mod_remoteip instead.
#
LogFormat "%v:%p %h %l %u %t \"%r\" %>s %O \"%{Referer}i\" \"%{User-Agent}i\"" vhost_combined
LogFormat "%h %l %u %t \"%r\" %>s %O \"%{Referer}i\" \"%{User-Agent}i\"" combined
LogFormat "%h %l %u %t \"%r\" %>s %O" common
LogFormat "%{Referer}i -> %U" referer
LogFormat "%{User-agent}i" agent

# Include of directories ignores editors' and dpkg's backup files,
# see README.Debian for details.

# Include generic snippets of statements
IncludeOptional conf-enabled/*.conf

# Include the virtual host configurations:
IncludeOptional sites-enabled/*.conf

# vim: syntax=apache ts=4 sw=4 sts=4 sr noet
```

Verify your **/etc/apache2/ports.conf** file is listening on both 80 and 443 
```
# If you just change the port or add more ports here, you will likely also
# have to change the VirtualHost statement in
# /etc/apache2/sites-enabled/000-default.conf

Listen 80

<IfModule ssl_module>
        Listen 443
</IfModule>

<IfModule mod_gnutls.c>
        Listen 443
</IfModule>

# vim: syntax=apache ts=4 sw=4 sts=4 sr noet
```

You can harden your config by adding/uncommenting directives in **/etc/apache2/conf-enabled/security.conf**, here is an example file: 
```
#
# Disable access to the entire file system except for the directories that
# are explicitly allowed later.
#
# This currently breaks the configurations that come with some web application
# Debian packages.
#
#<Directory />
#   AllowOverride None
#   Require all denied
#</Directory>

# Changing the following options will not really affect the security of the
# server, but might make attacks slightly more difficult in some cases.

#
# ServerTokens
# This directive configures what you return as the Server HTTP response
# Header. The default is 'Full' which sends information about the OS-Type
# and compiled in modules.
# Set to one of:  Full | OS | Minimal | Minor | Major | Prod
# where Full conveys the most information, and Prod the least.
#ServerTokens Minimal
ServerTokens Prod
#ServerTokens Full

#
# Optionally add a line containing the server version and virtual host
# name to server-generated pages (internal error documents, FTP directory
# listings, mod_status and mod_info output etc., but not CGI generated
# documents or custom error documents).
# Set to "EMail" to also include a mailto: link to the ServerAdmin.
# Set to one of:  On | Off | EMail
ServerSignature Off
#ServerSignature On

#
# Allow TRACE method
#
# Set to "extended" to also reflect the request body (only for testing and
# diagnostic purposes).
#
# Set to one of:  On | Off | extended
TraceEnable Off
#TraceEnable On

#
# Forbid access to version control directories
#
# If you use version control systems in your document root, you should
# probably deny access to their directories. For example, for subversion:
#
#<DirectoryMatch "/\.svn">
#   Require all denied
#</DirectoryMatch>

#
# Setting this header will prevent MSIE from interpreting files as something
# else than declared by the content type in the HTTP headers.
# Requires mod_headers to be enabled.
#
#Header set X-Content-Type-Options: "nosniff"
Header unset Server

#
# Setting this header will prevent other sites from embedding pages from this
# site as frames. This defends against clickjacking attacks.
# Requires mod_headers to be enabled.
#
#Header set X-Frame-Options: "sameorigin"
Header unset X-Powered-By

Header set X-Content-Type-Options: "nosniff"
Header set X-Permitted-Cross-Domain-Policies: "master-only"
Header set X-XSS-Protection: "1; mode=block"

# vim: syntax=apache ts=4 sw=4 sts=4 sr noet

# Disable the old and weak/broken protocols SSLv2 and SSLv3. Instead, the web server will use TLSv1.0, TLSv1.1 and TLSv1.2
SSLProtocol all -SSLv2 -SSLv3
# Define secure list of availabel Ciphers to choose from
SSLCipherSuite ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-DSS-AES128-GCM-SHA256:kEDH+AESGCM:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA:ECDHE-ECDSA-AES128-SHA:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA:ECDHE-ECDSA-AES256-SHA:DHE-RSA-AES128-SHA256:DHE-RSA-AES128-SHA:DHE-DSS-AES128-SHA256:DHE-RSA-AES256-SHA256:DHE-DSS-AES256-SHA:DHE-RSA-AES256-SHA:!aNULL:!eNULL:!EXPORT:!DES:!RC4:!3DES:!MD5:!PSK
# Honor = Choose Web Server Cipher choice over clients preferences (http://httpd.apache.org/docs/2.2/mod/mod_ssl.html#sslhonorcipherorder)
SSLHonorCipherOrder on
# Disable SSL compression to avoid CRIME attack - http://httpd.apache.org/docs/2.2/mod/mod_ssl.html#sslcompression
SSLCompression off

# OCSP stapling allows clients to check a certificates revocation status - https://en.wikipedia.org/wiki/OCSP_stapling
# OCSP Stapling, is only available in Apache 2.3.3 and later
SSLUseStapling on
SSLStaplingResponderTimeout 5
SSLStaplingReturnResponderErrors off
SSLStaplingCache shmcb:/var/run/ocsp(128000)
```

### Run 

Provided you have opened 80 and 443 in your Security Group, you should now reach **https://yourfqdn** 

```
sudo a2ensite <yourfqdn>
sudo systemctl restart apache2
sudo systemctl reload apache2
sudo systemctl status apache2
```

Your status should look like this: 
```
● apache2.service - The Apache HTTP Server
     Loaded: loaded (/lib/systemd/system/apache2.service; enabled; vendor preset: enabled)
     Active: active (running) since Fri 2022-06-10 11:02:31 UTC; 1h 12min ago
       Docs: https://httpd.apache.org/docs/2.4/
    Process: 482 ExecStart=/usr/sbin/apachectl start (code=exited, status=0/SUCCESS)
    Process: 764 ExecReload=/usr/sbin/apachectl graceful (code=exited, status=0/SUCCESS)
   Main PID: 597 (apache2)
      Tasks: 55 (limit: 19185)
     Memory: 13.4M
        CPU: 382ms
     CGroup: /system.slice/apache2.service
             ├─597 /usr/sbin/apache2 -k start
             ├─773 /usr/sbin/apache2 -k start
             └─774 /usr/sbin/apache2 -k start
```

