# Workflows de Développement Docker

## Introduction

L'intégration de Docker dans les workflows de développement transforme la manière dont les équipes créent, testent et déploient leurs applications. Docker permet de standardiser les environnements de développement, faciliter la collaboration entre développeurs, et garantir la cohérence entre les environnements de développement, test et production.

Cette documentation présente les meilleures pratiques pour intégrer Docker dans votre cycle de développement, optimiser la reconstruction d'images, et mettre en place des workflows efficaces pour améliorer la productivité des équipes de développement.

## Concepts clés

### Cycle de développement avec Docker

Le cycle de développement conteneurisé comprend plusieurs phases :
- **Développement local** : Environnement isolé et reproductible
- **Reconstruction itérative** : Build optimisé pour les modifications fréquentes
- **Tests conteneurisés** : Validation dans un environnement identique à la production
- **Debugging intégré** : Outils de debug adaptés aux conteneurs

### Séparation des environnements

La distinction entre développement et production nécessite :
- **Multi-stage builds** : Dockerfile avec étapes séparées
- **Configuration flexible** : Variables d'environnement et fichiers de configuration
- **Optimisations spécifiques** : Images légères en production, outils de debug en développement

### Intégration continue

Docker facilite l'automatisation avec :
- **Tests automatisés** : Exécution dans des conteneurs éphémères
- **Pipeline CI/CD** : Construction et déploiement automatisés
- **Environnements de test** : Création rapide d'environnements isolés

## Cas d'usage

### Développement local avec hot reload

```dockerfile
# Dockerfile.dev
FROM node:16-alpine

WORKDIR /app

# Installation des dépendances
COPY package*.json ./
RUN npm install

# Installation d'outils de développement
RUN npm install -g nodemon

# Le code source sera monté en volume
VOLUME ["/app/src"]

# Port de développement
EXPOSE 3000

# Commande avec hot reload
CMD ["nodemon", "--watch", "src", "src/index.js"]
```

```yaml
# docker-compose.dev.yml
version: '3.8'
services:
  app:
    build:
      context: ..
      dockerfile: Dockerfile.dev
    ports:
      - "3000:3000"
    volumes:
      - ./src:/app/src
      - ./package.json:/app/package.json
      - node_modules:/app/node_modules
    environment:
      - NODE_ENV=development
      - DEBUG=true

volumes:
  node_modules:
```

### Multi-stage Dockerfile pour dev/prod

```dockerfile
# Dockerfile avec séparation dev/prod
FROM node:16-alpine AS base
WORKDIR /app
COPY package*.json ./

# Stage de développement
FROM base AS development
RUN npm install
COPY . .
EXPOSE 3000
CMD ["npm", "run", "dev"]

# Stage de production
FROM base AS production
RUN npm ci --only=production && npm cache clean --force
COPY . .
RUN npm run build
EXPOSE 3000
CMD ["npm", "start"]

# Stage de test
FROM development AS test
RUN npm run test:ci
```

### Reconstruction itérative optimisée

```dockerfile
# Dockerfile optimisé pour le développement
FROM node:16-alpine

WORKDIR /app

# Copier d'abord les fichiers de dépendances (cache Docker)
COPY package*.json ./
RUN npm install

# Copier le code source en dernier
COPY . .

# Build uniquement si nécessaire
RUN npm run build

EXPOSE 3000
CMD ["npm", "start"]
```

```bash
# Script de build optimisé
#!/bin/bash
# build-dev.sh

echo "Building development image..."

# Build avec cache
docker build \
  --target development \
  --cache-from myapp:dev-cache \
  -t myapp:dev \
  .

# Taguer pour le cache
docker tag myapp:dev myapp:dev-cache

echo "Development build complete!"
```

### Environnement de développement complet

```yaml
# docker-compose.yml pour développement
version: '3.8'
services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
      target: development
    ports:
      - "3000:3000"
    volumes:
      - .:/app
      - /app/node_modules
    environment:
      - NODE_ENV=development
      - DATABASE_URL=postgresql://dev:dev@postgres:5432/myapp_dev
    depends_on:
      - postgres
      - redis

  postgres:
    image: postgres:13-alpine
    environment:
      - POSTGRES_DB=myapp_dev
      - POSTGRES_USER=dev
      - POSTGRES_PASSWORD=dev
    ports:
      - "5432:5432"
    volumes:
      - postgres_dev:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

  pgadmin:
    image: dpage/pgadmin4
    environment:
      - PGADMIN_DEFAULT_EMAIL=admin@dev.com
      - PGADMIN_DEFAULT_PASSWORD=admin
    ports:
      - "8080:80"
    depends_on:
      - postgres

volumes:
  postgres_dev:
```

### Tests automatisés avec containers

