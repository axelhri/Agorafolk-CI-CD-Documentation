# CD Pipeline script

**[CD-Script](/.github/workflows/cd.yml)**

Ce pipeline ne se déclenche que si la CI a réussi.

## Déclencheur

```yaml
on:
  workflow_run:
    workflows: ["ci"]
    types:
      - completed
```

- **workflow_run :** écoute les résultats d’un autre workflow (ici `"ci"`, qui devrait correspondre à `"CI Pipeline"` — il faudrait s'assurer que le nom matche).

- **if :**

```yaml
if: ${{ github.event.workflow_run.conclusion == 'success' }}
```

→ s’exécute uniquement si la CI a réussi.

## Job build

1. Récupération du code

```yaml
- uses: actions/checkout@v4
```

2. Connexion à Docker Hub

```yaml
- name: Docker login
  run: echo "${{ secrets.DOCKER_PASSWORD }}" | docker login -u "${{ secrets.DOCKER_USERNAME }}" --password-stdin
```

Utilise les secrets GitHub pour se connecter à Docker Hub.

3. Construction de l'image Docker

```yaml
- name: Build the Docker image
  run: docker build -t ${{ secrets.DOCKER_USERNAME }}/agorafolk-back .
```

Tagge l’image avec `utilisateur/agorafolk-back`.

4. Push de l'image sur Docker Hub

```yaml
- name: Push the image
  run: docker push ${{ secrets.DOCKER_USERNAME }}/agorafolk-back
```
