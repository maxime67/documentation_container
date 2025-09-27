# Gestion des conteneurs

## Introduction

La gestion efficace des conteneurs Docker constitue le cœur de l'utilisation pratique de cette technologie. Elle englobe la configuration des options de lancement, la gestion des variables d'environnement, et l'allocation des ressources système.

Cette maîtrise permet de créer des environnements d'exécution optimisés, sécurisés et adaptés aux besoins spécifiques de chaque application. La compréhension fine de ces mécanismes est essentielle pour déployer des applications robustes en production et gérer efficacement les ressources du serveur hôte.

## Concepts clés

### Options de docker run

**Mode détaché (-d)**
```bash
# Exécution en arrière-plan
docker run -d nginx

# Avec nom pour faciliter la gestion
docker run -d --name web-server nginx

# Récupération des logs d'un conteneur détaché
docker logs -f web-server
```

**Mapping de ports (-p)**
```bash
# Port simple : host:conteneur
docker run -d -p 8080:80 nginx

# Tous les ports exposés
docker run -d -P nginx

# Interface spécifique
docker run -d -p 127.0.0.1:8080:80 nginx

# Multiples ports
docker run -d -p 80:80 -p 443:443 nginx
```

**Montage de volumes (-v)**
```bash
# Volume nommé
docker run -d -v data-volume:/var/data nginx

# Bind mount (répertoire hôte)
docker run -d -v /host/path:/container/path nginx

# Mount en lecture seule
docker run -d -v /host/path:/container/path:ro nginx

# Volume temporaire (tmpfs)
docker run -d --tmpfs /tmp nginx
```

**Variables d'environnement (-e)**
```bash
# Variable simple
docker run -e NODE_ENV=production node-app

# Multiples variables
docker run -e NODE_ENV=production -e PORT=3000 node-app

# Variable depuis l'hôte, doit être définie en variable d'environnement sur la machine hôte
docker run -e USER=$USER node-app

# Fichier de variables
docker run --env-file .env node-app
```

**Nommer un conteneur (--name)**
```bash
# Attribution d'un nom explicite
docker run -d --name database -e POSTGRES_PASSWORD=pass  postgres

# Utilisation du nom pour les opérations
docker logs database
docker exec -it database psql
docker stop database
```

**Politique de redémarrage (--restart)**
```bash
# Pas de redémarrage automatique (défaut)
docker run --restart=no nginx

# Redémarrage automatique
docker run --restart=always nginx

# Redémarrage sauf si arrêt manuel
docker run --restart=unless-stopped nginx

# Redémarrage limité
docker run --restart=on-failure:3 nginx
```

**Mode interactif avec TTY (-it)**
```bash
# Session interactive
docker run -it ubuntu bash

# Commande ponctuelle interactive
docker run -it --rm alpine ping 8.8.8.8
```

### Variables d'environnement

**Définition avec -e ou --env-file**
```bash
# Variables individuelles
docker run -e DATABASE_URL=postgres://localhost/mydb \
           -e API_KEY=secret123 \
           -e DEBUG=true \
           mon-app

# Fichier d'environnement (.env)
# DATABASE_URL=postgres://localhost/mydb
# API_KEY=secret123
# DEBUG=true
docker run --env-file .env mon-app

# Combinaison des deux approches
docker run --env-file .env -e OVERRIDE_VAR=value mon-app
```

**Surcharge des variables du Dockerfile**
```dockerfile
# Dans le Dockerfile
ENV NODE_ENV=development
ENV PORT=3000
```
```bash
# Surcharge au runtime
docker run -e NODE_ENV=production -e PORT=8080 mon-app
```

**Bonnes pratiques pour les secrets**
```bash
# ❌ Ne pas définir les secrets en variables d'environnement
docker run -e API_SECRET=supersecret mon-app

# ✅ Utiliser des fichiers secrets montés
docker run -d -v ./secure/api-key:/run/secrets/api-key:ro nginx

# ✅ Variables d'environnement pour configuration non sensible
docker run -e LOG_LEVEL=info -e FEATURE_FLAG=enabled nginx
```

### Gestion des ressources

**Limite mémoire (--memory)**
```bash
# Limite en mégaoctets
docker run --memory=512m nginx

# Limite en gigaoctets
docker run --memory=2g nginx

# Limite avec réservation
docker run --memory=1g --memory-reservation=512m nginx

# Vérification de l'utilisation
docker stats --format "table {{.Container}}\t{{.MemUsage}}\t{{.MemPerc}}"
```

