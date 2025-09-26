# Architecture et composants Kubernetes

## Introduction

Kubernetes est une plateforme d'orchestration de conteneurs qui automatise le déploiement, la mise à l'échelle et la gestion des applications conteneurisées. Pour maîtriser Kubernetes, il est essentiel de comprendre son architecture distribuée et ses composants principaux.

Ce document présente les éléments fondamentaux de l'architecture Kubernetes : les composants du control plane, les worker nodes, ainsi que les concepts organisationnels comme les namespaces, labels et annotations qui permettent de structurer et gérer efficacement les ressources.

## Concepts clés

### Control Plane vs Worker Nodes

**Control Plane** : Cerveau du cluster qui prend toutes les décisions
- Gère l'état désiré du cluster
- Expose l'API Kubernetes
- Planifie les workloads

**Worker Nodes** : Machines qui exécutent les applications
- Hébergent les pods (unités de déploiement)
- Communiquent avec le control plane
- Exécutent les conteneurs

### Composants principaux

**API Server** : Point d'entrée unique pour toutes les interactions
- Interface REST pour tous les composants
- Valide et traite les requêtes
- Point central de communication

**etcd** : Base de données distribuée
- Stocke l'état de tout le cluster
- Sauvegarde la configuration
- Source unique de vérité

**kubelet** : Agent sur chaque worker node
- Gère les pods sur le node
- Communique avec l'API Server
- S'assure que les conteneurs fonctionnent

**kube-proxy** : Proxy réseau sur chaque node
- Gère les règles de routage réseau
- Implémente les services Kubernetes
- Équilibrage de charge

### Concepts organisationnels

**Namespaces** : Espaces de noms virtuels
- Isolation logique des ressources
- Séparation par environnement ou équipe
- Gestion des quotas et permissions

**Labels** : Paires clé-valeur attachées aux objets
- Système d'étiquetage flexible
- Utilisés pour organiser et sélectionner
- Exemples : `app=nginx`, `environment=prod`

**Selectors** : Filtres basés sur les labels
- Permettent de sélectionner des groupes d'objets
- Utilisés par les services et déploiements
- Syntaxe : `app=nginx,version=1.2`

**Annotations** : Métadonnées non-identifiantes
- Informations additionnelles sur les objets
- Non utilisées pour la sélection
- Exemples : documentation, outils externes

## Cas d'usage

### Séparation par environnements avec namespaces
```yaml
# Namespace développement
apiVersion: v1
kind: Namespace
metadata:
  name: development
---
# Namespace production
apiVersion: v1
kind: Namespace
metadata:
  name: production
```

### Organisation avec labels et selectors
```yaml
# Pod avec labels
apiVersion: v1
kind: Pod
metadata:
  name: web-app
  labels:
    app: nginx
    environment: prod
    version: "1.2"
spec:
  containers:
  - name: nginx
    image: nginx:1.20

---
# Service sélectionnant les pods
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
    environment: prod
  ports:
  - port: 80
```

### Utilisation d'annotations
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: annotated-pod
  annotations:
    description: "Pod nginx pour l'application web principale"
    maintainer: "equipe-devops@entreprise.com"
    last-updated: "2024-01-15"
spec:
  containers:
  - name: nginx
    image: nginx:1.20
```

## Conclusion

L'architecture Kubernetes repose sur une séparation claire entre le control plane qui orchestre et les worker nodes qui exécutent. Les composants principaux (API Server, etcd, kubelet, kube-proxy) travaillent ensemble pour maintenir l'état désiré du cluster.

Les concepts organisationnels (namespaces, labels, annotations) sont essentiels pour structurer et gérer efficacement les ressources dans un environnement de production. Ils permettent l'isolation, l'organisation et la documentation des workloads.

Points clés à retenir :
- Le control plane prend les décisions, les worker nodes les exécutent
- L'API Server est le point central de communication
- Les namespaces isolent logiquement les ressources
- Les labels permettent la sélection et l'organisation
- Les annotations ajoutent des métadonnées documentaires

## Ressources

- [Documentation officielle - Architecture](https://kubernetes.io/docs/concepts/overview/components/)
- [Guide des concepts Kubernetes](https://kubernetes.io/docs/concepts/)
- [Namespaces](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/)
- [Labels et Selectors](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/)
- [Annotations](https://kubernetes.io/docs/concepts/overview/working-with-objects/annotations/)