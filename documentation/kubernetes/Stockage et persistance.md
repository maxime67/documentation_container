# Stockage et persistance Kubernetes

## Introduction

La gestion du stockage constitue un défi majeur dans les environnements conteneurisés. Kubernetes propose un système de stockage flexible qui découple les applications des détails de l'infrastructure de stockage sous-jacente.

Ce document explore les différents mécanismes de stockage Kubernetes : depuis les volumes temporaires jusqu'aux solutions de stockage persistant. Comprendre ces concepts est essentiel pour gérer les données dans des applications conteneurisées, qu'il s'agisse de configuration, de secrets ou de données applicatives persistantes.

## Concepts clés

### Types de volumes

**emptyDir** : Volume temporaire
- Créé quand le pod démarre
- Partagé entre tous les conteneurs du pod
- Supprimé quand le pod est détruit
- Stockage local sur le node

**hostPath** : Montage d'un répertoire du node
- Accès direct au système de fichiers du node
- Données persistent même après suppression du pod
- Risques de sécurité et de portabilité

**configMap** : Données de configuration
- Monte des fichiers de configuration
- Lecture seule
- Mise à jour automatique possible

**secret** : Données sensibles
- Stockage chiffré des mots de passe, certificats
- Montage en tant que fichiers ou variables d'environnement
- Accès contrôlé par RBAC

### Persistent Volumes (PV)

**Définition** : Ressource de stockage dans le cluster
- Provisionnée par l'administrateur ou dynamiquement
- Cycle de vie indépendant des pods
- Différents types : NFS, iSCSI, cloud storage

**États du PV** :
- **Available** : Libre et disponible
- **Bound** : Lié à un PVC
- **Released** : PVC supprimé mais ressource non nettoyée
- **Failed** : Récupération automatique échouée

### Persistent Volume Claims (PVC)

**Rôle** : Demande de stockage par un utilisateur
- Spécifie la taille, le mode d'accès
- Kubernetes trouve un PV correspondant
- Abstraction pour les développeurs

**Modes d'accès** :
- **ReadWriteOnce (RWO)** : Lecture/écriture par un seul node
- **ReadOnlyMany (ROX)** : Lecture seule par plusieurs nodes
- **ReadWriteMany (RWX)** : Lecture/écriture par plusieurs nodes

### Storage Classes

**Objectif** : Provisioning dynamique du stockage
- Définit le type de stockage (SSD, HDD, cloud)
- Paramètres spécifiques au fournisseur
- Provisioning automatique des PV

**Reclaim Policy** :
- **Retain** : Conservation manuelle
- **Delete** : Suppression automatique
- **Recycle** : Réutilisation (déprécié)

## Cas d'usage

### Volume emptyDir pour cache temporaire
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-cache
spec:
  containers:
  - name: webapp
    image: nginx:1.20
    volumeMounts:
    - name: cache-volume
      mountPath: /tmp/cache
  - name: cache-warmer
    image: busybox:1.35
    command: ['sh', '-c', 'while true; do echo "cache warming" > /tmp/cache/status; sleep 60; done']
    volumeMounts:
    - name: cache-volume
      mountPath: /tmp/cache
  volumes:
  - name: cache-volume
    emptyDir:
      sizeLimit: 1Gi
```

### ConfigMap et Secret comme volumes
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  app.properties: |
    server.port=8080
    database.url=jdbc:mysql://db:3306/myapp
---
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
type: Opaque
data:
  db-password: cGFzc3dvcmQxMjM=  # password123 en base64
---
apiVersion: v1
kind: Pod
metadata:
  name: app-with-config
spec:
  containers:
  - name: webapp
    image: myapp:1.0
    volumeMounts:
    - name: config-volume
      mountPath: /etc/config
    - name: secrets-volume
      mountPath: /etc/secrets
      readOnly: true
  volumes:
  - name: config-volume
    configMap:
      name: app-config
  - name: secrets-volume
    secret:
      secretName: app-secrets
```

### PVC avec Storage Class
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: webapp-storage
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: fast-ssd
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp-persistent
spec:
  replicas: 1
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
      - name: webapp
        image: nginx:1.20
        volumeMounts:
        - name: webapp-storage
          mountPath: /usr/share/nginx/html
      volumes:
      - name: webapp-storage
        persistentVolumeClaim:
          claimName: webapp-storage
```

### StatefulSet avec stockage persistant
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: database-cluster
spec:
  serviceName: database-service
  replicas: 3
  selector:
    matchLabels:
      app: database
  template:
    metadata:
      labels:
        app: database
    spec:
      containers:
      - name: postgres
        image: postgres:13
        env:
        - name: POSTGRES_DB
          value: mydb
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: password
        volumeMounts:
        - name: postgres-storage
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
  - metadata:
      name: postgres-storage
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: fast-ssd
      resources:
        requests:
          storage: 20Gi
```

### Volume Snapshot
```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: webapp-snapshot
spec:
  volumeSnapshotClassName: csi-snapclass
  source:
    persistentVolumeClaimName: webapp-storage
---
# Restauration à partir d'un snapshot
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: webapp-restored
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  dataSource:
    name: webapp-snapshot
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
```

### Monitoring du stockage
```bash
# Vérifier les PV disponibles
kubectl get pv

# Détails d'un PVC
kubectl describe pvc webapp-storage

# Utilisation du stockage dans les pods
kubectl top pods --containers

# Événements liés au stockage
kubectl get events --field-selector reason=FailedMount
```

## Conclusion

Le système de stockage Kubernetes offre une abstraction puissante qui permet de découpler les applications de l'infrastructure de stockage. Les volumes temporaires répondent aux besoins de cache et de configuration, tandis que les PV/PVC assurent la persistance des données critiques.

Points clés à retenir :
- Choisir le bon type de volume selon le besoin (temporaire vs persistant)
- Les Storage Classes automatisent le provisioning et standardisent les types de stockage
- Les PVC permettent aux développeurs de demander du stockage sans connaître l'infrastructure
- Les StatefulSets intègrent naturellement le stockage persistant ordonné
- Les snapshots offrent protection et flexibilité pour la sauvegarde

Une stratégie de stockage bien pensée est cruciale pour la fiabilité et les performances des applications dans Kubernetes. Elle doit prendre en compte les besoins de performance, de durabilité et de sécurité.

## Ressources

- [Documentation Volumes](https://kubernetes.io/docs/concepts/storage/volumes/)
- [Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
- [Storage Classes](https://kubernetes.io/docs/concepts/storage/storage-classes/)
- [Volume Snapshots](https://kubernetes.io/docs/concepts/storage/volume-snapshots/)
- [Dynamic Volume Provisioning](https://kubernetes.io/docs/concepts/storage/dynamic-provisioning/)
- [CSI (Container Storage Interface)](https://kubernetes.io/docs/concepts/storage/volumes/#csi)