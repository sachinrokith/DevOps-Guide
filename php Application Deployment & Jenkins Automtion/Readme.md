1.AWS EC2 Server Setup
Login to server
     ssh -i key.pem ubuntu@EC2_PUBLIC_IP
     
Update system
     sudo apt update && sudo apt upgrade -y

2.Install Docker & Docker Compose
       sudo apt install -y ca-certificates curl gnupg
       
       sudo install -m 0755 -d /etc/apt/keyrings
       curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor
       -o /etc/apt/keyrings/docker.gpg
       
       sudo chmod a+r /etc/apt/keyrings/docker.gpg
       echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/
       docker.gpg] https://download.docker.com/linux/ubuntu $(. /etc/os-release &&
       echo $VERSION_CODENAME) stable" | sudo tee /etc/apt/sources.list.d/
       docker.list > /dev/null
       
       sudo apt update
       sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-
       plugin docker-compose-plugin

3.Start docker
       sudo systemctl enable docker
       sudo systemctl start docker

4.Allow users to run docker
       sudo usermod -aG docker ubuntu
       sudo usermod -aG docker jenkins
       sudo systemctl restart jenkins

5.Project Folder Structure on Server
      /var/lib/jenkins/app/Truse/truelysell-laravel-v1                                                     -> actual deployed project
      /home/ubuntu/                                                                                        -> sql backups & manual
files

We use Jenkins directory because Jenkins owns permissions.

6.Dockerfile;

FROM php:8.4-fpm

# Install dependencies
RUN apt-get update && apt-get install -y \
    git curl zip unzip libpng-dev libonig-dev libxml2-dev libzip-dev

# Install PHP extensions
RUN docker-php-ext-install pdo_mysql mbstring exif pcntl bcmath gd zip

# Install Composer
COPY --from=composer:latest /usr/bin/composer /usr/bin/composer

WORKDIR /var/www

COPY . .

RUN composer install --no-dev --optimize-autoloader

CMD ["php-fpm"]

7.docker-compose.yml;

version: '3.8'

services:

  app:
    build: .
    container_name: laravel_app
    restart: unless-stopped
    working_dir: /var/www
    volumes:
      - ./:/var/www
    networks:
      - laravel

  nginx:
    image: nginx:alpine
    container_name: laravel_nginx
    restart: unless-stopped
    ports:
      - "80:80"
    volumes:
      - ./:/var/www
      - ./nginx/default.conf:/etc/nginx/conf.d/default.conf
    depends_on:
      - app
    networks:
      - laravel

  db:
    image: mysql:8
    container_name: laravel_db
    restart: unless-stopped
    environment:
      MYSQL_DATABASE: laravel
      MYSQL_ROOT_PASSWORD: root
      MYSQL_USER: truelyuser
      MYSQL_PASSWORD: StrongPassword123
    ports:
      - "3306:3306"
    networks:
      - laravel

  phpmyadmin:
    image: phpmyadmin/phpmyadmin
    container_name: laravel_pma
    restart: unless-stopped
    ports:
      - "8081:80"
    environment:
      PMA_HOST: db
    networks:
      - laravel

networks:
  laravel:
    
8.nginx/ default.conf;

server {
    listen 80;
    server_name _;

    root /var/www/public;
    index index.php index.html index.htm;

    charset utf-8;

    # Max upload size (important for Laravel forms & images)
    client_max_body_size 100M;

    # Main Laravel routing
    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    # Handle PHP files
    location ~ \.php$ {
        fastcgi_pass laravel_app:9000;
        fastcgi_index index.php;
        include fastcgi_params;

        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        fastcgi_param DOCUMENT_ROOT $realpath_root;
        fastcgi_param PATH_INFO $fastcgi_path_info;

        fastcgi_buffers 16 16k;
        fastcgi_buffer_size 32k;
    }

    # Cache static files (improves performance)
    location ~* \.(jpg|jpeg|png|gif|ico|css|js|svg|woff|woff2|ttf|map)$ {
        expires max;
        log_not_found off;
    }

    # Security: block hidden files
    location ~ /\.(?!well-known).* {
        deny all;
    }

    # Favicon & robots
    location = /favicon.ico { access_log off; log_not_found off; }
    location = /robots.txt  { access_log off; log_not_found off; }

    # Send all 404 to Laravel
    error_page 404 /index.php;
}

9.Generate Laravel App Key
     docker-compose exec app php artisan key:generate
     docker-compose exec app php artisan optimize:clear
     docker-compose exec app php artisan tinker
     
10Import Database
     docker exec -i truelysell-mysql mysql -u truelyselluser -pStrongPassword123 truelysell < truelysell-laravel-new.sql

11.Run Migrations
     docker-compose exec app php artisan migrate

12.Fix Permissions
     docker-compose exec app chown -R www-data:www-data storage bootstrap/cache
     docker-compose exec app mkdir -p storage/framework/sessions
     docker-compose exec app mkdir -p storage/framework/views
     docker-compose exec app mkdir -p storage/logs
     docker-compose exec app chmod -R 775 storage bootstrap/cache

13.Jenkins

pipeline {
 agent any

 environment {
    APP_DIR="/home/ubuntu/truelysell-laravel-v1"
    BRANCH="main"
    SCANNER_HOME=tool 'SonarScanner'
 }

 stages {

  stage('Fix Permissions'){
   steps{
    sh 'sudo chown -R jenkins:jenkins $APP_DIR || true'
   }
  }

  stage('Update Repo'){
   steps{
    sh '''
    cd $APP_DIR
    git fetch origin
    git reset --hard origin/$BRANCH
    '''
   }
  }

  stage('Build Image'){
   steps{
    sh '''
    cd $APP_DIR
    docker compose build app
    '''
   }
  }

  stage('Recreate Containers'){
   steps{
    sh '''
    cd $APP_DIR
    docker compose rm -f app nginx || true
    docker compose up -d app nginx
    '''
   }
  }

  stage('Install Dependencies'){
   steps{
    sh '''
    cd $APP_DIR
    docker compose exec -T app composer install --optimize-autoloader
    docker compose exec -T app sh -c "[ -f .env ] || cp .env.example .env"
    docker compose exec -T app php artisan key:generate --force || true
    '''
   }
  }

  stage('Run Unit Tests'){
   steps{
    sh '''
    cd $APP_DIR
    docker compose exec -T -e XDEBUG_MODE=coverage app vendor/bin/phpunit --coverage-clover coverage.xml
    '''
   }
  }

  stage('SonarQube Analysis'){
   steps{
    withSonarQubeEnv('SonarQube'){
     sh """
     ${SCANNER_HOME}.........location \
      -Dsonar.projectKey=key name \
      -Dsonar.sources=${APP_DIR} \
      -Dsonar.php.coverage.reportPaths=${APP_DIR}/coverage.xml 
     """
    }
   }
  }

  stage('Quality Gate'){
   steps{
    timeout(time:15,unit:'MINUTES'){
     waitForQualityGate abortPipeline:true
    }
   }
  }

  stage('Laravel Setup'){
   steps{
    sh '''
    cd $APP_DIR
    docker compose exec -T app php artisan migrate --force || true
    docker compose exec -T app php artisan optimize || true
    '''
   }
  }
 }

 post{
  success{ echo 'Deployment successful' }
  failure{ echo 'Deployment failed' }
 }
}
