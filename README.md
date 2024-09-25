# Tutorial: Preparando uma Aplicação Laravel com Redis para Ambientes de Desenvolvimento e Homologação Usando GitLab CI/CD

## 1. Estrutura do Arquivo `.gitlab-ci.yml`

Este tutorial seguirá um processo adaptado para Laravel e Redis. A esteira inclui as seguintes fases:

1. **Build**: Instalação das dependências e preparação do ambiente.
2. **Testes (opcional)**: Rodar testes unitários e de integração.
3. **Deploy para desenvolvimento**: Realiza o deploy no ambiente de desenvolvimento usando ArgoCD.
4. **Deploy para homologação**: Realiza o deploy no ambiente de homologação usando ArgoCD.

---

## 2. Preparação do Ambiente de Desenvolvimento

### 2.1 Configuração de Variáveis de Ambiente no GitLab

Você precisará definir as variáveis de ambiente para Laravel e Redis no GitLab. Para isso:

1. Acesse **Settings > CI/CD > Variables** no seu repositório GitLab.
2. Adicione as seguintes variáveis:

   - `APP_ENV=development`
   - `APP_KEY`: A chave de criptografia do Laravel.
   - `DB_CONNECTION`: Ex.: `pgsql` ou `mysql`.
   - `DB_HOST`: O endereço do banco de dados de desenvolvimento.
   - `DB_PORT`: A porta usada pelo banco de dados.
   - `DB_DATABASE`: Nome do banco de dados.
   - `DB_USERNAME`: Nome de usuário do banco de dados.
   - `DB_PASSWORD`: Senha do banco de dados.
   - `REDIS_HOST`: Endereço do servidor Redis.
   - `REDIS_PASSWORD`: Senha do Redis (se houver).
   - `CI_REGISTRY_USER` e `CI_REGISTRY_PASSWORD`: Para autenticação no Docker Registry.
   - `ARGOCD_DEV_SERVER`: Endereço do servidor ArgoCD para o ambiente de desenvolvimento.
   - `ARGOCD_USERNAME`: Nome de usuário para autenticação no ArgoCD.
   - `ARGOCD_PASSWORD`: Senha para autenticação no ArgoCD.
   - `ARGOCD_STAGING_SERVER`: Endereço do servidor ArgoCD para o ambiente de homologação.

Certifique-se de que as variáveis de ambiente estejam configuradas corretamente para que o pipeline funcione corretamente.

### 2.2 Estrutura Básica do Arquivo `.gitlab-ci.yml`

Crie um arquivo `.gitlab-ci.yml` no diretório raiz do seu projeto Laravel com a seguinte estrutura básica:

```yaml
variables:
  CONTAINER_NAME: laravel-app
  DOCKER_DRIVER: overlay2
  PHP_VERSION: "8.1"
  REDIS_HOST: redis

stages:
  - build
  - test
  - deploy-dev
  - deploy-staging

cache:
  paths:
    - vendor/

services:
  - redis

before_script:
  - cp .env.example .env
  - composer install --no-interaction --prefer-dist --optimize-autoloader
  - php artisan key:generate
  - php artisan migrate

docker-build:
  stage: build
  image: php:${PHP_VERSION}-cli
  script:
    - docker build -t $CI_REGISTRY_IMAGE/$CONTAINER_NAME .
    - docker push $CI_REGISTRY_IMAGE/$CONTAINER_NAME
  tags:
    - docker
  only:
    - develop

test:
  stage: test
  image: php:${PHP_VERSION}-cli
  services:
    - mysql
  script:
    - cp .env.example .env
    - |
      echo "DB_CONNECTION=sqlite" >> .env
      echo "DB_DATABASE=:memory:" >> .env
    - composer install --no-interaction --prefer-dist
    - php artisan key:generate
    - php artisan migrate --env=testing --force
    - php artisan test
  tags:
    - docker
  only:
    - develop

deployment-dev:
  stage: deploy-dev
  image: argoproj/argocd
  script:
    - argocd --grpc-web login "${ARGOCD_DEV_SERVER}" --insecure --username "${ARGOCD_USERNAME}" --password "${ARGOCD_PASSWORD}"
    - argocd --grpc-web app actions run laravel-app restart --kind Deployment
  tags:
    - docker
  only:
    - develop

deployment-staging:
  stage: deploy-staging
  image: argoproj/argocd
  script:
    - argocd --grpc-web login "${ARGOCD_STAGING_SERVER}" --insecure --username "${ARGOCD_USERNAME}" --password "${ARGOCD_PASSWORD}"
    - argocd --grpc-web app actions run laravel-app restart --kind Deployment
  tags:
    - docker
  only:
    - staging
```

