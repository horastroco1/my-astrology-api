# Use Ubuntu 22.04 as the base image
FROM ubuntu:22.04

# Avoid prompts from apt
ENV DEBIAN_FRONTEND=noninteractive
ENV COMPOSER_ALLOW_SUPERUSER=1

# Set working directory
WORKDIR /var/www/api

# 1. Install system dependencies (Adding build-essential for compiling C code)
RUN apt-get update -y && \
    apt-get install -y \
    git \
    unzip \
    libzip-dev \
    libicu-dev \
    libonig-dev \
    libpng-dev \
    curl \
    software-properties-common \
    apt-transport-https \
    ca-certificates \
    gnupg \
    g++ \
    make \
    build-essential \
    nginx

# Add PHP repository
RUN add-apt-repository ppa:ondrej/php -y
RUN apt-get update -y

# Install PHP and extensions
RUN apt-get install -y \
    php7.4 \
    php7.4-fpm \
    php7.4-cli \
    php7.4-common \
    php7.4-mysql \
    php7.4-xml \
    php7.4-mbstring \
    php7.4-curl \
    php7.4-zip \
    php7.4-gd \
    php7.4-intl \
    php7.4-bcmath \
    php7.4-soap \
    php7.4-opcache

# Install Composer
RUN curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer

# Clean up
RUN apt-get clean && rm -rf /var/lib/apt/lists/*

# Copy composer files first
COPY composer.json ./
# COPY composer.lock ./ 

# Install dependencies
RUN composer install --no-scripts --no-autoloader

# Copy the rest of the application
COPY . .

# --- ⚡️ KEY FIX: COMPILE SWETEST MANUALLY ---
# We go into the source folder and compile a fresh binary for Linux
RUN cd swetest/src && \
    make clean && \
    make && \
    chmod +x swetest

# Create necessary directories and set permissions
RUN mkdir -p /var/www/api/var/cache /var/www/api/var/log /run/php && \
    chown -R www-data:www-data /var/www/api/var && \
    chmod -R 777 /var/www/api/var

# Dump autoload
RUN composer dump-autoload --optimize

# Configure Nginx
RUN echo ' \
server { \
    listen 9393; \
    root /var/www/api/public; \
    index index.php index.html index.htm; \
    server_name _; \
    \
    access_log /var/log/nginx/access.log; \
    error_log /var/log/nginx/error.log; \
    \
    location / { \
        try_files $uri $uri/ /index.php?$query_string; \
    } \
    \
    location ~ \.php$ { \
        fastcgi_pass unix:/run/php/php7.4-fpm.sock; \
        fastcgi_index index.php; \
        include fastcgi_params; \
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name; \
    } \
}' > /etc/nginx/sites-available/default

# Enable site
RUN rm -f /etc/nginx/sites-enabled/default && \
    ln -s /etc/nginx/sites-available/default /etc/nginx/sites-enabled/

# Fix PHP-FPM logging configuration
RUN sed -i 's/;error_log = php_errors.log/error_log = \/var\/log\/php7.4-fpm.log/' /etc/php/7.4/fpm/php-fpm.conf && \
    sed -i 's/listen = \/run\/php\/php7.4-fpm.sock/listen = \/run\/php\/php7.4-fpm.sock/' /etc/php/7.4/fpm/pool.d/www.conf

# Create log files
RUN touch /var/log/nginx/access.log /var/log/nginx/error.log /var/log/php7.4-fpm.log && \
    chown -R www-data:www-data /var/log/nginx /var/log/php7.4-fpm.log

# Install assets and clear cache
RUN php bin/console assets:install --symlink || true
RUN php bin/console cache:clear || true

EXPOSE 9393

# Start services
CMD service php7.4-fpm start && nginx -g 'daemon off;'
