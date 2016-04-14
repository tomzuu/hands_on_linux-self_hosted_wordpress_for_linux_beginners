# Set up a WordPress Site

Replace all instances of 'yourusername' with the system username that you'll use for this site, and all instances of 'yoursitename' with the same. It makes sense to use a truncated version of your domain for both of these, e.g. for 'tutorialinux.com' I would use 'tutorialinux'.


## Create a system user for this site

    adduser yourusername


## Create nginx vhost config file

Add the following content to /etc/nginx/conf.d/yoursitename.conf. Replace all occurrences of '{{ domain_name }}' with your actual domain name for this site:

    server {
        listen       80;
        server_name  www.{{ domain_name }};

        client_max_body_size 20m;

        index index.php index.html index.htm;
        root   /home/yourusername/public_html;

        location / {
            try_files $uri $uri/ /index.php?q=$uri&$args;
        }

        # pass the PHP scripts to FastCGI server
        location ~ \.php$ {
                # Basic
                try_files $uri =404;
                fastcgi_index index.php;

                # Create a no cache flag
                set $no_cache "";

                # Don't ever cache POSTs
                if ($request_method = POST) {
                  set $no_cache 1;
                }

                # Admin stuff should not be cached
                if ($request_uri ~* "/(wp-admin/|wp-login.php)") {
                  set $no_cache 1;
                }

                # If we are the admin, make sure nothing
                # gets cached, so no weird stuff will happen
                if ($http_cookie ~* "wordpress_logged_in_") {
                  set $no_cache 1;
                }

                # Cache and cache bypass handling
                fastcgi_no_cache $no_cache;
                fastcgi_cache_bypass $no_cache;
                fastcgi_cache microcache;
                fastcgi_cache_key $server_name|$request_uri|$args;
                fastcgi_cache_valid 200 60m;
                fastcgi_cache_valid 404 10m;
                fastcgi_cache_use_stale updating;


                # General FastCGI handling
                fastcgi_pass unix:/var/run/php-fpm/yoursitename.sock;
                fastcgi_pass_header Set-Cookie;
                fastcgi_pass_header Cookie;
                fastcgi_ignore_headers Cache-Control Expires Set-Cookie;
                fastcgi_split_path_info ^(.+\.php)(/.+)$;
                fastcgi_param SCRIPT_FILENAME $request_filename;
                fastcgi_intercept_errors on;
                include fastcgi_params;         
        }

        location ~* \.(js|css|png|jpg|jpeg|gif|ico|woff|ttf|svg|otf)$ {
                expires 30d;
                add_header Pragma public;
                add_header Cache-Control "public";
                access_log off;
        }

        # deny access to .htaccess files, if Apache's document root
        # concurs with nginx's one
        #
        #location ~ /\.ht {
        #    deny  all;
        #}
    }

    server {
        listen       80;
        server_name  {{ domain_name }};
        rewrite ^/(.*)$ http://www.{{ domain_name }}/$1 permanent;
    }





## Disable default nginx vhost

    rm /etc/nginx/sites-enabled/default 


## Create php-fpm vhost pool config file

Add the following content to a new php-fpm pool configuration file, at /etc/php5/fpm/pool.d/yoursitename.conf -- replace all occurrences of "yoursitename" with your truncated domain name, and all occurrences of "yourusername" with the name of the system user you've created for this website:

    [yoursitename]
    listen = /var/run/php-fpm/yoursitename.sock
    listen.owner = yourusername
    listen.group = www-data
    listen.mode = 0660
    user = yourusername
    group = www-data
    pm = dynamic
    pm.max_children = 75
    pm.start_servers = 8
    pm.min_spare_servers = 5
    pm.max_spare_servers = 20
    pm.max_requests = 500

    php_admin_value[upload_max_filesize] = 25M
    php_admin_value[error_log] = /home/yourusername/logs/phpfpm_error.log
    php_admin_value[open_basedir] = /home/yourusername:/tmp




## Create logfile

    touch /home/yourusername/logs/phpfpm.log




## Create database + DB user

Log into your mysql database with the root account, using the password you created earlier (during the mysql_secure_installation script run):

    mysql -u root -p

This will prompt you for the MySQL root user’s password, and then give you a database shell. This shell will let you enter the following commands to create the WordPress database and user, along with appropriate permissions. Swap out ‘yoursite’ for your truncated domain name. This name can't contain any punctuation or special characters. Replace 'chooseapassword' with a strong password:

    CREATE DATABASE yoursite;
    CREATE USER yoursite@localhost;
    SET PASSWORD FOR yoursite@localhost= PASSWORD("chooseapassword");
    GRANT ALL PRIVILEGES ON yoursite.* TO yoursite@localhost IDENTIFIED BY 'chooseapassword';
    FLUSH PRIVILEGES;


Great; you’re done! Hit ctrl-d to exit the MySQL shell.




## Install WordPress

Now it's time to actually download and install the WordPress application.


### Download WordPress

    su - yourusername
    cd
    wget https://wordpress.org/latest.tar.gz


### Extract Wordpress Archive (+ Clean Up)

    tar zxf latest.tar.gz
    rm latest.tar.gz


### Rename the extracted 'wordpress' directory

    rmdir public_html # remove in case it was created already
    mv wordpress public_html


### Create the site log directory

    mkdir /home/yourusername/logs
    chown yourusername:www-data /home/yourusername/public_html


### Set proper file permissions on your site files

    cd /home/yourusername/public_html
    chown -R yourusername:www-data .
    find . -type d -exec chmod 755 {} \;
    find . -type f -exec chmod 644 {} \;


### Secure the wp-config.php file so other users can’t read DB credentials

    chmod 640 wp-config.php




## Restart your services

    systemctl restart php5-fpm nginx
