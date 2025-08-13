# CI Pipeline script

## Structure générale

```yaml
name: CI Pipeline

on:
  push: # déclenché lors d’un push sur certaines branches
    branches: [main, develop]
  pull_request: # déclenché lors d’une PR vers ces branches
    branches: [main, develop]
```

**But :** Lancer automatiquement les tests et l’analyse de code pour éviter que du code cassé ou non conforme ne soit intégré.

## Job test

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
```

- **runs-on :** indique que le job tourne sur un runner GitHub basé sur Ubuntu.

## Services (dépendances externes)

```yaml
services:
  postgres:
    image: postgres:16-alpine
    env:
      POSTGRES_DB: test
      POSTGRES_USER: test
      POSTGRES_PASSWORD: ${{ secrets.POSTGRES_PASSWORD || 'test_secret' }}
    ports:
      - 5432:5432
    options: >-
      --health-cmd="pg_isready -U test -d test"
      --health-interval=10s
      --health-timeout=5s
      --health-retries=5

  redis:
    image: redis:alpine
    ports:
      - 6379:6379
```

**postgres :** base de données Postgres utilisée pour les tests.

**redis :** serveur Redis pour la mise en cache ou la file d’attente.

**options :** vérifie que Postgres est prêt avant de lancer les tests.

L’utilisation de `secrets.POSTGRES_PASSWORD` sécurise le mot de passe en production. Si le secret n’est pas défini, il utilise `test_secret`.

## Étapes (steps)

1. Récupération du code

```yaml
- uses: actions/checkout@v4
```

Clone le repo dans le runner.

2. Installation de PHP

```yaml
- name: Setup PHP
  uses: shivammathur/setup-php@v2
  with:
    php-version: "8.4"
    extensions: mbstring, xml, ctype, iconv, intl, pdo_pgsql, redis
    coverage: xdebug
```

Installe PHP 8.4 avec les extensions nécessaires et Xdebug pour la couverture de tests.

3. Cache des dépendances Composer

```yaml
- name: Cache Composer packages
  uses: actions/cache@v3
```

Évite de réinstaller toutes les dépendances à chaque run → gain de temps.

4. Installation des dépendances

```yaml
- run: composer install --prefer-dist --no-progress
```

5. Configuration des variables d'environnement

```yaml
- run: cp .env.test .env.local
```

Copie un `.env` de test.

6. Analyse statique

```yaml
- run: php ./vendor/bin/phpstan analyze
```

Détecte les erreurs potentielles sans exécuter le code.

7. Lint Symfony

```yaml
- run: php bin/console lint:container --env=prod
```

Vérifie que la configuration Symfony est correcte.

8. Exécution des tests

```yaml
- run: php ./bin/phpunit
  env:
    DATABASE_URL: "postgresql://..."
    REDIS_URL: "redis://..."
```

Lance PHPUnit avec les URLs vers Postgres et Redis configurées.
