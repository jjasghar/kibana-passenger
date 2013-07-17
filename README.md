#Deploying Kibana as a Rack app with Passenger

Credit where credit is due: [Thanks Nick Chappel](http://github.com/nickchappell/personal-projects-docs.git), I ran with it and made it my own.


###Modify older versions of Kibana to conform to Rack application standards

If you download an older version of Kibana, you may have to make some modifications to it in order for it to be loaded as a Rack application.

Download and extract the tarball for Kibana.

In the folder the tarball extracted to, rename `static` to `public`, create a `tmp` folder and create a `restart.txt` file inside of the `tmp` folder.

In `kibana.rb`, find the following line and change `static` to `public`:

```
 set :public_folder, Proc.new { File.join(root, "static") }
``` 

to:

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

Adapted from this guide: [Apache Users Guide](http://www.modrails.com/documentation/Users%20guide%20Apache.html)

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

NOTE: by default it goes to `/var/lib/gems/1.8/gems/kibana-0.2.0/` if you are using rvm or something else you'll need to change the following sections.

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

