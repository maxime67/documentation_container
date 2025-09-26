# Gestion du service Docker

## Introduction

La gestion du service Docker est un aspect crucial de l'administration système. Docker fonctionne comme un daemon système (dockerd) qui doit être correctement configuré, surveillé et maintenu pour garantir un fonctionnement optimal des conteneurs.

Cette gestion comprend le contrôle du cycle de vie du service via systemctl, la consultation des logs pour le diagnostic des problèmes, et la configuration appropriée du système de journalisation. Une maîtrise de ces aspects est essentielle pour maintenir un environnement Docker stable et performant.

## Concepts clés

### Commandes systemctl

**systemctl** est l'utilitaire standard pour gérer les services systemd sur les distributions Linux modernes. Docker s'intègre parfaitement dans ce système de gestion des services.

**Commandes de base** :

**Démarrage du service** :
```bash
# Démarrer le service Docker
sudo systemctl start docker

# Démarrer et activer au boot
sudo systemctl enable --now docker
```

**Arrêt du service** :
```bash
# Arrêter le service Docker
sudo systemctl stop docker

# Désactiver le démarrage automatique
sudo systemctl disable docker
```

**Redémarrage du service** :
```bash
# Redémarrer le service Docker
sudo systemctl restart docker

# Recharger la configuration sans redémarrer
sudo systemctl reload docker
```

**Vérification du statut** :
```bash
# Afficher le statut détaillé
sudo systemctl status docker

# Vérifier si le service est actif
sudo systemctl is-active docker

# Vérifier si le service est activé au boot
sudo systemctl is-enabled docker
```

### Logs du daemon Docker avec journalctl

**journalctl** permet de consulter les logs systemd, incluant ceux du daemon Docker. Ces logs sont essentiels pour diagnostiquer les problèmes et surveiller l'activité.

**Consultation des logs** :
```bash
# Afficher tous les logs du service Docker
sudo journalctl -u docker

# Afficher les logs en temps réel
sudo journalctl -u docker -f

# Afficher les logs depuis un moment précis
sudo journalctl -u docker --since "2024-01-01 10:00:00"
sudo journalctl -u docker --since "1 hour ago"

# Afficher les dernières entrées
sudo journalctl -u docker -n 50
```

**Filtrage avancé** :
```bash
# Logs avec niveau de priorité (erreurs uniquement)
sudo journalctl -u docker -p err

# Logs entre deux dates
sudo journalctl -u docker --since "2024-01-01" --until "2024-01-02"

# Logs avec format JSON pour parsing
sudo journalctl -u docker -o json
```

**Gestion de l'espace disque** :
```bash
# Voir l'utilisation du journal
sudo journalctl --disk-usage

# Nettoyer les anciens logs (garder 2 semaines)
sudo journalctl --vacuum-time=2weeks

# Limiter la taille du journal
sudo journalctl --vacuum-size=1G
```

### Configuration des logs du daemon

**Fichier de configuration principal** : `/etc/docker/daemon.json`

**Configuration des drivers de logging** :
```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
```

**Drivers de logging disponibles** :
- **json-file** : Format JSON (par défaut)
- **syslog** : Envoi vers syslog
- **journald** : Intégration avec systemd
- **none** : Désactivation des logs

**Configuration avancée** :
```json
{
  "log-driver": "journald",
  "log-opts": {
    "tag": "docker/{{.Name}}"
  },
  "debug": true,
  "log-level": "info"
}
```

**Configuration systemd du daemon** :
```bash
# Éditer la configuration systemd
sudo systemctl edit docker

# Ajouter des options personnalisées
[Service]
Environment="DOCKER_OPTS=--log-level=debug"
ExecStart=
ExecStart=/usr/bin/dockerd $DOCKER_OPTS
```

## Cas d'usage

### Démarrage automatique du service Docker
```bash
# Configuration pour démarrage automatique au boot
sudo systemctl enable docker

# Vérification de la configuration
sudo systemctl is-enabled docker
systemctl list-unit-files | grep docker
```

### Diagnostic de problèmes de démarrage
```bash
# Vérifier le statut en cas d'échec
sudo systemctl status docker --no-pager -l

# Consulter les logs d'erreur récents
sudo journalctl -u docker --since "10 minutes ago" -p err

# Redémarrer en mode debug
sudo systemctl stop docker
sudo dockerd --debug
```

### Surveillance des performances
```bash
# Monitoring en temps réel des logs
sudo journalctl -u docker -f | grep -E "(error|warning|fatal)"

# Analyse des redémarrages du service
sudo journalctl -u docker --since "24 hours ago" | grep "Started\|Stopped"

# Vérification des ressources utilisées
systemctl show docker --property=MemoryCurrent,CPUUsageNSec
```

### Configuration pour environnement de production
```bash
# Configuration optimisée pour la production
cat > /etc/docker/daemon.json << EOF
{
  "log-driver": "journald",
  "log-opts": {
    "tag": "docker/{{.Name}}/{{.ID}}"
  },
  "log-level": "warn",
  "storage-driver": "overlay2",
  "live-restore": true
}
EOF

# Redémarrage pour appliquer la configuration
sudo systemctl restart docker
```

### Rotation et gestion des logs
```bash
# Configuration de la rotation des logs conteneurs
cat > /etc/docker/daemon.json << EOF
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "50m",
    "max-file": "5",
    "compress": "true"
  }
}
EOF

# Vérification de l'application
sudo systemctl restart docker
docker info | grep -A 10 "Logging Driver"
```

## Conclusion

La gestion efficace du service Docker est fondamentale pour maintenir un environnement de conteneurisation stable et performant. La maîtrise des commandes systemctl permet un contrôle précis du cycle de vie du service, tandis que la consultation des logs via journalctl facilite le diagnostic et la surveillance.

La configuration appropriée du système de logging, tant au niveau du daemon que des conteneurs individuels, est cruciale pour la maintenance préventive et la résolution rapide des incidents. Ces compétences sont essentielles pour tout administrateur système travaillant avec Docker en production.

Une surveillance proactive et une configuration adaptée à l'environnement permettent d'anticiper les problèmes et de garantir la continuité de service des applications conteneurisées.

## Ressources

- [Documentation Docker - Configuration du daemon](https://docs.docker.com/config/daemon/)
- [Guide systemd pour administrateurs](https://www.freedesktop.org/software/systemd/man/systemctl.html)
- [Documentation journalctl](https://man7.org/linux/man-pages/man1/journalctl.1.html)
- [Configuration des drivers de logging Docker](https://docs.docker.com/config/containers/logging/configure/)
- [Bonnes pratiques de logging pour Docker](https://docs.docker.com/config/containers/logging/)
- [Troubleshooting Docker daemon](https://docs.docker.com/config/daemon/troubleshoot/)