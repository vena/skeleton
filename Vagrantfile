#############################################################
# Rawr! Vagrant Skeleton!
# 
# This Vagrantfile creates a test environment for
# LAMP/LEMP-based projects. 
# 
# This Vagrantfile supports and recommends the following
# Vagrant plugins:
#
# vagrant-auto_network plugin
# Automatically manages and assigns IP addresses to guests
# http://github.com/adrienthebo/vagrant-auto_network
#
# vagrant-hostsupdater
# Keeps track of host name => IP aliases in /etc/hosts
# https://github.com/cogitatio/vagrant-hostsupdater
# 
#############################################################

##
# Config
#
config = {

	# Set the host name for this box.
	# (REQUIRED)
	"hostname" => "skeleton",

	# Set a private IP address.
	# (OPTIONAL. Used if vagrant-auto_network is not installed)
	"ip" => "10.0.12.2",

	# Choose a web server to install.
	# (REQUIRED. Accepts "apache" or "nginx")
	"webserver" => "nginx",

	# Set the base directory for /vagrant, 
	# relative to the Vagrantfile.
	# (REQUIRED)
	"vagrant_basedir" => "./",

	# Set the web server's base directory, 
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


Vagrant.configure("2") do |vconfig|
	
	# Warn about missing recommended plugins on "up"
	if ARGV[0] == 'up'

		unless defined? AutoNetwork
			print "[INFO] vagrant-auto_network not detected!\n"
			print "       This plugin is recommended. If you'd like to install it now, ^C and run:\n"
			print "       $ vagrant plugin install vagrant-auto_network\n"
		end

		unless defined? VagrantPlugins::HostsUpdater
			print "[INFO] vagrant-hostsupdater not detected!\n"
			print "       This plugin is recommended. If you'd like to install it now, ^C and run:\n"
			print "       $ vagrant plugin install vagrant-hostsupdater\n"
		end

	end

	# Set display name
	vconfig.vm.define config["hostname"] do |t|
	end

	# Set VM name in VirtualBox UI
	vconfig.vm.provider "virtualbox" do |p|
		p.name = "vagrant::" + config["hostname"]
	end

	vconfig.vm.box = "ubuntu/trusty32"
	
	unless config.has_key?("ip")
		config["ip"] = "10.0.255.255"
	end
	vconfig.vm.network :private_network, ip: config["ip"], :auto_network => true
	
	vconfig.vm.hostname = config["hostname"]
	
	vconfig.vm.synced_folder config["vagrant_basedir"], "/vagrant", :create => true, :group => "www-data", :mount_options => ["dmode=775","fmode=664"]
	
	##
	# Base package install
	#
	vconfig.vm.provision :shell, :inline => "
	if [ ! -d /.provisioned ]; then
		mkdir -p /.provisioned
	fi
	if [ ! -f /.provisioned/system ]; then
		export DEBIAN_FRONTEND=noninteractive
		echo 'Setting timezone to UTC...'
		echo 'Etc/UTC' | tee /etc/timezone > /dev/null 2>&1 && dpkg-reconfigure --frontend noninteractive tzdata > /dev/null 2>&1

		echo 'Setting grub timeout...'
		echo 'GRUB_RECORDFAIL_TIMEOUT=3' >> /etc/default/grub && update-grub > /dev/null 2>&1
		
		echo 'Updating package cache...'
		apt-get update > /dev/null 2>&1

		echo 'Installing base packages...'
		apt-get install python-software-properties -y > /dev/null 2>&1
		apt-get install augeas-tools curl libnss-mdns ntp git-core puppet memcached -y > /dev/null 2>&1
		echo $(date) > /.provisioned/system
	fi
	"

	##
	# Install MariaDB
	#
	vconfig.vm.provision :shell, :inline => "
	if [ ! -f /.provisioned/mariadb ]; then
		echo '[MariaDB] Installing...'
		export DEBIAN_FRONTEND=noninteractive
		apt-key adv --recv-keys --keyserver hkp://keyserver.ubuntu.com:80 0xcbcb082a1bb943db > /dev/null 2>&1
		add-apt-repository 'deb http://mirrors.syringanetworks.net/mariadb/repo/5.5/ubuntu precise main' > /dev/null 2>&1
		apt-get update > /dev/null 2>&1
		apt-get install mariadb-server mariadb-client -y > /dev/null 2>&1
		
		echo '[MariaDB] Allowing all connections to root user.'
		/usr/bin/mysql -u root -e \"grant all privileges on *.* to 'root'@'%'; flush privileges;\"
		
		echo '[MariaDB] Binding MariaDB to external interfaces.'
		/usr/bin/perl -pi -e 's/^.*bind-address.*$/bind-address = 0.0.0.0/' '/etc/mysql/my.cnf'
		service mysql restart > /dev/null 2>&1

		echo '[MariaDB] Installation complete.'
		echo $(date) > /.provisioned/mariadb
	fi
	"

	##
	# Install PHP CLI, PHP-FPM and required modules
	#
	vconfig.vm.provision :shell, :inline => "
		if [ ! -f /.provisioned/php ]; then
			echo '[PHP] Installing PHP-CLI and PHP-FPM...'
			export DEBIAN_FRONTEND=noninteractive
			apt-get update > /dev/null 2>&1
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
	"

	##
	# Install Apache web server if configured
	#
	if config["webserver"] == "apache"
		vconfig.vm.provision :shell, :inline => "
		if [ ! -f /.provisioned/apache2 ]; then
			echo '[Apache] Installing...'
			export DEBIAN_FRONTEND=noninteractive
			apt-get update > /dev/null 2>&1
			apt-get install apache2-mpm-event -y > /dev/null 2>&1
			
			echo '[Apache] Enabling mod_rewrite.'
			a2enmod rewrite > /dev/null 2>&1
			
			echo '[Apache] Disabling default server config.'
			a2dissite default 000-default > /dev/null 2>&1
			
			echo '[Apache] Installing vagrant server config.'
			cat <<EOT >/etc/apache2/sites-available/vagrant.conf
EnableSendfile Off
ServerName %{hostname}
<VirtualHost *:80>
	ServerName %{hostname}
	DocumentRoot /vagrant/%{web_basedir}
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
EOT
			a2ensite vagrant > /dev/null 2>&1
			
			echo '[Apache/PHP-FPM] Installing mod_fastcgi...'
			cat <<EOT >> /etc/apt/sources.list
deb http://archive.ubuntu.com/ubuntu trusty multiverse
deb http://archive.ubuntu.com/ubuntu trusty-updates multiverse
deb http://security.ubuntu.com/ubuntu trusty-security multiverse
EOT
			apt-get update > /dev/null 2>&1
			apt-get install libapache2-mod-fastcgi > /dev/null 2>&1

			echo '[Apache/PHP-FPM] Enabling PHP-FPM...'
			a2enmod actions alias fastcgi > /dev/null 2>&1
			cat <<EOT >/etc/apache2/conf-available/php5-fpm.conf
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
EOT
			a2enconf php5-fpm > /dev/null 2>&1
			apache2ctl restart > /dev/null 2>&1

			echo '[Apache] Installation complete.'
			echo $(date) > /.provisioned/apache2
		fi
		" % {hostname: config["hostname"], web_basedir: config["web_basedir"]}
	end

	##
	# Install nginx web server if configured
	#
	if config["webserver"] == "nginx"
		vconfig.vm.provision :shell, :inline => "
		if [ ! -f /.provisioned/nginx ]; then
			echo '[nginx] Installing...'
			export DEBIAN_FRONTEND=noninteractive
			apt-get update > /dev/null 2>&1
			apt-get install nginx -y > /dev/null 2>&1
			
			echo '[nginx] Removing default web server config.'
			rm -rf /etc/nginx/sites-enabled/default
			
			echo '[nginx] Installing vagrant web server config.'
			cat <<EOT >/etc/nginx/sites-available/vagrant
server {
    listen 80 default_server;
    server_name _;
    root /vagrant/%{web_basedir};
    index index.php index.html index.htm;

    sendfile off;

    location / {
        try_files \\$uri /index.php?$query_string;
    }

    location ~ \.php($|/) {
        fastcgi_pass unix:/var/run/php5-fpm.sock;
        fastcgi_index index.php;
        include fastcgi_params;
    }

    location /nginx_status {
        stub_status on;
        access_log off;
    }

    include /etc/nginx/conf.vagrant/*.conf;

}
EOT
			ln -s /etc/nginx/sites-available/vagrant /etc/nginx/sites-enabled/vagrant
			mkdir /etc/nginx/conf.vagrant
			
			service nginx restart > /dev/null 2>&1

			echo '[nginx] Installation complete.'
			echo $(date) > /.provisioned/nginx
		fi
		" % {web_basedir: config["web_basedir"]}
	end

	##
	# Install Composer
	#
	vconfig.vm.provision :shell, :inline => "
	if [ ! -f /.provisioned/composer ]; then
		echo '[Composer] Installing...'
		augtool -s 'set /files/etc/php5/cli/php.ini/PHP/allow_url_fopen On'
		curl -s http://getcomposer.org/installer | php > /dev/null 2>&1
		
		echo '[Composer] Moving to /usr/local/bin/composer.'
		mkdir -p /usr/local/bin
		mv /home/vagrant/composer.phar /usr/local/bin/composer
		(crontab -l; echo '14 3 */2 * * /usr/local/bin/composer self-update') | crontab - > /dev/null 2>&1

		echo '[Composer] Installation complete.'
		echo $(date) > /.provisioned/composer
	fi
	"

	if config["upgrade_on_boot"] == "yes"
		vconfig.vm.provision :shell, :inline => "
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
		"
	end

end

# vim: syntax=ruby
