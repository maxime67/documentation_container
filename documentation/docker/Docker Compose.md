# Docker Compose

## Introduction

Docker Compose est un outil qui permet de définir et gérer des applications multi-conteneurs avec Docker. Il utilise un fichier YAML pour configurer les services de votre application, puis avec une seule commande, vous pouvez créer et démarrer tous les services à partir de votre configuration.

Docker Compose simplifie considérablement le déploiement d'applications complexes composées de plusieurs conteneurs interdépendants, en automatisant la création des réseaux, volumes et la gestion des dépendances entre services.

## Concepts clés

### Fichier docker-compose.yml

Le fichier `docker-compose.yml` est le cœur de Docker Compose. Il définit la structure de votre application multi-conteneurs :

- **Structure YAML** : Format lisible utilisant l'indentation pour définir la hiérarchie
- **Services** : Chaque conteneur est défini comme un service avec ses paramètres
- **Réseaux** : Configuration des communications entre conteneurs
- **Volumes** : Gestion du stockage persistant et partagé
- **Variables d'environnement** : Substitution dynamique de valeurs avec `${VARIABLE}`

### Gestion des dépendances

- **depends_on** : Contrôle l'ordre de démarrage des services
- **restart** : Définit la politique de redémarrage (no, always, on-failure, unless-stopped)
- **ports vs expose** : `ports` expose vers l'hôte, `expose` uniquement entre conteneurs

### Commandes essentielles

- **docker-compose up/down** : Démarrage et arrêt complet de l'application
- **docker-compose ps/logs** : Monitoring de l'état et des journaux
- **docker-compose exec** : Exécution de commandes dans les conteneurs
- **docker-compose build** : Construction des images personnalisées

## Cas d'usage

### Application web avec base de données

```yaml
version: '3.8'
services:
  web:
    # Ici on build un conteneur à partir de l'image du Dockerfile présent dans le même dossier
    build: ..
    ports:
      - "8000:8000"
    # Ici on séquence les démarages des conteneurs
    depends_on:
      - db
    # On surcharge les variables d'environnement
    environment:
      # On utilise la résolution DNS de docker via l'utilisation du nom du conteneur 'postgresql', traduit par l'addresse IP.
      - DATABASE_URL=postgresql://user:password@db:5432/myapp

  db:
    image: postgres:13
    environment:
      - POSTGRES_DB=myapp
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=password
    volumes:
      # On définit un volume docker pour conserver les données présentes dans '/var/lib/postgresql/data', répertoire par défaut des données de postgresql
      - postgres_data:/var/lib/postgresql/data

volumes:
  # On définit le volume mentionné plus tôt
  postgres_data:
```

### Utilisation des variables d'environnement

```yaml
# fichier .env présent dans le même dossier
DATABASE_PASSWORD=secretpassword
API_PORT=3000

# docker-compose.yml
services:
  api:
    image: node:16
    ports:
    # Docker résout la valeur de la variable 'API_PORT' à partir de sa valeur définit dans le fichier .env 
      - "${API_PORT}:3000"
    environment:
      - DB_PASSWORD=${DATABASE_PASSWORD}
```

### Commandes courantes

```bash
# Démarrer l'application
docker-compose up -d

# Voir les services en cours
docker-compose ps

# Consulter les logs
docker-compose logs -f web

# Exécuter une commande dans un service
docker-compose exec web bash

# Arrêter et supprimer
docker-compose down

# Scaling d'un service
docker-compose up --scale web=3
```

### Fichiers d'override pour différents environnements

```yaml
# docker-compose.override.yml (développement)
services:
  web:
    volumes:
      - .:/app
    environment:
      - DEBUG=true

# docker-compose.prod.yml (production)
services:
  web:
    restart: always
    environment:
      - DEBUG=false
```

## Conclusion

Docker Compose transforme la gestion d'applications multi-conteneurs en une tâche simple et reproductible. Il permet de définir l'infrastructure comme du code, facilitant le déploiement cohérent entre différents environnements.

Les points essentiels à retenir :
- Un seul fichier YAML pour définir toute l'architecture
- Gestion automatique des réseaux et dépendances
- Facilité de scaling et de monitoring
- Flexibilité avec les variables d'environnement et fichiers d'override

Pour approfondir, explorez les fonctionnalités avancées comme les healthchecks, les profiles, et l'intégration avec Docker Swarm pour l'orchestration en production.

## Ressources

- [Documentation officielle Docker Compose](https://docs.docker.com/compose/)
- [Référence du fichier Compose](https://docs.docker.com/compose/compose-file/)
- [Guide des meilleures pratiques](https://docs.docker.com/develop/dev-best-practices/)
- [Exemples Docker Compose](https://github.com/docker/awesome-compose)