### 2.3 Build da Aplicação (docker-build)

O job `docker-build` será responsável por:

1. Construir a imagem Docker da aplicação Laravel.
2. Enviar essa imagem para o Docker Registry.

Aqui, utilizamos o PHP 8.1 como base, mas isso pode ser ajustado de acordo com a versão utilizada no seu projeto.

### 2.4 Melhorias sugeridas no Dockerfile

Para construir a imagem Docker da aplicação Laravel, você precisará de um arquivo `Dockerfile` no diretório raiz do seu projeto. Aqui está um exemplo básico de um `Dockerfile` para uma aplicação Laravel:

```dockerfile
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

# Cria um usuário para o Composer e Artisan Commands e adiciona ao grupo www-data
RUN useradd -G www-data,root -u 1000 -d /var/www -s /bin/bash www \
    && mkdir -p /var/www/.composer && \
    chown -R www:www /var/www

# Copie o arquivo de configuração do Apache
COPY apache2.conf /etc/apache2/apache2.conf

# Ative o módulo de reescrita do Apache
RUN a2enmod rewrite

# Defina o diretório de trabalho
WORKDIR /var/www/html

# Copie os arquivos da aplicação para o diretório de trabalho
COPY . /var/www/html

# Defina as permissões corretas para o diretório de trabalho
RUN chown -R www-data:www-data /var/www/html \
    && chmod -R 755 /var/www/html/storage \
    && chmod -R 755 /var/www/html/bootstrap/cache

# Copie a configuração do Supervisord
COPY supervisord.conf /etc/supervisor/conf.d/supervisord.conf

# Exponha a porta 80 para o Apache
EXPOSE 80

# Inicie o Supervisord para gerenciar o Apache e outros processos
CMD ["/usr/bin/supervisord", "-c", "/etc/supervisor/conf.d/supervisord.conf"]

```

---

## 3. Rodando Testes Unitários (Opcional)

Se a sua aplicação Laravel inclui testes unitários, você pode configurar um job para rodar esses testes automaticamente antes de fazer o deploy.

### 3.1 Testes no Pipeline

Adicione a seguinte seção ao arquivo `.gitlab-ci.yml` para rodar os testes:

```yaml
test:
  stage: test
  image: php:${PHP_VERSION}-cli
  script:
    - cp .env.example .env
    - |
      echo "DB_CONNECTION=sqlite" >> .env
      echo "DB_DATABASE=:memory:" >> .env
    - composer install --no-interaction --prefer-dist
    - php artisan key:generate
    - php artisan migrate --env=testing --force
    - php artisan test
  tags:
    - docker
  only:
    - develop
```

Esse job será executado no branch `develop` sempre que houver um push.

---

## 4. Deploy no Ambiente de Desenvolvimento

Depois que a imagem for construída e os testes rodados (se houver), o deploy será realizado no ambiente de desenvolvimento. Usaremos o ArgoCD para fazer o deploy da aplicação no Kubernetes.

### 4.1 Configuração do ArgoCD

Para a fase de deploy, você precisa configurar o ArgoCD no ambiente de desenvolvimento:

1. Certifique-se de que as variáveis para o servidor ArgoCD estão definidas:

   - `ARGOCD_DEV_SERVER`
   - `ARGOCD_USERNAME`
   - `ARGOCD_PASSWORD`

2. Adicione o job de deploy no ambiente de desenvolvimento ao arquivo `.gitlab-ci.yml`:

```yaml
deployment-dev:
  stage: deploy-dev
  image: argoproj/argocd
  script:
    - argocd --grpc-web login "${ARGOCD_DEV_SERVER}" --insecure --username "${ARGOCD_USERNAME}" --password "${ARGOCD_PASSWORD}"
    - argocd --grpc-web app actions run laravel-app restart --kind Deployment
  tags:
    - docker
  only:
    - develop
```

Quando você fizer o push para o branch `develop`, o pipeline fará o deploy no ambiente de desenvolvimento automaticamente.

