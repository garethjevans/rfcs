# Restructuring PHP Buildpacks

## Proposal

The RFC proposes a new set of smaller buildpacks that, by themselves provides
more limited functionality, but in composition provides a powerful way to build
various kinds of images related to PHP.

## Motivation

1. Reducing the responsibility of each buildpack will help to make them easier
   to understand and maintain. We will end up with more buildpacks, each with
   simpler implementations.

1. Align buildplan philosophy with our other buildpacks. We have moved away
   from the buildpack unilaterally marking its layers as `build` or `launch`.
   Buildpacks should interact with each other via meaningful `build` and
   `launch` requirements. This builds a clean separation between the concerns
   of building and running an application image.

In the current state of the world, we have:

- a [php-dist](https://github.com/paketo-buildpacks/php-dist) buildpack that provides
  php.
- a [php-composer](https://github.com/paketo-buildpacks/php-composer) that provides
  composer and installs dependencies.
- a [php-web](https://github.com/paketo-buildpacks/php-web) that takes care of
  everything related to serving php on a web server.

## Buildpacks

The PHP family of buildpacks aims to serve the following functions:
* Build an image to run HTTPD web server with php
* Build an image to run Nginx web server with php
* Build an image to run Built-in php web server
* Build an image to run FPM[<sup>1</sup>](#note-1) process manager
* Build an image to run a PHP CLI script using Procfile.
* Optionally use composer as application level package manager
* Optionally use memcached/redis as session handler[<sup>3</sup>](#note-3)

The new structure would include the following buildpacks in addition to the
existing Procfile, Apache HTTPD Server and Nginx Server
buildpacks[<sup>2</sup>](#note-2):

* **php-dist**:
  Installs the [`php`](https://www.php.net) distribution, making it available on the `$PATH`
  * provides: `php`
  * requires: none

  This distribution must include popular modules like memcached, mongodb,
  mysqli, openssl, phpiredis, zlib etc; and a suitable `php.ini`. Downstream
  buildpacks are expected to obtain info (like extension dir) through the
  [`php-config`](https://www.php.net/manual/en/install.pecl.php-config.php)
  executable whenever possible.

* **composer**:
  Installs [`composer`](https://getcomposer.org), a dependency manager for PHP
  and makes it available on the `$PATH`
  * provides: `composer`
  * requires: none

* **composer-install**:
  Resolves project dependencies, and installs them using `composer`.
  * provides: none
  * requires: `php`, `composer` at build

* **php-fpm**:
  Configures `php-fpm.conf` (config file in `php.ini` syntax), and sets a start
  command (type `php-fpm`) to start FPM[<sup>1</sup>](#note-1).
  * provides: `php-fpm`
  * requires: `php` at build and launch

  Separation of FPM into a separate buildpack lets users run FPM in one
  container and web server in another container.
  The generated `php-fpm.conf` must accept requests on a tcp/ip socket by
  default, must "include" custom fpm configs specified by the user (e.g. via
  env variables or specific location in the app).

* **php-builtin-server**:
  Set up PHP's [built-in web
  server](https://www.php.net/manual/en/features.commandline.webserver.php) to
  serve PHP applications.
  * provides: none
  * requires: `php` at launch

  This buildpack sets a start command (type `web`) to start the built-in web
  server. This is the default web server.

* **php-httpd**:
  Sets up HTTPD as the web server to serve PHP applications.
  * provides: none
  * requires: `php` at build; `php`, `php-fpm`, `httpd` at launch

  This buildpack generates `httpd.conf` and sets up a start command (type
  `web`) to run PHP FPM and HTTPD Server. Users need to declare the intention to
  use httpd.

* **php-nginx**:
  Sets up Nginx as the web server to serve PHP applications.
  * provides: none
  * requires: `php` at build; `php`, `php-fpm`, `nginx` at launch

  This buildpack generates `nginx.conf` and sets up a start command (type
  `web`) to run PHP FPM and Nginx Server. Users need to declare the intention to
  use nginx.

* **php-memcached-session-handler**:
  Configures the given memcached service instance as a PHP session
  handler[<sup>3</sup>](#note-3). Memcached settings are to be provided through
  a suitable
  [binding](https://paketo.io/docs/buildpacks/configuration/#bindings).
  * provides: none
  * requires: `php` at launch

* **php-redis-session-handler**:
  Configures the given redis service instance as a PHP session
  handler[<sup>3</sup>](#note-3). Redis settings are to be provided through a
  suitable
  [binding](https://paketo.io/docs/buildpacks/configuration/#bindings).
  * provides: none
  * requires: `php` at launch

This would result in the following order groupings in the PHP language family meta-buildpack:

```toml
[[order]] # HTTPD web server

  [[order.group]]
    id = "paketo-buildpacks/php-dist"
    version = ""

  [[order.group]]
    id = "paketo-buildpacks/composer"
    version = ""
    optional = true

  [[order.group]]
    id = "paketo-buildpacks/composer-install"
    version = ""
    optional = true

  [[order.group]]
    id = "paketo-buildpacks/httpd"
    version = ""

  [[order.group]]
    id = "paketo-buildpacks/php-fpm"
    version = ""

  [[order.group]]
    id = "paketo-buildpacks/php-httpd"
    version = ""

  [[order.group]]
    id = "paketo-buildpacks/php-memcached-session-handler"
    version = ""
    optional = true

  [[order.group]]
    id = "paketo-buildpacks/php-redis-session-handler"
    version = ""
    optional = true

  [[order.group]]
    id = "paketo-buildpacks/procfile"
    version = ""
    optional = true

[[order]] # Nginx web server

  [[order.group]]
    id = "paketo-buildpacks/php-dist"
    version = ""

  [[order.group]]
    id = "paketo-buildpacks/composer"
    version = ""
    optional = true

  [[order.group]]
    id = "paketo-buildpacks/composer-install"
    version = ""
    optional = true

  [[order.group]]
    id = "paketo-buildpacks/nginx"
    version = ""

  [[order.group]]
    id = "paketo-buildpacks/php-fpm"
    version = ""

  [[order.group]]
    id = "paketo-buildpacks/php-nginx"
    version = ""

  [[order.group]]
    id = "paketo-buildpacks/php-memcached-session-handler"
    version = ""
    optional = true

  [[order.group]]
    id = "paketo-buildpacks/php-redis-session-handler"
    version = ""
    optional = true

  [[order.group]]
    id = "paketo-buildpacks/procfile"
    version = ""
    optional = true

[[order]] # Built-in web server (web), FPM (php-fpm) & Script (Procfile)

  [[order.group]]
    id = "paketo-buildpacks/php-dist"
    version = ""

  [[order.group]]
    id = "paketo-buildpacks/composer"
    version = ""
    optional = true

  [[order.group]]
    id = "paketo-buildpacks/composer-install"
    version = ""
    optional = true

  [[order.group]]
    id = "paketo-buildpacks/php-fpm"
    version = ""

  [[order.group]]
    id = "paketo-buildpacks/php-builtin-server"
    version = ""

  [[order.group]]
    id = "paketo-buildpacks/php-memcached-session-handler"
    version = ""
    optional = true

  [[order.group]]
    id = "paketo-buildpacks/php-redis-session-handler"
    version = ""
    optional = true

  [[order.group]]
    id = "paketo-buildpacks/procfile"
    version = ""
    optional = true
```

## Environment Variables

* `php-fpm` buildpack must expose the location of the `php-fpm.conf` file through
  an environment variable available at build and launch.
* Session handler buildpacks must append to `PHP_INI_SCAN_DIR` to modify the
  search path so PHP will locate any .ini configuration files required by the
  handler to operate.

## Unresolved Questions and Bikeshedding

* ~~php-httpd/php-nginx also sets a start command for fpm that is also set by
  php-fpm. Does the spec allow an elegant way for start cmds set by 2
  buildpacks to run together?~~ No

* ~~Is buildpack.yml the best way to detect if the app should be run on
  builtin/httpd/nginx?~~ Consensus is to use env variables.

## Notes

<a name="note-1">1</a>. There are a few ways for adding support for PHP to a
web server – as a native web server module, using CGI, using FastCGI. FPM
(FastCGI Process Manager) is a FastCGI implementation for PHP, bundled with the
official PHP distribution since version 5.3.3

<a name="note-2">2</a>. Per the [Web Server Buidpack Subteam
RFC](https://github.com/paketo-buildpacks/rfcs/blob/master/accepted/0006-web-servers.md),
the Apache HTTPD Server and Nginx Server buildpacks are no more considered to
be part of the PHP family of buildpacks.

<a name="note-3">3</a>. The session handler is responsible for storing data
from PHP sessions. By default, PHP uses files but they have severe scalability
limitations. With external session handlers, multiple application nodes can
connect to a central data store.