**Limite CPU (--cpus)**
```bash
# Limite à 1.5 CPU
docker run --cpus=1.5 nginx

# Priorité CPU
docker run --cpu-shares=512 nginx

# Affinité CPU (CPUs spécifiques)
docker run --cpuset-cpus=0,1 nginx
```

**Gestion du swap (--memory-swap)**
```bash
# Swap = 2x la mémoire (défaut)
docker run --memory=1g nginx

# Désactiver le swap
docker run --memory=1g --memory-swap=1g nginx

# Swap illimité
docker run --memory=1g --memory-swap=-1 nginx

# Swap personnalisé
docker run --memory=1g --memory-swap=2g nginx
```

**Monitoring des ressources**
```bash
# Statistiques temps réel
docker stats

# Alertes de dépassement
docker run --memory=100m --oom-kill-disable=false test-app
```

## Cas d'usage

### Base de données avec persistance
```bash
# Créer un password file en amont
echo "ekrvnaklerbnlknrjkbn" >> ./secrets/db_password

# PostgreSQL avec volume persistant
docker run -d \
  --name postgres-db \
  --restart=always \
  --memory=1g \
  -e POSTGRES_DB=myapp \
  -e POSTGRES_USER=appuser \
  -e POSTGRES_PASSWORD_FILE=/run/secrets/db_password \
  # Le volume mentionné ci-dessous sera crée à la volé
  -v postgres-data:/var/lib/postgresql/data \
  # On utilise ici le fichier crée précédement pour ne pas mentionner de secret en clair
  -v ./secrets/db_password:/run/secrets/db_password:ro \
  # Seul la machine hôte aura accès au service
  -p 127.0.0.1:5432:5432 \
  postgres:13
```

### Environnement de développement
```bash
# Conteneur de développement avec hot-reload pour une applicationb VueJS
docker run -it \
  --name dev-env \
  --rm \
  -p 80:80 \
  -v $(pwd):/app \
  -w /app \
  node:22-alpine \
  sh -c "apk add --no-cache python3 make g++ && npm install && npm run dev"
```

### Application avec configuration externe
```bash
# Fichier .env.prod
echo "DATABASE_URL=postgres://prod-server/mydb
REDIS_URL=redis://prod-cache:6379
API_TIMEOUT=30000
WORKERS=4" > .env.prod

# Lancement avec configuration
docker run -d \
  --name app-prod \
  --env-file .env.prod \
  -e INSTANCE_ID=$(hostname) \
  --memory=1g \
  --cpus=0.5 \
  mon-app:prod
```

### Tests d'intégration avec ressources limitées
```bash
# Conteneur de test avec ressources réduites
docker run --rm \
  --name integration-test \
  --memory=256m \
  --cpus=0.25 \
  -e NODE_ENV=test \
  -e CI=true \
  -v $(pwd):/app \
  -w /app \
  node:16-alpine \
  npm test
```

## Conclusion

La gestion efficace des conteneurs Docker repose sur la maîtrise des options de lancement et la configuration appropriée des ressources. L'utilisation judicieuse des variables d'environnement, combinée à une allocation optimale de la mémoire et du CPU, permet de créer des environnements d'exécution robustes et performants.

Les politiques de redémarrage et la gestion des volumes assurent la persistance et la disponibilité des applications, tandis que les bonnes pratiques de sécurité protègent les données sensibles. Cette approche structurée est essentielle pour des déploiements fiables en production.

La compréhension de ces concepts prépare à l'utilisation d'outils plus avancés comme Docker Compose et les orchestrateurs, qui automatisent et simplifient la gestion de ces configurations complexes.

## Ressources

- [Documentation docker run](https://docs.docker.com/engine/reference/run/)
- [Gestion des ressources Docker](https://docs.docker.com/config/containers/resource_constraints/)
- [Variables d'environnement et secrets](https://docs.docker.com/engine/swarm/secrets/)
- [Politiques de redémarrage](https://docs.docker.com/config/containers/start-containers-automatically/)
- [Bonnes pratiques de sécurité](https://docs.docker.com/engine/security/)
- [Monitoring des conteneurs](https://docs.docker.com/config/containers/runmetrics/)