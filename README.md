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
docker swarm init --advertise-addr 10.228.242.240
```

### Sur les Workers
```bash
docker swarm join --token <TOKEN_WORKER> 10.228.242.240:2377
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

```nginx
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

Pour que Nginx puisse démarrer sur les nœuds Workers sans erreur de montage du volume (`./nginx/default.conf`), je copie aussi ce dossier et ce fichier sur les deux Workers (`worker1` et `worker2`) :
```bash
mkdir -p /root/wordpress-swarm/nginx
# Copier le fichier default.conf dans /root/wordpress-swarm/nginx/default.conf
```

### Fichier docker compose 

J'ai documenté chaque instruction du fichier `docker-compose.yml` et ajouté les contraintes de placement pour que MariaDB tourne sur le manager et que WordPress et Nginx tournent sur les workers :

```yaml
# Version de syntaxe Compose compatible avec Docker Swarm
version: '3.8'

services:
  # ==============================================================================
  # Service Base de données MariaDB
  # ==============================================================================
  db:
    # Image officielle MariaDB (version 11.4 LTS stable)
    image: mariadb:11.4

    # Variables d'environnement pour la configuration de la base de données
    environment:
      # Chemin vers le secret Docker contenant le mot de passe root MariaDB
      MYSQL_ROOT_PASSWORD_FILE: /run/secrets/db_root_password
      # Nom de la base de données créée automatiquement pour WordPress
      MYSQL_DATABASE: wordpress
      # Nom de l'utilisateur dédié créé pour WordPress
      MYSQL_USER: wp_user
      # Chemin vers le secret Docker contenant le mot de passe utilisateur
      MYSQL_PASSWORD_FILE: /run/secrets/db_password

    # Déclaration des secrets Swarm montés dans le conteneur (/run/secrets/)
    secrets:
      - db_root_password
      - db_password

    # Volume persistant pour stocker les données MariaDB même en cas de redémarrage
    volumes:
      - db_data:/var/lib/mysql

    # Réseau overlay isolé pour autoriser uniquement la communication avec WordPress
    networks:
      - backend

    # Paramètres de déploiement Docker Swarm
    deploy:
      # Une seule réplique car MariaDB standard ne gère pas le multi-instance actif/actif
      replicas: 1
      # Contrainte de placement : s'exécute uniquement sur le nœud manager
      placement:
        constraints:
          - node.role == manager

  # ==============================================================================
  # Service WordPress (PHP-FPM)
  # ==============================================================================
  wordpress:
    # Image officielle WordPress PHP-FPM légère sous Alpine v6.8.2
    image: wordpress:6.8.2-fpm-alpine

    # Variables d'environnement pour connecter WordPress à MariaDB
    environment:
      # Hôte et port du service MariaDB sur le réseau backend
      WORDPRESS_DB_HOST: db:3306
      # Nom de la base de données MariaDB à utiliser
      WORDPRESS_DB_NAME: wordpress
      # Utilisateur MariaDB pour la connexion
      WORDPRESS_DB_USER: wp_user
      # Fichier secret contenant le mot de passe de l'utilisateur MariaDB
      WORDPRESS_DB_PASSWORD_FILE: /run/secrets/db_password

    # Secret Swarm monté dans le conteneur pour le mot de passe utilisateur
    secrets:
      - db_password

    # Volume nommé partagé contenant les fichiers PHP et médias de WordPress
    volumes:
      - wp_data:/var/www/html

    # Réseaux overlay : frontend (accès Nginx) et backend (accès MariaDB)
    networks:
      - frontend
      - backend

    # Paramètres de déploiement Docker Swarm
    deploy:
      # 2 répliques pour assurer la haute disponibilité et la tolérance aux pannes
      replicas: 2
      # Contrainte de placement : s'exécute sur les nœuds workers
      placement:
        constraints:
          - node.role == worker

  # ==============================================================================
  # Service Nginx (Reverse Proxy & Serveur Web)
  # ==============================================================================
  nginx:
    # Image officielle Nginx v1.28.0 sous Alpine Linux v3.21
    image: nginx:1.28.0-alpine3.21

    # Ports publiés sur le réseau hôte (Routing Mesh Swarm)
    ports:
      # Redirige le port 80 de l'hôte vers le port 80 du conteneur Nginx
      - "80:80"

    # Volumes montés dans le conteneur Nginx
    volumes:
      # Montage en lecture seule du fichier de configuration du reverse proxy
      - ./nginx/default.conf:/etc/nginx/conf.d/default.conf:ro
      # Montage en lecture seule des fichiers statiques WordPress
      - wp_data:/var/www/html:ro

    # Réseau overlay frontend pour communiquer avec les répliques WordPress
    networks:
      - frontend

    # Paramètres de déploiement Docker Swarm
    deploy:
      # 2 répliques Nginx réparties pour la répartition de charge
      replicas: 2
      # Contrainte de placement : s'exécute sur les nœuds workers
      placement:
        constraints:
          - node.role == worker

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

Je me place dans le dossier du projet et je lance le déploiement de la stack :
```bash
cd /root/wordpress-swarm
docker stack deploy -c docker-compose.yml wp_stack
```

Vérification de l'état des services :
```bash
docker stack services wp_stack
```

Vérification des tâches et sur quels nœuds elles tournent :
```bash
docker stack ps wp_stack
```

Au début, la base de données ne démarrait pas avec l'image `mariadb:12.0.1` demandée dans le cahier des charges parce que ce tag n'existe pas sur Docker Hub officiel, j'ai donc utilisé la version stable LTS `mariadb:11.4`.

