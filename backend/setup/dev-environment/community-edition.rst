:title: OroCommerce, OroCRM, OroPlatform Environment Setup for Community Edition

.. _environment-setup-community:

Environment Setup for Community Edition
=======================================

This topic provides a detailed description of the environment setup process for Community Edition of Oro applications.

Before proceeding, please refer to the :ref:`System Requirements <system-requirements>` for the recommended environmental components and their supported versions. If you use the same environment and components described in the System Requirements, you can reuse the commands provided in this guide without modification. Otherwise, please adjust them to match the syntax supported by the tools of your choice.

Prepare a Server with OS
------------------------

Get a dedicated physical or virtual server with at least 4Gb RAM with the Oracle Linux v8 installed. Ensure that you can run processes as a *root* user or user with *sudo* permissions.

Environment Setup
-----------------

Enable Required Package Repositories
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Add the EPEL and remi repositories by running:

.. code-block:: bash

   dnf -y install dnf-plugin-config-manager https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm https://rpms.remirepo.net/enterprise/remi-release-8.rpm
   dnf -y module enable postgresql:13 nodejs:16 php:remi-8.2
   dnf -y upgrade

Add Oro public repository:

.. code-block:: bash

    cat >"/etc/yum.repos.d/oropublic.repo" <<__EOF__
    [oropublic]
    name=OroPublic
    baseurl=https://nexus.oro.cloud/repository/oropublic/8/x86_64/
    enabled=1
    gpgcheck=0
    module_hotfixes=1
    __EOF__

Install Nginx, NodeJS, PHP, PostgreSQL Server
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. code-block:: bash

   dnf -y --setopt=install_weak_deps=False --best install pngquant jpegoptim findutils rsync glibc-langpack-en psmisc wget bzip2 unzip p7zip p7zip-plugins parallel patch nodejs npm git-core jq bc postgresql postgresql-server postgresql-contrib php-common php-cli php-fpm php-opcache php-mbstring php-mysqlnd php-pgsql php-pdo php-json php-process php-ldap php-gd php-intl php-bcmath php-xml php-soap php-sodium php-tidy php-imap php-pecl-zip php-pecl-mongodb
   dnf -y --setopt=install_weak_deps=False --best --nogpgcheck install oro-nginx oro-nginx-mod-http-cache_purge oro-nginx-mod-http-cookie_flag oro-nginx-mod-http-geoip oro-nginx-mod-http-gridfs oro-nginx-mod-http-headers_more oro-nginx-mod-http-naxsi oro-nginx-mod-http-njs oro-nginx-mod-http-pagespeed oro-nginx-mod-http-sorted_querystring oro-nginx-mod-http-testcookie_access oro-nginx-mod-http-xslt-filter

Install Composer
^^^^^^^^^^^^^^^^

Run the commands below, or use another Composer installation process described in the |official documentation|.

.. code-block:: bash

   php -r "copy('https://getcomposer.org/installer', 'composer-setup.php');" && php composer-setup.php
   php -r "unlink('composer-setup.php');"
   mv composer.phar /usr/bin/composer

Perform Security Configuration
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Configure *SELinux*
~~~~~~~~~~~~~~~~~~~

For the production environment, it is strongly recommended to keep *SELinux* enabled in the *enforcing* mode.

.. warning:: The actual SELinux configuration depends on the real production server environment and should be configured by an experienced system administrator.

In this guide, we are loosening the SELinux mode to simplify installation in the local and development environment by setting the permissive option for the **setenforce** mode.

However, your environment configuration may differ. If that is the case, please adjust the commands that will follow in the next sections to match your configuration.

.. code-block:: bash

   sed -i 's/SELINUX=enforcing/SELINUX=permissive/g' /etc/selinux/config
   setenforce permissive

Configure Users Permissions
~~~~~~~~~~~~~~~~~~~~~~~~~~~

For security reasons, we recommend performing all Oro application-related processes on behalf of two different linux users:

* **Administrative user** (for example, oroadminuser) --- A user should be able to perform administration operations like application installation, update, etc.
* **Application user** (for example, nginx) ---  A user should be able to perform runtime operations that require no changes in the application source code files.

In this guide, to simplify installation in the local and development environment, we are loosening this requirement and using the superuser permissions to perform Oro application administrative tasks. However, for your staging or production environment, please adjust the commands that will follow in the next sections to run environment management commands as well as application install and update via a dedicated admin user.

