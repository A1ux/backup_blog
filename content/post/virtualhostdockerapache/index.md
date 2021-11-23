+++
author = "Alux"
title = "Multiples contenedores y Proxy Inverso Apache + HTTPS"
date = "2021-11-21"
description = "Cuando se cuenta con varios contenedores y queremos que accedan a traves de un virtual host se hace complicado, por eso esta es la forma de realizar este metodo para redirigir varias paginas web a traves del mismo puerto"
tags = [
    "docker",
    "apache",
    "virtual host",
    "proxy",
]
categories = [
    "infraestructure",
]
series = ["Infraestructure"]
image = "head.png"
+++

# Problema

Actualmente cuento con un vps el cual uso muy poco pero queria utilizarlo mas seguido, por lo cual me di a la tarea de tener un dns el cual me bloquee todo el trafico malicioso o de publicidad, la primera opcion fue `Pi-Hole` que utilice anteriormente aunque existen otras opciones como `Adguard Home` pero que algun dia probare. Para esto queria que todo estuviera centralizado y no tener que acceder a traves de puertos a los servicios como pihole o algun otro docker que quiera levantar mas adelante. ya que no quiero acceder por dominio.com:port para cada servicio sino que segun lo que ingrese se me redirija hacia el puerto 443 automaticamente.

Para eso lo que hare es que cuando se acceda a los siguientes dominios, sea automaticamente redirigido a un docker por el proxy reverso de apache y evitar colocar puertos y que se maneje todo por subdominios y forzando las comunicaciones a https.

1. https://pi.dominio.com/
2. https://wordpress.dominio.com/

# Resolucion

Para esto me apoyare del siguiente software para solucionar este problema.

- Apache
- Docker
- Certbot

## Configuracion de contenedores docker

Actualmente solo tengo 1 contenedor arriba, que es pihole pero hare un ejemplo si tuviera un wordpress adicional a este para poder acceder a traves del dominio y todos con un certificado de letsencrypt.

### Pi-Hole Container

Para pihole existe el siguiente script para poder ejecutarlo, que lo que hara es levantar uno en el puerto 3141 de nuestro vps, pero sin exponerlo a toda internet.

```bash
#!/bin/bash

# https://github.com/pi-hole/docker-pi-hole/blob/master/README.md

PIHOLE_BASE="${PIHOLE_BASE:-$(pwd)}"
[[ -d "$PIHOLE_BASE" ]] || mkdir -p "$PIHOLE_BASE" || { echo "Couldn't create storage directory: $PIHOLE_BASE"; exit 1; }

# Note: ServerIP should be replaced with your external ip.
docker run -d \
    --name pihole \
    -p 53:53/tcp -p 53:53/udp \
    -p 127.0.0.1:3141:80 \
    -e TZ="America/Chicago" \
    -v "${PIHOLE_BASE}/etc-pihole/:/etc/pihole/" \
    -v "${PIHOLE_BASE}/etc-dnsmasq.d/:/etc/dnsmasq.d/" \
    --dns=127.0.0.1 --dns=1.1.1.1 \
    --restart=unless-stopped \
    --hostname pi.hole \
    -e VIRTUAL_HOST="pi.hole" \
    -e PROXY_LOCATION="pi.hole" \
    -e ServerIP="127.0.0.1" \
    pihole/pihole:latest

printf 'Starting up pihole container '
for i in $(seq 1 20); do
    if [ "$(docker inspect -f "{{.State.Health.Status}}" pihole)" == "healthy" ] ; then
        printf ' OK'
        echo -e "\n$(docker logs pihole 2> /dev/null | grep 'password:') for your pi-hole: https://${IP}/admin/"
        exit 0
    else
        sleep 3
        printf '.'
    fi

    if [ $i -eq 20 ] ; then
        echo -e "\nTimed out waiting for Pi-hole start, consult your container logs for more info (\`docker logs pihole\`)"
        exit 1
    fi
done;
```

### Wordpress container

Ahora tenemos otro contenedor con wordpress que lo que hace es iniciar una instancia de word y siempre solamente local en el puerto 8002, para esto ya tenemos dos contenedores en los cuales trabajar.

```yaml
version: "3"
services:
  WordPress:
    image: wordpress
    links:
      - mariadb:mysql
    environment:
      - WORDPRESS_DB_PASSWORD=password
    ports:
      - 127.0.0.1:8001:80
    volumes:
      - ./html:/var/www/html

  mariadb:
    image: mariadb
    environment:
      - MYSQL_ROOT_PASSWORD=password
      - MYSQL_DATABASE=wordpress
    volumes:
      - ./database:/var/lib/mysql
```

