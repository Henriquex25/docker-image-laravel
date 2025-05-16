ARG PHP_VERSION
FROM php:${PHP_VERSION:-8.3.2-fpm-bullseye}

## Argumentos para sincronizar permissões entre host e container
ARG WWWUSER=1000
ARG WWWGROUP=1000

## Versão da Lib do Redis para PHP
ARG REDIS_LIB_VERSION=6.2.0

## Versão do NODE
ARG NODE_VERSION=23

## Diretório da aplicação
ARG APP_DIR=/var/www

### Instalações básicas
RUN apt-get update -y && apt-get install -y --no-install-recommends \
    apt-utils \
    supervisor \
    zlib1g-dev \
    libzip-dev \
    unzip \
    libpng-dev \
    libpq-dev \
    libxml2-dev \
    libbrotli-dev \
    default-mysql-client \
    nginx \
    nano \
    git && \
    rm -rf /var/lib/apt/lists/*

# Instalando extensões do PHP
RUN docker-php-ext-install mysqli pdo pdo_mysql session xml intl zip simplexml pcntl gd

# Redis e Swoole
RUN pecl channel-update pecl.php.net \
    && pecl install redis-${REDIS_LIB_VERSION} \
    && docker-php-ext-enable redis

# RUN pecl install swoole && docker-php-ext-enable swoole

# Composer
RUN curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer

### Configuração de usuário/grupo para sincronizar com o host
RUN if [ ${WWWUSER} -ne 0 ]; then \
    usermod -u ${WWWUSER} www-data && \
    groupmod -g ${WWWGROUP} www-data; \
    fi

### Configurações do projeto
WORKDIR $APP_DIR

# Cria diretórios críticos e ajusta permissões ANTES de copiar o código
RUN mkdir -p storage bootstrap/cache && \
    chown -R ${WWWUSER}:${WWWGROUP} storage bootstrap/cache && \
    chown -R ${WWWUSER}:${WWWGROUP} /var/www/storage && \
    chmod -R 775 storage bootstrap/cache /var/www/storage

# Copia arquivos de configuração
COPY ./docker/supervisord/supervisord.conf /etc/supervisor/conf.d/supervisord.conf
COPY ./docker/php/extra-php.ini "$PHP_INI_DIR/conf.d/99_extra.ini"
COPY ./docker/nginx/server.conf /etc/nginx/sites-enabled/default
COPY ./docker/nginx/error.html /var/www/html/error.html

# Copia o código-fonte com permissões corretas
COPY --chown=www-data:www-data . .

# Configura o Git para aceitar o diretório (como root)
RUN if [ -d ".git" ]; then \
    git config --global --add safe.directory /var/www && \
    git config --global --add safe.directory '*' && \
    chown -R www-data:www-data /var/www; \
    fi

# Instala dependências como www-data
USER www-data
RUN composer install --ignore-platform-reqs --no-dev --no-interaction --prefer-dist --optimize-autoloader --no-progress

# Limpeza final (como root)
USER root
RUN apt-get clean && \
    rm -rf /var/lib/apt/lists/* && \
    chown -R www-data:www-data $APP_DIR && \
    find $APP_DIR -type f -exec chmod 664 {} \; && \
    find $APP_DIR -type d -exec chmod 775 {} \;

CMD ["/usr/bin/supervisord", "-c", "/etc/supervisor/supervisord.conf"]
