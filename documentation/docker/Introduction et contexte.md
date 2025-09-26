# Introduction et contexte

## Introduction

[//]: # (TODO: Revoir la formulation pour quelle soit moin formelle et 'jolie', faire cela de manière pro et simple, et moin pompeu)
Docker révolutionne le déploiement d'applications en proposant une approche de conteneurisation légère et portable. Face aux défis traditionnels de déploiement où une application fonctionnant parfaitement en développement peut échouer en production, Docker apporte une solution élégante : empaqueter l'application avec toutes ses dépendances dans un conteneur standardisé.

Cette technologie s'appuie sur des mécanismes natifs du noyau Linux pour créer des environnements isolés, légers et reproductibles, transformant la façon dont nous développons, testons et déployons les applications modernes.

## Concepts clés

### Problématiques de déploiement et de portabilité

**Syndrome "ça marche sur ma machine"** : Les différences entre environnements de développement, test et production génèrent des dysfonctionnements difficilement prévisibles.

**Dépendances complexes** : Versions de bibliothèques, configurations système et variables d'environnement différentes entre serveurs.

**Gestion des versions** : Coexistence difficile de plusieurs versions d'une même application ou de ses composants.

### Conteneurisation vs virtualisation traditionnelle

**Virtualisation traditionnelle** : Chaque machine virtuelle embarque un système d'exploitation complet, consommant des ressources importantes.

**Conteneurisation** : Les conteneurs partagent le noyau de l'hôte, n'embarquant que les bibliothèques et binaires nécessaires à l'application.

**Avantages de la conteneurisation** :
- Démarrage quasi-instantané (secondes vs minutes)
- Empreinte mémoire réduite
- Densité plus élevée sur un même serveur
- Portabilité garantie entre environnements

### Architecture Docker sur Linux

**dockerd (daemon Docker)** : Service principal qui gère les conteneurs, images, réseaux et volumes.

**docker (client)** : Interface en ligne de commande qui communique avec le daemon via API REST.

**containerd** : Runtime de conteneurs de niveau intermédiaire, responsable du cycle de vie des conteneurs.

**runc** : Runtime de bas niveau qui crée et exécute les conteneurs selon les spécifications OCI (Open Container Initiative).

### Namespaces et cgroups Linux

**Namespaces** : Mécanismes d'isolation qui partitionnent les ressources système :
- PID : isolation des processus
- Network : pile réseau dédiée
- Mount : système de fichiers isolé
- User : espaces utilisateurs séparés
- UTS : hostname et domaine distincts

**Cgroups (Control Groups)** : Limitation et monitoring des ressources :
- CPU : allocation de temps processeur
- Mémoire : limitation de la consommation RAM
- I/O : contrôle des accès disque et réseau
- Devices : restriction d'accès aux périphériques

## Cas d'usage

### Développement d'application web
```bash
# Création d'un environnement Node.js isolé
docker run -it --rm -v $(pwd):/app -w /app node:16 npm install
docker run -it --rm -v $(pwd):/app -w /app -p 3000:3000 node:16 npm start
```

### Déploiement microservices
```bash
# Déploiement d'une API et sa base de données
docker run -d --name postgres-db -e POSTGRES_PASSWORD=secret postgres:13
docker run -d --name api-service --link postgres-db -p 8080:8080 mon-api:latest
```

### Tests d'intégration
```bash
# Environnement de test temporaire
docker run --rm -e NODE_ENV=test -v $(pwd):/app mon-app:test npm test
```

### Migration d'applications legacy
Conteneurisation d'applications existantes pour faciliter leur migration vers le cloud sans refactoring complet.

## Conclusion

Docker transforme fondamentalement l'approche du déploiement d'applications en résolvant les problèmes de portabilité et de cohérence entre environnements. En s'appuyant sur les mécanismes natifs du noyau Linux, Docker offre une solution à la fois performante et élégante.

La compréhension de l'architecture Docker et des concepts sous-jacents comme les namespaces et cgroups est essentielle pour maîtriser cette technologie. Cette base solide permet d'appréhender sereinement les aspects plus avancés comme l'orchestration, la sécurité et l'optimisation des performances.

Docker s'impose aujourd'hui comme un standard de facto dans l'industrie, ouvrant la voie vers les architectures cloud-native et l'orchestration avec Kubernetes.

## Ressources

- [Documentation officielle Docker](https://docs.docker.com/)
- [Spécifications OCI (Open Container Initiative)](https://opencontainers.org/)
- [Guide des namespaces Linux](https://man7.org/linux/man-pages/man7/namespaces.7.html)
- [Documentation cgroups du noyau Linux](https://www.kernel.org/doc/Documentation/cgroup-v1/)
- [Architecture containerd](https://containerd.io/docs/)
- [Comparaison conteneurs vs machines virtuelles - Red Hat](https://www.redhat.com/en/topics/containers/containers-vs-vms)