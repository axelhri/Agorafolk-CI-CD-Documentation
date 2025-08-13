# Docker Compose

**[docker-compose](/Agorafolk-CI-CD-Documentation/docker-compose.yaml)**

## Service `php`

```yaml
php:
  build:
    context: .
    dockerfile: ./.docker/Dockerfile
  container_name: symfony_php
  working_dir: /var/www
  volumes:
    - .:/var/www
  depends_on:
    - database
  environment:
    APP_ENV: dev
```

**Rôle :**  
C'est le conteneur qui fait tourner PHP et Symfony.

- build (context & dockerfile) : le conteneur est construit à partir du Dockerfile situé dans `.docker/Dockerfile`

- container_name : nom fixe `symfony_php` pour simplifier les commandes (`docker exec symfony_php`)

- working_dir : `/var/www` devient le répertoire courant par défaut

- volumes : monte le projet local(`.`) dans `/var/www` pour que les modifications soient visibles sans rebuild

- depends_on : démarre seulement après la base de données

- environment (APP_ENV) : définit Symfony en mode développement (`dev`)

## Service `nginx`

```yaml
nginx:
  image: nginx:stable-alpine
  container_name: symfony_nginx
  ports:
    - "8002:80"
  volumes:
    - .:/var/www
    - ./.docker/nginx/conf.d:/etc/nginx/conf.d:ro
  labels:
    - "traefik.enable=true"
    ...
```

**Rôle :**  
C'est le serveur web qui sert les ficihiers Symfony et relaie les requêtes à PHP-FPM

- image : utilise la version stable et légère de `Nginx`

- ports : exposé localement sur `8002` -> `http://localhost:8002`

- volumes :

  - code source monté dans `/var/www`
  - configuration Nginx depuis `.docker/nginx/conf.d` (en lecture seule `:ro`)

- labels Traefik : permettent à Traefik de router le trafic HTTPS vers ce conteneur
  - `traefik.http.routers.api` -> `api.agorafolk.local`
  - `traefik.http.routers.web` -> `web.agorafolk.local`
  - `traefik.http.services.nginx.loadbalancer.server.port=80` -> indique à Traefik d’envoyer le trafic vers le port interne 80 d’Nginx

## Service `traefik`

```yaml
traefik:
  image: traefik:v2.11
  container_name: traefik
  restart: always
  command:
    - "--api.insecure=true"
    - "--providers.docker=true"
    - "--entrypoints.web.address=:80"
    - "--entrypoints.websecure.address=:443"
  ports:
    - "443:443"
    - "80:80"
  volumes:
    - /var/run/docker.sock:/var/run/docker.sock
    - ./.docker/traefik/traefik.yml:/etc/traefik/traefik.yml:ro
    - ./.docker/traefik/certs:/certs:ro
```

**Rôle :**
Reverse proxy et gestionnaire de certificats HTTPS.

- image : version 2.11 de Traefik

- command :

  - --api.insecure=true : active l’interface d’admin sur :8080 (non sécurisé, à éviter en prod)
  - --providers.docker=true : détecte automatiquement les services via leurs labels
  - --entrypoints.web et websecure : définit les ports d’écoute (80 HTTP, 443 HTTPS)

- ports : expose HTTP et HTTPS à l’extérieur

- volumes :
  - accès au socket Docker pour détecter les conteneurs
  - configuration statique (traefik.yml)
  - certificats SSL (certs)

## Service `database`

```yaml
database:
  image: postgres:${POSTGRES_VERSION:-16}-alpine
  environment:
    POSTGRES_DB: ${POSTGRES_DB:-app}
    POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-!ChangeMe!}
    POSTGRES_USER: ${POSTGRES_USER:-app}
  volumes:
    - database_data:/var/lib/postgresql/data:rw
```

**Rôle :**
Base de données PostgreSQL pour l’application Symfony.

- image : version Alpine (légère), configurable via variable d’environnement

- environment :

  - nom de la DB
  - utilisateur et mot de passe (à changer en prod)

- volumes : database_data stocke les données de façon persistante même si le conteneur est recréé

## Volumes et réseaux

```yaml
volumes:
  database_data:

networks:
  default:
```

- database_data : volume persistant pour PostgreSQL

- network default : tous les services partagent un réseau Docker interne par défaut (communication via nom de service)

## Schéma de fonctionnement

1. Traefik écoute sur 80 et 443 → envoie le trafic aux bons conteneurs selon le domaine (api.agorafolk.local ou web.agorafolk.local).

2. Nginx sert les requêtes et envoie celles en PHP vers le service php.

3. PHP exécute Symfony et interagit avec PostgreSQL.
