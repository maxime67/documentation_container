# Debugging et Résolution de Problèmes Docker

## Introduction

Le debugging des conteneurs Docker est une compétence essentielle pour identifier et résoudre rapidement les problèmes en développement et en production. Docker fournit de nombreux outils intégrés pour diagnostiquer les dysfonctionnements, analyser les configurations et surveiller le comportement des conteneurs.

Cette documentation présente les techniques de debugging les plus efficaces, les problèmes courants rencontrés avec Docker, et les outils de diagnostic pour maintenir des environnements conteneurisés stables et performants.

## Concepts clés

### Techniques de debugging

Le debugging Docker s'articule autour de plusieurs approches :
- **Accès direct aux conteneurs** : Shell interactif pour explorer l'environnement
- **Analyse des logs** : Consultation des sorties d'erreur et de debug
- **Inspection de la configuration** : Vérification des paramètres et métadonnées
- **Tests de connectivité** : Validation des communications réseau

### Problèmes courants

Les problèmes les plus fréquents incluent :
- **Échec de démarrage** : Erreurs de configuration ou dépendances manquantes
- **Problèmes de permissions** : Conflits d'utilisateurs et droits d'accès
- **Conflits de ressources** : Ports occupés, espace disque insuffisant
- **Problèmes réseau** : Isolation des conteneurs, résolution DNS

### Outils de diagnostic

Docker propose des commandes spécialisées :
- **docker inspect** : Métadonnées complètes des objets Docker
- **docker system events** : Flux d'événements en temps réel
- **docker top** : Processus actifs dans les conteneurs

## Cas d'usage

### Accès shell et exploration

```bash
# Accéder à un conteneur en cours d'exécution
docker exec -it monconteneur bash

# Si bash n'est pas disponible (Alpine Linux)
docker exec -it monconteneur sh

# Exécuter des commandes spécifiques
docker exec monconteneur ls -la /app
docker exec monconteneur ps aux
docker exec monconteneur cat /etc/hostname

# Accéder en tant qu'utilisateur root
docker exec -it --user root monconteneur bash
```

### Analyse des logs d'erreur

```bash
# Consulter tous les logs
docker logs monconteneur

# Logs avec timestamps
docker logs -t monconteneur

# Suivre les logs en temps réel
docker logs -f monconteneur

# Dernières lignes et suivi
docker logs --tail 100 -f monconteneur

# Logs depuis une période spécifique
docker logs --since 2023-01-01T10:00:00 monconteneur
docker logs --since 1h monconteneur
```

### Inspection de configuration

```bash
# Inspection complète d'un conteneur
docker inspect monconteneur

# Extraire des informations spécifiques
docker inspect --format='{{.State.Status}}' monconteneur
docker inspect --format='{{.NetworkSettings.IPAddress}}' monconteneur
docker inspect --format='{{.Config.Env}}' monconteneur

# Inspection d'une image
docker inspect nginx:alpine

# Inspection d'un réseau
docker inspect bridge
```

### Tests de connectivité réseau

```bash
# Lister les réseaux
docker network ls

# Inspecter un réseau
docker network inspect bridge

# Tester la connectivité depuis un conteneur
docker exec monconteneur ping google.com
docker exec monconteneur nslookup database
docker exec monconteneur telnet database 5432

# Vérifier les ports ouverts
docker exec monconteneur netstat -tlnp
docker exec monconteneur ss -tlnp
```

### Diagnostic des problèmes de démarrage

```bash
# Vérifier l'état des conteneurs
docker ps -a

# Analyser les raisons d'arrêt
docker inspect --format='{{.State.ExitCode}}' monconteneur
docker inspect --format='{{.State.Error}}' monconteneur

# Tenter un démarrage en mode interactif
docker run -it --rm monimage bash

# Vérifier l'image et ses layers
docker history monimage
```

### Résolution des problèmes de permissions

```bash
# Vérifier l'utilisateur du conteneur
docker exec monconteneur whoami
docker exec monconteneur id

# Lister les permissions
docker exec monconteneur ls -la /app

# Corriger les permissions temporairement
docker exec --user root monconteneur chown -R app:app /app
docker exec --user root monconteneur chmod -R 755 /app

# Vérifier les volumes montés
docker inspect --format='{{.Mounts}}' monconteneur
```

### Diagnostic des conflits de ports

```bash
# Vérifier les ports utilisés sur l'hôte
netstat -tlnp | grep :8080
lsof -i :8080

# Voir les ports mappés des conteneurs
docker port monconteneur

# Lister tous les conteneurs avec leurs ports
docker ps --format "table {{.Names}}\t{{.Ports}}\t{{.Status}}"

# Tester l'accessibilité d'un port
telnet localhost 8080
curl -v http://localhost:8080
```

