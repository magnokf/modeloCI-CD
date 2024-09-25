# Use a imagem oficial do PHP com Apache
FROM php:8.1-apache

ENV DEBIAN_FRONTEND=noninteractive
ENV TZ=America/Sao_Paulo

RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone

# Atualize o sistema e instale as dependências necessárias
RUN apt-get update && apt-get install -y --no-install-recommends \
    tzdata \
    bash \
    supervisor \
    postgresql-server-dev-all \
    libfreetype6-dev \
    libjpeg62-turbo-dev \
    libwebp-dev \
    libxpm-dev \
    libxml2-dev \
    libzip-dev \
    libonig-dev \
    unzip \
    git \
    curl \
    libpng-dev \
    libpq-dev \
    libicu-dev \
    libldap2-dev \
    && docker-php-ext-configure gd --with-freetype --with-jpeg --with-webp \
    && docker-php-ext-install -j$(nproc) \
        pgsql \
        pdo_pgsql \
        pdo_mysql \
        mbstring \
        exif \
        pcntl \
        bcmath \
        gd \
        sockets \
        zip \
    && pecl install redis \
    && docker-php-ext-enable redis \
    && apt-get clean && rm -rf /var/lib/apt/lists/*

# Instale extensões adicionais do PHP
RUN docker-php-ext-install -j$(nproc) soap ldap intl

# Instale o Composer (copiado da imagem oficial do Composer)
COPY --from=composer:latest /usr/bin/composer /usr/bin/composer

# Defina o diretório de trabalho
WORKDIR /var/www/html

# Copie os arquivos da aplicação para o diretório de trabalho
COPY . /var/www/html

# Copie o arquivo .env da pasta development para o diretório da aplicação
COPY development/.env /var/www/html/.env

# Instale as dependências do Composer
RUN composer install --no-dev --no-interaction --no-progress --optimize-autoloader

# Gere a chave de criptografia do Laravel
RUN php artisan key:generate

# Crie o diretório de logs do Laravel
RUN mkdir -p storage/logs

# Crie o diretório de cache do Laravel
RUN mkdir -p bootstrap/cache

# Crie o diretório de armazenamento do Laravel
RUN mkdir -p storage/app

# Crie o diretório de armazenamento temporário do Laravel
RUN mkdir -p storage/framework/sessions

# Crie o diretório de armazenamento temporário do Laravel
RUN mkdir -p storage/framework/views

# Crie o diretório de armazenamento temporário do Laravel
RUN mkdir -p storage/framework/cache

# Crie o diretório de armazenamento temporário do Laravel
RUN mkdir -p storage/framework/cache/data

# Defina as permissões corretas para o diretório de trabalho
RUN chown -R www-data:www-data /var/www/html \
    && chmod -R 755 /var/www/html/storage \
    && chmod -R 755 /var/www/html/bootstrap/cache

# Copie o arquivo de configuração do Apache
COPY apache2.conf /etc/apache2/apache2.conf

# Ative o módulo de reescrita do Apache
RUN a2enmod rewrite

# Copie a configuração do Supervisord
COPY supervisord.conf /etc/supervisor/conf.d/supervisord.conf

# Exponha a porta 80 para o Apache
EXPOSE 80

# Inicie o Supervisord para gerenciar o Apache e outros processos
CMD ["/usr/bin/supervisord", "-c", "/etc/supervisor/conf.d/supervisord.conf"]
