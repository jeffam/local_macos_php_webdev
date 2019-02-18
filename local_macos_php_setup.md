# Setting up a local development site
 
## Mac setup
 
One option for creating a local development environment on a Mac is to use
VirtualBox and Vagrant to create a VM in which your site can run. However,
keeping track of what's running inside the VM and what's running on the Mac
can be tricky, as can mapping files and ports between the two.
 
Here are instructions for installing a Drupal 8 development site without using
Vagrant or another VM system as of 4/6/2018.
 
### Install Homebrew
 
If you don't already have it installed, run this command (which is all on one
line):
 
```
/usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
```
 
Here are some useful Homebrew commands:
 
```
brew search STRING
```
 
Look for Homebrew packages with names that contain the string. Package names
that end with `@` and a number will install a specific version of the program.
 
```
brew install PACKAGENAME
```
 
Install the latest version of the package (or if `PACKAGENAM` includes `@`
and a version number, that version).
 
```
brew upgrade
```
 
Upgrade all the programs that were installed via Homebrew to the latest
versions.
 
Homebrew Services works with the macOS Launchctl Manager to run programs when
the system is started, which you'll want PHP, Apache, and MariaDB to do.
 
```
brew services list
```
 
Display a list of running programs that were installed via Homebrew.
 
```
brew services start PACKAGENAME
brew services stop PACKAGENAME
brew services restart PACKAGENAME
```
 
Start, stop, or restart one of those programs.
 
### Make sure git is up to date
 
Use which git to see what version of git you are running. If it's old, use
`brew install` git to update it. (As of this writing in April 2018, we are
using git version 2.17.) It should be installed in `/usr/local/bin/git`.
 
### Install PHP 7.1 via Homebrew
 
In order to get PHP into the picture, install a new version of it with
Homebrew:
 
```
brew install php@7.1
```
 
We are using PHP 7.1 currently, and this installs that version. The `php.ini`
and `php-fpm.ini` file can be found in `/usr/local/etc/php/7.1/`.
 
Start this version of PHP:
 
```
brew services start php@7.1
```
 
Use php -v to check that that's the version of PHP you are running. You'll
probably need to add these lines to your `~/.bash_profile` file:
 
```
export PATH="/usr/local/opt/php@7.1/bin:$PATH"
export PATH="/usr/local/opt/php@7.1/sbin:$PATH"
```
 
Set the user that PHP will run as to be you; it means that Drupal will run as
that user, too, and you'll have fewer permissions issues. Edit the
`/usr/local/etc/php/7.1/php-fpm.d/www.conf` file and look for these lines:
 
```
user = _www
group = _www
```
 
Change `_www` to your username (which you can find out with `whoami`).
 
Also tell PHP to listen on port 9071, which is what we'll tell Apache to
connect on. Find this line:
 
```
listen = 127.0.0.1:9000
```
 
Change 9000 to 9071.
 
### Install Apache
 
Apache comes with MacOS, but use Homebrew to install another one.
 
Stop running the existing Apache:
 
```
sudo launchctl unload -w /System/Library/LaunchDaemons/org.apache.httpd.plist 2>/dev/null
```
 
Install Apache via Homebrew instead:
 
```
brew install httpd
```
 
