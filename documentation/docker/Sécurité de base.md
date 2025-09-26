# Sécurité de Base Docker

## Introduction

La sécurité des conteneurs Docker est un aspect critique qui nécessite une attention particulière dès la conception de vos applications. Docker fournit plusieurs mécanismes pour renforcer la sécurité, mais il est essentiel de comprendre les bonnes pratiques et de les appliquer systématiquement.

Cette documentation présente les principes fondamentaux de sécurité Docker, incluant l'exécution sécurisée des conteneurs, la gestion appropriée des secrets, et l'application du principe du moindre privilège pour minimiser les risques de sécurité.

## Concepts clés

### Principe du moindre privilège

Le **principe du moindre privilège** consiste à accorder uniquement les permissions minimales nécessaires au fonctionnement d'une application :
- **Utilisateur non-root** : Ne pas utiliser d'utilisateur privilégiés
- **Limitation des capacités** : Restreindre les capacités système avec `--cap-drop` et `--cap-add`
- **Isolation des ressources** : Limiter l'accès aux ressources système

### Gestion des images sécurisées

- **Images de base à jour** : Utiliser des images récentes avec les correctifs de sécurité
- **Images minimales** : Privilégier des images légères comme Alpine Linux
- **Scan de vulnérabilités** : Analyser régulièrement les images pour prendre connaissance des vulnérabilitées

### Gestion des secrets

Les **secrets** (mots de passe, clés API, certificats) ne doivent jamais être exposés :
- **Les secrets dans les images** : Ne pas inclure de secrets dans les layers
- **Variables d'environnement** : Solution simple mais avec limitations
- **Montage de fichiers** : Alternative plus sécurisée pour les secrets sensibles

## Cas d'usage

### Exécution avec utilisateur non-root

```dockerfile
# Dockerfile sécurisé
FROM node:16-alpine

# Créer un utilisateur non-privilégié
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nextjs -u 1001

# Copier et configurer l'application
COPY --chown=nextjs:nodejs .. /app
WORKDIR /app

# Passer à l'utilisateur non-root
USER nextjs

# Commande d'exécution
CMD ["node", "server.js"]
```

### Configuration sécurisée avec Docker Compose

```yaml
version: '3.8'
services:
  app:
    build: .
    user: "1001:1001"  # UID:GID non-root
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE  # Seulement si nécessaire
    read_only: true
    tmpfs:
      - /tmp
    security_opt:
      - no-new-privileges:true
    
  database:
    image: postgres:13-alpine
    user: "999:999"
    environment:
      - POSTGRES_DB=myapp
    volumes:
      - db_data:/var/lib/postgresql/data
    cap_drop:
      - ALL
    cap_add:
      - SETUID
      - SETGID
      - DAC_OVERRIDE

volumes:
  db_data:
```

### Limitation des capacités

```bash
# Supprimer toutes les capacités par défaut
docker run --cap-drop=ALL nginx

# Ajouter seulement les capacités nécessaires
docker run --cap-drop=ALL --cap-add=NET_BIND_SERVICE nginx

# Exemple avec plusieurs capacités
docker run --cap-drop=ALL \
  --cap-add=NET_BIND_SERVICE \
  --cap-add=CHOWN \
  --cap-add=SETUID \
  my-app
```

### Gestion sécurisée des secrets

```dockerfile
# ❌ MAUVAISE pratique - Secret dans l'image
FROM alpine
ENV API_KEY=secret123  # Visible dans l'historique 
RUN echo "password123" > /app/config  # Stocké dans les layers !

# ✅ BONNE pratique - Pas de secrets dans l'image
FROM alpine
# Configuration sans secrets
COPY app/ /app/
# Les secrets seront injectés à l'exécution
```

### Injection de secrets à l'exécution

```bash
# Via variables d'environnement (attention aux logs)
docker run -e DATABASE_PASSWORD=secret123 myapp

# Via fichiers montés (plus sécurisé)
echo "secret123" | docker secret create db_password -
docker service create --secret db_password myapp

# Avec Docker Compose et fichiers externes
docker run -v /host/secrets:/app/secrets:ro myapp
```

### Docker Compose avec secrets externes

```yaml
version: '3.8'
services:
  app:
    image: myapp:latest
    environment:
      - DATABASE_HOST=db
    secrets:
      - db_password
      - api_key
    volumes:
      - /etc/ssl/certs:/etc/ssl/certs:ro  # Certificats en lecture seule

secrets:
  db_password:
    file: ./secrets/db_password.txt
  api_key:
    external: true
    external_name: myapp_api_key
```

### Scan de sécurité et mise à jour

```bash
# Vérifier les vulnérabilités avec Docker Scout
docker scout cves myapp:latest

# Mise à jour des images de base
docker pull node:16-alpine
docker build --no-cache -t myapp:latest .

# Nettoyer les anciennes images
docker image prune -f
```

### Configuration de sécurité avancée

```bash
# Exécution avec options de sécurité renforcées
docker run \
  --user 1001:1001 \
  --cap-drop=ALL \
  --cap-add=NET_BIND_SERVICE \
  --security-opt=no-new-privileges:true \
  --read-only \
  --tmpfs /tmp \
  --tmpfs /var/run \
  myapp:latest
```

### Dockerfile optimisé pour la sécurité

```dockerfile
FROM node:16-alpine AS base

# Mise à jour des packages système
RUN apk update && apk upgrade && \
    apk add --no-cache dumb-init && \
    rm -rf /var/cache/apk/*

# Créer un utilisateur dédié
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nextjs -u 1001

FROM base AS dependencies
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production && npm cache clean --force

FROM base AS runtime
WORKDIR /app

# Copier les dépendances et le code
COPY --from=dependencies --chown=nextjs:nodejs /app/node_modules ./node_modules
COPY --chown=nextjs:nodejs . .

# Passer à l'utilisateur non-root
USER nextjs

# Utiliser dumb-init pour la gestion des signaux
ENTRYPOINT ["dumb-init", "--"]
CMD ["node", "server.js"]
```

### Validation des entrées et variables

```bash
# Script de validation dans le conteneur
#!/bin/sh
# validate-env.sh

if [ -z "$DATABASE_URL" ]; then
    echo "ERROR: DATABASE_URL is required"
    exit 1
fi

if [ -z "$API_KEY" ]; then
    echo "ERROR: API_KEY is required"
    exit 1
fi

# Valider le format des variables
if ! echo "$DATABASE_URL" | grep -qE '^postgresql://'; then
    echo "ERROR: Invalid DATABASE_URL format"
    exit 1
fi

echo "Environment validation passed"
```

## Conclusion

La sécurité Docker repose sur l'application rigoureuse de bonnes pratiques dès la conception. L'adoption du principe du moindre privilège, la gestion appropriée des secrets et la maintenance régulière des images sont essentielles pour maintenir un environnement sécurisé.

Points essentiels à retenir :
- Toujours exécuter les conteneurs avec des utilisateurs non-root
- Limiter les capacités système au strict nécessaire
- Ne jamais inclure de secrets dans les images Docker
- Maintenir les images de base à jour avec les correctifs de sécurité
- Utiliser des outils de scan pour détecter les vulnérabilités

## Ressources

- [Guide de sécurité Docker officiel](https://docs.docker.com/engine/security/)
- [Docker Bench Security](https://github.com/docker/docker-bench-security)
- [OWASP Container Security](https://owasp.org/www-project-container-security/)
- [CIS Docker Benchmark](https://www.cisecurity.org/benchmark/docker)