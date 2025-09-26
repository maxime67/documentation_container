# Stockage et volumes

## Introduction

La gestion du stockage est un aspect critique des applications conteneurisées. Par nature, les conteneurs Docker sont éphémères : toutes les données écrites dans leur système de fichiers disparaissent lors de leur suppression. Les volumes et les différents types de montage permettent de résoudre cette limitation en fournissant une persistance des données.

Docker propose plusieurs mécanismes de stockage, chacun adapté à des cas d'usage spécifiques : bind mounts pour lier directement des répertoires de l'hôte, volumes Docker pour un stockage géré et optimisé, et tmpfs pour des données temporaires en mémoire. La maîtrise de ces concepts est essentielle pour architecturer des applications robustes et maintenir la persistance des données critiques.

## Concepts clés

### Types de montage

**Bind mounts : `-v /host/path:/container/path`**
```bash
# Montage simple d'un répertoire
docker run -d -v /var/log/nginx:/var/log/nginx nginx

# Montage en lecture seule
docker run -d -v /etc/nginx:/etc/nginx:ro nginx

# Répertoire de travail pour développement
docker run -it -v $(pwd):/app -w /app node:16 npm install

# Montage avec propriétaire spécifique (avec --user)
docker run -d --user 1000:1000 -v /data:/app/data mon-app
```

**Volumes Docker : stockage géré par Docker**
```bash
# Création d'un volume nommé
docker volume create mon-volume

# Utilisation d'un volume nommé
docker run -d -v mon-volume:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=pass mysql:8

# Volume anonyme (créé automatiquement)
docker run -d -v /var/lib/mysql -e MYSQL_ROOT_PASSWORD=pass mysql:8

```

**tmpfs : stockage en mémoire**
```bash
# Montage tmpfs simple
docker run -d --tmpfs /tmp nginx

# tmpfs avec options spécifiques
docker run -d --tmpfs /tmp:rw,noexec,nosuid,size=100m nginx

# Montage tmpfs pour données sensibles
docker run -d --tmpfs /run/secrets:noexec,nosuid,size=10m nginx

# Alternative avec --mount (syntaxe plus explicite)
docker run -d --mount type=tmpfs,destination=/tmp,tmpfs-size=100m nginx
```

### Gestion des volumes

**docker volume create/ls/rm : gestion des volumes**
```bash
# Créer un volume simple
docker volume create data-volume

# Créer un volume avec options
docker volume create \
  --driver local \
  --opt type=ext4 \
  --opt device=/dev/sdb1 \
  production-data

# Lister tous les volumes
docker volume ls

# Filtrer les volumes
docker volume ls --filter "dangling=true"  # Volumes orphelins
docker volume ls --filter "driver=local"

# Supprimer un volume
docker volume rm data-volume

# Supprimer tous les volumes non utilisés
docker volume prune

# Suppression forcée (attention !)
docker volume rm -f data-volume
```

**docker volume inspect : inspection des volumes**
```bash
# Inspection détaillée d'un volume
docker volume inspect data-volume

# Format personnalisé
docker volume inspect --format '{{.Mountpoint}}' data-volume

# Inspection multiple
docker volume inspect vol1 vol2 vol3

# Vérifier la taille utilisée (approximative)
docker system df -v
```

**Partage de volumes entre conteneurs**
```bash
# Volume partagé entre plusieurs conteneurs
docker volume create shared-data

# Premier conteneur écrit des données
docker run -d --name writer \
  -v shared-data:/data \
  -e ROLE=writer \
  data-processor

# Deuxième conteneur lit les données
docker run -d --name reader \
  -v shared-data:/data:ro \
  -e ROLE=reader \
  data-processor

# Pattern volumes-from (legacy)
docker run -d --name data-container \
  -v /var/lib/mysql \
  mysql:8 echo "Data container"

docker run -d --volumes-from data-container mysql:8
```

### Persistance des données

**Séparation données/application**
```bash
# Architecture avec séparation claire
# Base de données avec volume persistant
docker run -d --name postgres-db \
  -v postgres-data:/var/lib/postgresql/data \
  -e POSTGRES_DB=myapp \
  postgres:13

# Application stateless
docker run -d --name app \
  -e DATABASE_URL=postgresql://postgres-db:5432/myapp \
  --link postgres-db \
  mon-app:latest

# Upgrade de l'application sans perte de données
docker stop app
docker rm app
docker run -d --name app \
  -e DATABASE_URL=postgresql://postgres-db:5432/myapp \
  --link postgres-db \
  mon-app:v2.0
```

**Localisation des volumes sur le système hôte**
```bash
# Emplacement par défaut des volumes Docker
ls -la /var/lib/docker/volumes/

# Chemin complet d'un volume
docker volume inspect --format '{{.Mountpoint}}' mon-volume

# Accès direct aux données (en tant que root)
sudo ls -la $(docker volume inspect --format '{{.Mountpoint}}' mon-volume)

# Volume avec emplacement personnalisé (bind mount)
docker run -d -v /opt/app-data:/data mon-app
```

