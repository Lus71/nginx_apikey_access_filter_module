nginx_apikey_access_filter_module
=================================

Nginx filter to restrict the access at your backend APIs

Status
======

This module is under active development.

Synopsis
========

      # let's filter all the requests to our backend API s(i.e. a tornado web app)
      location ~* \.do$ {

         # enable the apikey access filter
         apikey_access_filter on;

         # set the connection string for the client_id / client_secret provider
         apikey_access_filter_dburl "mysql://apikeyadmin:apikeyadmin@127.0.0.1:3306/APIKEYS";

         # common reverse proxy config (i.e. for tornado)
         proxy_pass_header Server;
         proxy_set_header Host $http_host;
         proxy_redirect off;
         proxy_set_header X-Real-IP $remote_addr;
         proxy_set_header X-Scheme $scheme;
         proxy_pass http://backends;
      }


Description
===========

This module could be useful to anyone who develops backends APIs wich are consumed by Javascript, iOS, Android and other client apps. Rather then implements access controls for each backend, you can enble this filter at the relative location in your nginx configuration file.

Every time a client app call one of yours backend API, has to pass a token (named `X-AuthAPI`):

    client_id|issued_time|HAMAC_SHA1(client_id|request_uri_path, client_secret)

Where `client_id` and `client_secret` are strings generated as you desire. The `client_secret` will be kept safe in your own database.

Each client application that have to call your API has the `client_id` and a`client_secret`; when issue the request, has to pass this token. The token can be passed as Request Header or as Cookie, the module will check cookies first and then request headers.

Here an example. The client application has:

    client_id     = LUCACLIENTID1971
    client_secret = MyCl13n7S3Cr377001233aaER

and wants to call the backend API `http://your.cool.domain.com/get_all_photos.do?q=cat&max_results=40`; let's suppose that the `issued_time = 1335104889` in unix epoch time, so the token will be:

    LUCACLIENTID1971|1335104889|f0679855e23b2c338a41491c0b8e0576cc8397dd

Here the http request using wget:

     wget --header="X-AuthAPI:LUCACLIENTID1971|1335104889|f0679855e23b2c338a41491c0b8e0576cc8397dd" \
         -O - "http://your.cool.domain.com/get_all_photos.do?q=cat&max_results=40"

If the token is invalid (tampered, expired etc.) this module will repond with an ERROR 403: Forbidden otherwise the request will be passed to the desired url.

Actually the module needs a MySQL database as client id / client secret provider, but I'd like to use [Redis](http://redis.io/). Redis is asynchronous while opening and closing the MySQL connection is a blocking operation. If someone would like to help me..That's will be cool! :-)

Install
=======
 
 * Download the latest version of this module [HERE](https://github.com/Lus71/nginx_apikey_access_filter_module/zipball/master).
 * Download the latest version of Nginx [HERE](http://nginx.org/)

Build the source with this module:

    wget 'http://nginx.org/download/nginx-1.0.15.tar.gz'
    tar -xzvf nginx-1.0.15.tar.gz
    cd nginx-1.0.15/

    # Here we assume Nginx is to be installed under /opt/nginx/.
    ./configure --prefix=/opt/nginx \
        --add-module=/path/to/nginx_apikey_access_filter_module

    make

    make install

Create the MySQL database holding the client_id / client_secret pairs:

    CREATE DATABASE IF NOT EXISTS APIKEYS DEFAULT CHARACTER SET utf8;
    GRANT CREATE,INSERT,DELETE,UPDATE,SELECT on APIKEYS.* to apikeyadmin@localhost IDENTIFIED BY 'apikeyadmin';

    DROP TABLE IF EXISTS `API_USER`;

    CREATE TABLE `API_USER` (
      `id` varchar(40) NOT NULL ,
      `secret` varchar(80) NOT NULL,
      `active` tinyint(3) unsigned NOT NULL DEFAULT '1',
      `created` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
      PRIMARY KEY (`id`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8;

Insert some values; for example:

    INSERT INTO API_USER (id, secret) VALUES ('LUCACLIENTID1971', 'MyCl13n7S3Cr377001233aaER');

Add the directives to your location in your nginx configuration file (see Synopsis).

TODO
====

 * Adding an `expire time` directive (so the API token will have an expiration time)

 * Replacing MySQL with Redis


