Multiple Wordpress Sites in XAMPP
=================================

A very rough and very incomplete guide to setting up multiple separate virtual hosts in XAMPP and installing wordpress into them.

If you're confused about terms, check the [glossary](#glossary).

If something doesn't work or you encounter some errors or I've forgotten something, let me know via a GitHub issue or a pull request.


Contents
--------

1. [Tl;dr; version](#tldr-version)
2. [Longer version](#longer-version)
  1. [Creating a domain](#1-creating-a-domain)
  2. [Setting up a vhost](#2-setting-up-a-vhost)
  3. [Testing the vhost](#3-testing-the-vhost)
  4. [Setting up a database for WordPress](#4-setting-up-a-database-for-wordpress)
  5. [Preparing to install WordPress](#5-preparing-to-install-wordpress)
  6. [Installing WordPress](#6-installing-wordpress)
  7. [Rejoice!](#7-rejoice)
3. [Glossary](#glossary)


Tl;dr; version
--------------

First make sure xampp is installed and working and that apache is configured to support name-based virtual hosts. Then, every time you want a wordpress site

1. Create a domain (e.g. in `/etc/hosts`)
2. Create a virtualhost with a document root
3. Create a database for wordpress (and optionally a user and password)
4. Unzip wordpress into the document root
5. Restart apache
6. Go through the [wordpress installation instructions](http://codex.wordpress.org/Installing_WordPress#Famous_5-Minute_Install)
7. Rejoice


Longer version
--------------

#### 1. Creating a domain

Creating a domain is generally quite an involved process involving credit cards and registrars and other fun things. Luckily there's a convenient shortcut - the hosts file.

Your operating system will have a hosts file (`/etc/hosts` for linux, `C:\Windows\System32\drivers\etc\hosts` for Windows, dunno for OS X). This is basically a short-cut file that can totally avoid DNS resolution and lets you point any domain at any IP you want. For example, you can make `www.google.com` resolve to any IP you like just by adding a hosts entry like

    127.0.0.1 www.google.com

But with that specific example you'll hit some issues with the SSL certificate. Anyway, to set up your new wordpress site you need to decide on a domain. If you're only setting up one then you already have your very own domain that nobody else can access: `localhost`. If you're setting up multiple sites then they'll each need their own domain.

You can make this domain whatever you want but common choices are `<clientname>.local` or `<clientname-test>.localhost`. The top level domain doesn't matter - you could call it `<clientname>.wheee` or even dispense with the dot and call it just `whee` but some browsers are likely to take issue with just visiting a single TLD so I always follow the `<something>.<something else>` format.

So in summary, if you're making a site called 'foo' then you probably want a domain like `foo.local`.

The next step is to figure out your IP. In most cases if you're setting this up for local development then the `127.0.0.1` IP will be fine - this is the standard address of the IPv4 loopback interface. If you run into issues then it's probably because apache has bound to your IPv6 interface, in which case you probably want `::1`. There's no harm in having an IPv4 and an IPv6 entry for the same domain in your hosts file.

So, if you want to visit your site at `foo.local` then you probably want the following added to the bottom of your hosts file:

    127.0.0.1 foo.local
    ::1 foo.local


#### 2. Setting up a vhost

Now that you have a domain pointing at your IP (probably your local IP) you need a web server to serve files on that domain. Apache is XAMPP's web server and it's pretty good, so use that.

By default a web server binds to port 80 and accepts any incoming connections on that port, speaking HTTP on those connections. This is fine if you only have one website on your server but not so convenient if you want to host multiple sites. To get around this apache has a concept of 'name based virtual hosts' which allows it to serve entirely different sites based purely on the domain name that the browser requests.

To explain this a bit more, a single apache web server can be configured to show everying in `/var/www/vhosts/dogs` to people asking for `dogs.com` and everything in `/var/www/vhosts/cats` to people asking for `cats.com`, all over the same port and on the same server.

This is the technique you should probably use. Before setting it up there's one more thing you need to do - decide where on your filesystem apache will be serving a particular virtual host from (the document root). You'll have to configure this for every vhost. On Linux, this is most commonly in `/var/www/vhosts/<site name>` e.g. `/var/www/vhosts/foo.local`. Once you've decided, create a directory in that location and remember, write down or copy the full absolute path to it. On Windows that will include the drive letter, e.g. `C:/vhosts/foo.local`. 

Notice that the backslashes in typical Windows paths are expressed as foward slashes. This is necessary with Apache and very common in programming. Microsoft, long ago, decided that backslashes were the way forwards and the rest of the world now disagrees, so in most cases you'll find that programming languages expect forward slashes.

Anway, now that you have your document root ready to go you can set up the vhost. You need to find your apache's vhosts config file (`<xampp>/apache/conf/extras/httpd-vhosts.conf`, I think). Either way the following code needs to make it into the apache conf somehow:

```
<VirtualHost *:80>    
    ServerName [domain]
    DocumentRoot "[path to your document root]"
    <Directory "[path to your document root]">
        Require all granted
    </Directory>
</VirtualHost>
```

and with our example:

```
<VirtualHost *:80>    
    ServerName foo.local
    DocumentRoot "C:/vhosts/foo.local"
    <Directory "C:/vhosts/foo.local">
        Require all granted
    </Directory>
</VirtualHost>
```

There's quite a lot there and explaining it all is beyond the scope of this document. 
* The `*:80` bit means 'this virtual host is for all ip addresses on port 80'. 
* The `ServerName` directive tells Apache which domain this vhost is more (you can specify others with the `ServerAlias` directive). 
* The `DocumentRoot` directive tells Apache where the site is located on disk.
* The `<Directory>` block tells Apache that that path is a directory it can serve files from.
* The `Require all granted` bit means 'let everyone see everything in here'.

This is definitely a very insecure and poor set up for any production site so ***don't use it in production***. It'll be fine for local development though.

#### 3. Testing the vhost

Now is probably a good time to test that the vhost is working as you expect. By default when you visit `http://foo.local` or any other site, Apache will serve an 'index' document. The list of valid extensions for an index document is unknown to me, but I know that `index.html` works, so to test your vhost put a file called `index.html` in your document root and put the following code in it:

```
<html>
  <head>
    <title>Test</title>
  </head>
  <body>
    Test
  </body>
</html>
```

If you then restart Apache and go to `http://foo.local` or whatever your domain is then you should see a page with the word 'Test' on it and the title 'Test' in the browser. If so then shout "woohoo" (optional) because everything's working well.


#### 4. Setting up a database for WordPress

XAMPP's database is MariaDB, which is an open source binary-compatible fork of MySQL. In general you can normally just use it as if it is MySQL and virtually all of the advice and instructions online about MySQL will apply to MariaDB equally well.

MariaDB works by having a database server that you and other things can connect to. By default XAMPP will give you a command line client you can use to interact with the MariaDB database server. This is how you can set up the database for WordPress (each WordPress installation should really have its own database).

To create a database, use XAMPP to load up the shell (can't remember how to do this) and type `mysql -u root -p` and enter the root password (might be blank by default, can't remember).

Once at your prompt (`mysql>`, I think) you can create the database with:

    create database <database name>;

Where database name is any valid database name, e.g. `foo_local`.

If you want to create a separate database user for your site (pretty much mandatory for production, not that important in local development) then you can use:

    grant all privileges on [database].[tables] to '[username]'@'[host]' identified by '[password]';

e.g.

    grant all privileges on foo_local.* to 'foo'@'localhost' identified by 'foo-password';


Note that this doesn't configure their permissions securely at all and shouldn't be used in production. It's fine for local development though.

You can confirm that the database exists by trying `show databases` and you should see it in the list.


#### 5. Preparing to install WordPress

Now comes the really tough stuff. You need to unzip the wordpress installation into your document root. If you like you can put it within a subfolder in the document root.


#### 6. Installing WordPress

By now you should have:

* A database (and, optionally, a user and password for it)
* A vhost that works
* An unzipped WordPress in the document root of the vhost
* Low expecations

You can now restart Apache (to make sure that all the config changes are picked up - it's probably not necessary though) and follow the [WordPress installation instructions](http://codex.wordpress.org/Installing_WordPress#Famous_5-Minute_Install). 

#### 7. Rejoice!

Praising the sun is optional, but encouraged.


Glossary
--------

Coming soon!

accept
apache conf
application
binding
browser
computer
connection
database
database server
directive
dns resolution
dns server
domain
domain name
editor
host
hosts file
http
ip
ipv4
ipv6
localhost
mariadb
mysql
name-based virtual host
operating system
port 443
port 80
program
protocol
registrar
server
shell
slashes
ssl
ssl certificate
tcp
top level domain
vhost
web server
wordpress
xampp
binary-compatible
open source
fork