**Sauvegarde et restauration des volumes**
```bash
# Sauvegarde d'un volume via conteneur temporaire
docker run --rm \
  -v mon-volume:/data \
  -v $(pwd):/backup \
  alpine tar czf /backup/backup-$(date +%Y%m%d).tar.gz -C /data .

# Restauration d'un volume
docker run --rm \
  -v mon-volume:/data \
  -v $(pwd):/backup \
  alpine tar xzf /backup/backup-20240101.tar.gz -C /data

# Sauvegarde avec timestamp et rotation
BACKUP_FILE="backup-$(date +%Y%m%d_%H%M%S).tar.gz"
docker run --rm \
  -v production-data:/source:ro \
  -v /backups:/dest \
  alpine tar czf /dest/$BACKUP_FILE -C /source .

# Migration de volume entre environnements
# Export
docker run --rm \
  -v source-volume:/data:ro \
  -v $(pwd):/backup \
  alpine tar czf /backup/migration.tar.gz -C /data .

# Import (nouvel environnement)
docker volume create target-volume
docker run --rm \
  -v target-volume:/data \
  -v $(pwd):/backup \
  alpine tar xzf /backup/migration.tar.gz -C /data
```

## Cas d'usage

### Base de données avec persistance
```bash
# Création du volume pour PostgreSQL
docker volume create postgres-data

# Lancement de la base avec volume persistant
docker run -d --name postgres-db \
  --restart=unless-stopped \
  -v postgres-data:/var/lib/postgresql/data \
  -e POSTGRES_DB=production \
  -e POSTGRES_USER=app \
  -e POSTGRES_PASSWORD_FILE=/run/secrets/db_password \
  -v /secrets/db_password:/run/secrets/db_password:ro \
  postgres:13

# Sauvegarde régulière automatisée
docker run --rm \
  -v postgres-data:/var/lib/postgresql/data:ro \
  -v /backups:/backup \
  postgres:13 \
  pg_dump -h postgres-db -U app production > /backup/backup-$(date +%Y%m%d).sql
```

### Environnement de développement avec code source
```bash
# Développement Node.js avec hot-reload
docker run -it --rm \
  --name dev-environment \
  -v $(pwd):/app \
  -v node_modules:/app/node_modules \
  -v ~/.npm:/root/.npm \
  -p 3000:3000 \
  -w /app \
  node:16 \
  sh -c "npm install && npm run dev"

# Avantages :
# - Code source synchronisé avec l'hôte
# - node_modules persistant entre redémarrages
# - Cache npm réutilisé
```

### Application web avec assets et logs
```bash
# Création des volumes nécessaires
docker volume create web-assets
docker volume create web-logs

# Serveur web avec stockage séparé
docker run -d --name nginx-server \
  --restart=unless-stopped \
  -v web-assets:/usr/share/nginx/html \
  -v web-logs:/var/log/nginx \
  -v $(pwd)/nginx.conf:/etc/nginx/nginx.conf:ro \
  -p 80:80 \
  nginx:alpine

# Processus de déploiement
docker run --rm \
  -v web-assets:/target \
  -v $(pwd)/dist:/source \
  alpine cp -r /source/* /target/
```

### Partage de données entre microservices
```bash
# Volume partagé pour communication via fichiers
docker volume create shared-uploads

# Service de traitement d'images
docker run -d --name image-processor \
  -v shared-uploads:/uploads \
  image-service:latest

# Service web qui reçoit les uploads
docker run -d --name web-app \
  -v shared-uploads:/app/uploads \
  -p 8080:8080 \
  web-app:latest

# Service de nettoyage périodique
docker run -d --name cleanup-service \
  -v shared-uploads:/data \
  -e CLEANUP_INTERVAL=3600 \
  cleanup-service:latest
```

### Migration et backup de production
```bash
# Script de sauvegarde complète
#!/bin/bash
BACKUP_DIR="/backups/$(date +%Y%m%d)"
mkdir -p $BACKUP_DIR

# Sauvegarde de tous les volumes de production
for volume in $(docker volume ls -q --filter name=prod_); do
  echo "Sauvegarde du volume $volume..."
  docker run --rm \
    -v $volume:/data:ro \
    -v $BACKUP_DIR:/backup \
    alpine tar czf /backup/$volume.tar.gz -C /data .
done

# Sauvegarde des configurations
docker run --rm \
  -v /etc/docker:/docker-config:ro \
  -v $BACKUP_DIR:/backup \
  alpine tar czf /backup/docker-config.tar.gz -C /docker-config .

echo "Sauvegarde terminée dans $BACKUP_DIR"
```

## Conclusion

La gestion efficace du stockage et des volumes est fondamentale pour créer des applications Docker robustes et pérennes. Les différents types de montage offrent une flexibilité adaptée à chaque besoin : bind mounts pour le développement et l'intégration avec l'hôte, volumes Docker pour la persistance gérée, et tmpfs pour les données temporaires sensibles.

La séparation claire entre les données et l'application permet des déploiements sans interruption et facilite la maintenance. Les stratégies de sauvegarde et de restauration garantissent la protection des données critiques, tandis que le partage de volumes entre conteneurs facilite l'architecture de systèmes distribués.

La maîtrise de ces concepts est essentielle pour évoluer vers des architectures plus complexes et prépare l'utilisation d'orchestrateurs comme Docker Swarm ou Kubernetes qui s'appuient sur ces mêmes principes de gestion du stockage.

## Ressources

- [Documentation Docker Storage](https://docs.docker.com/storage/)
- [Gestion des volumes Docker](https://docs.docker.com/storage/volumes/)
- [Bind mounts vs volumes](https://docs.docker.com/storage/bind-mounts/)
- [Drivers de stockage](https://docs.docker.com/storage/storagedriver/)
- [Bonnes pratiques de stockage](https://docs.docker.com/develop/dev-best-practices/#where-and-how-to-persist-application-data)
- [Sauvegarde et migration](https://docs.docker.com/storage/volumes/#backup-restore-or-migrate-data-volumes)