# PHP docker image

Can be used for any PHP application inherit official [yii2-docker](https://github.com/yiisoft/yii2-docker) and based on the PHP Apache Debian version. With configuration made to be compatible with [openshift security policy](https://docs.openshift.com/container-platform/3.11/creating_images/guidelines.html). 

If you wan't to do some modification in the image, you `Dockerfile` should look something like that:

```
FROM linkbn/php-openshift:X.X-apache
ARG USER_ID=2000
USER root
COPY src/ /app/
RUN /docker-bin/docker-build.sh
USER ${USER_ID}
```

## Entry-point specificity

Entry-point script can run:

* **Apache HTTPD** server (default behavior if no command or args)
* **cron daemon** with environment variable properly setup
* [Yii CLI application](https://www.yiiframework.com/doc/guide/2.0/en/tutorial-console) with your custom arguments
* `bash`, `php` and `composer` command are also allowed
* `loop` execute the command after 

The entry-point script is also providing those helpers:

## Wait for a list of service availability

Before running your service you may be need to wait for other to be up and listening (example wait for you database server to be up and running on port 3306). You can provide the environment variable `WAIT_FOR_IT_LIST` with the list of service to test before starting up the application.

If you want to wait for a mysql server on port 3306 and an SMTP server on port 25, just do:

```
WAIT_FOR_IT_LIST=mysql:3306,smtp:25
```

## Image Configuration at buildtime

With docker build arguments (`docker build --build-arg VAR_NAME=VALUE`), if you wan't to change some of them you will need to run as root in your Dockerfile inheriting from that image the script `/docker-bin/docker-build.sh`.

### System configuration (buildtime)

* **USER_ID**: Id of the user that will run the container (default: `2000`)
* **USER_HOME**: Home diretory of the user defined by `USER_ID` (default: `/home/user`)
* **TZ**: System timezone will be used for cron and logs (default: `Europe/Paris`, done by `docker-build.sh`)

### Apache HTTPD configuration  (buildtime)

Log format by default is `combined` on container stdout and apache is listening on port 8080 (http) or 8443 (https). Document root of Apache is `${APP_DIR}/web`.

* **remoteip**: By default remoteip configuration is enabled, see runtime part of the documentation to configure it.
* **serve-cgi-bin**: Is disabled by default.
* **syslog**: You can enable Apache HTTPD loging to syslog, using `a2enconf syslog` in your docker buld.

### Cron configuration (buildtime)

We're using [supercronic](https://github.com/aptible/supercronic) as cron dameon. You can put your cronfile in:
*  `/etc/cron.d/` in the normal cron format '`minute` `hour` `day of month` `month` `day of week` `user` (NB: user will not be taken in consideration in our cron is not run as root) will be merge by `docker-build.sh` script at build time.
* or create the file `/etc/crontab` in [supercronic supported format](https://github.com/gorhill/cronexpr).

### Php configuration (buildtime)

List of already embed modules (the one with a (`*`) are loaded by default):

* bcmath (`*`)
* exif
* gd
* gearman
* gmp (`*`)
* imagick
* intl (`*`)
* mongodb
* pcntl
* pdo_mysql
* pdo_pgsql
* soap
* sockets
* sodium (`*`)
* yaml (`*`)
* xdebug
* Zend OPcache (`*`)
* zip

If you want for your specific application to enable one of them just do:

```
docker-php-ext-enable extension-name
```

If the module you need is missing you can just add them in your `Dockerfile`, see [php docker](https://hub.docker.com/_/php/) image documentation for "[How to install more PHP extensions](https://github.com/docker-library/docs/blob/master/php/README.md#how-to-install-more-php-extensions)".

* **PHP_VERSION**: Version of php used to do the build (default: `latest`).
* **PHP_OPCACHE_MAX_ACCELERATED_FILES_DEFAULT**: 

### Application configuration (buildtime)

The `docker-build.sh` script will (if `APP_DIR` is not empty):
* Remove the `.git*` file and directories,
* Set execution right to shell script inside `tests` directory if it exist.
* Give read right on files and directories inside `APP_DIR`,
* If there is `composer.json` file, composer will be run,
* Define a value for `PHP_OPCACHE_MAX_ACCELERATED_FILES_DEFAULT` based on the number of php file.

* **APP_DIR**: You PHP application should be installed in that directory (default: `/app`).
* **APP_WRITE_DIRECTORIES**: List of directory in which the application need to be able to write (default: `runtime web/assets web/runtime tests/_output tests/_support/_generated`).

### Documentation Generation configuration (buildtime)

You can generate a static documentation with [daux.io](https://github.com/dauxio/daux.io) to be include in your application, if you've got a documentation directory inside your code.

* **DOC_GENERATE**: Do you want to generate the static doc (default: `yes`).
* **DOC_DIR_SRC**: Source directory of your documentation inside the `/app/` (default: `docs`).
* **DOC_DIR_DST**: Destination directory of the generated documentation in `/app/web/` (default: `doc`).

### Composer configuration (buildtime)

* **COMPOSER_DEV**: Install composer development dependencies (default: `no`, done by `docker-build.sh`).

### Yii configuration (buildtime)

* **YII_ENV**: if set at buildtime and is set to `test` or `dev` **COMPOSER_DEV** will be set to `yes`, so composer development dependencies are installed (`gii`, `codeception`, ...).







## Image Configuration at runtime

With environment variables (`docker run  -e VAR_NAME=VALUE`).

### System configuration (runtime)

* **USER_NAME**: Name of the user that will run the container will have the id defined by **USER_ID** and home defined by **USER_HOME** (default: `default`)

### Apache HTTPD configuration (runtime)

* **APACHE_RUN_USER**: Username of the user that will run apache (default: `$USER_NAME`).
* **APACHE_SERVER_NAME**: Set Apache ServerName (default: `__default__`).
* **APACHE_REMOTE_IP_HEADER**: Set `RemoteIPHeader` directive of the [remote_ip module](https://httpd.apache.org/docs/trunk/mod/mod_remoteip.html) (default: `X-Forwarded-For`)
* **APACHE_REMOTE_IP_TRUSTED_PROXY**: Set `RemoteIPtrustedProxy` directive of the [remote_ip module](https://httpd.apache.org/docs/trunk/mod/mod_remoteip.html) (default: `10.0.0.0/8 172.16.0.0/12 192.168.0.0/16`)
* **APACHE_REMOTE_IP_INTERNAL_PROXY**: Set `RemoteIPInternalProxy` directive of the [remote_ip module](https://httpd.apache.org/docs/trunk/mod/mod_remoteip.html) (default: `10.0.0.0/8 172.16.0.0/12 192.168.0.0/16`)

#### Apache HTTPD syslog configuration

Will be use only if you add `a2enconf syslog` in your `Dockerfile`.

* **APACHE_SYSLOG_HOST**: Ip or dns of the UDP syslog server (default: `none`).
* **APACHE_SYSLOG_PORT**: Port of syslog server (default: `514`).
* **APACHE_PROGRAM_NAME**: Value of logsource field in syslog (default: `httpd`).

### Cron configuration (runtime)

* **CRON_DEBUG**: Enable debug mode of [supercronic](https://github.com/aptible/supercronic).

### PHP configuration (runtime)

You can enable extension at runtime with the environment variable `PHP_ENABLE_EXTENSION`, you can provide a list of extension by separating them with with comma (`,`).

```
PHP_ENABLE_EXTENSION=gd,exif
```

You can override some PHP configuration setting by defining the following environment variable:

#### General configuration

* **PHP_TIMEZONE**: [date.timezone](http://php.net/manual/en/datetime.configuration.php#ini.date.timezone) (default: `$TZ`)
* **PHP_UPLOAD_MAX_FILESIZE**: [upload_max_filesize](http://php.net/manual/en/ini.core.php#ini.upload-max-filesize) (default: `2m`)
* **PHP_POST_MAX_SIZE**: [post_max_size](http://php.net/manual/en/ini.core.php#ini.post-max-size) (default: `8m`)
* **PHP_MAX_EXECUTION_TIME**: [max_execution_time](http://php.net/manual/en/info.configuration.php#ini.max-execution-time) (default: `30`)
* **PHP_MEMORY_LIMIT**: [memory_limit](http://php.net/manual/en/ini.core.php#ini.memory-limit) (default: `64m`)
* **PHP_REALPATH_CACHE_SIZE**: [realpath_cache_size](http://php.net/manual/en/ini.core.php#ini.realpath-cache-size) (default: `256k`)
* **PHP_REALPATH_CACHE_TTL**: [realpath_cache_ttl](http://php.net/manual/en/ini.core.php#ini.realpath-cache-ttl) (default: `3600`)

#### Opcache configuration

* **PHP_OPCACHE_ENABLE**: [opcache.enable](http://php.net/manual/en/opcache.configuration.php#ini.opcache.enable) enable Opcache (default: `1` = On)
* **PHP_OPCACHE_ENABLE_CLI**: [opcache.enable_cli](http://php.net/manual/en/opcache.configuration.php#ini.opcache.enable-cli) enable opcache for PHP in CLI (default: `1` = On)
* **PHP_OPCACHE_MEMORY**: [opcache.memory_consumption](http://php.net/manual/en/opcache.configuration.php#ini.opcache.memory-consumption) (default: `64m`)
* **PHP_OPCACHE_VALIDATE_TIMESTAMP**: [opcache.validate_timestamps](http://php.net/manual/en/opcache.configuration.php#ini.opcache.validate-timestamps) (default: `0` = Off)
* **PHP_OPCACHE_REVALIDATE_FREQ**: [opcache.revalidate_freq](http://php.net/manual/en/opcache.configuration.php#ini.opcache.revalidate-freq) (default: `600` seconds)
* **PHP_OPCACHE_MAX_ACCELERATED_FILES**: [opcache.max_accelerated_files](http://php.net/manual/en/opcache.configuration.php#ini.opcache.max-accelerated-files) (default: `PHP_OPCACHE_MAX_ACCELERATED_FILES_DEFAULT` calculate at buildtime by `docker-build.sh`)

### Yii configuration (runtime)

* **YII_DB_MIGRATE**: Enable database schema migration at container start up if set to `true` for more detail refer to [Yii2 databse migration](https://www.yiiframework.com/doc/guide/2.0/en/db-migrations).
* **YII_RBAC_MIGRATE**: Eanble creation/update of your list of static role and permission on your authManager at container start up if set to `true` for more detail refer to [macfly/yii2-rbac-cli](https://github.com/marty-macfly/yii2-rbac-cli).
