# syntax=docker/dockerfile:1

FROM public.ecr.aws/amazonlinux/amazonlinux:2.0.20231116.0 as builder

RUN <<EOF
    amazon-linux-extras install -y nginx1 php8.1
    yum groupinstall -y "Development Tools"
    yum install -y php-devel php-{opcache,curl,gd,intl,mbstring,soap,xml,zip,dom,imagick,posix,pear,redis}
    yum clean all && rm -rf /var/cache/yum
    echo ls /usr/lib64/php/modules
    ls /usr/lib64/php/modules
    pecl install igbinary && echo "extension=igbinary.so" >> /etc/php.d/igbinary.ini
    pecl install redis && echo "extension=redis.so" >> /etc/php.d/redis.ini
    ls /usr/lib64/php/modules
    curl -sS https://raw.githubusercontent.com/composer/getcomposer.org/9bef96e8ce65b79bd29c525fa918980889c9a124/web/installer | php -- --quiet
    mv composer.phar /usr/local/bin/composer
EOF

ADD --chown=999:999 --chmod=755 run.sh /srv/run.sh
ADD --chown=999:999 --chmod=755 bedrock/ /srv/bedrock/
ADD --chown=999:999 --chmod=755 opcache/ /srv/opcache/

RUN cd /srv/bedrock && composer install --no-dev

FROM public.ecr.aws/amazonlinux/amazonlinux:2.0.20231116.0
COPY --from=public.ecr.aws/awsguru/aws-lambda-adapter:0.3.3 /lambda-adapter /opt/extensions/lambda-adapter
ARG DISABLE_OPCACHE_PRE_COMPILATION=false

RUN <<EOF
    amazon-linux-extras install -y nginx1 php8.1
    yum install -y sudo php-{opcache,curl,gd,intl,mbstring,soap,xml,zip,dom,imagick,posix,redis} less
    yum clean all && rm -rf /var/cache/yum
    echo "extension=igbinary.so" >> /etc/php.d/igbinary.ini
    echo "extension=redis.so" >> /etc/php.d/redis.ini
EOF

RUN curl -O https://raw.githubusercontent.com/wp-cli/builds/gh-pages/phar/wp-cli.phar && \
    chmod +x wp-cli.phar && \
    mv wp-cli.phar /usr/local/bin/wp

ADD --chmod=755 config/nginx/ /etc/nginx/
ADD --chmod=755 config/php/ /etc/
COPY --from=builder /srv /srv
COPY --from=builder /usr/lib64/php/modules /usr/lib64/php/modules

RUN <<EOF
    if [[ "${DISABLE_OPCACHE_PRE_COMPILATION}" = "false" ]]; 
    then 
        mkdir /tmp/opcache 
        chmod 777 /tmp/opcache
        # sudo -u nginx -g tty /usr/sbin/php-fpm && nginx
        /usr/sbin/php-fpm && nginx
        curl --no-progress-meter localhost:8080/app/prime_opcache.php
        rm /srv/bedrock/web/app/prime_opcache.php
        chmod -R 777 /tmp/opcache && rm -rf /srv/opcache && mv /tmp/opcache /srv/
    fi

    rm -rf /srv/bedrock/web/app/uploads
    mkdir -p /mnt/share/uploads
    ln -sfn /mnt/share/uploads /srv/bedrock/web/app/uploads
    ls -l /srv
EOF

CMD ["/srv/run.sh"]
