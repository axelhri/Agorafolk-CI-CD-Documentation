# Dockerfile

## Image de base

```docker
FROM php:8.3-fpm-alpine
```

- **php:8.3-fpm-alpine :**

  - PHP 8.3

  - mode FPM (FastCGI Process Manager) → optimisé pour servir via Nginx

  - version Alpine → image très légère (~5 Mo de base)

## Installation des dépendances système

```docker
RUN apk add --no-cache \
    bash \
    git \
    zip \
    unzip \
    libzip-dev \
    icu-dev \
    oniguruma-dev \
    libpq-dev \
    && docker-php-ext-install \
        pdo \
        pdo_pgsql \
        intl \
        zip \
        opcache
```

- **apk add --no-cache :** installe des paquets Alpine Linux sans garder de cache (économie de taille)

- **Paquets installés :**

  - bash → shell avancé (plus confortable que sh)

  - git → gestion de versions

  - zip, unzip → compression/décompression

  - libzip-dev → nécessaire pour l’extension PHP zip

  - icu-dev → nécessaire pour intl (internationalisation)

  - oniguruma-dev → nécessaire pour le moteur regex de PHP

  - libpq-dev → librairie cliente PostgreSQL pour PHP

- **docker-php-ext-install :** compile et active des extensions PHP :

  - pdo et pdo_pgsql → accès à PostgreSQL via PDO

  - intl → gestion des formats de date, nombre, texte selon la locale

  - zip → gestion des archives ZIP

  - opcache → amélioration des performances PHP

## Installation de Composer

```docker
RUN curl -sS https://getcomposer.org/installer \
    | php -- --install-dir=/usr/local/bin --filename=composer
```

- Télécharge et installe Composer globalement sous /usr/local/bin/composer

- Composer = gestionnaire de dépendances PHP (indispensable pour Symfony)

## Répoertoire de travail

```docker
WORKDIR /var/www
```

- Définit /var/www comme dossier de travail par défaut dans le conteneur

- Toute commande docker exec ou docker run commencera ici

- C’est aussi là que le volume du code source sera monté dans le docker-compose.yml

## Résumé

Ce Dockerfile :

1. Part d’une base PHP 8.3 FPM légère

2. Ajoute les outils nécessaires pour un projet Symfony avec PostgreSQL

3. Installe Composer pour gérer les dépendances PHP

4. Prépare un répertoire de travail /var/www
