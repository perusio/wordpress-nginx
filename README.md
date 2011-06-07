# Nginx configuration for WordPress

## Introduction 

   This is a nginx configuration for running [WordPress](http://wordpress.org "WordPress").
   
   It **differs** from the _usual_ configuration, like the
   [one](http://wiki.nginx.org/Wordpress "Nginx Wiki WordPress
   config") available on the [Nginx Wiki](http://wiki.nginx.org "Nginx
   Wiki").
   
   It makes use of **nested locations** with named capture groups
   instead of [fastcgi_split\_path\_info](http://wiki.nginx.org/HttpFcgiModule#fastcgi_split_path_info
   "FastCGI split path info").

   This example configuration assumes that the site is called
   `example.com`. Change accordingly to reflect your server setup.
   
## Features

   1. Filtering of invalid HTTP `Host` headers.

   2. Access to install files, like `install.php,` is protected using
      [HTTP Basic Auth](http://wiki.nginx.org/NginxHttpAuthBasicModule
      "Basic Auth Nginx Module").  

   3. Protection of all the _internal_ directories, like version
      control repositories and the `readme` file(s) 
      that come with WP or an external plugin.

   4. Faster and more secure handling of PHP FastCGI by Nginx using
      named groups in regular expressions instead of using
      [fastcgi_split\_path\_info](http://wiki.nginx.org/HttpFcgiModule#fastcgi_split_path_info
      "FastCGI split path info"). Requires Nginx version &ge; 0.8.25.

   5. Compatible with the WordPress plugin
      [wp-super-cache](http://wordpress.org/extend/plugins/wp-super-cache "WordPress
      SuperCache") for serving static pages to anonymous users.
      
   6. [Upload Progress](http://wiki.nginx.org/NginxHttpUploadProgressModule
      "Upload progress Nginx module") support.   
   
   7. Possibility of using **Apache** as a backend for dealing with
      PHP. Meaning using Nginx as
      [reverse proxy](http://wiki.nginx.org/HttpProxyModule "Nginx Proxy Module").

## Basic Auth for access to restricted files like install.php

   `install.php` and the WordPress `readme.html` are protected using
   Basic Auth. The readme file discloses the version number of
   WordPress.
   
   Not only `install.php`, but any PHP file that has **install.php**
   as the ending is protected. This way if, for example, there's a
   permission problem with `wp-config.php` and WP can't read the file
   it will invoke `install.php` since it assumes that if no specific
   configuration information is available then the site must not yet
   be installed. Now imagine that this happens on your site and that
   someone stumbles on the `install.php`? If not protected by the
   Basic Auth, information disclosure would be the least potential
   problem.

   You have to create the `.htpasswd-users` file with the user(s) and
   password(s). For that, if you're on Debian or any of its
   derivatives like Ubuntu you need the
   [apache2-utils](http://packages.debian.org/search?suite%3Dall&section%3Dall&arch%3Dany&searchon%3Dnames&keywords%3Dapache2-utils)
   package installed. Then create your password file by issuing:

          htpasswd -d -b -c .htpasswd-users <user> <password>

   You should delete this command from your shell history
   afterwards with `history -d <command number>` or alternatively
   omit the `-b` switch, then you'll be prompted for the password.

   This creates the file (there's a `-c` switch). For adding
   additional users omit the `-c`.

   Of course you can rename the password file to whatever you want,
   then accordingly change its name in the virtual host config
   file, `example.com`.

## Nginx as a Reverse Proxy: Proxying to Apache for PHP

   If you **absolutely need** to use the rather _bad habit_ of
   deploying web apps relying on `.htaccess`, or you just want to use
   Nginx as a reverse proxy. The config allows you to do so. Note that
   this provides some benefits over using only Apache, since Nginx is
   much faster than Apache. Furthermore you can use the proxy cache
   and/or use Nginx as a load balancer. 
   
## Installation

   1. Move the old `/etc/nginx` directory to `/etc/nginx.old`.
   
   2. Clone the git repository from github:
   
      `git clone https://github.com/perusio/nginx-wordpress.git`
   
   3. Edit the `sites-available/example.com` configuration file to
      suit your requirements. Namely replacing `example.com` with
      **your** domain.
   
   4. Setup the PHP handling method. It can be:
   
      + Upstream HTTP server like Apache with mod_php. To use this
        method comment out the `include upstream_phpcgi.conf;`
        line in `nginx.conf` and uncomment the lines:
        
            include reverse_proxy.conf;
            include upstream_phpapache.conf;

        Now you must set the proper address and port for your
        backend(s) in the `upstream_phpapache.conf`. By default it
        assumes the loopback `127.0.0.1` interface on port
        `8080`. Adjust accordingly to reflect your setup.

        Comment out **all**  `fastcgi_pass` directives in either
        `drupal_boost.conf` or `drupal_boost_drush.conf`, depending
        which config layout you're using. Uncomment out all the
        `proxy_pass` directives. They have a comment around them,
        stating these instructions.
      
      + FastCGI process using php-cgi. In this case an
        [init script](https://github.com/perusio/php-fastcgi-debian-script
        "Init script for php-cgi") is
        required. This is how the server is configured out of the
        box. It uses UNIX sockets. You can use TCP sockets if you prefer.
      
      + [PHP FPM](http://www.php-fpm.org "PHP FPM"), this requires you
        to configure your fpm setup, in Debian/Ubuntu this is done in
        the `/etc/php5/fpm` directory.
        
      Check that the socket is properly created and is listening. This
      can be done with `netstat`, like this for UNIX sockets:
      
         `netstat --unix -l`
         
      And like this for TCP sockets:   
         
         `netstat -t -l`
   
      It should display the PHP CGI socket.
   
      Note that the default socket type is UNIX and the config assumes
      it to be listening on `unix:/tmp/php-cgi/php-cgi.socket`, if
      using the `php-cgi`, or in `unix:/var/run/php-fpm.sock` using
      `php-fpm` and that you should **change** to reflect your setup
      by editing `upstream_phpcgi.conf`.
   
   
   5. Create the `/etc/nginx/sites-enabled` directory and enable the
      virtual host using one of the methods described below.

      Note that if you're using the
      [nginx_ensite](http://github.com/perusio/nginx_ensite) script
      described below it **creates** the `/etc/nginx/sites-enabled`
      directory if it doesn't exist the first time you run it for
      enabling a site.    

   6. Reload Nginx:
   
      `/etc/init.d/nginx reload`
   
   7. Check that WordPress is working by visiting the configured site
      in your browser.
   
   8. Remove the `/etc/nginx.old` directory.
   
   9. Done.
   
## Enabling and Disabling Virtual Hosts

   I've created a shell script
   [nginx_ensite](http://github.com/perusio/nginx_ensite) that lives
   here on github for quick enabling and disabling of virtual hosts.
   
   If you're not using that script then you have to **manually**
   create the symlinks from `sites-enabled` to `sites-available`. Only
   the virtual hosts configured in `sites-enabled` will be available
   for Nginx to serve.
   
## Acessing the php-fpm status and ping pages

   You can get the
   [status and a ping](http://forum.nginx.org/read.php?3,56426) pages
   for the running instance of `php-fpm`. There's a
   `php_fpm_status.conf` file with the configuration for both
   features.
   
   + the **status page** at `/fpm-status`;
     
   + the **ping page** at `/ping`.

   For obvious reasons these pages are acessed only from a given set
   of IP addresses. In the suggested configuration only from
   localhost and non-routable IPs of the 192.168.1.0 network.
    
   To enable the status and ping pages uncomment the line in the
   `example.com` virtual host configuration file.   
   
   
## Getting the latest Nginx packaged for Debian or Ubuntu

   I maintain a [debian repository](http://debian.perusio.net/unstable
   "my debian repo") with the
   [latest](http://nginx.org/en/download.html "Nginx source download")
   version of Nginx. This is packaged for Debian **unstable** or
   **testing**. The instructions for using the repository are
   presented on this [page](http://debian.perusio.net/debian.html
   "Repository instructions").
 
   It may work or not on Ubuntu. Since Ubuntu seems to appreciate more
   finding semi-witty names for their releases instead of making clear
   what's the status of the software included. Is it **stable**? Is it
   **testing**? Is it **unstable**? The package may work with your
   currently installed environment or not. I don't have the faintest
   idea which release to advise. So you're on your own. Generally the
   APT machinery will sort out for you any dependencies issues that
   might exist.

## My other Nginx configs on github

   + [Drupal](https://github.com/perusio/drupal-with-nginx "Drupal
     Nginx configuration")
     
   + [Piwik](https://github.com/perusio/piwik-nginx "Piwik Nginx
     configuration")
     
   + [Chive](https://github.com/perusio/chive-nginx "Chive Nginx
     configuration")

## Securing your PHP configuration

   I have created a small shell script that parses your `php.ini` and
   sets a sane environment, be it for **development** or
   **production** settings. 
   
   Grab it [here](https://github.com/perusio/php-ini-cleanup "PHP
   cleanup script").
