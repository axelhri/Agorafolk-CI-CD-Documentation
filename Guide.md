# Guide d’utilisation

## 1. Prérequis

Avant de commencer, assure-toi d’avoir :

- Docker et Docker Compose installés

- OpenSSL installé (pour générer le certificat si besoin)

- Les noms de domaine locaux (`agorafolk.local`, `web.agorafolk.local`, `api.agorafolk.local`) ajoutés dans ton fichier `hosts

## 2. Configurer les domaines locaux

Ouvre ton fichier `/etc/hosts` (**Linux/Mac**) ou `C:\Windows\System32\drivers\etc\hosts` (**Windows**) et ajoute :

```lua
127.0.0.1   agorafolk.local
127.0.0.1   web.agorafolk.local
127.0.0.1   api.agorafolk.local
```

## 3. Générer le certificat TLS local

Si tu n’as pas encore le certificat dans `.docker/traefik/certs/`, exécute :

```bash
mkdir -p .docker/traefik/certs

openssl req -x509 -nodes -days 365 \
  -newkey rsa:2048 \
  -keyout .docker/traefik/certs/agorafolk.key \
  -out .docker/traefik/certs/agorafolk.crt \
  -config .docker/traefik/openssl.cnf
```

Ce certificat est **auto-signé** — ton navigateur te demandera de l’accepter manuellement.

## 4. Lancer l'environnement Docker

```bash
docker-compose up -d
```

## 5. Installer les dépendances Symfony

```bash
docker exec -it symfony_php composer install
```

- Cela installe toutes les dépendances PHP listées dans `composer.json`.

## 6. Préparer la base de données

```bash
docker exec -it symfony_php php bin/console doctrine:database:create
docker exec -it symfony_php php bin/console doctrine:migrations:migrate
docker exec -it symfony_php php bin/console doctrine:fixtures:load
```

- Crée la base **PostgreSQL** et applique les migrations.

## 7. Accéder à l'application

- **Front-end / Web** → https://web.agorafolk.local

- **API** → https://api.agorafolk.local

- **Dashboard Traefik** → http://localhost:8080
