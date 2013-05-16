#Deploying Kibana as a Rack app with Passenger

Credit where credit is due: git://github.com/nickchappell/personal-projects-docs.git

Thanks Nick Chappell, I ran with it and made it my own.


###Modify older versions of Kibana to conform to Rack application standards

If you download an older version of Kibana, you may have to make some modifications to it in order for it to be loaded as a Rack application.

Download and extract the tarball for Kibana.

In the folder the tarball extracted to, rename `static` to `public`, create a `tmp` folder and create a `restart.txt` file inside of the `tmp` folder.

In `kibana.rb`, find the following line and change `static` to `public`:

```
 set :public_folder, Proc.new { File.join(root, "static") }
``` 

…to:

```
set :public_folder, Proc.new { File.join(root, "public") }
``` 

##Preliminary steps

Update everything first:

`sudo apt-get update; sudo apt-get upgrade`

Install Ruby Gems:

`sudo apt-get install rubygems`

Install Passenger:

`gem install passenger`

###Path stuff

We have to add the gems folder to our path so the Passenger-specific commands later on will work.

Edit `/etc/bash.bashrc` and add the following line to the end:

```
export PATH=$PATH:/usr/local/bin:/var/lib/gems/1.8/bin:/var/lib/gems/1.9/bin
```

Once you're done, paste it into your terminal as a command and run it. This is just so we don't have to log out and then log back in for it to take effect.

##Apache steps

Adapted from this guide: [http://www.modrails.com/documentation/Users%20guide%20Apache.html](http://www.modrails.com/documentation/Users%20guide%20Apache.html)

###Install packages

Passenger comes with a script to configure itself with Apache automatically, but you'll need to have Apache and some other packages installed first.

Install them:

```
apt-get install apache2-mpm-prefork apache2-prefork-dev libapr1-dev libaprutil1-dev
``` 

###Use the Passenger script to download, compile and install the Apache2 Passenger module

Run `passenger-install-apache2-module`. 

###Create .load and .conf files for module loading

To have Passenger available as a module that can be loaded and unloaded, we have to create 2 files in `/etc/apache2/mods-available`.

To allow Apache to load the Passenger module, create `passenger.load` and give it the following contents:

```
LoadModule passenger_module /var/lib/gems/1.8/gems/passenger-3.0.19/ext/apache2/mod_passenger.so
``` 

To set some Passenger module options, create `passenger.conf` and give it the following contents:

```
PassengerRoot /var/lib/gems/1.8/gems/passenger-3.0.19
PassengerRuby /usr/bin/ruby1.8
``` 

To load the Passenger module, type the following command and then restart Apache:

```
sudo a2enmod passenger
/etc/init.d/apache2 restart
```

**Note:** The above steps aren't required to have Passenger loaded as a module. You can paste the contents of `passenger.load` and `passenger.conf` in as one chunk at the bottom of `/etc/apache2/apache2.conf`. However, this would load Passenger every time Apache started and you would have to comment out or remove the lines to stop Passenger from loading, as opposed to just using the `a2enmod` and `a2dismod` commands.

Using the `mods-available` folder and the separate `.load` and `.conf` files isn't required but provides a little more granularity with respect to module loading/unloading.

###Install Kibana

Clone Kibana from Github to get the latest version:

`git clone git://github.com/rashidkpc/Kibana.git`

####Create the kibana gem

Build the gem

```shell
cd Kibana
gem build kibana.gemspec
```

Install the gem

```shell
sudo gem install kibana-0.2.0.gem
```

(by default it goes to `/var/lib/gems/1.8/gems/kibana-0.2.0/` if you are using rvm or something else you'll need to change the following sections.

###Creating a Kibana virtual host

Create `/etc/apache2/sites-available/kibana`, and place the following in there.

```
<VirtualHost *:80>
  ServerName myservername.com
  DocumentRoot /var/lib/gems/1.8/gems/kibana-0.2.0/public
    <Directory "/var/lib/gems/1.8/gems/kibana-0.2.0/public">
      Allow from all
      Options FollowSymLinks
      Options -MultiViews
    </Directory>

</VirtualHost>
```

Run the following to enable the site and disable default.

```
sudo a2dissite default
sudo a2ensite kibana
sudo /etc/init.d/apache2 restart
```

##Nginx steps (TODO)

Adapted from this guide: [http://www.modrails.com/documentation/Users%20guide%20Nginx.html](http://www.modrails.com/documentation/Users%20guide%20Nginx.html)

###Let Passenger compile and install Nginx with Passenger support

Passenger has an interactive script that does this, `passenger-install-nginx-module` (this is in `/usr/local/bin`, which we added to our path above). If it runs into any problems, it stops and tells you want to do before running it again.

In my case, it wanted some development OpenSSL libraries to be installed:

```
Installation instructions for required software

 * To install OpenSSL development headers:
   Please run apt-get install libssl-dev as root.

If the aforementioned instructions didn't solve your problem, then please take
a look at the Users Guide:

  /var/lib/gems/1.8/gems/passenger-3.0.19/doc/Users guide Nginx.html
```

I ran `sudo apt-get install libssl-dev` and then ran `passenger-install-nginx-module` again so it could pick up from where it left off.

When it asks if you want a simple or advanced install, select simple. The default answers for questions it asks for any questions (in brackets) are fine.

Nginx will be installed and will start running automatically. To be able to start and stop it, you'll want to create an **init.d script**.

Create a new file, `/etc/init.d/nginx`, grab the init.d script from [this page](http://wiki.nginx.org/Nginx-init-ubuntu) and paste it into the file.

Then, run `/usr/sbin/update-rc.d -f nginx defaults` to install it. You should see the output below if it's successful:

```
root@my-server: ~ > /usr/sbin/update-rc.d -f nginx defaults
 Adding system startup for /etc/init.d/nginx ...
   /etc/rc0.d/K20nginx -> ../init.d/nginx
   /etc/rc1.d/K20nginx -> ../init.d/nginx
   /etc/rc6.d/K20nginx -> ../init.d/nginx
   /etc/rc2.d/S20nginx -> ../init.d/nginx
   /etc/rc3.d/S20nginx -> ../init.d/nginx
   /etc/rc4.d/S20nginx -> ../init.d/nginx
   /etc/rc5.d/S20nginx -> ../init.d/nginx
``` 

Try starting, stopping and restarting Nginx via the init.d script. If you get a `command not found` error, you may have to edit the init script to point to the different locations of the Nginx binary and its PID files. The locations may be different because of where the Passenger install script placed them.

In my case: 

* the Nginx binary was located at `/opt/nginx/sbin/nginx`
* the Nginx config file was located at `/opt/nginx/conf/nginx.conf`
* the PID file was located at `/opt/nginx/logs/nginx.pid`

The following lines in the init.d script from the page linked to above have to be modified:

```
DAEMON=/opt/nginx/sbin/nginx
...
PIDSPATH=/opt/nginx/logs/
...
NGINX_CONF_FILE="/opt/nginx/conf/nginx.conf"
``` 
