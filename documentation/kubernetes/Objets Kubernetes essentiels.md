# Objets Kubernetes essentiels

## Introduction

Kubernetes propose un ensemble d'objets API qui permettent de décrire et gérer les applications conteneurisées. Ces objets constituent les briques fondamentales pour déployer, exposer et maintenir des applications dans un cluster.

Ce document explore les objets essentiels de Kubernetes organisés en trois catégories principales : les Pods (unités de base), les Workload Controllers (gestion du cycle de vie) et les Services (réseau et exposition). Comprendre ces objets et leurs interactions est crucial pour maîtriser le déploiement d'applications sur Kubernetes.

## Concepts clés

### Pods

**Définition** : Plus petite unité déployable dans Kubernetes
- Contient un ou plusieurs conteneurs partageant réseau et stockage
- Possède une adresse IP unique dans le cluster
- Les conteneurs du pod communiquent via localhost

**Cycle de vie** :
1. **Pending** : Pod accepté mais conteneurs pas encore créés
2. **Running** : Pod assigné à un node, conteneurs en cours d'exécution
3. **Succeeded** : Tous les conteneurs terminés avec succès
4. **Failed** : Au moins un conteneur a échoué
5. **Unknown** : État du pod impossible à déterminer

**Quality of Service (QoS)** :
- **Guaranteed** : Requests = Limits pour toutes les ressources
- **Burstable** : Au moins une ressource avec requests < limits
- **BestEffort** : Aucune ressource request/limit spécifiée

### Workload Controllers

**Deployments** : Gestion déclarative des applications
- Déploiement en rolling update
- Rollback automatique
- Scaling horizontal

**ReplicaSets** : Maintient un nombre spécifié de répliques
- Généralement géré par les Deployments
- Sélection par labels

**StatefulSets** : Applications avec état
- Identité stable pour chaque pod
- Stockage persistant ordonné
- Déploiement et suppression ordonnés

**DaemonSets** : Un pod par node
- Logs, monitoring, stockage
- Mise à jour automatique sur nouveaux nodes

**Jobs/CronJobs** : Tâches batch
- Jobs : exécution unique jusqu'au succès
- CronJobs : planification type cron

### Services et réseau

**Types de Services** :
- **ClusterIP** : Exposition interne au cluster
- **NodePort** : Exposition sur un port de chaque node
- **LoadBalancer** : Équilibreur de charge externe
- **ExternalName** : Mapping vers un nom DNS externe

**Service Discovery** : DNS automatique
- Format : `service.namespace.svc.cluster.local`
- Variables d'environnement injectées

**Ingress** : Exposition HTTP/HTTPS
- Routage basé sur l'URL ou l'hôte
- Terminaison TLS
- Load balancing applicatif

## Cas d'usage

### Pod avec init container et ressources
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: webapp-pod
spec:
  initContainers:
  - name: init-db
    image: busybox:1.35
    command: ['sh', '-c', 'until nslookup db-service; do sleep 2; done;']
  containers:
  - name: webapp
    image: nginx:1.20
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
  - name: sidecar-logging
    image: fluentd:v1.14
    volumeMounts:
    - name: logs
      mountPath: /var/log
  volumes:
  - name: logs
    emptyDir: {}
```

### Deployment avec stratégie de déploiement
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.20
        ports:
        - containerPort: 80
```

### StatefulSet pour base de données
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres-statefulset
spec:
  serviceName: postgres-service
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:13
        env:
        - name: POSTGRES_DB
          value: mydb
  volumeClaimTemplates:
  - metadata:
      name: postgres-storage
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 1Gi
```

### Service avec différents types
```yaml
# ClusterIP (par défaut)
apiVersion: v1
kind: Service
metadata:
  name: nginx-clusterip
spec:
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
---
# NodePort
apiVersion: v1
kind: Service
metadata:
  name: nginx-nodeport
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080
```

### Ingress avec routage
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: webapp-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: webapp.exemple.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
```

### CronJob pour sauvegarde
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: database-backup
spec:
  schedule: "0 2 * * *"  # Tous les jours à 2h
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: postgres:13
            command:
            - /bin/bash
            - -c
            - pg_dump -h db-service mydb > /backup/backup-$(date +%Y%m%d).sql
          restartPolicy: OnFailure
```

## Conclusion

Les objets Kubernetes essentiels forment un écosystème cohérent pour gérer les applications conteneurisées. Les Pods constituent l'unité de base, les Workload Controllers gèrent leur cycle de vie, et les Services assurent la connectivité réseau.

Points clés à retenir :
- Les Pods sont éphémères, utilisez les Controllers pour la durabilité
- Les Deployments conviennent aux applications stateless
- Les StatefulSets sont nécessaires pour les applications avec état
- Les Services découplent la connectivité réseau des Pods
- L'Ingress permet l'exposition HTTP/HTTPS avec routage avancé
- Les ressources requests/limits assurent la QoS et la stabilité

La maîtrise de ces objets et de leurs interactions permet de concevoir des architectures robustes et scalables sur Kubernetes.

## Ressources

- [Documentation Pods](https://kubernetes.io/docs/concepts/workloads/pods/)
- [Workload Controllers](https://kubernetes.io/docs/concepts/workloads/controllers/)
- [Services et réseau](https://kubernetes.io/docs/concepts/services-networking/)
- [Gestion des ressources](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
- [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)
- [Jobs et CronJobs](https://kubernetes.io/docs/concepts/workloads/controllers/job/)