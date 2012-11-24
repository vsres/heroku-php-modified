Modified Apache/PHP build pack 
========================

This is a custom build pack for Heroku for our PHP API's. It is modified for short script timeout, small posts, high availability and PHP etensions. This is based on [heroku-buildpack-silex] (https://github.com/ingeting/heroku-buildpack-silex) which is based on [heroku-buildpack-php] (https://github.com/heroku/heroku-buildpack-php).

Compiling binaries
------------------

In order to create the packages used by the build pack, you must compile Apache, PHP and their extensions with the following steps.

    Mac OSX Terminal -> command prompt
    $ heroku run bash -a heroku-build-server

    # Bash shell is now running on a remote Heroku dyno (our build server) - the filesystem
    # is read-only except for the /app and /tmp folders.  We will install/compile everything within 
    # the /app folder (libraries, etc).  Then at the end, tar up the main folders
    # /app/apache, /app/php, /app/pear, /app/libmcrypt, etc and scp them to
    # a publically accessible location (S3, webserver, dropbox, etc)
    ~ $ curl -O http://www.us.apache.org/dist/httpd/httpd-2.2.23.tar.gz
    ~ $ tar -xzf httpd-2.2.23.tar.gz
    ~ $ cd httpd-2.2.23
    ~ $ ./configure --prefix=/app/apache --enable-rewrite --enable-ssl --enable-setenvif
    ~ $ make
    ~ $ make install
    ~ $ cd ..

    # Install libmcrypt libraries
    ~ $ curl -O https://dl.dropbox.com/u/13311057/libmcrypt-2.5.8.tar.gz
    ~ $ tar -xzf libmcrypt-2.5.8.tar.gz
    ~ $ cd libmcrypt-2.5.8
    ~ $ ./configure --prefix=/app/libmcrypt
    ~ $ make
    ~ $ make install
    

    # Copy compiled libmcrypt libraries to our app for packaging
    ~ $ mkdir /app/libmcrypt
    ~ $ cp /usr/local/lib/libmcrypt* /app/libmcrypt -R
    
    # Download and install PHP
    # (future option for 5.4 --with-vpx-dir=/usr/include )
    ~ $ curl -O https://dl.dropbox.com/u/13311057/php-5.3.19.tar.gz
    ~ $ tar -xzf php-5.3.19.tar.gz
    ~ $ ./configure --prefix=/app/php --with-apxs2=/app/apache/bin/apxs --with-mysql --with-mysqli --with-pdo-mysql --with-iconv --with-gd --with-curl=/usr/lib --with-config-file-path=/app/php --enable-soap=shared --with-zlib --with-jpeg-dir=/usr/include --with-readline --enable-cli --enable-pcntl --with-mcrypt=/app/libmcrypt/lib --enable-static=yes --with-openssl
    ~ $ make
    ~ $ make install
    ~ $ cd ..

    # PHP Extensions
    ~ $ mkdir /app/php/ext
    ~ $ cp /usr/lib/libmysqlclient.so.16 /app/php/ext/

    ~ $ curl -O https://dl.dropbox.com/u/13311057/go-pear.phar
    ~ $ php/bin/php go-pear.php  # (accept defaults)

    ~ $ export PATH=/usr/local/bin:/usr/bin:/bin:/app/php/bin:/app/apache/bin:/app/pear/bin

    ~ $ pear/bin/pear config-set php_dir /app/php
    ~ $ pear/bin/pecl install apc
    ~ $ pecl install memcache
    ~ $ mkdir /app/php/include/php/ext/apc
    ~ $ cp /app/php/lib/php/extensions/no-debug-non-zts-20090626/apc.so /app/php/ext/
    ~ $ cp /app/php/include/php/ext/apc/apc_serializer.h /app/php/ext
    ~ $ cp /app/php/lib/php/extensions/no-debug-non-zts-20090626/memcache.so /app/php/ext    

    # Create Heroku Build Pack (hbp) packages. Note that these files will be loaded by
    # our bin/compile script in our BuildPack in GIT.  Whenever we upload new files to 
    # our Heroku Dyno, the compile script runs, imports the package files, decompresses them
    # and rebuilds our apache/php/etc environment to essentially as it appeared in the 
    # above Bash Shell.  
    ~ $ cd /app
    ~ $ echo $APACHE_VERSION > apache/VERSION
    ~ $ echo $PHP_VERSION > php/VERSION
    ~ $ echo $LIBMCRYPT_VERSION > libmcrypt/VERSION
    ~ $ echo $PEAR_VERSION > pear/VERSION
    ~ $ tar -zcvf hbp-apache-$APACHE_VERSION.tar.gz apache
    ~ $ tar -zcvf hbp-php-$PHP_VERSION.tar.gz php
    ~ $ tar -zcvf hbp-libmcrypt-$LIBMCRYPT_VERSION.tar.gz libmcrypt
    ~ $ tar -zcvf hbp-pear-$PEAR_VERSION.tar.gz pear

    # COPY your hbp-*.tar.gz packages to a public avail location (S3, DropBox, etc)
    ~ $ scp hbp-.* user@somedomain:/~ #(provide password)

    # EXIT from the Heroku Shell
    ~ $ exit

Modifying Apache/PHP and/or new Libraries
-------

1. Open a terminal
2. Open a heroku bash shell like above
3. curl -O hbp-*.tar.gz in the /app folder
4. tar -xzf hbp-*.tar.gz to uncompress each of the packages
5. now install any new dependency libraries
6. rerun the ./configure on apache and/or php as needed (see above) adding new options as needed
7. once everything compiled, tar packages back up
8. upload new tar packages back to the publicly accessible location
9. modify the compile script (GIT Buildpack) with new version numbers (if they changed)

Hacking
-------

To change this buildpack, fork it on Github. Push up changes to your fork, then create a test app with --buildpack <your-github-url> and push to it.


Meta
----

Original build pack created by Pedro Belo. Composer support by Henri Bergius. Silex build pack by Klaus Silveira.
