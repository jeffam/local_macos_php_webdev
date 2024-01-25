# PHP Web Development Setup for macOS

This repo contains an Ansible playbook for the configuration of PHP web development for macOS.

Currently, it runs PHP and other services directly on macOS. This is done for simplicity and performance.

## Requirements

- Homebrew
- Ansible (`brew install ansible`)
- macOS 10+

## What is installed and configured

- Caddy 2.x web server
- PHP-FPM with 8 pools
  - PHP 7.4 with xdebug enabled (port 9740)
  - PHP 7.4 with xdebug disabled (port 9741)
  - PHP 8.1 with xdebug enabled (port 9810)
  - PHP 8.1 with xdebug disabled (port 9081)
  - PHP 8.2 with xdebug enabled (port 9820)
  - PHP 8.2 with xdebug disabled (port 9821)
  - PHP 8.3 with xdebug enabled (port 9830)
  - PHP 8.3 with xdebug disabled (port 9831)
- pecl/apcu
- MariaDB, a MySQL-compatibledatabase server
- Dnsmasq for automatic `.test` TLD that resolves to localhost
- Mailhog for local email debugging and testing

## Running the Ansible playbook

Run the following to install both the above software and related configuration files:

```
$ ansible-playbook main.yml
```

You can run the playbook multiple times, if new versions of the required software are available, they will be automatically updated. 

If you get an error about a missing `community.general` collection, run the following:

```
$ ansible-galaxy install -r requirements.yml
```

## Starting the services

This is left as a manual task at present, so that the configured software can be enabled or disabled as desired.

```
$ brew services start php
$ brew services start php@8.2
$ brew services start php@8.1
$ brew services start php@7.4
$ brew services start caddy
$ brew services start mariadb@10.11
$ brew services start mailhog
$ sudo brew services start dnsmasq
```

# Configuring sites

By default, Caddy will use PHP 8.2 and look for sites in:

```
/Users/$USER/sites/{host}/web/
```

For example, a file in the following location:

```
/Users/jeff/sites/demo.test/web/info.php
```

...would be viewed in a browser at http://demo.test/info.php

For this to work, `dnsmasq` should be running and configured to resolve all `.test` domains to `127.0.0.1`. Alternatively, hostnames could be added to `/etc/hosts` if `dnsmasq` is not used.

## Using different PHP versions or otherwise modifying Caddy config

Additional configuration can be added to `/usr/local/etc/Caddy.conf.d/`. Any files ending in `.conf` will be loaded into config.

### Example: Run a site in PHP 8.3

```
http://php-8.3.test {
	@xdebug header_regexp Cookie XDEBUG_TRIGGER
	root * /Users/jeff/sites/{host}/web
	encode gzip
	php_fastcgi @xdebug 127.0.0.1:9831
	php_fastcgi 127.0.0.1:9830
	file_server
	handle_errors {
		respond "{http.error.status_code} {http.error.status_text}"
	}
}
```

Note the `php_*` lines and the selected port numbers.

## Using xdebug

This configuration sets up two `php-fpm` pools for each version of PHP: One with `xdebug` enable and the other without. This is done because `xdebug` can degrade PHP performance.

Caddy is configured to switch to the`php-fpm` pool with `xdebug` enabled when the `XDEBUG_TRIGGER` cookie is present.

You can use your browser dev tools to set the cookie (a value of `1` is fine). Or you can install a browser extension to toggle the cookie, if you are comfortable with the requested permissions. Search for 'xdebug browser extension'.

# Roadmap

- Install and manage `launchd` services automatically.
- Add a similar container-based setup for Docker or Podman. Caddy and dnsmasq could still run on the macOS host.
