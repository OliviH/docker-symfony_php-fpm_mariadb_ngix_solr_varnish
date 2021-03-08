# DOCKER FOR SYMFONY

## Base de données

> BDD: sympho

> User: root

> mdp: example

## STRUCTURE

```bash
mkdir -p struct/php && mkdir struct/nginx && mkdir -p struct/logs/nginx && mkdir -p struct/varnish && mkdir symfony && touch struct/php/Dockerfile && touch struct/nginx/Dockerfile && touch struct/nginx/default.conf && touch struct/docker-compose.yml && touch struct/.env && touch struct/varnish/default.vcl
```

# FILES

> struct/docker-compose.yml

```yaml
version: '3'

services:

  db:
    container_name: "mariadb"
    image: mariadb
    networks:
      - netsymph
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: example

  adminer:
    container_name: "adminer"
    image: adminer
    networks:
      - netsymph
    depends_on: 
      - db
    restart: always
    ports:
      - 8091:8080

  solr:
    container_name: "solr"
    image: solr:slim
    restart: always
    ports:
      - 8983:8983
    networks:
      - netsymph

  php:
    container_name: "php-fpm"
    build:
      context: ./php
    depends_on:
      - db
    networks:
      - netsymph
    environment:
      - APP_ENV=${APP_ENV}
      - APP_SECRET=${APP_SECRET}
    volumes:
      - ${APP_FOLDER}:/var/www

  nginx:
    container_name: "nginx"
    build:
      context: ./nginx
    networks:
      - netsymph
    volumes:
      - ${APP_FOLDER}:/var/www
      - ./nginx/default.conf:/etc/nginx/nginx.conf
      - ./logs:/var/log
    depends_on:
      - php
#    ports:
#      - "8095:80"

  varnish:
    image: varnish
    restart: unless-stopped
    volumes:
      - ./varnish/default.vcl:/etc/varnish/default.vcl:ro
    tmpfs: /var/lib/varnish:exec
    networks:
      - netsymph
    ports:
      - 8090:80
    depends_on:
      - nginx

networks:
  netsymph:
    driver: bridge
```

> struct/.env

```
APP_FOLDER=../symfony
APP_ENV=dev
APP_SECRET=4b9fd56e9eb4e6f3c0935e8adb63751f1761173d4e99283dbfa29c202f930228
```

> struct/nginx/nginx.conf/Dockerfile

```
FROM nginx:alpine
WORKDIR /var/www
CMD ["nginx"]
EXPOSE 80
```

> struct/nginx/nginx.conf/default.conf

```
user nginx;
worker_processes  4;
daemon off;

error_log  /var/log/nginx/error.log warn;
pid        /var/run/nginx.pid;

events {
    worker_connections  1024;
}

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;
    access_log  /var/log/nginx/access.log;
    sendfile        on;
    keepalive_timeout  65;

    server {
        listen 80 default_server;
        listen [::]:80 default_server ipv6only=on;

        server_name localhost;
        root /var/www/public;
        index index.php index.html index.htm;

        location / {
            try_files $uri $uri/ /index.php$is_args$args;
        }

        location ~ \.php$ {
            try_files $uri /index.php =404;
            fastcgi_pass php-fpm:9000;
            fastcgi_index index.php;
            fastcgi_buffers 16 16k;
            fastcgi_buffer_size 32k;
            fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
            fastcgi_read_timeout 600;
            include fastcgi_params;
        }

        location ~ /\.ht {
            deny all;
        }
    }
}
```

> struct/php/Dockerfile

```
FROM php:8.0.0RC3-fpm-alpine
RUN apk --update --no-cache add git
RUN docker-php-ext-install pdo_mysql
COPY --from=composer /usr/bin/composer /usr/bin/composer
WORKDIR /var/www
CMD composer install ;  php-fpm
EXPOSE 9000
```

> struct/varnish/default.vcl

```
vcl 4.0;
backend default {
  .host = "nginx";
  .port = "80";
}

sub vcl_deliver {
  # Display hit/miss info
  if (obj.hits > 0) {
    set resp.http.V-Cache = "HIT";
  }
  else {
    set resp.http.V-Cache = "MISS";
  }
}
```

## Other commands

```bash 
docker-compose up -d --build
docker-compose ps
docker-compose down
docker-compose logs nginx
```

# To create sympfony skeleton

```bash
docker-compose run php composer create-project symfony/website-skeleton:"^4.4" /var/www
```

# To create database - see .env to determinate login, password and database name

```bash
docker-compose run php bin/console doctrine:database:create
```
# To create datas in database

```bash
docker-compose run php bin/console make:entity
```


# To use console

