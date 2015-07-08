# -*- mode: ruby -*-
# vi: set ft=ruby :

#############################################################
# Rawr! Vagrant Skeleton!
# 
# This Vagrantfile creates my dev environment for
# LAMP/LEMP-based projects. 
# 
# This Vagrantfile supports and recommends the landrush
# vagrant plugin for managing guest DNS resolution.
#
# https://github.com/phinze/landrush
# 
#############################################################

##
# Config
#
vm_config = {

	# Set the host name for this box.
	# (REQUIRED)
	"hostname" => "skeleton",

	# Set the TLD for landrush
	# (OPTIONAL. Used if landrush is installed, appended to hostname)
	"tld" => "vm",

	# Set a private IP address.
	# (OPTIONAL. Used if landrush is not installed)
	"ip" => "10.0.12.2",

	# Choose a web server to install.
	# (REQUIRED. Accepts "apache" or "nginx")
	"webserver" => "nginx",

	# Set the base directory for /vagrant on the guest filesystem, 
	# relative to the Vagrantfile.
	# (REQUIRED)
	"vagrant_basedir" => "./",

	# Set the guest web server's base directory, 
	# relative to the above vagrant_basedir.
	# (OPTIONAL)
	"web_basedir" => "public/",

	# Optionally dist-upgrade on boot.
	# (OPTIONAL)
	"upgrade_on_boot" => "yes"
}
#
# End Config
##

