Installing Vagrant
==========

Download the relevant package for your specific OS: 

**Windows** [Universal (32 and 64-bit)](https://releases.hashicorp.com/vagrant/1.9.3/vagrant_1.9.3.msi)
	
>If you're using Windows, you may need to log out of your user profile then log back in to get it working.

**Debian** [32-bit](https://releases.hashicorp.com/vagrant/1.9.3/vagrant_1.9.3_i686.deb)|[64-bit](https://releases.hashicorp.com/vagrant/1.9.3/vagrant_1.9.3_x86_64.deb)

**Centos** [32-bit](https://releases.hashicorp.com/vagrant/1.9.3/vagrant_1.9.3_i686.rpm)|[64-bit](https://releases.hashicorp.com/vagrant/1.9.3/vagrant_1.9.3_x86_64.rpm)

**Mac OS X** [Universal (32 and 64-bit)](https://releases.hashicorp.com/vagrant/1.9.3/vagrant_1.9.3_x86_64.dmg)

Now open up your terminal (you can do this by hitting *command+space* at the same time and type **terminal** if you're using a Mac), copy and paste the following and hit enter.

`mkdir vagrantUbuntuXenial && cd vagrantUbuntuXenial && vagrant init && vagrant box add box ubuntu/xenial64`

+ This creates a working directory at the root of your home directory called `~/vagrantUbuntuXenial/`, then changes the working directory to that and initializes Vagrant, and finally adds the box `ubuntu/xenial64`.
+ To destroy this vagrant, you can run `vagrant destroy`
+ For more information on getting started, head [here](https://www.vagrantup.com/intro/getting-started/index.html)
+ Here's a [list of available boxes](https://atlas.hashicorp.com/ubuntu/boxes/xenial64)

Configuring Vagrant
==========

At this point, you should have a couple of things in your `vagrantUbuntuXenial` directory:

	$ ls -a vagrantUbuntuXenial
	.		..		.vagrant	Vagrantfile

The configuration file for Vagrant is the `Vagrantfile`, which is what you'll need to edit. Again, open this file up with your favorite text editor (I highly recommend [sublime](https://www.sublimetext.com/Sublime%20Text%201.4%20Setup.exe))

Once you have the file open, delete it's contents and replace it with the following:

```
Vagrant.configure("2") do |config|

  config.vm.box = "ubuntu/xenial64"
  config.vm.box_check_update = true
  config.vm.network "forwarded_port", guest: 80, host: 8080

  config.vm.provider "virtualbox" do |vb|
    vb.gui = true
    vb.memory = "512"
  end

  config.vm.provision :shell, path: "bootstrap.sh"
end
```

Since the ubuntu/xenial box is not actively maintained, there's a [bug](https://bugs.launchpad.net/cloud-images/+bug/1569237) that needs to be addressed, we can can include the [fix](https://github.com/joelhandwell/ubuntu_vagrant_boxes/blob/master/u16/Vagrantfile) in our provisioning script labelled `bootstrap.sh`. To create the file, run the following in your terminal:

`touch /vagrantUbuntuXenial/bootstrap.sh`

Now open the file, copy the following code, and paste it into the file.

```
#!/usr/bin/env bash

### The following addresses the bug in ubuntu/xenial vagrant boxes that prevents you from logging in through VirtualBox 	###
### Source: https://github.com/joelhandwell/ubuntu_vagrant_boxes/blob/master/u16/Vagrantfile 								###

echo "create user vagrant"
adduser --disabled-password --gecos "" vagrant
echo 'vagrant:vagrant' | chpasswd
ls -al /home/
echo "add sudo privilege to user vagrant"
cp /etc/sudoers.d/90-cloud-init-users /etc/sudoers.d/admin
chmod +w /etc/sudoers.d/admin
ls -al /etc/sudoers.d/
sed -i 's/ubuntu/vagrant/g' /etc/sudoers.d/admin
cat /etc/sudoers.d/admin
echo "enable ssh access for user vagrant"
mkdir /home/vagrant/.ssh
chown vagrant:vagrant /home/vagrant/.ssh
cat /home/ubuntu/.ssh/authorized_keys > /home/vagrant/.ssh/authorized_keys
chown vagrant:vagrant /home/vagrant/.ssh/authorized_keys
su - vagrant -c "cat /home/vagrant/.ssh/authorized_keys"
chmod 600 /home/vagrant/.ssh/authorized_keys
ls -al /home/vagrant/.ssh
chmod 700 /home/vagrant/.ssh
ls -al /home/vagrant
sudo apt update -y
sudo apt full-upgrade -y
sudo apt-get autoremove -y
sudo systemctl disable apt-daily.service
sudo systemctl disable apt-daily.timer

#sudo apt-get install python-software-properties software-properties-common
#sudo LC_ALL=C.UTF-8 add-apt-repository ppa:ondrej/php

sudo dpkg --configure -a
sudo apt-get update

##Scripting the installation of a LAMP stack causes more issues than it solves ##

#sudo apt-get purge php5-common -y
#sudo apt-get install -y apache2 mysql-server php7.0 php7.0-fpm php7.0-mysql

if ! [ -L /var/www ]; then
  rm -rf /var/www
  ln -fs /vagrant /var/www
fi
```

The tl;dr of this script is that it allows us to set the username:password to vagrant:vagrant. Without this, we would not be able to login using the default user when using VirtualBox. It's also setting up the DocumentRoot to be shared with the Vagrant root. You can read more about shared folders [here](https://www.vagrantup.com/docs/synced-folders/). 

Now let's start our Vagrant box by running the following in your terminal:

`vagrant up`

You should see a bunch of output. Once that's complete, go ahead and ssh into your Vagrant box, you should see the following:

>**NOTE**: I will be prepending **ubuntu@ubuntu-xenial:~$** with each command that needs to be ran from the Xenial box 

```
$ vagrant ssh
Welcome to Ubuntu 16.04.2 LTS (GNU/Linux 4.4.0-72-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  Get cloud support with Ubuntu Advantage Cloud Guest:
    http://www.ubuntu.com/business/services/cloud

3 packages can be updated.
0 updates are security updates.


Last login: Sat Apr 15 01:42:24 2017 from 10.0.2.2
ubuntu@ubuntu-xenial:~$
```

Configuring the LAMP Stack
==========

Now that you're in, run the following to install the webserver your local WordPress site will be using:

`**ubuntu@ubuntu-xenial:~$** sudo apt-get install -y apache2 mysql-client mysql-server php7.0 php7.0-fpm php7.0-mysql libapache2-mod-php7.0 php7.0-cli php7.0-cgi php7.0-gd`

You should be taken to the mysql installation screen (it's pinkish) where you'll be asked to choose a mysql root password. I set my root password to be `root`.

You can safely ignore this from the output:

```
NOTICE: Not enabling PHP 7.0 FPM by default.
NOTICE: To enable PHP 7.0 FPM in Apache2 do:
NOTICE: a2enmod proxy_fcgi setenvif
NOTICE: a2enconf php7.0-fpm
```

At this point, you should be able to access the index.html file located at the webroot, you can do this a number of ways:

1. `**ubuntu@ubuntu-xenial:~$** wget -qO- 127.0.0.1`
2. `**ubuntu@ubuntu-xenial:~$** curl 127.0.0.1`

From Vagrant's [getting-started](https://www.vagrantup.com/intro/getting-started/provisioning.html):

>This works because in the shell script above, we installed Apache and setup the default DocumentRoot of Apache to point to our /vagrant directory, which is the default synced folder setup by Vagrant.

The guide is specifically referring to this code block in the shell script:

```
if ! [ -L /var/www ]; then
  rm -rf /var/www
  ln -fs /vagrant /var/www
fi
```

>**NOTE**: we didn't install Apache in our shell script in this example, but directly from the box once ssh'd in

You can also visit the webroot from your browser at this point, just head to <http://localhost:8080>

Installing WordPress
==========

>most of this was gathered from this guide [here](http://www.tecmint.com/install-wordpress-on-ubuntu-16-04-with-lamp/)

First, we'll want to test whether or not PHP is working properly, so in order to do this, we'll test by creating a php info page. Open up a new terminal tab by pressing `command+t` and run the following:

`cd /var/www/html && touch info.php && echo "<?php phpinfo(); ?>" >> info.php && curl -IL http://localhost/info.php`

You should see the following output:

```
HTTP/1.1 200 OK
Date: Sat, 15 Apr 2017 02:41:25 GMT
Server: Apache/2.4.18 (Ubuntu)
Content-Type: text/html; charset=UTF-8
```

I've decided that it's best to download WordPress to the root then rsync the files to the web root from there. To do this, run the following:

`cd /vagrant/ && wget -c http://wordpress.org/latest.tar.gz && tar -xzvf latest.tar.gz && sudo rsync -av wordpress/* /var/www/html/`

Now we'll need to set permissions for the `www-data` user:

`sudo chown -R www-data:www-data /var/www/html/ && sudo chmod -R 755 /var/www/html/`

>this isn't actually doing anything in Vagrant since we haven't created the `www-data` user, but it's an important step and should always be taken when installing WordPress.

To set us up for the database creation, we'll need to edit the wp-config.php file, but WordPress only comes with a sample `.php` file, so let's copy it to wp-config.php with:

`cp wp-config-sample.php wp-config.php`

Open up the `wp-config.php` file using whatever text editor you're comfortable with. The important things we'll be doing here are updating the db credentials and salts. Feel free to use the same credentials

```
define('DB_NAME', 'wp_myblog');
.
.
define('DB_USER', 'myblog');
.
.
define('DB_PASSWORD', 'myblog');
```

You can visit this page <https://api.wordpress.org/secret-key/1.1/salt/> to obtain the salts, replace what you get from that page with the defines that are already there.

Once this is done, you can access mysql by running `mysql -u root -p`. You should see the following (you'll enter the password you chose for your mysql root user):

```
Enter password:
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 8
Server version: 5.7.17-0ubuntu0.16.04.2 (Ubuntu)
```

Once you're in, run the following commands. If you didn't decide to set the database, user, and password with the same values that are used in this guide, use the ones you went with, but the commands are the same.

```
mysql> CREATE DATABASE wp_myblog;
mysql> GRANT ALL PRIVILEGES ON wp_myblog.* TO 'myblog'@'localhost' IDENTIFIED BY 'myblog';
mysql> FLUSH PRIVILEGES;
mysql> EXIT;
```

Once you've done this, you'll need to restart both apache and mysql, you can do this by running:

`sudo systemctl restart apache2.service && sudo systemctl restart mysql.service`

At this point, you'll be able to head to your browser to finish the installation of WordPress:

<http://localhost:8080/>


Errors
==========

If you already have a vagrant running, you may see the following error when attempting to run a second vagrant:

	$ vagrant up
	Bringing machine 'default' up with 'virtualbox' provider...
	Vagrant cannot forward the specified ports on this VM, since they
	would collide with some other application that is already listening
	on these ports. The forwarded port to 2222 is already in use
	on the host machine.

	To fix this, modify your current project's Vagrantfile to use another
	port. Example, where '1234' would be replaced by a unique host port:

	  config.vm.network :forwarded_port, guest: 22, host: 1234

	Sometimes, Vagrant will attempt to auto-correct this for you. In this
	case, Vagrant was unable to. This is usually because the guest machine
	is in a state which doesn't allow modifying port forwarding. You could
	try 'vagrant reload' (equivalent of running a halt followed by an up)
	so vagrant can attempt to auto-correct this upon booting. Be warned
	that any unsaved work might be lost.

Do exactly what it says, then run:

	$ vagrant reload

You'll see that it fixes this issue automatically:

	`==> default: Fixed port collision for 22 => 2222. Now on port 2200.`
