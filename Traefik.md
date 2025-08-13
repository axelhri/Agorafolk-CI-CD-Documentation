# Configuration Traefik

**[Traefik.yml](/Agorafolk-CI-CD-Documentation/docker/traefik/traefik.yml)**
**[openssl.cnf](/Agorafolk-CI-CD-Documentation/docker/traefik/traefik.yml)**
**[cert](/Agorafolk-CI-CD-Documentation/docker/traefik/certs/agorafolk.crt)**
**[key](/Agorafolk-CI-CD-Documentation/docker/traefik/certs/agorafolk.key)**

## traefik.yml

Ce fichier est la configuration statique de Traefik (chargée au démarrage).

**Entrées réseau (entryPoints)**

### Entrées réseau (entryPoints)

```yaml
entryPoints:
  web:
    address: ":80"
  websecure:
    address: ":443"
```

- `web` → écoute sur le port 80 (HTTP)

- `websecure` → écoute sur le port 443 (HTTPS)

- Ces noms (`web`, `websecure`) sont ensuite utilisés dans les labels Docker (`traefik.http.routers...entrypoints=websecure`).

### Providers

```yaml
providers:
  docker:
    endpoint: "unix:///var/run/docker.sock"
    exposedByDefault: false
```

- docker : Traefik va détecter automatiquement les conteneurs Docker et leurs labels

- `endpoint` : accès au socket Docker (`/var/run/docker.sock`)

- `exposedByDefault: false` : par défaut, Traefik n’expose pas un service.

  - Il faut mettre explicitement `traefik.enable=true `dans les labels pour l’activer (ce qui est fait dans `docker-compose.yml`).

### API et Dashboard

```yaml
api:
  dashboard: true
```

- Active l’interface web de Traefik (dashboard)

- Elle est accessible uniquement si un routeur est configuré pour la publier ou si `--api.insecure=true` est utilisé dans `docker-compose.yml` (ce qui est le cas → exposée sur le port `:8080`).

### TLS et certificats

```yaml
tls:
  certificates:
    - certFile: /certs/agorafolk.crt
      keyFile: /certs/agorafolk.key
```

- Active TLS (HTTPS) avec un certificat auto-signé

- `/certs/agorafolk.crt` → fichier du certificat public

- `/certs/agorafolk.key` → clé privée associée

- Ces fichiers sont montés dans le conteneur via le volume :

```yaml
- ./.docker/traefik/certs:/certs:ro
```

## openssl.cnf

Ce fichier sert à générer un certificat auto-signé correspondant à ton domaine local.

### Configuration générale

```ini
[req]
default_bits       = 2048
prompt             = no
default_md         = sha256
distinguished_name = dn
req_extensions     = req_ext
```

- default_bits : clé RSA de 2048 bits (sécurité standard)

- prompt = no : toutes les infos sont prises depuis ce fichier, pas d’interaction

- default_md : algorithme de hachage SHA-256

- distinguished_name = dn : la section `[dn]` définit les infos de l’organisation

- req_extensions = req_ext : ajoute les extensions définies dans `[req_ext]`

### Indentité du certificat

```ini
[dn]
C  = FR
ST = France
L  = Paris
O  = Agorafolk
CN = agorafolk.local
```

- C : pays (FR)

- ST : état/région (France)

- L : localité (Paris)

- O : organisation (Agorafolk)

- CN (Common Name) : nom principal du certificat (`agorafolk.local`)

### Extensions (SAN)

```ini
[req_ext]
subjectAltName = @alt_names

[alt_names]
DNS.1 = agorafolk.local
DNS.2 = web.agorafolk.local
DNS.3 = api.agorafolk.local
```

- **subjectAltName** (SAN) : liste des noms de domaine que le certificat couvre

- Ici :

  - agorafolk.local (domaine principal)

  - web.agorafolk.local

  - api.agorafolk.local

Ces SAN permettent d’utiliser le même certificat pour plusieurs sous-domaines.

### Fonctionnement global

1. OpenSSL (via openssl.cnf) génère un certificat auto-signé avec plusieurs noms (agorafolk.local, web.agorafolk.local, api.agorafolk.local).

2. Ce certificat est placé dans .docker/traefik/certs/.

3. Traefik charge ce certificat et écoute en HTTPS sur :443 (websecure).

4. Quand une requête arrive :

   - Traefik choisit le bon service en fonction des labels Docker

   - Il gère la terminaison TLS (présente le certificat au client)

   - Il transmet la requête au conteneur cible (ex. Nginx).