```yaml
# docker-compose.test.yml
version: '3.8'
services:
  test:
    build:
      context: .
      dockerfile: Dockerfile
      target: test
    environment:
      - NODE_ENV=test
      - DATABASE_URL=postgresql://test:test@postgres-test:5432/myapp_test
    depends_on:
      - postgres-test
    command: npm run test:all

  postgres-test:
    image: postgres:13-alpine
    environment:
      - POSTGRES_DB=myapp_test
      - POSTGRES_USER=test
      - POSTGRES_PASSWORD=test
    tmpfs:
      - /var/lib/postgresql/data

  integration-test:
    build:
      context: .
      dockerfile: Dockerfile.test
    environment:
      - API_URL=http://app:3000
    depends_on:
      - app
    command: npm run test:integration
```

```bash
# Script de test automatisé
#!/bin/bash
# run-tests.sh

echo "Running unit tests..."
docker-compose -f docker-compose.test.yml run --rm test

echo "Running integration tests..."
docker-compose -f docker-compose.test.yml up -d app postgres-test
docker-compose -f docker-compose.test.yml run --rm integration-test

echo "Cleaning up..."
docker-compose -f docker-compose.test.yml down -v
```

### Debugging d'applications conteneurisées

```dockerfile
# Dockerfile.debug
FROM node:16-alpine

# Installation d'outils de debug
RUN apk add --no-cache bash curl netcat-openbsd

WORKDIR /app

COPY package*.json ./
RUN npm install

# Installation du debugger
RUN npm install -g inspector

COPY . .

# Port pour le debugger
EXPOSE 3000 9229

# Commande avec debug activé
CMD ["node", "--inspect=0.0.0.0:9229", "src/index.js"]
```

```yaml
# Configuration pour debugging
version: '3.8'
services:
  app-debug:
    build:
      context: .
      dockerfile: Dockerfile.debug
    ports:
      - "3000:3000"
      - "9229:9229"  # Port debugger
    volumes:
      - ./src:/app/src
    environment:
      - NODE_ENV=development
      - DEBUG=*
```

### Workflow de développement avec Makefile

```makefile
# Makefile pour workflow Docker
.PHONY: dev build test clean

# Développement
dev:
	docker-compose -f docker-compose.dev.yml up --build

# Build de production
build:
	docker build --target production -t myapp:latest .

# Tests
test:
	docker-compose -f docker-compose.test.yml run --rm test

# Tests d'intégration
test-integration:
	docker-compose -f docker-compose.test.yml up -d
	docker-compose -f docker-compose.test.yml run --rm integration-test
	docker-compose -f docker-compose.test.yml down

# Nettoyage
clean:
	docker-compose down -v
	docker system prune -f

# Setup initial
setup:
	docker-compose -f docker-compose.dev.yml build
	docker-compose -f docker-compose.dev.yml run --rm app npm install

# Logs de développement
logs:
	docker-compose -f docker-compose.dev.yml logs -f

# Shell dans le conteneur
shell:
	docker-compose -f docker-compose.dev.yml exec app sh
```

### Configuration avec variables d'environnement

```bash
# .env.development
NODE_ENV=development
DEBUG=true
DATABASE_URL=postgresql://dev:dev@localhost:5432/myapp_dev
REDIS_URL=redis://localhost:6379
LOG_LEVEL=debug
WEBPACK_DEV_SERVER=true

# .env.production
NODE_ENV=production
DEBUG=false
LOG_LEVEL=error
WEBPACK_DEV_SERVER=false
```

### Optimisation du cache Docker

```dockerfile
# Optimisation des layers pour le cache
FROM node:16-alpine AS deps
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

FROM node:16-alpine AS dev-deps
WORKDIR /app
COPY package*.json ./
RUN npm ci

FROM node:16-alpine AS builder
WORKDIR /app
COPY --from=dev-deps /app/node_modules ./node_modules
COPY . .
RUN npm run build

FROM node:16-alpine AS runtime
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY --from=builder /app/dist ./dist
COPY package*.json ./

EXPOSE 3000
CMD ["npm", "start"]
```

## Conclusion

L'intégration de Docker dans les workflows de développement apporte une standardisation et une efficacité considérables. L'utilisation de multi-stage builds, de volumes pour le développement, et d'outils de test automatisés permet de créer des environnements de développement robustes et reproductibles.

Points essentiels à retenir :
- Séparer clairement les configurations de développement et production
- Utiliser les volumes pour le hot reload et la productivité
- Optimiser les Dockerfile pour le cache et la reconstruction rapide
- Automatiser les tests dans des environnements conteneurisés
- Intégrer Docker dans les pipelines CI/CD

Pour maximiser l'efficacité, considérez l'utilisation d'outils comme Docker Compose Override, des scripts Makefile pour automatiser les tâches courantes, et l'intégration avec des IDE pour un debugging seamless.

## Ressources

- [Guide de développement Docker](https://docs.docker.com/develop/)
- [Multi-stage builds](https://docs.docker.com/develop/dev-best-practices/)
- [Docker Compose pour le développement](https://docs.docker.com/compose/compose-file/)
- [Debugging avec Docker](https://docs.docker.com/config/containers/)