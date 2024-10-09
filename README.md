
#!/bin/bash


SSL_DIR="sail-ssl"


mkdir -p $SSL_DIR


openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
    -keyout $SSL_DIR/server.key \
    -out $SSL_DIR/server.crt \
    -subj "/C=US/ST=State/L=City/O=Organization/OU=Unit/CN=localhost"


DOCKER_COMPOSE_FILE="docker-compose.yml"


if ! grep -q "443:443" $DOCKER_COMPOSE_FILE; then
    sed -i '/ports:/a \ \ \ \ - "443:443"' $DOCKER_COMPOSE_FILE
fi


if ! grep -q "$SSL_DIR:/etc/ssl/private" $DOCKER_COMPOSE_FILE; then
    sed -i "/laravel.test:/a \ \ \ \ volumes:\n\ \ \ \ \ \ \ \ - ./$SSL_DIR:/etc/ssl/private" $DOCKER_COMPOSE_FILE
fi


APACHE_CONF_FILE="docker/apache/default.conf"


cat <<EOL > $APACHE_CONF_FILE
<VirtualHost *:80>
    ServerName localhost
    Redirect permanent / https://localhost/
</VirtualHost>

<VirtualHost *:443>
    ServerName localhost

    SSLEngine on
    SSLCertificateFile /etc/ssl/private/server.crt
    SSLCertificateKeyFile /etc/ssl/private/server.key

    DocumentRoot /var/www/html/public
    <Directory /var/www/html/public>
        AllowOverride All
        Require all granted
    </Directory>
</VirtualHost>
EOL

# restart
./vendor/bin/sail down
./vendor/bin/sail up -d