---

## 5. Preparando o Ambiente de Homologação

Após configurar o ambiente de desenvolvimento, você pode adicionar o job de deploy para homologação. Este processo é semelhante ao deploy de desenvolvimento.

### 5.1 Adicionando Job de Deploy para Homologação

Adicione um novo estágio para deploy em homologação no arquivo `.gitlab-ci.yml`:

```yaml
deployment-staging:
  stage: deploy-staging
  image: argoproj/argocd
  script:
    - argocd --grpc-web login "${ARGOCD_STAGING_SERVER}" --insecure --username "${ARGOCD_USERNAME}" --password "${ARGOCD_PASSWORD}"
    - argocd --grpc-web app actions run laravel-app restart --kind Deployment
  tags:
    - docker
  only:
    - staging
```

### 5.2 Configuração das Variáveis para Homologação

Certifique-se de que as variáveis específicas do ambiente de homologação estejam configuradas no GitLab:

- `ARGOCD_STAGING_SERVER`
- `ARGOCD_USERNAME`
- `ARGOCD_PASSWORD`

### 5.3 Criar o Branch de Homologação

Crie o branch `staging` para deploy no ambiente de homologação:

```bash
git checkout -b staging
git push origin staging
```

Com o push para o branch `staging`, o job de deploy será acionado, e a aplicação será reiniciada no ambiente de homologação.

---

## 6. Supervisord para Laravel

Se você precisa usar o Supervisord para gerenciar processos no ambiente de CI/CD com o GitLab, você pode configurá-lo no arquivo .gitlab-ci.yml. Isso pode ser útil para aplicações que precisam rodar múltiplos processos ao mesmo tempo, como um servidor de fila (queue worker) e o próprio servidor da aplicação Laravel.
Aqui está como você poderia configurar o Supervisord no job de build ou deployment, por exemplo:

```yaml
services:
  - name: redis:latest
  - name: mysql:latest
  - name: php:8.1-cli
    alias: php

before_script:
  - composer install --no-interaction --prefer-dist --optimize-autoloader
  - php artisan key:generate
  - cp .env.example .env
  - php artisan migrate

supervisord:
  stage: build
  image: php:8.1-cli
  script:
    - apt-get update
    - apt-get install -y supervisor
    - cp /path/to/supervisord.conf /etc/supervisor/conf.d/supervisord.conf
    - supervisord -c /etc/supervisor/conf.d/supervisord.conf
  tags:
    - docker
  only:
    - develop
```

Neste exemplo, o job `supervisord` instala o Supervisord e copia um arquivo de configuração `supervisord.conf` para o diretório `/etc/supervisor/conf.d/`. O arquivo de configuração deve conter as configurações necessárias para rodar os processos que você deseja gerenciar com o Supervisord.

### 6.1 Exemplo de Arquivo de Configuração do Supervisord

Aqui está um exemplo de arquivo de configuração `supervisord.conf` que define um grupo de programas para rodar um servidor de fila (queue worker) e o servidor da aplicação Laravel:

```conf
[program:laravel-queue-worker]
command=php /path/to/artisan queue:work
autostart=true
autorestart=true
user=www-data
redirect_stderr=true
stdout_logfile=/path/to/queue-worker.log

[program:laravel-server]
command=php /path/to/artisan serve --host=
autostart=true
autorestart=true
user=www-data
redirect_stderr=true
stdout_logfile=/path/to/server.log
```

Neste exemplo, o Supervisord gerencia dois programas: um servidor de fila (queue worker) e o servidor da aplicação Laravel. O `command` define o comando a ser executado para cada programa, e o `autostart` e `autorestart` definem se o programa deve ser iniciado automaticamente e reiniciado em caso de falha. O `user` define o usuário sob o qual o programa será executado, e o `stdout_logfile` define o arquivo de log para a saída padrão do programa.

Você pode personalizar o arquivo de configuração do Supervisord de acordo com as necessidades da sua aplicação.

### 6.2 Rodando o Supervisord no Pipeline

Depois de configurar o arquivo de configuração do Supervisord e o job no arquivo `.gitlab-ci.yml`, você pode rodar o Supervisord no pipeline. O Supervisord irá gerenciar os processos definidos no arquivo de configuração, garantindo que eles estejam em execução durante o deploy da aplicação.

