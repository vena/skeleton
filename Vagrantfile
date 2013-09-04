#############################################################
# Rawr! Vagrant Skeleton!
# 
# This Vagrantfile creates a test environment for
# LAMP/LEMP-based projects. 
# 
# This Vagrantfile supports the vagrant-auto_network plugin
# 
# http://github.com/adrienthebo/vagrant-auto_network
# 
# If you're not using that, be sure to set the "ip" config
# option to something local and unused.
#############################################################

##
# Config
#
config = {

	# Set the host name for this box.
	# (REQUIRED)
	"hostname" => "skeleton",

	# Set a private IP address.
	# (REQUIRED. Used if vagrant-auto_network is not installed)
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
		p.name = config["hostname"] + "_vagrant"
	end

	vconfig.vm.box = "precise32"
	vconfig.vm.box_url = "http://files.vagrantup.com/precise32.box"
	
	vconfig.vm.network :private_network, ip: config["ip"], :auto_network => true
	
	vconfig.vm.hostname = config["hostname"]
	
	vconfig.vm.synced_folder config["vagrant_basedir"], "/vagrant/base", :group => "www-data", :extra => "dmode=775,fmode=664"
	
	##
	# Base package install
	#
	vconfig.vm.provision :shell, :inline => "
	if [ ! -d /.provisioned ]; then
		mkdir -p /.provisioned
	fi
	if [ ! -f /.provisioned/system ]; then
		export DEBIAN_FRONTEND=noninteractive
		echo 'Setting timezone...'
		echo 'Etc/UTC' | tee /etc/timezone > /dev/null 2>&1 && dpkg-reconfigure --frontend noninteractive tzdata

		echo 'Setting grub timeout...'
		echo 'GRUB_RECORDFAIL_TIMEOUT=3' >> /etc/default/grub && update-grub > /dev/null 2>&1
		
		echo 'Updating package cache...'
		apt-get update > /dev/null 2>&1

		echo 'Installing base packages...'
		apt-get install python-software-properties -y > /dev/null 2>&1
		add-apt-repository ppa:apt-fast/stable -y > /dev/null 2>&1
		apt-get update > /dev/null 2>&1
		apt-get install apt-fast -y > /dev/null 2>&1
		apt-fast install augeas-tools curl libnss-mdns ntp git-core puppet -y > /dev/null 2>&1
		echo $(date) > /.provisioned/system
	fi
	"

	##
	# Install MYSQL
	#
	vconfig.vm.provision :shell, :inline => "
	if [ ! -f /.provisioned/mysql ]; then
		echo '[MySQL] Installing...'
		export DEBIAN_FRONTEND=noninteractive
		apt-fast update > /dev/null 2>&1
		apt-fast install mysql-server mysql-client -y > /dev/null 2>&1
		
		echo '[MySQL] Allowing all connections to root user.'
		/usr/bin/mysql -uroot -e \"grant all privileges on *.* to 'root'@'%'; flush privileges;\"
		
		echo '[MySQL] Binding MySQL to all interfaces.'
		/usr/bin/perl -pi -e 's/^.*bind-address.*$/bind-address = 0.0.0.0/' '/etc/mysql/my.cnf'
		service mysql restart > /dev/null 2>&1
		echo $(date) > /.provisioned/mysql
	fi
	"

	##
	# Install PHP command line tool and modules
	#
	vconfig.vm.provision :shell, :inline => "
		if [ ! -f /.provisioned/php ]; then
			echo '[PHP] Installing command line tool...'
			export DEBIAN_FRONTEND=noninteractive
			apt-fast update > /dev/null 2>&1
			apt-fast install php5-cli -y > /dev/null 2>&1
			
			echo '[PHP] Installing modules...'
			apt-fast update > /dev/null 2>&1
			apt-fast install php5-mysql php5-gd php5-mcrypt php5-suhosin php5-tidy php5-curl php5-xsl php5-sqlite php-apc -y > /dev/null 2>&1	
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
			apt-fast update > /dev/null 2>&1
			apt-fast install apache2 -y > /dev/null 2>&1
			
			echo '[Apache] Enabling mod_rewrite.'
			a2enmod rewrite > /dev/null 2>&1
			
			echo '[Apache] Disabling default server config.'
			a2dissite default > /dev/null 2>&1
			
			echo '[Apache] Installing vagrant server config.'
			cat <<EOT >/etc/apache2/sites-available/vagrant
EnableSendfile Off

<VirtualHost *:80>
	ServerName %{hostname}
	DocumentRoot /vagrant/base/%{web_basedir}
	<Directory /www>
		AllowOverride all
		Options FollowSymlinks
		Order allow,deny
		Allow from all
	</Directory>
</VirtualHost>
EOT
			a2ensite vagrant > /dev/null 2>&1
			
			echo '[PHP] Installing mod_php...'
			apt-fast update > /dev/null 2>&1
			apt-fast install libapache2-mod-php -y > /dev/null 2>&1
			apache2ctl restart
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
			apt-fast update > /dev/null 2>&1
			apt-fast install nginx -y > /dev/null 2>&1
			
			echo '[nginx] Removing default web server config.'
			rm -rf /etc/nginx/sites-enabled/default
			
			echo '[nginx] Installing vagrant web server config.'
			cat <<EOT >/etc/nginx/sites-available/vagrant
server {
    listen 80 default_server;
    server_name _;
    root /vagrant/base/%{web_basedir};
    index index.php index.html index.htm;

    sendfile off;

    location / {
        try_files \\$uri /index.php;
    }

    location ~ \.php($|/) {
        fastcgi_pass 127.0.0.1:9000;
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
			
			echo '[PHP] Installing php-fpm...'
			apt-fast update > /dev/null 2>&1
			apt-fast install php5-fpm -y > /dev/null 2>&1
			service nginx restart
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
		/usr/bin/perl -pi -e 's/^.*suhosin.executor.include.whitelist.*$/suhosin.executor.include.whitelist = phar/' '/etc/php5/conf.d/suhosin.ini'
		augtool -s 'set /files/etc/php5/cli/php.ini/PHP/allow_url_fopen On'
		curl -s http://getcomposer.org/installer | php > /dev/null 2>&1
		
		echo '[Composer] Moving to /usr/local/bin/composer.'
		mkdir -p /usr/local/bin
		mv /home/vagrant/composer.phar /usr/local/bin/composer
		(crontab -l; echo '14 3 */2 * * /usr/local/bin/composer self-update') | crontab - > /dev/null 2>&1
		echo $(date) > /.provisioned/composer
	fi
	"

	if config["upgrade_on_boot"] == "yes"
		vconfig.vm.provision :shell, :inline => "
			echo ''
			echo 'Upgrading installed packages in the background.'
			echo 'This will take a while, and you will need to reload'
			echo 'the VM manually when it is complete using:'
			echo ''
			echo 'vagrant reload --no-provision'
			echo ''

			apt-fast update > /dev/null 2>&1
			sudo nohup sh -c 'sudo apt-get update && sudo DEBIAN_FRONTEND=noninteractive apt-get -y -o Dpkg::Options::=\"--force-confdef\" -o Dpkg::Options::=\"--force-confold\" dist-upgrade' > /dev/null 2>&1 &
		"
	end

end

# vim: syntax=ruby