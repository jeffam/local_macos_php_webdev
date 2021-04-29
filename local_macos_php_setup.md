# macOS + php-fpm + apache

These are the notes for my PHP (mainly Drupal) development setup on macOS. I've tried Vagrant and Docker, and I sure wish they were worth it, but filesystem performace issues are too noticeable.

I prefer this native setup along with a robust development/staging server for catching any glitches from infrastructure differences.

## Overview

- PHP via [Homebrew][]
- Apache 2.4.x (included with macOS)

## PHP

Install PHP 7.3 via Homebrew:

```
$ brew install php@7.3
```

If you have an older version of PHP installed, run the following to ensure the `php` CLI version is 7.3, too.

```
$ brew link --force --overwrite php@7.3
```

The run `php -v` to see check the CLI version.

### Configure PHP

You may want to edit some settings in `/usr/local/etc/php/7.3/php.ini`. Here are my modifications:

```
max_execution_time = 90
max_input_time = 90
memory_limit = 1024M
```

Also consider running PHP as your user. This can reduce file permissions issues in Drupal and other PHP applications. Edit the `/usr/local/etc/php/7.3/php-fpm.d/www.conf` file and look for these lines:

```
user = _www
group = _www
```

Change `_www` to your username (which you can find out with `whoami`).

### Start PHP

```
brew services start php@7.3
```

### xdebug

These instructions enable step debugging only. Be sure to read the full [xdebug documentation][] for profiling, etc.

```
$ pecl install xdebug
```

Check the file `/usr/local/etc/php/7.3/php.ini` to see if the line `zend_extension="xdebug.so"` was added at the top of the file. Remove it if it's there.

Add the following to `/usr/local/etc/php/7.3/conf.d/ext-xdebug.ini`.

```
[xdebug]
zend_extension="xdebug.so"

xdebug.mode=debug
xdebug.start_with_request=yes
```

And restart PHP for the settings to take effect:

```
brew services restart php@7.3
```

#### Xdebug Performance

Xdebug causes a performance hit on PHP requests, both web and CLI.

To reduce the hit, we can run two `php-fpm` pools - one with xdebug enabled and the other with xdebug disabled.

Homebrew PHP includes a default `www` pool.

- Copy `/usr/local/etc/php/7.3/php-fpm.d/www.conf` to `/usr/local/etc/php/7.3/php-fpm.d/www-xdebug.conf`.
- Edit `www-xdebug.conf`.
- Change `[www]` to `[www-xdebug]`.
- Change the line `listen = 127.0.0.1:9000` to `listen = 127.0.0.1:9009`.
- Edit `/usr/local/etc/php/7.3/php-fpm.d/www.conf` and add the following to a new line at the bottom of the file: `php_admin_value[xdebug.mode] = off`

Restart PHP for the settings to take effect:

```
brew services restart php@7.3
```

After these changes, you'll have two `php-fpm` pools running on ports `9000` and `9009`. Only the `php-fpm` running on port `9009` will have Xdebug enabled.

To disable Xdebug for command line scripts, set the following environment variable:

```
export XDEBUG_MODE=off
```

You can run that as needed, or add it permanently to your `.zshrc` or `.bash_profile` files.

## Apache

Apache comes with macOS. You can verify that and get the installed version with:

```
$ httpd -v
Server version: Apache/2.4.41 (Unix)
Server built:   Jun  5 2020 23:42:06
```

That's the result from macOS `10.15.7`.

Many guides suggest installing Apache (or nginx) from Homebrew, but I've had no trouble with the version included in the OS.

### Configure Apache

Apache's configuration files are in `/etc/apache2`. To avoid having your changes overwritten by updates to Apache, create the new directory `/etc/apache2/other` to contain your custom configuration.

Ensure that the following line is at the end of `/etc/apache2/httpd.conf` (the main Apache configuration file):

```
Include /private/etc/apache2/other/*.conf
```

Create a file in the `other` directory called `ANYFILENAME.conf` (e.g. `local.conf`). Add these lines to the file:

```
# Local settings

# Overridden fron httpd.conf. Replace USERNAME with the output from `whoami`.
User USERNAME

# Overridden from httpd.conf to bind to only loopback address.
Listen 127.0.0.1:80

# Set a server name for logs
ServerName localdev.test:80

# Add Apache modules
LoadModule actions_module libexec/apache2/mod_actions.so
LoadModule vhost_alias_module libexec/apache2/mod_vhost_alias.so
LoadModule rewrite_module libexec/apache2/mod_rewrite.so
LoadModule proxy_module libexec/apache2/mod_proxy.so
LoadModule proxy_fcgi_module libexec/apache2/mod_proxy_fcgi.so

# The directory where all development sites live, with the pattern
# ./hostname/web
# where hostname is usually something like mysite.test.
Define webdir /Users/USERNAME/Sites/

<Directory ${webdir}>
  Options +Indexes -Multiviews +FollowSymLinks
  AllowOverride All
  Require all granted
</Directory>

<VirtualHost *:80>
  ServerAlias *
  VirtualDocumentRoot ${webdir}%0/web

  RewriteEngine On

  # Set a default PHP_FPM_PORT.
  RewriteRule ^ - [E=PHP_FPM_PORT:9000]

  # If the XDEBUG_TRIGGER cookie is set, use an php-fpm pool with xdebug enabled.
  RewriteCond %{HTTP_COOKIE} \XDEBUG_TRIGGER\b
  RewriteRule ^ - [E=PHP_FPM_PORT:9009]

  Timeout 90

  <FilesMatch "\.php$">
    SetHandler proxy:fcgi://localhost:%{ENV:PHP_FPM_PORT}
  </FilesMatch>
</VirtualHost>
```

