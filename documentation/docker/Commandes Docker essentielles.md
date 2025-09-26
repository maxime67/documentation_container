# Dockerfile et construction d'images

## Introduction

Le Dockerfile est le fichier de configuration central qui définit comment construire une image Docker. Il contient une série d'instructions qui sont exécutées séquentiellement pour créer une image personnalisée à partir d'une image de base.

La maîtrise du Dockerfile et des bonnes pratiques de construction est essentielle pour créer des images optimisées, sécurisées et maintenables. Cette section couvre les instructions fondamentales, les techniques d'optimisation et les stratégies pour réduire la taille des images finales.

## Concepts clés

### Instructions Dockerfile essentielles

**FROM** : Image de base
```dockerfile
# Image officielle depuis Docker Hub
FROM node:16-alpine

# Image spécifique avec tag
FROM ubuntu:20.04

# Image depuis un registry privé
FROM mon-registry.com/base-image:latest

# Multi-stage build
FROM node:16 AS build-stage
FROM nginx:alpine AS production-stage
```

**RUN** : Exécuter des commandes lors du build
```dockerfile
# Commande simple
RUN apt-get update

# Commandes multiples (optimisation des layers)
RUN apt-get update && \
    apt-get install -y curl wget && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

# Forme exec (recommandée pour scripts)
RUN ["npm", "start"]
```

**COPY et ADD** : Copier des fichiers
```dockerfile
# COPY (recommandé) - copie simple
COPY package.json /app/
COPY src/ /app/src/

# COPY avec pattern
COPY *.json /app/

# ADD - avec fonctionnalités étendues (extraction d'archives)
ADD https://example.com/file.tar.gz /tmp/
ADD archive.tar.gz /app/

# Copie avec utilisateur spécifique
COPY --chown=node:node package.json /app/
```

**WORKDIR** : Définir le répertoire de travail
```dockerfile
# Définir le répertoire de travail
WORKDIR /app

# Créer et utiliser un répertoire
WORKDIR /app/backend

# Utilisation de variables
WORKDIR ${APP_HOME}
```

**EXPOSE** : Documenter les ports
```dockerfile
# Port TCP (par défaut)
EXPOSE 8080

# Port UDP explicite
EXPOSE 53/udp

# Multiples ports
EXPOSE 80 443 8080
```

**ENV** : Variables d'environnement
```dockerfile
# Variable simple
ENV NODE_ENV production

# Multiples variables
ENV NODE_ENV=production \
    PORT=3000 \
    DEBUG=false

# Variable avec valeur par défaut
ENV API_URL=${API_URL:-http://localhost:3000}
```

**CMD vs ENTRYPOINT** : Commande par défaut
```dockerfile
# CMD - peut être surchargée par docker run
CMD ["npm", "start"]
CMD npm start  # forme shell

# ENTRYPOINT - point d'entrée fixe
ENTRYPOINT ["node", "server.js"]

# Combinaison ENTRYPOINT + CMD
ENTRYPOINT ["node"]
CMD ["server.js"]  # argument par défaut

# ENTRYPOINT avec script
ENTRYPOINT ["./docker-entrypoint.sh"]
```

### Bonnes pratiques de construction

**Optimisation des layers**
```dockerfile
# ❌ Mauvais - chaque RUN crée une layer
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get install -y wget
RUN apt-get clean

# ✅ Bon - regroupement des commandes
RUN apt-get update && \
    apt-get install -y \
        curl \
        wget && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*
```

**Utilisation du cache Docker**

```dockerfile
# ✅ Copier package.json en premier pour cache des dépendances
COPY package.json package-lock.json ./
RUN npm ci --only=production

# Copier le code ensuite
COPY .. .
```

**Fichier .dockerignore**
```dockerignore
# Fichiers de développement
node_modules
npm-debug.log
.git
.gitignore

# Fichiers de build
dist/
build/
*.log

# Documentation
README.md
docs/

# Fichiers sensibles
.env
*.key
```

**Images de base minimales**
```dockerfile
# Alpine Linux (très légère)
FROM node:16-alpine

# Distroless (sans shell)
FROM gcr.io/distroless/nodejs:16

# Scratch (vide - pour binaires statiques)
FROM scratch
COPY app /app
ENTRYPOINT ["/app"]
```

**Principe du single responsibility**
```dockerfile
# ✅ Un service par conteneur
FROM nginx:alpine
COPY nginx.conf /etc/nginx/nginx.conf
COPY html/ /usr/share/nginx/html/
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

### Build context et optimisation

**Comprendre le contexte de build**
```bash
# Le contexte = répertoire envoyé au daemon Docker
docker build .  # Contexte = répertoire courant

# Spécifier un contexte différent
docker build -f ./docker/Dockerfile ./src

# URL Git comme contexte
docker build https://github.com/user/repo.git#branch
```

**Minimiser la taille du contexte**
```dockerfile
# Structure optimale du projet
project/
├── .dockerignore    # Exclure fichiers inutiles
├── Dockerfile
├── src/            # Code source uniquement
└── package.json
```

**Ordre optimal des instructions**
```dockerfile
# 1. Image de base (change rarement)
FROM node:16-alpine

# 2. Variables d'environnement globales
ENV NODE_ENV=production

# 3. Installation des dépendances système
RUN apk add --no-cache dumb-init

# 4. Création utilisateur
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nextjs -u 1001

# 5. Répertoire de travail
WORKDIR /app

# 6. Fichiers de dépendances (cache layer)
COPY package*.json ./