Vagrant.configure("2") do |config|

	config.vm.box = "ubuntu/vivid32"

	# Generate hostname if one is not provided
	unless vm_config.has_key?("hostname")
		vm_config["hostname"] = "vagrant-" + rand(36*8).to_s(36)
	end

	# Hacky way to set vagrant display name
	config.vm.define vm_config["hostname"] do |t|
	end

	#config.vm.hostname = vm_config["hostname"]
	config.vm.provision :shell, inline: "hostnamectl set-hostname " + vm_config["hostname"]

	config.vm.provider "virtualbox" do |p|
		p.name = "vagrant::" + vm_config["hostname"]
	end

	unless Vagrant.has_plugin?("landrush")
		if ARGV[0] == "up"
			print "\e[33m[INFO] landrush plugin not detected!\e[0m\n"
			print "       This plugin is recommended. If you'd like to install it now, ^C and run:\n"
			print "       $ vagrant plugin install landrush\n"
		end

		# Set an IP if none is set in vm_config
		unless vm_config.has_key?("ip")
			vm_config["ip"] = "10.0.255.255"
		end

		config.vm.network :private_network, ip: vm_config["ip"]
	else
		config.landrush.enabled = true
		if vm_config.has_key?("tld") && !vm_config["tld"].empty?
			config.landrush.tld = vm_config["tld"]
		end
		case ARGV[0]
			when 'up'
				print "\e[32m[INFO] landrush plugin detected.\e[0m\n"
				print "\e[32m       Guest will be accessible at http://%s.%s\e[0m\n" % [vm_config["hostname"], config.landrush.tld.is_a?(String) ? config.landrush.tld : Landrush::Config::DEFAULTS[:tld]]
			when 'destroy', 'down'
				print "\e[32m[INFO] landrush plugin detected.\e[0m\n"
				print "\e[32m       DNS entries for guest will be removed.\e[0m\n"
		end
	end

	config.vm.synced_folder vm_config["vagrant_basedir"], "/vagrant", :create => true, :group => "www-data", :mount_options => ["dmode=775", "fmode=664"]
	config.ssh.shell = "bash -c 'BASH_ENV=/etc/profile exec bash'"

	# Base package installation
	if (ARGV[0] == 'up' || ARGV[0] == 'resume')
		print "\e[35m[PROVISIONING] Initializing guest...\e[0m\n"
		config.vm.provision "shell", inline: <<-SHELL
			if [ ! -d /.provisioned ]; then
				mkdir -p /.provisioned
			fi
			if [ ! -f /.provisioned/system ]; then
				export DEBIAN_FRONTEND=noninteractive
				sed -i 's/^mesg n$/tty -s \&\& mesg n/g' /root/.profile

				echo '[PROVISIONING] Setting timezone to UTC'
				echo 'Etc/UTC' | tee /etc/timezone > /dev/null 2>&1 && dpkg-reconfigure --frontend noninteractive tzdata > /dev/null 2>&1

				echo '[PROVISIONING] Setting grub timeout'
				echo 'GRUB_RECORDFAIL_TIMEOUT=3' >> /etc/default/grub && update-grub > /dev/null 2>&1

				echo '[PROVISIONING] Updating package cache'
				apt-get update > /dev/null 2>&1

				echo '[PROVISIONING] Installing base system packages'
				apt-get install python-software-properties -y > /dev/null 2>&1
				apt-get install augeas-tools curl libnss-mdns ntp git-core puppet memcached -y > /dev/null 2>&1
				echo $(date) > /.provisioned/system
			fi
		SHELL

		# Install MariaDB
		config.vm.provision "shell", inline: <<-SHELL
			if [ ! -d /.provisioned/mariadb ]; then
				export DEBIAN_FRONTEND=noninteractive

				echo -e "\033[33;35m[MariaDB] Installing...\033[0m\n"

				apt-key adv --recv-keys --keyserver hkp://keyserver.ubuntu.com:80 0xcbcb082a1bb943db > /dev/null 2>&1
				add-apt-repository 'deb http://mirrors.syringanetworks.net/mariadb/repo/5.5/ubuntu precise main' > /dev/null 2>&1
				apt-get update > /dev/null 2>&1
				apt-get install mariadb-server mariadb-client -y > /dev/null 2>&1
				
				echo '[MariaDB] Allowing all local subnet connections to root user.'
				/usr/bin/mysql -u root -e \"grant all privileges on *.* to 'root'@'192.168.%'; grant all privileges on *.* to 'root'@'10.%'; grant all privileges on *.* to 'rooot'@'172.16.%'; flush privileges;\"
				
				echo '[MariaDB] Binding MariaDB to external interfaces.'
				/usr/bin/perl -pi -e 's/^.*bind-address.*$/bind-address = 0.0.0.0/' '/etc/mysql/my.cnf'
				service mysql restart > /dev/null 2>&1

				echo '[MariaDB] Installation complete.'
				echo $(date) > /.provisioned/mariadb
			fi
		SHELL

		# PHP CLI + PHP-FPM + modules
		config.vm.provision "shell", inline: <<-SHELL
			apt-get update > /dev/null 2>&1
			if [ ! -f /.provisioned/php ]; then
				export DEBIAN_FRONTEND=noninteractive

				echo -e "\033[33;35m[PHP] Installing PHP-CLI and PHP-FPM...\033[0m\n"

				apt-get install php5-cli php5-fpm php5 -y > /dev/null 2>&1
				
				echo '[PHP] Installing modules...'
				apt-get update > /dev/null 2>&1
				apt-get install php5-mysql php5-gd php5-mcrypt php5-tidy php5-curl php5-xsl php5-sqlite php5-memcached -y > /dev/null 2>&1
				php5enmod mcrypt mysql gd tidy curl xsl sqlite3 memcached

				echo '[PHP] Restarting PHP-FPM...'
				service php5-fpm restart > /dev/null 2>&1

				echo '[PHP] Installation complete.'
				echo $(date) > /.provisioned/php
			fi
		SHELL

		# Install apache if configured
		if vm_config["webserver"] == "apache"
			config.vm.provision "shell", inline: <<-SHELL
				if [ ! -f /.provisioned/apache2 ]; then
					export DEBIAN_FRONTEND=noninteractive

					if [ -f /.provisioned/nginx ]; then
						echo -e "\033[33;33m[Apache] Uninstalling nginx...\033[0m\n"
						apt-get remove nginx -y > /dev/null 2>&1
						rm /.provisioned/nginx
					fi

					echo -e "\033[33;35m[Apache] Installing...\033[0m\n"
					apt-get install apache2-mpm-event -y > /dev/null 2>&1
					
					echo '[Apache] Enabling mod_rewrite.'
					a2enmod rewrite > /dev/null 2>&1
					
					echo '[Apache] Disabling default server config.'
					a2dissite default 000-default > /dev/null 2>&1
					
					echo '[Apache] Installing vagrant server config.'
					cat <<EOF > /etc/apache2/sites-available/vagrant.conf
EnableSendfile Off
ServerName #{vm_config["hostname"]}
<VirtualHost *:80>
	ServerName #{vm_config["hostname"]}
	DocumentRoot /vagrant/#{vm_config["web_basedir"]}
	<Directory />
		AllowOverride all
		Options FollowSymlinks
		Order allow,deny
		Allow from all
		Require all granted
	</Directory>
	ErrorLog /var/log/apache2/vagrant-error.log
	CustomLog /var/log/apache2/vagrant-access.log combined
</VirtualHost>
EOF
					
					a2ensite vagrant > /dev/null 2>&1

					echo '[Apache/PHP-FPM] Installing mod_fastcgi...'
					cat <<EOF >> /etc/apt/sources.list
deb http://archive.ubuntu.com/ubuntu $(lsb_release -cs) multiverse
deb http://archive.ubuntu.com/ubuntu $(lsb_release -cs)-updates multiverse
deb http://security.ubuntu.com/ubuntu $(lsb_release -cs)-security multiverse
EOF
					apt-get update > /dev/null 2>&1
					apt-get install libapache2-mod-fastcgi > /dev/null 2>&1

					echo '[Apache/PHP-FPM] Enabling PHP-FPM...'
					a2enmod actions alias fastcgi > /dev/null 2>&1
					cat <<EOF >/etc/apache2/conf-available/php5-fpm.conf