### Monitoring des ressources et espace disque

```bash
# Vérifier l'utilisation des ressources
docker stats monconteneur

# Utilisation de l'espace disque
docker system df
docker system df -v

# Identifier les gros consommateurs
docker images --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}"

# Nettoyer l'espace disque
docker system prune
docker container prune
docker image prune
docker volume prune
```

### Surveillance des événements système

```bash
# Surveiller les événements en temps réel
docker system events

# Filtrer les événements par type
docker system events --filter type=container
docker system events --filter container=monconteneur

# Événements depuis un moment donné
docker system events --since 1h

# Événements avec format JSON
docker system events --format json
```

### Analyse des processus

```bash
# Processus dans un conteneur
docker top monconteneur

# Processus avec format personnalisé
docker top monconteneur aux

# Processus de tous les conteneurs
for container in $(docker ps -q); do
  echo "=== $container ==="
  docker top $container
done
```

### Script de diagnostic complet

```bash
#!/bin/bash
# docker-debug.sh

CONTAINER_NAME=$1

if [ -z "$CONTAINER_NAME" ]; then
    echo "Usage: $0 <container_name>"
    exit 1
fi

echo "=== DIAGNOSTIC POUR $CONTAINER_NAME ==="

echo "1. État du conteneur:"
docker ps -a --filter name=$CONTAINER_NAME

echo -e "\n2. Logs récents:"
docker logs --tail 20 $CONTAINER_NAME

echo -e "\n3. Configuration réseau:"
docker inspect --format='{{.NetworkSettings.IPAddress}}' $CONTAINER_NAME

echo -e "\n4. Variables d'environnement:"
docker inspect --format='{{.Config.Env}}' $CONTAINER_NAME

echo -e "\n5. Processus actifs:"
docker top $CONTAINER_NAME 2>/dev/null || echo "Conteneur non démarré"

echo -e "\n6. Utilisation des ressources:"
docker stats --no-stream $CONTAINER_NAME 2>/dev/null || echo "Statistiques non disponibles"

echo -e "\n7. Volumes montés:"
docker inspect --format='{{.Mounts}}' $CONTAINER_NAME

echo -e "\n8. Ports exposés:"
docker port $CONTAINER_NAME 2>/dev/null || echo "Aucun port exposé"
```

### Debugging avec Docker Compose

```bash
# Logs de tous les services
docker-compose logs

# Logs d'un service spécifique
docker-compose logs web

# État des services
docker-compose ps

# Recréer un service problématique
docker-compose up --force-recreate web

# Accéder à un service
docker-compose exec web bash

# Vérifier la configuration
docker-compose config
```

### Techniques de debugging avancées

```bash
# Créer une image de debug avec outils
FROM alpine:latest
RUN apk add --no-cache curl wget netcat-openbsd bash procps

# Lancer un conteneur de debug sur le même réseau
docker run -it --rm --network container:monconteneur alpine sh

# Copier des fichiers depuis/vers un conteneur
docker cp monconteneur:/app/config.log ./debug/
docker cp debug-script.sh monconteneur:/tmp/

# Sauvegarder l'état d'un conteneur
docker commit monconteneur debug-snapshot
docker run -it debug-snapshot bash
```

## Conclusion

La maîtrise des techniques de debugging Docker est cruciale pour maintenir des applications conteneurisées fiables. L'utilisation méthodique des outils de diagnostic permet d'identifier rapidement les causes des dysfonctionnements et d'appliquer les corrections appropriées.

Points essentiels à retenir :
- Utiliser `docker logs` comme premier réflexe pour diagnostiquer les problèmes
- Exploiter `docker inspect` pour analyser les configurations en détail
- Tester la connectivité réseau avec les outils intégrés aux conteneurs
- Surveiller les ressources système pour prévenir les problèmes de performance
- Adopter une approche méthodique avec des scripts de diagnostic

Pour des environnements complexes, considérez l'intégration d'outils de monitoring externes, l'utilisation de profilers applicatifs, et la mise en place de dashboards de surveillance pour anticiper les problèmes avant qu'ils n'impactent la production.

## Ressources

- [Guide de troubleshooting Docker](https://docs.docker.com/config/troubleshooting/)
- [Documentation Docker CLI](https://docs.docker.com/engine/reference/commandline/)
- [Best practices de debugging](https://docs.docker.com/develop/dev-best-practices/)
- [Outils de monitoring Docker](https://docs.docker.com/config/containers/resource_constraints/)