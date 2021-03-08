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
