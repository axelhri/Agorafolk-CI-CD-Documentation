# Nginx

## Bloc `server`

```nginx
server {
    listen 80;
    server_name api.agorafolk.local web.agorafolk.local;
```

- **listen 80** → Nginx écoute en HTTP sur le port 80 (dans le conteneur, Traefik ou Docker port-mapping redirige vers lui).

- **server_name** → ce serveur répond aux domaines :

  - `api.agorafolk.local`

  - `web.agorafolk.local`

## Racine et fichier index

```nginx
    root /var/www/public;
    index index.php index.html;
```

- **root** → répertoire racine du site : `/var/www/public` (dossier public de Symfony).

- **index** → fichiers d’index par défaut (priorité à `index.php`, sinon `index.html`).

## Gestion des requçetes principales

```nginx
    location / {
        try_files $uri /index.php$is_args$args;
    }
```

- **location /** → toutes les requêtes HTTP qui ne correspondent pas à un autre bloc.

- **try_files $uri** →

  - Si le fichier demandé existe (`$uri`), Nginx le sert directement.

  - Sinon, redirection vers `index.php` (Symfony gère la route).

  - `$is_args$args` → conserve les paramètres GET (`?page=...`).

## Traitement PHP-FPM

```nginx
    location ~ ^/index\.php(/|$) {
        fastcgi_pass symfony_php:9000;
        fastcgi_split_path_info ^(.+\.php)(/.*)$;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param HTTPS on;
        internal;
    }
```

- **location ~ ^/index.php(/|$)** → capture uniquement `index.php` (et éventuellement un chemin après).

- **fastcgi_pass symfony_php:9000** → envoie la requête PHP au conteneur `symfony_php` (service `php` dans Docker) sur le port 9000 (PHP-FPM).

- **fastcgi_split_path_info** → sépare le chemin du script (`.php`) et le reste de l’URL.

- **include fastcgi_params** → charge les paramètres standards FastCGI.

- **fastcgi_param SCRIPT_FILENAME ...** → indique à PHP-FPM le chemin complet du script exécuté.

- **fastcgi_param HTTPS on** → informe PHP que la requête est HTTPS (important car Traefik termine le SSL).

- **internal** → cette route ne peut pas être appelée directement par le client, seulement en interne via `try_files`.

## Sécurité : fichiers cachés

```nginx
    location ~ /\. {
        deny all;
    }
```

- Bloque l’accès aux fichiers/dossiers commençant par `.` (ex. `.env`, `.git`).

## Caching des assets statiques

```nginx
    location ~* \.(?:css|js|jpg|jpeg|gif|png|ico|svg|woff|woff2|ttf|eot)$ {
        expires 1M;
        access_log off;
        add_header Cache-Control "public";
    }
```

- **location ~\*** → insensible à la casse (`.CSS`, `.Js`, etc.).

- **Fichiers concernés** : CSS, JS, images, polices, etc.

- **expires 1M** → met en cache côté navigateur pendant 1 mois.

- **access_log off** → n’enregistre pas les requêtes pour ces fichiers dans les logs (gain de performance).

- **add_header Cache-Control "public"** → permet au cache de proxy/CDN de stocker la ressource.

## Schéma de fonctionnement

1. **Traefik** reçoit la requête HTTPS → l’envoie à Nginx sur port 80 interne.

2. **Nginx** :

   - Sert directement les fichiers statiques s’ils existent (`try_files`).

   - Sinon, envoie vers `index.php` → `PHP-FPM` dans le conteneur `symfony_php`.

3. Symfony prend le relais pour générer la réponse.