```bash
docker-compose run php bin/console

---

Symfony 4.4.19 (env: dev, debug: true)

Usage:
  command [options] [arguments]

Options:
  -h, --help            Display this help message
  -q, --quiet           Do not output any message
  -V, --version         Display this application version
      --ansi            Force ANSI output
      --no-ansi         Disable ANSI output
  -n, --no-interaction  Do not ask any interactive question
  -e, --env=ENV         The Environment name. [default: "dev"]
      --no-debug        Switches off debug mode.
  -v|vv|vvv, --verbose  Increase the verbosity of messages: 1 for normal output, 2 for more verbose output and 3 for debug

Available commands:
  about                                      Displays information about the current project
  help                                       Displays help for a command
  list                                       Lists commands
 assets
  assets:install                             Installs bundles web assets under a public directory
 cache
  cache:clear                                Clears the cache
  cache:pool:clear                           Clears cache pools
  cache:pool:delete                          Deletes an item from a cache pool
  cache:pool:list                            List available cache pools
  cache:pool:prune                           Prunes cache pools
  cache:warmup                               Warms up an empty cache
 config
  config:dump-reference                      Dumps the default configuration for an extension
 dbal
  dbal:run-sql                               Executes arbitrary SQL directly from the command line.
 debug
  debug:autowiring                           Lists classes/interfaces you can use for autowiring
  debug:config                               Dumps the current configuration for an extension
  debug:container                            Displays current services for an application
  debug:event-dispatcher                     Displays configured listeners for an application
  debug:form                                 Displays form type information
  debug:router                               Displays current routes for an application
  debug:translation                          Displays translation messages information
  debug:twig                                 Shows a list of twig functions, filters, globals and tests
 doctrine
  doctrine:cache:clear-collection-region     Clear a second-level cache collection region
  doctrine:cache:clear-entity-region         Clear a second-level cache entity region
  doctrine:cache:clear-metadata              Clears all metadata cache for an entity manager
  doctrine:cache:clear-query                 Clears all query cache for an entity manager
  doctrine:cache:clear-query-region          Clear a second-level cache query region
  doctrine:cache:clear-result                Clears result cache for an entity manager
  doctrine:database:create                   Creates the configured database
  doctrine:database:drop                     Drops the configured database
  doctrine:database:import                   Import SQL file(s) directly to Database.
  doctrine:ensure-production-settings        Verify that Doctrine is properly configured for a production environment
  doctrine:mapping:convert                   [orm:convert:mapping] Convert mapping information between supported formats
  doctrine:mapping:import                    Imports mapping information from an existing database
  doctrine:mapping:info                      
  doctrine:migrations:current                [current] Outputs the current version
  doctrine:migrations:diff                   [diff] Generate a migration by comparing your current database to your mapping information.
  doctrine:migrations:dump-schema            [dump-schema] Dump the schema for your database to a migration.
  doctrine:migrations:execute                [execute] Execute one or more migration versions up or down manually.
  doctrine:migrations:generate               [generate] Generate a blank migration class.
  doctrine:migrations:latest                 [latest] Outputs the latest version
  doctrine:migrations:list                   [list-migrations] Display a list of all available migrations and their status.
  doctrine:migrations:migrate                [migrate] Execute a migration to a specified version or the latest available version.
  doctrine:migrations:rollup                 [rollup] Rollup migrations by deleting all tracked versions and insert the one version that exists.
  doctrine:migrations:status                 [status] View the status of a set of migrations.
  doctrine:migrations:sync-metadata-storage  [sync-metadata-storage] Ensures that the metadata storage is at the latest version.
  doctrine:migrations:up-to-date             [up-to-date] Tells you if your schema is up-to-date.
  doctrine:migrations:version                [version] Manually add and delete migration versions from the version table.
  doctrine:query:dql                         Executes arbitrary DQL directly from the command line
  doctrine:query:sql                         Executes arbitrary SQL directly from the command line.
  doctrine:schema:create                     Executes (or dumps) the SQL needed to generate the database schema
  doctrine:schema:drop                       Executes (or dumps) the SQL needed to drop the current database schema
  doctrine:schema:update                     Executes (or dumps) the SQL needed to update the database schema to match the current mapping metadata
  doctrine:schema:validate                   Validate the mapping files
 lint
  lint:container                             Ensures that arguments injected into services match type declarations
  lint:twig                                  Lints a template and outputs encountered errors
  lint:xliff                                 Lints a XLIFF file and outputs encountered errors
  lint:yaml                                  Lints a file and outputs encountered errors
 make
  make:auth                                  Creates a Guard authenticator of different flavors
  make:command                               Creates a new console command class
  make:controller                            Creates a new controller class
  make:crud                                  Creates CRUD for Doctrine entity class
  make:docker:database                       Adds a database container to your docker-compose.yaml file
  make:entity                                Creates or updates a Doctrine entity class, and optionally an API Platform resource
  make:fixtures                              Creates a new class to load Doctrine fixtures
  make:form                                  Creates a new form class
  make:message                               Creates a new message and handler
  make:messenger-middleware                  Creates a new messenger middleware
  make:migration                             Creates a new migration based on database changes
  make:registration-form                     Creates a new registration form system
  make:reset-password                        Create controller, entity, and repositories for use with symfonycasts/reset-password-bundle
  make:serializer:encoder                    Creates a new serializer encoder class
  make:serializer:normalizer                 Creates a new serializer normalizer class
  make:subscriber                            Creates a new event subscriber class
  make:test                                  [make:unit-test|make:functional-test] Creates a new test class
  make:twig-extension                        Creates a new Twig extension class
  make:user                                  Creates a new security user class
  make:validator                             Creates a new validator and constraint class
  make:voter                                 Creates a new security voter class
 router
  router:match                               Helps debug routes by simulating a path info match
 secrets
  secrets:decrypt-to-local                   Decrypts all secrets and stores them in the local vault.
  secrets:encrypt-from-local                 Encrypts all local secrets to the vault.
  secrets:generate-keys                      Generates new encryption keys.
  secrets:list                               Lists all secrets.
  secrets:remove                             Removes a secret from the vault.
  secrets:set                                Sets a secret in the vault.
 security
  security:encode-password                   Encodes a password.
 server
  server:dump                                Starts a dump server that collects and displays dumps in a single place
  server:log                                 Starts a log server that displays logs in real time
 translation
  translation:update                         Updates the translation file
```

