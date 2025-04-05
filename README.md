# 🚀 Laravel Docker Environment with Docker Compose

Este entorno contiene todo lo necesario para correr una aplicación Laravel con Docker: PHP, NGINX, MySQL, phpMyAdmin, Composer y Artisan. Ideal para desarrollo local.

---

## 🧱 Servicios Incluidos

| Servicio        | Descripción                                                 | Puerto         |
|-----------------|-------------------------------------------------------------|----------------|
| **NGINX**       | Servidor web para Laravel                                   | `8080:80`      |
| **PHP**         | PHP 8.2 FPM con extensiones necesarias                      | —              |
| **MySQL**       | Base de datos MySQL 8.0                                     | `3306:3306`    |
| **phpMyAdmin**  | Interfaz para MySQL                                         | `8090:80`      |
| **Composer**    | Instalador de dependencias PHP                              | —              |
| **Artisan**     | CLI de Laravel                                              | —              |

---

## 📁 Estructura del Proyecto

.
├── docker-compose.yml
├── dockerfiles/
│   ├── nginx.dockerfile
│   ├── php.dockerfile
│   └── composer.dockerfile
├── mysql/
│   ├── .env
│   └── data/
├── nginx/
│   └── default.conf
└── src/               ← Tu proyecto Laravel va aquí

---

## 📦 Contenido del docker-compose.yml

networks:
  laravel_network:

services:
  server:
    build:
      context: .
      dockerfile: dockerfiles/nginx.dockerfile
    ports:
      - 8080:80
    volumes:
      - ./src:/var/www/html
      - ./nginx/default.conf:/etc/nginx/conf.d/default.conf:ro
    depends_on:
      - php
      - mysql
    container_name: laravel_server
    networks:
      - laravel_network

  php:
    build:
      context: .
      dockerfile: dockerfiles/php.dockerfile
    volumes:
      - ./src:/var/www/html:delegated
    container_name: laravel_php
    networks:
      - laravel_network

  mysql:
    image: mysql:8.0.1
    restart: unless-stopped
    tty: true
    container_name: laravel_mysql
    env_file:
      - mysql/.env
    ports:
      - 3306:3306
    networks:
      - laravel_network
    volumes:
      - ./mysql/data:/var/lib/mysql

  phpmyadmin:
    image: phpmyadmin/phpmyadmin:latest
    restart: always
    container_name: laravel_phpmyadmin
    depends_on:
      - mysql
    ports:
      - 8090:80
    environment:
      - PMA_HOST=laravel_mysql
      - PMA_USER=root
      - PMA_PASSWORD=root.pa55
    networks:
      - laravel_network

  composer:
    build:
      context: .
      dockerfile: dockerfiles/composer.dockerfile
    volumes:
      - ./src:/var/www/html
    depends_on:
      - php
    networks:
      - laravel_network

  artisan:
    build:
      context: .
      dockerfile: dockerfiles/php.dockerfile
    volumes:
      - ./src:/var/www/html
    entrypoint: ["php", "/var/www/html/artisan"]
    networks:
      - laravel_network

---

## 🐘 dockerfiles/php.dockerfile

FROM php:8.2-fpm-alpine

WORKDIR /var/www/html
COPY src .

RUN apk add --no-cache mysql-client msmtp perl wget procps shadow libzip libpng libjpeg-turbo libwebp freetype icu
RUN apk add --no-cache --virtual build-essentials \
    icu-dev icu-libs zlib-dev g++ make automake autoconf libzip-dev \
    libpng-dev libwebp-dev libjpeg-turbo-dev freetype-dev && \
    docker-php-ext-configure gd --enable-gd --with-freetype --with-jpeg --with-webp && \
    docker-php-ext-install gd mysqli pdo_mysql intl bcmath opcache exif zip && \
    apk del build-essentials && rm -rf /usr/src/php*

RUN apk add --no-cache pcre-dev $PHPIZE_DEPS && \
    pecl install redis && \
    docker-php-ext-enable redis.so

RUN addgroup -g 1000 laravel && adduser -G laravel -g laravel -s /bin/sh -D laravel
RUN chown -R laravel /var/www/html
USER laravel

---

## 🌐 dockerfiles/nginx.dockerfile

FROM nginx:stable-alpine

WORKDIR /etc/nginx/conf.d
COPY nginx/default.conf .
RUN mv default.conf default.conf

WORKDIR /var/www/html
COPY src .

---

## 📦 dockerfiles/composer.dockerfile

FROM composer:latest

RUN addgroup -g 1000 laravel && adduser -G laravel -g laravel -s /bin/sh -D laravel
USER laravel

WORKDIR /var/www/html

ENTRYPOINT [ "composer", "--ignore-platform-reqs" ]