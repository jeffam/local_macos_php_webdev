# PHP Web Development Setup for macOS

This repo contains an Ansible playbook for the configuration of PHP web development for macOS.

Currently, it runs PHP and other services directly on macOS. This is done for simplicity and performance.

## Requirements

- Homebrew
- Ansible (`brew install ansible`)
- macOS 10+

## What is installed and configured

- Caddy 2.x web server
- PHP-FPM with 4 pools
  - PHP 7.x with xdebug enabled (port 9070)
  - PHP 7.x with xdebug disabled (port 9071)
  - PHP 8.x with xdebug enabled (port 9080)
  - PHP 8.x with xdebug disabled (port 9081)
- MariaDB, a MySQL-compatible database server
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
$ brew services start php@7.4
$ brew services start caddy
$ brew services start mariadb@10.4
$ brew services start mailhog
$ sudo brew services start dnsmasq
```

# Roadmap

- Install and manage `launchd` services automatically.
- Add a similar container-based setup for Docker or Podman. Caddy and dnsmasq could still run on the macOS host.
