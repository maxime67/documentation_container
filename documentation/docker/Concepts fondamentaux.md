# Concepts fondamentaux

## Introduction

Docker repose sur quatre concepts fondamentaux qui forment l'écosystème de la conteneurisation : les images, les conteneurs, les registries et les layers. Ces éléments travaillent ensemble pour créer un système cohérent de packaging, distribution et exécution d'applications.

Comprendre ces concepts est essentiel pour maîtriser Docker, car ils définissent la façon dont les applications sont empaquetées, stockées, partagées et exécutées. Cette approche modulaire et hiérarchique permet une gestion efficace des ressources et une réutilisation optimale des composants.

## Concepts clés

### Images : templates immuables

**Définition** : Une image Docker est un template en lecture seule contenant tout le nécessaire pour exécuter une application : code, runtime, bibliothèques, variables d'environnement et fichiers de configuration.

**Caractéristiques** :
- **Immuables** : Une fois créée, une image ne peut pas être modifiée
- **Versionnées** : Identifiées par des tags (latest, v1.0, stable)
- **Composées de layers** : Construites par empilement de couches
- **Réutilisables** : Servent de base pour créer des conteneurs

### Conteneurs : instances d'exécution d'images

**Définition** : Un conteneur est une instance d'exécution d'une image Docker. Il ajoute une couche inscriptible au-dessus de l'image en lecture seule.

**Cycle de vie** :
- **Créé** : Conteneur créé mais non démarré
- **En cours d'exécution** : Processus principal actif
- **Arrêté** : Processus terminé, conteneur conservé
- **Supprimé** : Conteneur définitivement effacé

**Différences image/conteneur** :
- Image = classe, Conteneur = instance
- Une image peut générer plusieurs conteneurs
- Les modifications dans un conteneur n'affectent pas l'image source

### Registries : stockage et distribution d'images

**Définition** : Les registries sont des services de stockage et distribution d'images Docker, permettant le partage entre développeurs et environnements.

**Types de registries** :
- **Docker Hub** : Registry public officiel
- **Registries privés** : Pour les entreprises (AWS ECR, Azure ACR, Google GCR)
- **Registry local** : Déployé sur infrastructure interne

### Layers : système de couches des images Docker

**Définition** : Les layers (couches) constituent le système de fichiers en couches des images Docker. Chaque instruction dans un Dockerfile crée une nouvelle couche.

**Fonctionnement** :
- **Union File System** : Les couches sont empilées pour former l'image finale
- **Partage de couches** : Les couches identiques sont partagées entre images
- **Optimisation d'espace** : Évite la duplication des données communes
- **Cache de build** : Réutilise les couches non modifiées lors de la construction

## Conclusion

Les quatre concepts fondamentaux de Docker forment un écosystème cohérent et puissant. Les images fournissent des templates reproductibles, les conteneurs permettent l'exécution isolée, les registries facilitent la distribution, et les layers optimisent le stockage et les performances.

Cette architecture modulaire offre une flexibilité exceptionnelle pour le développement, les tests et le déploiement d'applications. La compréhension de ces mécanismes est cruciale pour tirer pleinement parti de Docker et éviter les écueils courants comme la prolifération d'images volumineuses ou la mauvaise gestion du cache.

Ces concepts préparent la base pour aborder des sujets plus avancés comme l'optimisation des Dockerfiles, la sécurité des images, et l'orchestration de conteneurs.

## Ressources

- [Documentation Docker - Images et conteneurs](https://docs.docker.com/get-started/overview/)
- [Docker Hub - Registry officiel](https://hub.docker.com/)
- [Bonnes pratiques pour les Dockerfiles](https://docs.docker.com/develop/dev-best-practices/)
- [Compréhension du système de layers](https://docs.docker.com/storage/storagedriver/)
- [Guide des registries Docker](https://docs.docker.com/registry/)
- [Union File Systems expliqués](https://docs.docker.com/storage/storagedriver/overlayfs-driver/)