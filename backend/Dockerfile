# Use PHP 8.1 with FPM for Laravel
FROM php:8.2-fpm

# Install system dependencies
RUN apt-get update && apt-get install -y     libpng-dev libjpeg-dev libfreetype6-dev libxml2-dev     libzip-dev unzip git procps     && docker-php-ext-install pdo_mysql gd xml zip

# Install Composer
RUN curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer

# Set working directory
WORKDIR /var/www

# Copy application code
COPY . .
COPY .env .env
COPY php-fpm.conf /usr/local/etc/php-fpm.d/www.conf

# Install Laravel dependencies
RUN composer install --optimize-autoloader --no-dev

# Generate APP_KEY
RUN php artisan key:generate

# Set permissions
RUN chown -R www-data:www-data /var/www/storage /var/www/bootstrap/cache

# Expose port for PHP-FPM
EXPOSE 9000

# Start PHP-FPM
CMD ["php-fpm"]