<IfModule mod_fastcgi.c>
	AddHandler php5-fcgi .php
	Action php5-fcgi /php5-fcgi
	Alias /php5-fcgi /usr/lib/cgi-bin/php5-fcgi
	FastCgiExternalServer /usr/lib/cgi-bin/php5-fcgi -socket /var/run/php5-fpm.sock -pass-header Authorization
	<Directory /usr/lib/cgi-bin>
		Options ExecCGI FollowSymLinks
		SetHandler fastcgi-script
		Require all granted
	</Directory>
</IfModule>
EOF
					a2enconf php5-fpm > /dev/null 2>&1

					apache2ctl restart > /dev/null 2>&1

					echo '[Apache] Installation complete.'
					echo $(date) > /.provisioned/apache2
				fi
			SHELL
		end

		# Install nginx if configured
		if vm_config["webserver"] == "nginx"
			config.vm.provision "shell", inline: <<-SHELL
				if [ ! -f /.provisioned/nginx ]; then
					export DEBIAN_FRONTEND=noninteractive

					if [ -f /.provisioned/apache ]; then
						echo -e "\033[33;33m[nginx] Uninstalling Apache...\033[0m\n"
						apt-get remove apache2-mpm-event libapache2-mod-fastcgi -y > /dev/null 2>&1
						rm /.provisioned/apache
					fi

					echo -e "\033[33;35m[nginx] Installing...\033[0m\n"
					apt-get install nginx -y > /dev/null 2>&1

					echo '[nginx] Installing vagrant web server config.'
					rm -rf /etc/nginx/sites-enabled/default
					cat <<EOF > /etc/nginx/sites-available/vagrant
server {
	listen 80 default_server;
	listen [::]:80 default_server ipv6only=on;

	root /vagrant/#{vm_config["web_basedir"]};
	
	index index.html index.htm index.php;

	server_name #{vm_config["hostname"]};

	charset utf-8;

	sendfile off;

	client_max_body_size 100m;

	location / {
		try_files \\$uri \\$uri/ /index.php?\\$query_string;
	}

	location = /favicon.ico { access_log off; log_not_found off; }
	location = /robots.txt { access_log off; log_not_found off; }

	location ~ [^/]\\.php(/|$) {
		fastcgi_split_path_info ^(.+?\\.php)(/.*)$;
		if (!-f \\$document_root\\$fastcgi_script_name) {
			return 404;
		}
		include fastcgi.conf;
		fastcgi_pass unix:/var/run/php5-fpm.sock;
		include fastcgi_params;
		fastcgi_intercept_errors off;
		fastcgi_buffer_size 16k;
		fastcgi_buffers 4 16k;
	}

	location /nginx_status {
		stub_status on;
		access_log off;
	}

	include /etc/nginx/conf.vagrant/*.conf;
}
EOF
					ln -s /etc/nginx/sites-available/vagrant /etc/nginx/sites-enabled/vagrant
					mkdir /etc/nginx/conf.vagrant

					service nginx restart > /dev/null 2>&1

					echo '[nginx] Installation complete.'
					echo $(date) > /.provisioned/nginx
				fi
			SHELL
		end

		# Install Composer
		config.vm.provision "shell", inline: <<-SHELL
			if [ ! -f /.provisioned/composer ]; then
				echo -e "\033[33;35m[Composer] Installing...\033[0m\n"
				augtool -s 'set /files/etc/php5/cli/php.ini/PHP/allow_url_fopen On'
				mkdir -p /usr/local/bin
				curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer > /dev/null 2>&1

				echo "[Composer] Setting cron to auto-update."
				(crontab -l > /dev/null 2>&1; echo '14 3 */2 * * /usr/local/bin/composer self-update') | crontab - > /dev/null 2>&1

				echo "[Composer] Installation complete."
				echo $(date) > /.provisioned/composer
			fi
		SHELL

		# Update on boot if configured
		if vm_config["upgrade_on_boot"] == "yes"
			config.vm.provision "shell", inline: <<-SHELL
				export DEBIAN_FRONTEND=noninteractive
				echo ''
				echo 'Upgrading installed packages in the background.'
				echo 'This will take a while, show no progress information,'
				echo 'and you will need to reload the VM manually when it'
				echo 'is complete using:'
				echo ''
				echo 'vagrant reload --no-provision'
				echo ''

				apt-get update > /dev/null 2>&1
				sudo nohup sh -c 'sudo apt-get update && sudo DEBIAN_FRONTEND=noninteractive apt-get -y -o Dpkg::Options::=\"--force-confdef\" -o Dpkg::Options::=\"--force-confold\" dist-upgrade' > /dev/null 2>&1 &
			SHELL
		end

	end
end