Commands for running the web server, php-fpm process, cron commands, background processes, etc., are executed via the dedicated *application user* (*nginx*). Reuse them without modification if you keep the same username. Otherwise, adjust them accordingly.

Enable Installed Services
^^^^^^^^^^^^^^^^^^^^^^^^^

.. code-block:: bash

   systemctl start postgresql php-fpm nginx
   systemctl enable postgresql php-fpm nginx

Environment Configuration
-------------------------

.. include:: ./prepare-postgresql.rst

Configure Web Server
^^^^^^^^^^^^^^^^^^^^

For the production mode, it is strongly recommended to use the HTTPS protocol for the Oro application's public websites and reserve the HTTP mode for development and testing purposes only.

The samples of Nginx configuration for HTTPS and HTTP mode are provided below. Update the `/etc/nginx/conf.d/default.conf` file with the content that matches the type of your environment.

**Sample nginx Configuration for HTTP Websites (Use in Development and Staging Environment Only)**

.. code-block:: text

    server {
        server_name <your-domain-name> www.<your-domain-name>;
        root  <application-root-folder>/public;

        index index.php;

        gzip on;
        gzip_proxied any;
        gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;
        gzip_vary on;

        location / {
            # try to serve file directly, fallback to index.php
            try_files $uri /index.php$is_args$args;
        }

        location ~ ^/(index|index_dev|config|install)\.php(/|$) {
            fastcgi_pass php-fpm;
            # or
            # fastcgi_pass unix:/var/run/php/php7-fpm.sock;
            fastcgi_split_path_info ^(.+\.php)(/.*)$;
            include fastcgi_params;
            fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
            fastcgi_param HTTPS off;
            fastcgi_buffers 64 64k;
            fastcgi_buffer_size 128k;
        }

        location ~* ^[^(\.php)]+\.(jpg|jpeg|gif|png|ico|css|pdf|ppt|txt|bmp|rtf|js)$ {
            access_log off;
            expires 1h;
            add_header Cache-Control public;
        }

        error_log /var/log/nginx/<your-domain-name>_error.log;
        access_log /var/log/nginx/<your-domain-name>_access.log;
    }

**Sample nginx Configuration for HTTPS Websites (Safe for Production Environment)**

.. code-block:: text

    server {
        listen 80;
        server_name <your-domain-name> www.<your-domain-name>;
        return 301 https://$server_name$request_uri;
    }

    server {
        listen 443 ssl;
        server_name <your-domain-name> www.<your-domain-name>;

        ssl_certificate_key /etc/ssl/private/example.com.key;
        ssl_certificate /etc/ssl/private/example.com.crt.fullchain;
        ssl_protocols TLSv1.2;
        ssl_ciphers EECDH+AESGCM:EDH+AESGCM:AES2;

        root <application-root-folder>/public;

        index index.php;

        sendfile on;
        tcp_nopush on;
        tcp_nodelay on;

        # Increase this value in file uploads is allowed for larger files
        client_max_body_size 8m;

        gzip on;
        gzip_proxied any;
        gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;
        gzip_vary on;

        try_files $uri $uri/ @rewrite;

        location @rewrite {
            rewrite ^/(.*)$ /index.php/$1;
        }

        location ~ /\.ht {
            deny all;
        }

        location ~* ^[^(\.php)]+\.(jpg|jpeg|gif|png|ico|css|txt|bmp|js)$ {
            add_header Cache-Control public;
            expires 1h;
            access_log off;
        }

        location ~ [^/]\.php(/|$) {
            fastcgi_split_path_info ^(.+?\.php)(/.*)$;
            if (!-f $document_root$fastcgi_script_name) {
                return 404;
            }
            include                         fastcgi_params;
            fastcgi_pass                    php-fpm;
            fastcgi_index                   index.php;
            fastcgi_intercept_errors        on;
            fastcgi_connect_timeout         300;
            fastcgi_send_timeout            300;
            fastcgi_read_timeout            300;
            fastcgi_buffer_size             128k;
            fastcgi_buffers                 4   256k;
            fastcgi_busy_buffers_size       256k;
            fastcgi_temp_file_write_size    256k;
            fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;
            fastcgi_param  PATH_INFO        $fastcgi_path_info;
            fastcgi_param  HTTPS            on;
        }

        # Websockets connection path (configured in the .env-app.local file)
        location /ws {
            reset_timedout_connection on;

            # prevents 502 bad gateway error
            proxy_buffers 8 32k;
            proxy_buffer_size 64k;

            # redirect all HTTP traffic to localhost:8080;
            proxy_set_header Host $http_host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-NginX-Proxy true;
            proxy_set_header X-Forwarded-Proto $scheme;

            proxy_pass http://127.0.0.1:8080/;
            proxy_redirect off;
            proxy_read_timeout 86400;

            # enables WS support
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";

            error_log /var/log/nginx/<your-domain-name>_wss_error.log;
            access_log /var/log/nginx/<your-domain-name>_wss_access.log;
        }

        error_log /var/log/nginx/<your-domain-name>_https_error.log;
        access_log /var/log/nginx/<your-domain-name>_https_access.log;
    }

