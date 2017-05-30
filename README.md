# Nextcloud on Arch Linux | LEMP | NGINX | MariaDB | PHP7




1. Setup SD Card with Arch Linux for Raspberry Pi 3
	https://archlinuxarm.org/platforms/armv8/broadcom/raspberry-pi-3

	Replace sdX in the following instructions with the device name for the SD card as it appears on your computer.

	Start fdisk to partition the SD card:

		fdisk /dev/sdX

	At the fdisk prompt, delete old partitions and create a new one:

		Type o. This will clear out any partitions on the drive.
		Type p to list partitions. There should be no partitions left.
		Type n, then p for primary, 1 for the first partition on the drive, press ENTER to accept the default first sector, then type +100M for the last sector.
		Type t, then c to set the first partition to type W95 FAT32 (LBA).
		Type n, then p for primary, 2 for the second partition on the drive, and then press ENTER twice to accept the default first and last sector.
		Write the partition table and exit by typing w.
	Create and mount the FAT filesystem:

		mkfs.vfat /dev/sdX1
		mkdir boot
		mount /dev/sdX1 boot

	Create and mount the ext4 filesystem:

		mkfs.ext4 /dev/sdX2
		mkdir root
		mount /dev/sdX2 root

	Download and extract the root filesystem (as root, not via sudo):
	
		wget http://os.archlinuxarm.org/os/ArchLinuxARM-rpi-3-latest.tar.gz
		bsdtar -xpf ArchLinuxARM-rpi-3-latest.tar.gz -C root
		sync
	Move boot files to the first partition:
	
		mv root/boot/* boot
		Unmount the two partitions:
		umount boot root
	Insert the SD card into the Raspberry Pi, connect ethernet, and apply 5V power.
	
	Connect with SSH to your PI with given host or IP	
	Login as the default user alarm with the password alarm.
	The default root password is root.

2. Initial system setup

	Set locales
	Unkcomment in /etc/locale.gen

		en_US.UTF-8 UTF-8
	generate locales: 

		locale-gen
	set timezone	

		timedatectl set-timezone Europe/Berlin

	Upgrade packages

		pacman -Syu
	Install required packages (expand this list to fit your own preferences)

		pacman -S vim wget unzip

	2.1 1TB USB drive
	
	Format and Mount 1TB USB drive (assume it is partitioned with 1 partition). Replace sdX in the following instructions with the device name for the drive.
	
	Create mount point

		mkdir /mnt/wddrive
	Format to ext4

		mkfs.ext4 /dev/sdX1
	Show drive UUID and copy it

		blkid
	Make entry in /etc/fstab

		UUID=<the copied uuid> /mnt/wddrive ext4 defaults,noatime 0  0
	Reboot and proof that the drive is mounted 

		reboot
		lsblk

3. Install Nginx, MariaDB, PHP7 (LEMP) on Arch Linux 

	https://www.linuxbabe.com/linux-server/install-lemp-nginx-mariadb-php7-arch-linux-server
	
	3.1 NGINX
	
	Install

		pacman -S nginx-mainline
	Start and enable service

		systemctl start nginx
		systemctl enable nginx	
		systemctl status nginx
	Check if nginx is running, browse to http://<server ip>/

	3.2 MariaDB
	
	Install
	
		pacman -S mariadb
	Initialize the MariaDB data directory prior to starting the service.
	
		mysql_install_db --user=mysql --basedir=/usr --datadir=/var/lib/mysql
	Start and enable service
	
		systemctl start mysqld
		systemctl enable mysqld
		systemctl status mysqld

	Run the post-installation security script.
	
		mysql_secure_installation

	3.3 PHP7
	
	Install
	
		pacman -S php-fpm
	After it’s installed, we need to tell Nginx to run PHP using php-fpm.
	
		nano /etc/nginx/nginx.conf
	Find the location ~ \.php$ section and modify it to the following:
	
		location ~ \.php$ {
		    root           /usr/share/nginx/html;
		    fastcgi_pass   unix:/run/php-fpm/php-fpm.sock;
		    fastcgi_index  index.php;
		    fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;
		    include        fastcgi_params;
		}

	Then start and enable php-fpm 
	
		systemctl start php-fpm
		systemctl enable php-fpm
		systemctl status php-fpm

	Test PHP processing
	
		echo "<?php phpinfo(); ?>" >> /usr/share/nginx/html/test.php
		systemctl reload nginx
	
	Browse to http://<server ip>/test.php
	
	Enable extensions
	
		vim /etc/php/php.ini
	Uncomment the following 2 lines
	
		;extension=mysqli.so
		;extension=pdo_mysql.so
	Reload php-fpm service
	
		systemctl reload php-fpm

4. Install and Setup Nextcloud Server on Arch Linux

	https://www.linuxbabe.com/cloud-storage/nextcloud-server-arch-linux-nginx-mariadb-php7

	4.1 Install Nextcloud server	
	
	Download
	
		wget https://download.nextcloud.com/server/releases/latest.zip

	Extract
	
		unzip latest.zip -d /usr/share/nginx/

	Give the NGINX user http write permissions
	
		chown http:http /usr/share/nginx/nextcloud/ -R
		chown http:http /mnt/wddrive -R

	4.2 NC MariaDB setup	

	Log into MariaDB database server
	
		mysql -u root -p
	Then create a database for Nextcloud.
	
		create database nextcloud;

	Create the database user. Replace <user> and <password> with your preferred values.
	
		create user <user>@localhost identified by '<password>';
	Grant this user all privileges on the nextcloud database (replace <user> and <password>)
	
		grant all privileges on nextcloud.* to <user>@localhost identified by '<password>';
	Flush the privileges table and exit.
	
		flush privileges;
		exit;
	Enable Binary Logging in MariaDB
	In /etc/mysql/my.cnf in the [mysql] section, check if the following 2 lines are there
	
		log-bin        = mysql-bin
		binlog_format  = mixed
	Restart service
	
		systemctl restart mysqld

	4.3 NC NGINX setup

	Create a conf.d directory for individual Nginx config files.

		mkdir /etc/nginx/conf.d

	Create a config file for Nextcloud.

		vim /etc/nginx/conf.d/nextcloud.conf

	Put the following text into the file:

	    upstream php-handler {
		server unix:/run/php-fpm/php-fpm.sock;
	    }

	    server {
		listen 80;
		server_name nextcloud.your-domain.com;

		# Add headers to serve security related headers
		add_header X-Content-Type-Options nosniff;
		add_header X-Frame-Options "SAMEORIGIN";
		add_header X-XSS-Protection "1; mode=block";
		add_header X-Robots-Tag none;
		add_header X-Download-Options noopen;
		add_header X-Permitted-Cross-Domain-Policies none;

		# Path to the root of your installation
		root /usr/share/nginx/nextcloud/;

		location = /robots.txt {
	      allow all;
	      log_not_found off;
	      access_log off;
		}

		# The following 2 rules are only needed for the user_webfinger app.
		# Uncomment it if you're planning to use this app.
		#rewrite ^/.well-known/host-meta /public.php?service=host-meta last;
		#rewrite ^/.well-known/host-meta.json /public.php?service=host-meta-json
		# last;

		location = /.well-known/carddav {
	      return 301 $scheme://$host/remote.php/dav;
		}
		location = /.well-known/caldav {
		   return 301 $scheme://$host/remote.php/dav;
		}

		location ~ /.well-known/acme-challenge {
		  allow all;
		}

		# set max upload size
		client_max_body_size 512M;
		fastcgi_buffers 64 4K;

		# Disable gzip to avoid the removal of the ETag header
		gzip off;

		# Uncomment if your server is build with the ngx_pagespeed module
		# This module is currently not supported.
		#pagespeed off;

		error_page 403 /core/templates/403.php;
		error_page 404 /core/templates/404.php;

		location / {
		   rewrite ^ /index.php$uri;
		}

		location ~ ^/(?:build|tests|config|lib|3rdparty|templates|data)/ {
		   deny all;
		}
		location ~ ^/(?:\.|autotest|occ|issue|indie|db_|console) {
		   deny all;
		 }

		location ~ ^/(?:index|remote|public|cron|core/ajax/update|status|ocs/v[12]|updater/.+|ocs-provider/.+|core/templates/40[34])\.php(?:$|/) {
		   include fastcgi_params;
		   fastcgi_split_path_info ^(.+\.php)(/.*)$;
		   fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
		   fastcgi_param PATH_INFO $fastcgi_path_info;
		   #Avoid sending the security headers twice
		   fastcgi_param modHeadersAvailable true;
		   fastcgi_param front_controller_active true;
		   fastcgi_pass php-handler;
		   fastcgi_intercept_errors on;
		   fastcgi_request_buffering off;
		}

		location ~ ^/(?:updater|ocs-provider)(?:$|/) {
		   try_files $uri/ =404;
		   index index.php;
		}

		# Adding the cache control header for js and css files
		# Make sure it is BELOW the PHP block
		location ~* \.(?:css|js)$ {
	      try_files $uri /index.php$uri$is_args$args;
	      add_header Cache-Control "public, max-age=7200";
	      # Add headers to serve security related headers (It is intended to
	      # have those duplicated to the ones above)        
	      add_header X-Content-Type-Options nosniff;
	      add_header X-Frame-Options "SAMEORIGIN";
	      add_header X-XSS-Protection "1; mode=block";
	      add_header X-Robots-Tag none;
	      add_header X-Download-Options noopen;
	      add_header X-Permitted-Cross-Domain-Policies none;
	      # Optional: Don't log access to assets
	      access_log off;
	       }

	       location ~* \.(?:svg|gif|png|html|ttf|woff|ico|jpg|jpeg)$ {
	      try_files $uri /index.php$uri$is_args$args;
	      # Optional: Don't log access to other assets
	      access_log off;
	       }
	    }
	Replace the correct value for the server_name attribute.

	Next, edit /etc/nginx/nginx.conf file. 
	Add the following line in the http section so that individual Nginx config files will be loaded.

		include /etc/nginx/conf.d/*.conf;

	Reload service

		systemctl reload nginx
5. NC install PHP modules
	
	Install
	
		pacman -S php-gd

	Uncomment the following line in /etc/php/php.ini to enable the module
	
		;extension=gd.so  

	Reload service
	
		systemctl reload php-fpm
6. Nextcloud post installation setup
	
	6.1 Set PHP environment variables properly

	Uncomment in /etc/php/php-fpm.d/www.conf the following lines
	
		;env[HOSTNAME] = $HOSTNAME
		;env[PATH] = /usr/local/bin:/usr/bin:/bin
		;env[TMP] = /tmp
		;env[TMPDIR] = /tmp
		;env[TEMP] = /tmp

	Reload php-fpm service
	
		systemctl reload php-fpm

	6.2 HTTP header X-Frame-Options "SAMEORIGIN". (Double set header fields issue).

	In file /usr/share/nginx/nextcloud/.htaccess

	    Header always set X-Content-Type-Options "nosniff"
	    Header always set X-Frame-Options "SAMEORIGIN"

	In file /etc/nginx/conf.d/nextcloud.conf

	    # add_header X-Content-Type-Options nosniff;
	    # add_header X-Frame-Options "SAMEORIGIN";

	6.3 PHP Caching

	https://wiki.archlinux.org/index.php/PHP#Caching
	
	Install the php-apcu package.
	
		pacman -S php-apcu

	Uncomment in /etc/php/php.ini
	
		zend_extension=opcache.so

	Add in /etc/php/php.ini
	
		extension=apcu.so
		apc.enabled=1 
		apc.shm_size=32M
		apc.ttl=7200
		apc.enable_cli=1

	Add in /usr/share/nginx/nextcloud/config/config.php
	
		'memcache.local' => '\OC\Memcache\APCu',

	Restart services
	
		systemctl restart php-fpm
		systemctl restart nginx
	6.4 Use CRON
	
	Install
	
		pacman -S cronie 
	Add crontab entry
	
		crontab -u http -e
		*/15  *  *  *  * php -f /usr/share/nginx/nextcloud/cron.php
	Start and enable service
	
		systemctl start cronie.service
		systemctl enable cronie.service
	Set CRON radio button at Nextcloud Admin page

	6.5 Uploading files up to 16GB 
	
	In /usr/share/nginx/nextcloud/.user.ini
	
		upload_max_filesize = 16G
		post_max_size = 16G
		memory_limit=512M
		output_buffering=0

	In /etc/php/php.ini
	
		post_max_size = 16G
		upload_max_filesize = 16G
		max_input_time = 3600
		max_execution_time = 3600
		output_buffering = Off
		upload_tmp_dir = /mnt/wddrive/upload_tmp_dir

	In /etc/nginx/conf.d/nextcloud.conf
	
		client_max_body_size 16G;

	In /etc/nginx/nginx.conf
	
		client_body_temp_path /mnt/wddrive/upload_tmp_dir
	In /etc/nginx/nginx.conf add to PHP location block
	
		fastcgi_read_timeout 600;
	Restart services
	
		systemctl restart nginx
		systemctl restart php-fpm
	Create upload_tmp_dir on a place with enough free space and set write permission
	
		mkdir /mnt/wddrive/upload_tmp_dir
		chown http:http /mnt/wddrive/upload_tmp_dir/ -R