Pour Nginx, comme les répliques tournent sur les workers, le fichier `default.conf` doit être présent au même chemin sur `worker1` et `worker2` pour éviter une erreur de montage de volume. Une fois le fichier copié et les contraintes de placement appliquées, les 3 services tournent parfaitement :
- `wp_stack_db` : 1/1 (sur le manager)
- `wp_stack_wordpress` : 2/2 (1 réplique sur worker1, 1 réplique sur worker2)
- `wp_stack_nginx` : 2/2 (1 réplique sur worker1, 1 réplique sur worker2)

### Configuration de l'accès client Windows

Pour pouvoir accéder au site avec le nom de domaine `www.my-wordpress.ch` depuis la machine Windows hôte :

**Option 1 : Si on a les droits Administrateur sur Windows**
1. Ouvrir le Bloc-notes Windows en mode **Administrateur**.
2. Ouvrir le fichier : `C:\Windows\System32\drivers\etc\hosts`.
3. Ajouter la ligne suivante à la fin du fichier :
```text
10.228.242.240 www.my-wordpress.ch
```
4. Sauvegarder le fichier.

**Option 2 : Sur un PC ETML (sans droits Administrateur)**
Comme je n'ai pas les droits admin sur le PC de l'école pour modifier le fichier `hosts`, je peux soit accéder directement via l'IP `http://10.228.242.240/`, soit lancer Chrome ou Edge avec une règle de résolution DNS locale sans aucun droit admin dans PowerShell :
```powershell
Start-Process "chrome.exe" -ArgumentList '--user-data-dir="$env:TEMP\wp-chrome" --host-resolver-rules="MAP www.my-wordpress.ch 10.228.242.240" http://www.my-wordpress.ch'
```

### Test de fonctionnement

J'ouvre un navigateur ou je fais une requête curl vers `http://www.my-wordpress.ch` :
```bash
curl -I http://www.my-wordpress.ch/
```

WordPress répond bien avec un code HTTP `302 Found` et redirige vers `/wp-admin/install.php` pour lancer l'installation du site.

### Problème rencontré lors de la connexion (boucle infinie de login)

Après avoir installé WordPress, je n'arrivais pas à me connecter au tableau de bord avec mon compte admin `root`, la page se rechargeait en boucle avec l'erreur `&reauth=1` dans l'URL.

**Cause du problème :**  
Comme WordPress tourne avec 2 répliques (`worker1` et `worker2`), chaque conteneur avait généré son propre fichier `wp-config.php` avec des clés secrètes de session (`AUTH_KEY`, `AUTH_SALT`, etc.) différentes. Quand je me connectais sur `worker1`, Swarm envoyait ma requête suivante sur `worker2`, qui refusait le cookie car les clés de chiffrement ne correspondaient pas.

**Solution :**  
J'ai synchronisé le fichier `wp-config.php` entre `worker1` et `worker2` pour que les deux nœuds partagent les mêmes clés secrètes :
```bash
# Copie du wp-config.php de worker1 vers worker2
# Puis redémarrage du service WordPress
docker service update --force wp_stack_wordpress
```
Après le redémarrage du service, la connexion au tableau de bord `/wp-admin/` a fonctionné directement du premier coup.

### Déploiement de Swarmpit (Supervision du cluster Swarm)

Conformément au cahier des charges, l'outil open-source **Swarmpit** a été déployé pour superviser l'état du cluster Docker Swarm (services, conteneurs, nœuds, volumes, métriques).

1. **Prérequis matériel :**  
Le nœud manager héberge à la fois la base MariaDB de WordPress et les services de Swarmpit (Java/Clojure, CouchDB, InfluxDB). Pour éviter toute saturation mémoire et dépassement de délai (timeouts Swarm), le manager nécessite 2 Go de RAM (et 1 Go par worker).

2. **Fichier Compose Swarmpit (`/root/swarmpit/docker-compose.yml`) :**
Les services déployés sont :
- `app` : Application Web Swarmpit (`swarmpit/swarmpit:latest`) exposée sur les ports `888` et `8888`
- `db` : Base de données CouchDB (`couchdb:2.3.0`)
- `influxdb` : Base métriques temporelles (`influxdb:1.8`)
- `agent` : Agent déployé en mode global (`swarmpit/agent:latest`) sur l'ensemble des 3 nœuds

3. **Déploiement depuis le manager :**
```bash
cd /root/swarmpit
docker stack deploy -c docker-compose.yml swarmpit
```

4. **Vérification de l'état des services :**
```bash
docker stack services swarmpit
docker stack ps swarmpit
```
Tous les composants doivent afficher le statut `Running` :
- `swarmpit_app` : 1/1
- `swarmpit_db` : 1/1
- `swarmpit_influxdb` : 1/1
- `swarmpit_agent` : 3/3 (1 réplique par nœud)

5. **Accès au tableau de bord :**
Depuis le navigateur de la machine hôte :
- `http://10.228.242.207:8888` (ou `http://10.228.242.207:888`)
Créez votre compte administrateur lors de votre première connexion pour accéder à l'interface de gestion du cluster.

### Commandes pour supprimer totalement la stack et ses données

Pour supprimer complètement la stack Swarm ainsi que les volumes, secrets et réseaux associés :

```bash
# Suppression de la stack WordPress
docker stack rm wp_stack

# Suppression de la stack Swarmpit
docker stack rm swarmpit

# Suppression des volumes persistants
docker volume rm wp_stack_db_data wp_stack_wp_data swarmpit_db-data swarmpit_influx-data

# Suppression des secrets Docker
docker secret rm db_root_password db_password

# Suppression des réseaux overlay
docker network rm frontend backend swarmpit_net
```