### 6.3 Arquivo .gitlab-ci.yml Completo com Supervisord

Aqui está um exemplo completo de um arquivo `.gitlab-ci.yml` que inclui a configuração do Supervisord:

```yaml
variables:
  CONTAINER_NAME: laravel-app
  DOCKER_DRIVER: overlay2
  PHP_VERSION: "8.1"
  REDIS_HOST: redis

stages:
  - build
  - test
  - deploy-dev
  - deploy-staging

cache:
  paths:
    - vendor/

services:
  - redis

before_script:
  - composer install --no-interaction --prefer-dist --optimize-autoloader
  - php artisan key:generate
  - cp .env.example .env
  - php artisan migrate

supervisord:
  stage: build
  image: php:8.1-cli
  script:
    - apt-get update
    - apt-get install -y supervisor
    - cp /path/to/supervisord.conf /etc/supervisor/conf.d/supervisord.conf
    - supervisord -c /etc/supervisor/conf.d/supervisord.conf
  tags:
    - docker
  only:
    - develop

docker-build:
  stage: build
  image: php:${PHP_VERSION}-cli
  script:
    - docker build -t $CI_REGISTRY_IMAGE/$CONTAINER_NAME .
    - docker push $CI_REGISTRY_IMAGE/$CONTAINER_NAME
  tags:
    - docker
  only:
    - develop

test:
  stage: test
  image: php:${PHP_VERSION}-cli
  services:
    - mysql
  script:
    - php artisan test
  tags:
    - docker
  only:
    - develop

deployment-dev:
  stage: deploy-dev
  image: argoproj/argocd
  script:
    - argocd --grpc-web login "${ARGOCD_DEV_SERVER}" --insecure --username "${ARGOCD_USERNAME}" --password "${ARGOCD_PASSWORD}"
    - argocd --grpc-web app actions run laravel-app restart --kind Deployment
  tags:
    - docker
  only:
    - develop

deployment-staging:
  stage: deploy-staging
  image: argoproj/argocd
  script:
    - argocd --grpc-web login "${ARGOCD_STAGING_SERVER}" --insecure --username "${ARGOCD_USERNAME}" --password "${ARGOCD_PASSWORD}"
    - argocd --grpc-web app actions run laravel-app restart --kind Deployment
  tags:
    - docker
  only:
    - staging
```

Neste exemplo, o job `supervisord` instala o Supervisord e copia um arquivo de configuração `supervisord.conf` para o diretório `/etc/supervisor/conf.d/`. O arquivo de configuração define dois programas: um servidor de fila (queue worker) e o servidor da aplicação Laravel. O job `supervisord` é executado no estágio `build` do pipeline.

O Supervisord irá gerenciar os processos definidos no arquivo de configuração, garantindo que eles estejam em execução durante o deploy da aplicação.

### 6.4 Considerações Finais

O uso do Supervisord no pipeline do GitLab pode ser útil para aplicações que precisam rodar múltiplos processos ao mesmo tempo, como um servidor de fila (queue worker) e o próprio servidor da aplicação Laravel. O Supervisord simplifica a gestão desses processos, garantindo que eles estejam em execução durante o deploy da aplicação.

O exemplo fornecido neste tutorial é apenas um ponto de partida. Você pode personalizar o arquivo de configuração do Supervisord e o job no arquivo `.gitlab-ci.yml` de acordo com as necessidades específicas da sua aplicação.

---

## 7. Conclusão

Neste tutorial, você aprendeu como configurar um pipeline de CI/CD no GitLab para uma aplicação Laravel com Redis. Você configurou os estágios de build, testes, deploy para desenvolvimento e deploy para homologação, e aprendeu como usar o ArgoCD para fazer o deploy da aplicação no Kubernetes.

Além disso, você viu como configurar o Supervisord para gerenciar processos no ambiente de CI/CD, garantindo que eles estejam em execução durante o deploy da aplicação.

Esperamos que este tutorial tenha sido útil e que você possa aplicar esses conceitos em seus próprios projetos. O GitLab oferece uma série de recursos poderosos para automatizar o processo de desenvolvimento e deploy de aplicações, e esperamos que você possa aproveitar ao máximo essas ferramentas em seus projetos.

Obrigado por ler e boa sorte em seus projetos de desenvolvimento!

---