See
[macOS 10.13 High Sierra Apache Setup: Multiple PHP Versions](https://getgrav.org/blog/macos-sierra-apache-multiple-php-versions)
for details of configuring Apache on a Mac.
 
To start the Apache daemon (`httpd`) now and restart in at login:
 
```
brew services start httpd
```
 
Apache is listening on `http://localhost:8080`. See whether that URL displays
an Apache message saying that it can't find a site. (Yet.)
 
### Configure Apache
 
Apache's configuration files are in `/usr/local/etc/httpd`. To avoid having
your changes overwritten by updates to Apache, create a new directory called
`other` in `/usr/local/etc/httpd` to contain your configuration files. (You can
call it whatever you want; substitute the name you chose in the following
instructions.
 
Then edit `/usr/local/etc/httpd/httpd.conf` (the main Apache configuration
file) to include all files in the other directory. Add these lines at the end
of the `httpd.conf` file:
 
```
# Local settings
Include /usr/local/etc/httpd/other/*.conf
```
 
You may have to re-add these lines when Apache is updated.
 
Create a file in the `other` directory called whatever you want (maybe
`local.conf`). Paste these lines into the file:
 
```
  # Local settings
  # Listen on the standard web port
  Listen 80
 
  # Set a server name for logs
  ServerName localmac.test:80
 
  # Add Apache modules modules
  LoadModule vhost_alias_module lib/httpd/modules/mod_vhost_alias.so
  LoadModule actions_module lib/httpd/modules/mod_actions.so
  LoadModule proxy_module lib/httpd/modules/mod_proxy.so
  LoadModule proxy_fcgi_module lib/httpd/modules/mod_proxy_fcgi.so
  LoadModule rewrite_module lib/httpd/modules/mod_rewrite.so
 
  # The directory where all development sites live, with the pattern
  # ./hostname/docroot
  # where hostname is usually something like mysite.test.
  Define webdir /Users/jeff/Sites/
 
  <Directory ${webdir}>
    Options +Indexes -Multiviews +FollowSymLinks
    AllowOverride All
    Require all granted
  </Directory>
 
  <VirtualHost *:80>
    ServerAlias *
    VirtualDocumentRoot ${webdir}%0/docroot
 
    <FilesMatch "\.php$">
      Require all granted
      SetHandler proxy:fcgi://127.0.0.1:9071
    </FilesMatch>
  </VirtualHost>
```
 
Edit the line `Define webdir /Users/jeff/Sites/`, replacing
`/Users/jeff/Sites/` with the directory where you want your websites to live.
Don't forget the trailing slash.
 
This tells Apache to:
 
- listen on port 80 instead of 8080
- include some required Apache modules
- set your “webdir” to the directory where your sites will live
- look in all the subdirectories of that directory for a folder named docroot
- from which to serve the sites
- use the proxy:fcgi Apache module to get PHP into the mix, using the
- aforementioned port 9071.
 
### Update your /etc/hosts file
 
In the directory you defined as “webdir”, you will be able to create as many
directories as you want (or more likely, clone repositories of as many projects
as you want). Be sure that each directory ends with `.test`.
 
Edit your /etc/hosts file and add this line:
 
```
  127.0.0.1 test.test
```
 
You can add additional directory names for your various projects, like this:
 
```
  127.0.0.1 test.test foo.test
```
 
To test whether Apache is configured correctly, create a `test.test` directory
in your webdir and put an `index.html` file and an `index.php file` in there.
 
- Recommended contents of the former: `<h1>CoLab rocks!</h1>`
- Recommended contents of the latter: `<?php phpinfo(); `
 
Restart Apache and check whether it can see your `test.test/index.html` by
pointing your browser at `http://test.test`. If that works, point it at
`http://test.test/index.php` to see whether PHP is working with Apache.
 
### Install MariaDB
 
MariaDB is a replacement for mySQL; it works just the same but faster.
Platform.sh and other web hosts are switching to it. To install and run it,
run:
 
```
brew install mariadb
brew services start mariadb
```
 
MariaDB is so much like mySQL that its command line interface is called
`mysql`. By default it listens only to localhost, and its admin user is “root”
with no password. Test the connection with this command:
 
```
mysql -u root
```
 
You should see a mySQL command prompt. Type `quit;` to quit. (Don't forget the
semicolon.)
 
### Troubleshooting
 
If PHP runs out of memory, you can increase it in your
`/usr/local/etc/php/7.1/php.ini` file. Find this line:
 
```
  memory_limit = 128M
```
 
and change the number to 256M or more.