## Generacion de certificados de SSL

Para esto utilizaremos certbot que nos solucionara la vida, ya que permitira descargar los certificados generados por letsencrypt en nuestra carpeta de archivos, todos estos se guardan en `/etc/letsencrypt/live/`

Lo primero es en nuestro DNS agregar el CNAME de wordpress y pi apuntando a nuestra ip ya que certbot se basa en esto sino no dejara generar los certificados necesarios ya que hace una comprobacion de la ip y de nombre de DNS. Para esto los generamos con los siguientes comandos:

```bash
certbot certonly --apache -d pi.dominio.com
certbot certonly --apache -d wordpress.dominio.com
```
Y si todo esta correctamente son guardados en la carpeta anterior mencionada, y desde ahi se podran importar a apache. Para configurar que las conexiones solo sean manejadas por HTTPS.

## Configurar Apache como proxy reverso

Para eso debemos de ejecutar las siguientes instrucciones con permisos de `root`:

```bash
apt install apache
a2enmod proxy       
a2enmod proxy_http
a2enmod proxy_balancer
a2enmod ssl
a2enmod rewrite
```

Lo que hara es instalar apache, habilitar los modulos necesarios para redireccion y que actue como gateway, permitir proxear y balancear todo, aparte el habilitar modulo de SSL para que las comunicaciones vayan cifradas por HTTPS y que se pueda redirigir a estas. Ahora solo queda desactivar el puerto 80 y manejar siempre por 443 que es HTTPS, para eso editamos el archivo `/etc/apache2/ports.conf`.

```
# If you just change the port or add more ports here, you will likely also
# have to change the VirtualHost statement in
# /etc/apache2/sites-enabled/000-default.conf

#Listen 80

<IfModule ssl_module>
        Listen 443
</IfModule>

<IfModule mod_gnutls.c>
        Listen 443
</IfModule>

# vim: syntax=apache ts=4 sw=4 sts=4 sr noet
```

Ya que habilitamos e instalamos apache, vamos a lo siguiente que es agregar la configuracion para los sitios web a los que se quiere acceder:

```
cd /etc/apache2/sites-available/
touch wordpress.conf
touch pihole.conf
```
Y en pihole.conf agregamos el siguiente texto que lo que hace es colocar el nombre del virtual host al que accederemos igual al nombre que se coloca en el DNS, la ip y puerto en la que esta expuesto que en nuestro caso es localhost y puerto 3141 y los certificados SSL anteriormente generados con certbot.

pihole.conf
```
<VirtualHost *:443>
    ProxyPreserveHost On
    ServerName pi.dominio.com
    ProxyPass / http://127.0.0.1:3141/
    ProxyPassReverse / http://127.0.0.1:3141/
    SSLEngine On
    SSLProxyEngine On
    SSLCertificateFile "/etc/letsencrypt/live/pi.dominio.com/cert.pem"
    SSLCertificateChainFile "/etc/letsencrypt/live/pi.dominio.com/chain.pem"
    SSLCertificateKeyFile "/etc/letsencrypt/live/pi.dominio.com/privkey.pem"
</VirtualHost>
```
Para wordpress es similar y solo se cambia de puerto

wordpress.conf
```
<VirtualHost *:443>
    ProxyPreserveHost On
    ServerName wordpress.dominio.com
    ProxyPass / http://127.0.0.1:8001/
    ProxyPassReverse / http://127.0.0.1:8001/
    SSLEngine On
    SSLProxyEngine On
    SSLCertificateFile "/etc/letsencrypt/live/wordpress.dominio.com/cert.pem"
    SSLCertificateChainFile "/etc/letsencrypt/live/wordpress.dominio.com/chain.pem"
    SSLCertificateKeyFile "/etc/letsencrypt/live/wordpress.dominio.com/privkey.pem"
</VirtualHost>
```
teniendo esto ya solamente queda reiniciar apache o si ya lo habra pedido antes tambien hacerlo si es necesario.

```bash
systemctl reload apache2
```

Y si accedemos a traves de internet a pi.dominio.com y wordpress.dominio.com ya deberia entrar sin problema y a traves de HTTPS con certificados de Lets Encrypt.