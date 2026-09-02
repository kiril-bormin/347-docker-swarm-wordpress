# P_DevOps 347

### Opérations sur chaque VM
Mise à jour du système 
```bash
apk update && apk upgrade
```

Installation de docker sur
```bash
apk add docker
```

### Initialisation de Docker Swarm
### Sur la VM Manager
```bash
docker swarm init --advertise-addr 10.228.242.211
```

### Sur les Workers
```bash
docker swarm join --token SWMTKN-1-5negsl1ce27rxlsdty7ogsiyjtfckf36vcmhfllyx6bmfg47qg-b
o41phm2cl2dzkaqc4w49r0nr 10.228.242.211:2377
```

Vérification de la présence des VM 
```bash
docker node ls
```

### Configuration des réseau Frontend et Backend

Réseau pour la communication entre Nginx et WP
```bash
docker network create --driver overlay frontend
```
Réseau pour la communication isolé entre WP et MariaDB
```bash
docker network create --driver overlay backend
```

### Configuration des secrets Docker

```bash
echo "mdp" | docker secret create db_root_password -
```
hash - jsy51fkmgqicoga2oyewmn7j7

```bash
echo "mdp" | docker secret create db_password -
```

hash - r4rrz70lifp7avrfbflrin1w3

### Préparation de Nginx

Création d'un fichier de config
```bash
mkdir -p /root/wordpress-swarm/nginx
cd /root/wordpress-swarm/nginx
touch default.conf
```
J'utilise nano pour modifier le contenu du fichier
```bash
nano default.conf
```

Je met le contenu suivant: 

```bash
server {
    listen 80;
    server_name www.my-wordpress.ch;
    root /var/www/html;
    index index.php index.html;

    location / {
        try_files $uri $uri/ /index.php?$args;
    }

    location ~ \.php$ {
        fastcgi_pass wordpress:9000;
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    }
}
```

Fichier docker compose 
```bash
version: '3.8'

services:
  # DB
  db:
    image: mariadb:11.4
    # Utilisation des secrets Docker 
    environment:
      MYSQL_ROOT_PASSWORD_FILE: /run/secrets/db_root_password
      MYSQL_DATABASE: wordpress
      MYSQL_USER: wp_user
      MYSQL_PASSWORD_FILE: /run/secrets/db_password
    secrets:
      - db_root_password
      - db_password
    volumes:
      - db_data:/var/lib/mysql
    networks:
      - backend
    deploy:
      replicas: 1
      placement:
        constraints:
          - node.role == manager

  # Serveur WordPress (PHP)
  wordpress:
    image: wordpress:6.8.2-fpm-alpine
    environment:
      WORDPRESS_DB_HOST: db:3306
      WORDPRESS_DB_NAME: wordpress
      WORDPRESS_DB_USER: wp_user
      WORDPRESS_DB_PASSWORD_FILE: /run/secrets/db_password
    secrets:
      - db_password
    volumes:
      - wp_data:/var/www/html
    networks:
      - frontend
      - backend
    deploy:
      replicas: 2

  # Reverse proxy Nginx
  nginx:
    image: nginx:1.28.0-alpine3.21
    ports:
      - "80:80"
    volumes:
      - ./nginx/default.conf:/etc/nginx/conf.d/default.conf:ro
      - wp_data:/var/www/html:ro
    networks:
      - frontend
    deploy:
      replicas: 2

# Déclaration des volumes gérés par Swarm
volumes:
  db_data:
  wp_data:

# Déclaration des réseaux overlay préexistants
networks:
  frontend:
    external: true
  backend:
    external: true

# Déclaration des secrets créés précédemment
secrets:
  db_root_password:
    external: true
  db_password:
    external: true
```

### Déployer les services

```
docker stack deploy -c docker-compose.yml wp_stack
```

Vérificaiton de l'état 
```
docker stack services wp_stack
```

En premier temps les services nginx et wp ont bien démarrés sur les deux workers, mais la db n'a pas démmaré du premier coup parce que la version 12.0.1 n'était pas téléchargée, donc j'ai mis une version plus ancienne qui est 11.4.