* Replace **<application-root-folder>** with the absolute path where you are going to install the Oro application.
* Replace **<your-domain-name>** with the configured domain name that would be used for the Oro application.
* Change *ssl_certificate_key* and *ssl_certificate* with the actual values of your active SSL certificate.

Optionally, you can enable and configure |Apache PageSpeed module| for Nginx to improve web page latency as described in the :ref:`Performance Optimization of the Oro Application Environment <installation--optimize-runtime-performance>` article.

.. note:: If you choose the Apache web server instead of the Nginx one, the example of the web server configuration you can find in the :ref:`Web Server Configuration <installation--web-server-configuration>` article.

For the changes to take effect, restart `nginx` by running:

.. code-block:: bash

   systemctl restart nginx

Configure Domain Name Resolution
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

If you are going to use the Oro application in the local environment only, modify the */etc/hosts* file on the server by adding the following line:

.. code-block:: none

   127.0.0.1 localhost <your-domain-name>

After this change, the <your-domain-name> URLs opened in the local environment are handled by the local webserver.

To make your Oro application accessible from remote locations, configure a DNS server to point your domain name to your server IP address.

Configure PHP
^^^^^^^^^^^^^

To configure PHP, perform the following changes in the configuration files:

* In the `www.conf` file (*/etc/php-fpm.d/www.conf*) --- Change the user and the group for PHP-FPM to *nginx* and set recommended values for other parameters.

  .. code-block:: none

     user = nginx
     group = nginx
     catch_workers_output = yes

* In the `php.ini` file (*/etc/php.ini*) --- Change the memory limit and cache configuration to the following:

  .. code-block:: none

     memory_limit = 1024M
     realpath_cache_size=4096K
     realpath_cache_ttl=600

* In the `opcache.ini` file (*/etc/php.d/10-opcache.ini*) --- Modify the OPcache parameter to match the following values:

  .. code-block:: none

     opcache.enable=1
     opcache.enable_cli=0
     opcache.memory_consumption=512
     opcache.interned_strings_buffer=32
     opcache.max_accelerated_files=32531
     opcache.save_comments=1

For the changes to take effect, restart PHP-FPM by running:

.. code-block:: bash

   systemctl restart php-fpm

Configure Storage For Import Files
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Product images and File entity field files can be imported by a path to an image or a file during the import.

This path can be either:

- a URL,
- an absolute path,
- a relative path. In this case, the files are searched in the Gaufrette filesystem configured to store files to import. By default, it is configured to use the ``{PROJECT}/var/data/import_files`` local directory as storage.

This path can be reconfigured with |Gaufrette| adapter configuration.

For example, to change the path location, add a new configuration of the **import_files** |Gaufrette| adapter in the `Resources/config/oro/app.yml` file of your bundle:

.. code-block:: yaml

    knp_gaufrette:
        adapters:
            import_files:
                local:
                    directory: '/new/path/to/import_files'

Use the Gaufrette filesystem abstraction layer as storage, this configuration can be changed to use any supported filesystem
adapter supported by |Gaufrette| library.

For example, the configuration to use :ref:`GridFS <bundle-docs-platform-gridfs-config-bundle>` storage can be the following:

.. code-block:: yaml

    knp_gaufrette:
        adapters:
            import_files:
                oro_gridfs:
                    mongodb_gridfs_dsn: 'mongodb://127.0.0.1:27017/import_data_files'




.. admonition:: Business Tip

   Looking to take advantage of the recent trend in digital commerce? Learn more about a |wholesale B2B marketplace| and how you can implement it for your business.


What's Next
-----------

* :ref:`Install the Oro Application via the Command-Line Interface <installation>`

.. include:: /include/include-links-dev.rst
   :start-after: begin

.. include:: /include/include-links-seo.rst
   :start-after: begin