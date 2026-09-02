
# Déploiement WordPress sur Docker Swarm

Documentation étape par étape du déploiement de l'infrastructure Swarm à 3 nœuds.

## 1. Initialisation du cluster Docker Swarm

### Sur la VM Manager
Initialisation du cluster sur l'interface réseau locale :
```sh
docker swarm init --advertise-addr 10.228.242.211

```

### Sur les VM Workers (worker1 et worker2)

Connexion des deux nœuds workers au cluster :

```sh
docker swarm join --token <TOKEN_OBTENU_SUR_LE_MANAGER> 10.228.242.211:2377

```

### Vérification du cluster (Manager)

```sh
docker node ls

```

---

## 2. Configuration des réseaux Overlay

Création des réseaux isolés pour la segmentation frontend et backend :

```sh
# Réseau pour la communication Nginx <-> WordPress
docker network create --driver overlay frontend

# Réseau isolé pour la communication WordPress <-> MariaDB
docker network create --driver overlay backend

```

---

## 3. Configuration des secrets Docker

Création des identifiants chiffrés pour la base de données :

```sh
# Mot de passe root MariaDB
echo "k8#mZ9!vP4$wQ2@xL" | docker secret create db_root_password -

# Mot de passe utilisateur MariaDB pour WordPress
echo "tY7&rE3*uN9%sD1!fG" | docker secret create db_password -

```

Vérification des secrets créés :

```sh
docker secret ls

```

```

```