# 7. Installation dépendances
RUN npm i --only=production

# 8. Code source (change souvent)
COPY . .

# 9. Configuration runtime
USER nextjs
EXPOSE 3000

# 10. Commande de démarrage
CMD ["dumb-init", "node", "server.js"]
```

**Multi-stage builds**
```dockerfile
# Stage 1: Build
FROM node:22 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm i
COPY . .
RUN npm run build

# Stage 2: Production
FROM node:22-alpine AS production
WORKDIR /app
COPY package*.json ./
RUN npm i --only=production && npm cache clean --force
# On récupère le résultat du traitement effectué sur le conteneur de build
COPY --from=builder /app/dist ./dist
USER node
EXPOSE 3000
CMD ["node", "dist/server.js"]
```

## Cas d'usage

### Application Nextjs optimisée
```dockerfile
# =============================================================================
# IMAGE DE BASE - Partagée entre tous les stages pour la cohérence
# =============================================================================
FROM node:22-alpine AS base

# Installation des dépendances système nécessaires pour certains packages npm
RUN apk add --no-cache libc6-compat

WORKDIR /app

# Création d'un utilisateur non-root pour la sécurité
# - nodejs (groupe) : GID 1001
# - nextjs (utilisateur) : UID 1001, membre du groupe nodejs
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nextjs -u 1001 -G nodejs

# =============================================================================
# STAGE DE BUILD - Installation complète des dépendances et compilation
# =============================================================================
FROM base AS builder

# OPTIMISATION: Copier d'abord package.json pour exploiter le cache Docker
# Si package.json n'a pas changé, Docker réutilise cette layer en cache
COPY ./mon-app-next/package*.json ./

# npm ci est plus fiable que npm i pour les environnements de production
# - Installe exactement les versions du package-lock.json
# - Plus rapide et déterministe
RUN npm ci

# Copier le code source APRÈS l'installation des dépendances
# Permet de reconstruire seulement si le code change (pas les deps)
COPY ./mon-app-next .

# Exclure les fichiers inutiles avec .dockerignore :
# node_modules, .git, .next, coverage, etc.

# Construction de l'application Next.js
# Génère les fichiers optimisés dans .next/
RUN npm run build

# =============================================================================
# STAGE DE PRODUCTION - Image finale allégée
# =============================================================================
FROM base AS production

# Variables d'environnement pour optimiser Node.js en production
ENV NODE_ENV=production
ENV NEXT_TELEMETRY_DISABLED=1

# OPTIMISATION: Installer seulement les dépendances de production
# --omit=dev exclut devDependencies (TypeScript, ESLint, etc.)
COPY ./mon-app-next/package*.json ./
RUN npm ci --omit=dev && \
    npm cache clean --force

# Copier les artefacts de build depuis le stage builder
# IMPORTANT: .next contient l'application compilée
COPY --from=builder --chown=nextjs:nodejs /app/.next ./.next

# Copier les fichiers statiques (images, favicon, etc.)
COPY --from=builder --chown=nextjs:nodejs /app/public ./public

# SÉCURITÉ: Basculer vers l'utilisateur non-root
USER nextjs

# Port par défaut de Next.js
EXPOSE 3000

CMD ["npm", "start"]

```

### Application Python avec dépendances
```dockerfile
# Version simplifiée sans multi-stage (mais optimisée)
FROM python:3.8-alpine

# Variables d'environnement
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PIP_NO_CACHE_DIR=1

# Installer les dépendances système
RUN apk add --no-cache --virtual .build-deps gcc musl-dev libffi-dev openssl-dev \
    && apk add --no-cache libffi openssl

# Copier requirements et installer les dépendances
COPY requirements.txt .
RUN pip install --upgrade pip \
    && pip install -r requirements.txt \
    && apk del .build-deps

# Créer utilisateur non-root
RUN addgroup -g 1001 -S appgroup && \
    adduser -u 1001 -S appuser -G appgroup

# Configurer le répertoire de travail
WORKDIR /app
COPY --chown=appuser:appgroup . /app

# Passer à l'utilisateur non-root
USER appuser

EXPOSE 8000
CMD ["python", "app.py"]
```

### Application avec assets statiques
```dockerfile
# Build des assets
FROM node:16 AS frontend
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Serveur web
FROM nginx:alpine
COPY --from=frontend /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/nginx.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

## Conclusion

La construction d'images Docker efficaces nécessite une compréhension approfondie des instructions Dockerfile et des mécanismes de cache. L'application des bonnes pratiques permet de créer des images légères, sécurisées et performantes.

Les multi-stage builds représentent une technique avancée particulièrement utile pour séparer les environnements de build et de production, réduisant significativement la taille des images finales. L'optimisation du contexte de build et l'utilisation judicieuse du cache Docker accélèrent considérablement les temps de construction.

Ces compétences sont essentielles pour créer des images de qualité production, optimisées pour les déploiements à grande échelle et les pipelines CI/CD automatisés.

## Ressources

- [Référence Dockerfile](https://docs.docker.com/engine/reference/builder/)
- [Bonnes pratiques Dockerfile](https://docs.docker.com/develop/dev-best-practices/)
- [Multi-stage builds](https://docs.docker.com/develop/dev-best-practices/#use-multi-stage-builds)
- [Optimisation des images Docker](https://docs.docker.com/develop/dev-best-practices/#keep-images-small)
- [Guide .dockerignore](https://docs.docker.com/engine/reference/builder/#dockerignore-file)
- [Images distroless de Google](https://github.com/GoogleContainerTools/distroless)