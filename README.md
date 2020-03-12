# ISLE PHP-NGINX BASE

A Docker base image which adds NGINX in a PHP image container then use [supervisord](http://supervisord.org/) as the process manager to run both services.

This image can be used for any php app but it has some php and nginx [confd](http://www.confd.io/) templates which can be initialized in the app image extending this one.

## Working with this image

Here is a pseudo [multi-stage](https://docs.docker.com/develop/develop-images/multistage-build/) `Dockerfile` which describes how you can use this image to build an image containing your codes. Ideally your php app code dependencies should be managed by [composer](https://getcomposer.org/).

```
ARG build_environment=prod
ARG code_dir=./codebase
# This image tag
ARG base_image_tag=latest
ARG composer_version=1.9.3

#
# Stage 1: PHP Dependencies
#
FROM composer:${composer_version} as composer-build
ARG code_dir
ARG build_environment
# Run composer install here with all the needed flags.
# You can also use build_environment to add a --no-dev flag
# when building for production deployment
WORKDIR /app
COPY ${code_dir}/composer.json ${code_dir}/composer.lock ./
RUN set -eux; \
  flags="--no-suggest --prefer-dist --no-interaction --ignore-platform-reqs"; \
  if [ "$build_environment" == "prod" ]; then \
    flags="${flags} --no-dev"; \
  fi; \
  composer install $flags

#
# Stage 2: Any node related dependencies can be build here. e.g. any css preprocessor build for a theme.
#
FROM node:${node_version}
ARG code_dir
ARG build_environment

#
# Stage 3: The base app/drupal
#
FROM <this-image-name>:${base_image_tag} as base
# Copy the needed artifacts from stage 1 and 2 in this stage

#
# Stage 4: The production setup
#
FROM base AS prod
ENV APP_ENV=prod

# Using the production php.ini
RUN mv ${PHP_INI_DIR}/php.ini-production ${PHP_INI_DIR}/php.ini

#
# Stage 5: The dev setup
#
FROM base AS dev

ENV APP_ENV=dev \
  DEBUG=true

# Install development tools.
RUN pecl install xdebug-2.7.1; \
  docker-php-ext-enable xdebug; \
  # Adding the dev php.ini
  mv ${PHP_INI_DIR}/php.ini-development ${PHP_INI_DIR}/php.ini

# Copy composer binary from official Composer image. Notice we didn't need composer for prod stage.
COPY --from=composer:1.9.3 /usr/bin/composer /usr/bin/composer
```

## Note

If you are planning to use the configurations files unde `config/confd` and stored in the base image under `/confd_templates`. You will need to:

- Move the files to `/etc/confd/conf.d` inside the app dockerfile:

```bash
RUN mkdir -p /etc/confd/conf.d /etc/confd/templates; \
    mv /confd_templates/nginx/conf.d/* /etc/confd/conf.d/; \
    mv /confd_templates/php/conf.d/* /etc/confd/conf.d/; \
    mv /confd_templates/nginx/templates/* /etc/confd/templates/; \
    mv /confd_templates/php/templates/* /etc/confd/templates/
```

- If this app is not drupal provide a default nginx virtual host configuration file template to replace the one at `/etc/confd/templates/vhost.conf.tmpl`.