Edit the line `Define webdir /Users/USERNAME/Sites/`, replacing `/Users/USERNAME/Sites/` with the directory where you want your websites to live. Don't forget the trailing slash.

This tells Apache to:

- Include some required Apache modules
- Configure [dynamic mass virtual hosts][] via the `VirtualDocumentRoot` directive.
- Use the `proxy_fcgi_module` Apache module to enable PHP for all sites via the `php-fpm` service.
- Send PHP requests to one of two `php-fpm` pools - one with xdebug disabled and one with it enabled. This is determined by the presence of the `XDEBUG_TRIGGER` cookie. See this [simple xdebug cookie webextension][] for a way to toggle that cookie for specific `.test` domains.

### Start Apache

To start the Apache daemon (`httpd`):

```
$ sudo apachectl start
```

Other options for `apachectl` are `stop`, `restart`, and `configtest`, which can be useful for troubleshooting changes to you Apache configuration files.

To start on system startup, you may need to do the following:

```
sudo launchctl load /System/Library/LaunchDaemons/org.apache.httpd.plist
```

## Test it out

1. Create a directory at `/Users/USERNAME/Sites/example.test/web/`. Ensure that `/Users/USERNAME/Sites/` is actually the value of the `webdir` variable set above.
2. Place a file in that directory called `index.php` with the contents `<?php phpinfo();`.
3. Add the line `127.0.0.1	example.test` to your `/etc/hosts` file.
4. Visit http://example.test in a browser.

You should see the output of [phpinfo][].

You can add as many sites as you want to your `webdir` (e.g. `/Users/USERNAME/Sites/`) and Apache will serve them without a restart.

In other words, Apache will serve anything from `/Users/USERNAME/Sites/*.test/web/` at `http://*.test`, as long as you edit your `/etc/hosts` file to resolve `*.test` to `127.0.0.1`.

## Bonus Stuff

### MariaDB

You'll probably need a MySQL-compatible database server, and MariaDB is a good choice. To install and enable it,
run:

```
brew install mariadb
brew services start mariadb
```

MariaDB is so much like mySQL that its command line interface is called `mysql`. By default it listens only to localhost, and its admin user is "root" with no password. Test the connection with this command:

```
mysql -u root
```

You should see a mySQL command prompt. Type `quit;` to quit.

### Dnsmasq

If you create many sites in your `webdir` and don't want to constantly modify `/etc/hosts`, you can install [dnsmasq][] and configure your system to resolve the `.test` top level domain to your localhost (127.0.0.1).

You can install and enable it with Homebrew:

```
$ brew install dnsmasq
$ sudo brew services start dnsmasq
```

Open `/usr/local/etc/dnsmasq.conf` and add the following line:

```
address=/.test/127.0.0.1
```

And restart dnsmasq:

```
$ sudo brew services restart dnsmasq
```

Test it with:

```
dig example.test @127.0.0.1
```

You should see something like the following:

```
;; ANSWER SECTION:
example.test.		0	IN	A	127.0.0.1
```

Now that it's working, we need to add your local dnsmasq DNS server to the system resolver.

Ensure that the `/etc/resolver/` directory exists:

```
sudo mkdir /etc/resolver
```

Add the file `/etc/resolver/test` with the following contents:

```
nameserver 127.0.0.1
```

You can test the new resolver via:

```
scutil --dns
```

You should see something like the following:

```
resolver #8
  domain   : test
  nameserver[0] : 127.0.0.1
  flags    : Request A records, Request AAAA records
  reach    : 0x00030002 (Reachable,Local Address,Directly Reachable Address)
```

At this point, `*.test` (e.g. `foo.test`, `whatever.test`, `myproject.test`) should resolve to `127.0.0.1` and you won't need to touch your `/etc/hosts` file whenever you add a new site to your `webdir`.


[Homebrew]: https://brew.sh/
[xdebug documentation]: https://xdebug.org/docs
[dynamic mass virtual hosts]: https://httpd.apache.org/docs/2.4/vhosts/mass.html
[simple xdebug cookie webextension]: https://github.com/jeffam/xdebug_trigger_extension
[phpinfo]: https://www.php.net/manual/en/function.phpinfo.php
[dnsmasq]: http://www.thekelleys.org.uk/dnsmasq/doc.html
