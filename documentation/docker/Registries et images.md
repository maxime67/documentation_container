# Registries et Images Docker

## Introduction

Les registries Docker sont des dépôts centralisés qui stockent et distribuent les images de conteneurs. Docker Hub est le registry public par défaut, mais il existe également des registries privés et alternatifs. La gestion des images et de leurs tags est cruciale pour maintenir des déploiements cohérents et sécurisés.

Cette documentation couvre la recherche, le téléchargement, la gestion des tags, ainsi que les meilleures pratiques pour choisir et utiliser les images Docker appropriées selon vos besoins.

## Concepts clés

### Docker Hub et registries

**Docker Hub** est le registry public officiel de Docker qui contient :
- **Images officielles** : Maintenues par Docker Inc., vérifiées et optimisées
- **Images communautaires** : Créées par des utilisateurs et organisations
- **Images privées** : Accessibles uniquement aux comptes autorisés

### Tags et versioning

Les **tags** permettent d'identifier différentes versions d'une même image :
- **latest** : Pointe vers la version la plus récente (par défaut)
- **Versions spécifiques** : Ex. `nginx:1.21.6`, `node:16.14`
- **Tags descriptifs** : Ex. `alpine`, `slim`, `bullseye`

### Images de base courantes

- **Systèmes d'exploitation** : ubuntu, debian, alpine, centos
- **Applications** : nginx, apache, mysql, postgres, redis
- **Langages de programmation** : node, python, java, golang

## Cas d'usage

### Recherche et téléchargement d'images

```bash
# Rechercher une image sur Docker Hub
docker search nginx

# Télécharger une image
docker pull nginx:latest
docker pull nginx:1.21.6-alpine

# Lister les images locales
docker images

# Voir les détails d'une image
docker inspect nginx:latest
```

### Gestion des tags

```bash
# Convention de nommage
registry.com/namespace/repository:tag

# Exemples de tags
docker pull nginx:latest           # Version la plus récente
docker pull nginx:1.21            # Version majeure.mineure
docker pull nginx:1.21.6          # Version spécifique
docker pull nginx:alpine          # Variante basée sur Alpine Linux

# Créer un tag local
docker tag nginx:latest monapp/nginx:v1.0

# Pousser vers un registry
docker push monregistry.com/monapp/nginx:v1.0
```

### Choix d'images appropriées

```bash
# Image Ubuntu complète (plus lourde)
FROM ubuntu:20.04

# Image Alpine (légère, ~5MB)
FROM alpine:3.16

# Image officielle Node.js
FROM node:16-alpine

# Image officielle avec tag spécifique
FROM postgres:13.8-alpine
```

### Authentification et registries privés

```bash
# Connexion à Docker Hub
docker login

# Connexion à un registry privé
docker login private-registry.com

# Utilisation d'un registry privé
docker pull private-registry.com/myapp/api:v2.1
docker tag myapp:latest private-registry.com/myapp/api:v2.1
docker push private-registry.com/myapp/api:v2.1
```

### Exemple d'utilisation avec Docker Compose

```yaml
version: '3.8'
services:
  web:
    image: nginx:1.21.6-alpine
    ports:
      - "80:80"
    
  database:
    image: postgres:13.8-alpine
    environment:
      - POSTGRES_DB=myapp
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=password
    
  cache:
    image: redis:7-alpine
    command: redis-server --appendonly yes
```

### Meilleures pratiques pour les tags

```dockerfile
# Dockerfile avec image de base spécifique
FROM node:16.14.2-alpine3.15

# Éviter 'latest' en production
# ❌ Mauvais
FROM node:latest

# ✅ Bon
FROM node:16.14.2-alpine3.15
```

## Conclusion

La maîtrise des registries et des images Docker est essentielle pour un développement et un déploiement efficaces. L'utilisation judicieuse des tags permet de garantir la reproductibilité des environnements et la stabilité des applications.

Points essentiels à retenir :
- Privilégier les images officielles pour la sécurité et la fiabilité
- Utiliser des tags spécifiques en production pour éviter les surprises
- Choisir des images adaptées à vos besoins (Alpine pour la légèreté)
- Comprendre la différence entre images officielles et communautaires

Pour aller plus loin, explorez les registries privés, la signature d'images et les outils de scanning de vulnérabilités pour sécuriser votre chaîne d'approvisionnement d'images.

## Ressources

- [Docker Hub](https://hub.docker.com/)
- [Documentation des images officielles](https://docs.docker.com/docker-hub/official_images/)
- [Meilleures pratiques pour les images](https://docs.docker.com/develop/dev-best-practices/)
- [Guide des registries privés](https://docs.docker.com/registry/)