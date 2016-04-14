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

First make sure XAMPP is installed and working and that Apache is configured to support name-based virtual hosts. Then, every time you want a WordPress site

1. Create a domain (e.g. in `/etc/hosts`)
2. Create a virtual host with a document root
3. Create a database for WordPress (and optionally a user and password)
4. Unzip WordPress into the document root
5. Restart Apache
6. Go through the [WordPress installation instructions](http://codex.wordpress.org/Installing_WordPress#Famous_5-Minute_Install)
7. Rejoice


Longer version
--------------

#### 1. Creating a domain

Creating a domain is generally quite an involved process involving credit cards and registrars and other fun things. Luckily there's a convenient shortcut - the hosts file.

Your operating system will have a hosts file (`/etc/hosts` for Linux, `C:\Windows\System32\drivers\etc\hosts` for Windows, dunno for OS X). This is basically a short-cut file that can totally avoid DNS resolution and lets you point any domain at any IP you want. For example, you can make `www.google.com` resolve to any IP you like just by adding a hosts entry like

    127.0.0.1 www.google.com

But with that specific example you'll hit some issues with the SSL certificate. Anyway, to set up your new WordPress site you need to decide on a domain. If you're only setting up one then you already have your very own domain that nobody else can access: `localhost`. If you're setting up multiple sites then they'll each need their own domain.

You can make this domain whatever you want but common choices are `<clientname>.local` or `<clientname-test>.localhost`. The top level domain doesn't matter - you could call it `<clientname>.wheee` or even dispense with the dot and call it just `whee` but some browsers are likely to take issue with just visiting a single TLD so I always follow the `<something>.<something else>` format.

So in summary, if you're making a site called 'foo' then you probably want a domain like `foo.local`.

The next step is to figure out your IP. In most cases if you're setting this up for local development then the `127.0.0.1` IP will be fine - this is the standard address of the IPv4 loopback interface. If you run into issues then it's probably because Apache has bound to your IPv6 interface, in which case you probably want `::1`. There's no harm in having an IPv4 and an IPv6 entry for the same domain in your hosts file.

So, if you want to visit your site at `foo.local` then you probably want the following added to the bottom of your hosts file:

    127.0.0.1 foo.local
    ::1 foo.local


#### 2. Setting up a vhost

Now that you have a domain pointing at your IP (probably your local IP) you need a web server to serve files on that domain. Apache is XAMPP's web server and it's pretty good, so use that.

By default a web server binds to port 80 and accepts any incoming connections on that port, speaking HTTP on those connections. This is fine if you only have one website on your server but not so convenient if you want to host multiple sites. To get around this Apache has a concept of 'name based virtual hosts' which allows it to serve entirely different sites based purely on the domain name that the browser requests.

To explain this a bit more, a single Apache web server can be configured to show everying in `/var/www/vhosts/dogs` to people asking for `dogs.com` and everything in `/var/www/vhosts/cats` to people asking for `cats.com`, all over the same port and on the same server.

This is the technique you should probably use. Before setting it up there's one more thing you need to do - decide where on your filesystem Apache will be serving a particular virtual host from (the document root). You'll have to configure this for every vhost. On Linux, this is most commonly in `/var/www/vhosts/<site name>` e.g. `/var/www/vhosts/foo.local`. Once you've decided, create a directory in that location and remember, write down or copy the full absolute path to it. On Windows that will include the drive letter, e.g. `C:/vhosts/foo.local`. 

Notice that the backslashes in typical Windows paths are expressed as foward slashes. This is necessary with Apache and very common in programming. Microsoft, long ago, decided that backslashes were the way forwards and the rest of the world now disagrees, so in most cases you'll find that programming languages expect forward slashes.

Anway, now that you have your document root ready to go you can set up the vhost. You need to find your Apache's vhosts config file (`<xampp>/apache/conf/extras/httpd-vhosts.conf`, I think). Either way the following code needs to make it into the Apache conf somehow:

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
* The `*:80` bit means 'this virtual host is for all IP addresses on port 80'. 
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

Now comes the really tough stuff. You need to unzip the WordPress installation into your document root. If you like you can put it within a subfolder in the document root.


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

**accept**

When a server receives a connection on a port and starts listening and talking to it in some sort of protocol


**Apache**

Sometimes the open source software organisation giving us many great things, and sometimes the Apache HTTP Server which just gets called 'Apache'. Confusing


**Apache conf**

The collection of configuration files Apache reads to know what to do. Normally starts with httpd.conf or Apache.com and the others get `include`d


**application**

A program that runs on a computer


**binary-compatible**

A fancy word that means you can replace the program with another binary-compatible program and everything should work exactly as it used to


**binding**

When a server starts listening for connections on a port. Not to be confused with `BIND`, the DNS server implementation


**browser**

A program on a user's computer that knows how to speak HTTP and display web sites and other things


**client**

Sometimes a customer, but more often a program that talks to a server (via a connection)


**computer**

Really?


**connection**

An open communication, typically between a client and a server. Most connections for the web are over TCP


**database**

A program that is good at storing data records and relating them together in useful and interesting ways for you


**database server**

A server that provides access to a database


**directive**

A statement in an Apache conf file that means something to Apache


**DNS**

Domain Name System - a way of using human-readable names to represent resources on a network


**DNS resolution**

Given a domain name, finding the IP address it points to, e.g. for me www.google.com -> 216.58.213.131


**DNS server**

A server that provides DNS resolution


**domain**, **domain name**

A textual representation for a resource, usually an IP address - e.g. www.google.com -> 216.58.213.131


**editor**

A program that's good at editing text-based files (e.g. code). Emacs is the best one, but Notepad++ is decent too. Vi is probably good. On Windows, Notepad is the devil


**fork**

A copy of a project's source code, normally created so that someone can do different things that the project maintainer/owner wouldn't allow them to in the main project. Often used to submit changes to a project


**host**

Either the stuff that lets someone see a website, or a specific server on which a website resides. As a verb it means 'to provide to clients'


**hosts file**

The super special file that is checked before any DNS resolution is attempted. Think of it as a basic and private DNS server and you're halfway there


**HTTP**

Hypertext Transfer Protocol - a protocol for serving web pages


**IP**

Internet Protocol - a protocol for communicating data over a network


**ipv4**

The old and outdated 4-byte addressing scheme, e.g. 8.8.4.4 (Google's DNS server)


**ipv6**

The new and swanky 16-byte addressing scheme, e.g. 2001:4860:4860::8844


**localhost**

A special domain that exclusively means 'this computer'


**MariaDB**, **MySQL**

A popular database server and client


**name-based virtual host**

An Apache concept allowing different sites to be served to different domains on the same Apache server


**open source**

A style of licensing encouraging collaboration and sharing. Free as in freedom, and often free as in beer


**operating system**

The thing you use to interact with your computer hardware. Popular examples include Windows, Linux, Android and OS X


**port 443**

The normal port for HTTPS connections


**port 80**

The normal port for HTTP connections


**program**

A set of instructions your operating system knows how to execute in order to achieve something. Examples include Minesweeper, Apache and Bash. More loosely it can refer to a set of code that the computer can execute in some way, e.g. a PHP script


**protocol**

A set of rules for communicating. The rules typically define a message structure that allows for efficient and effective relaying of data and actions depending on the requirements. Examples include HTTP, HTTPS, FTP and IP


**registrar**

A company or organisation that's responsible for tracking who gets to decide which domains point at which IPs and other DNS-related things


**server**

Either a program that listens for connections and speaks to clients, or sometimes a physical computer that runs a server (program). E.g. Apache is a server (program) and my previous company had about 30 servers (physical or virtual machines) each of which ran one or two Apache servers (program). Confusing, eh?


**shell**

A program that typically makes it easy to run other programs. Examples include Bash, dash and PowerShell


**slashes**

Either / or \. The first is a forward slash and is widely used in every popular OS other than Windows and also on the web. The second is a backslash typically used for indicating that the following character is special in some way, or for separating path elements on Windows


**SSL**

Secure Socket Layer - a means of encrypting a connection so that only the server and client can understand what's being communicated


**SSL certificate**

A certificate that helps a client talking to a server over SSL trust that the server is who it says it is. For example, SSL would allow me to securely talk to www.google.com but it won't help me know that www.google.com is actually some horrible attacker from next door. SSL certificates will warn me that although I appear to be talking to www.google.com I'm probably talking to www.hacker-next-door.com


**tcp**

Transmission Control Protocol - An impossible-to-remember acronym and a protocol that adds error correction and synchronous communication on top of IP. It's so commonly used with IP that generally things claim to use TCP/IP. An alternative to TCP for different requirements is UDP


**TLD**, **top level domain**

The last bit of a domain name, e.g. the `com` in `google.com`. There are now many more of these than there used to be


**vhost**, **virtual host**, **virtualhost**

A website that Apache knows to serve based on the domain you ask for. Also can refer to the vhost config or the root directory of the vhost


**web server**

A server (program) that knows how to speak HTTP, HTTPS and sometimes some other things. Serves web pages


**WordPress**

Some sort of PHP-based blog-slash-CMS type thing


**XAMPP**

A bundle of technologies (I think run by Apache?) that includes an Apache web server, MariaDB database server and client, and PHP. Maybe some other things
