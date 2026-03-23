# Event Logging - Configuration Wazuh

## Table des Matières

1. [Introduction](#introduction)
2. [Concepts Fondamentaux](#concepts-fondamentaux)
3. [Event Logging vs Alert Management](#event-logging-vs-alert-management)
4. [Configuration d'Event Logging](#configuration-devent-logging)
5. [Types d'Événements Enregistrés](#types-dévénements-enregistrés)
6. [Fichiers de Log](#fichiers-de-log)
7. [Format des Logs](#format-des-logs)
8. [Rotation et Maintenance des Logs](#rotation-et-maintenance-des-logs)
9. [Analyse des Logs](#analyse-des-logs)
10. [Bonnes Pratiques](#bonnes-pratiques)

---

## Introduction

**Event Logging** est le système de Wazuh qui enregistre **TOUS** les événements détectés par le Manager et les agents, indépendamment de leur sévérité. 

Alors que **Alert Management** enregistre seulement les alertes importantes (niveau 3 et plus), **Event Logging** enregistre tout.

Event Logging fournit un historique complet de toute l'activité du système Wazuh, utile pour le debugging, l'audit, et la conformité.

---

## Concepts Fondamentaux

### Différence entre Event et Alert

#### Un Événement
Un **Événement** est chaque information détectée par Wazuh, peu importe son importance:
- Un fichier modifié
- Une tentative de connexion
- Un processus démarrant
- Un port ouvert
- Une modification de registre

#### Une Alerte
Une **Alerte** est un Événement qui a passé les filtres d'Alert Management:
- Un événement de niveau 3 ou plus (enregistré)
- Un événement de niveau 7 ou plus (notifié par email)
- Un événement considéré important

### Relation entre Logging et Alerting

```
Événement détecté
       ↓
   Event Logging
   (tout est enregistré)
       ↓
   Analyse et Règles
       ↓
   Alert Management
   (décision: enregistrer? notifier?)
       ↓
   Stockage des Alertes
   (dans alerts.json/alerts.log)
```

- **Event Logging** enregistre les entrées brutes
- **Alert Management** décide quoi faire avec

---

## Event Logging vs Alert Management

### Alert Management

**Paramètres:**
```xml
<alerts>
  <log_alert_level>3</log_alert_level>
  <email_alert_level>7</email_alert_level>
</alerts>
```

**Résultat:**
- Seulement les alertes importantes sont enregistrées
- Les alertes niveau 1-2 sont ignorées
- Réduction du volume de données

**Fichiers:**
- `/var/ossec/logs/alerts/alerts.json`
- `/var/ossec/logs/alerts/alerts.log`

### Event Logging

**Paramètres:**
- Aucun paramètre (tout est enregistré)

**Résultat:**
- Tous les événements sont enregistrés
- Aucun filtrage sur la sévérité
- Volume de données très important

**Fichiers:**
- `/var/ossec/logs/ossec.log` (log principal du Manager)
- `/var/ossec/logs/archives/` (archives compressées)

---

## Configuration d'Event Logging

### Section du Fichier de Configuration

**Fichier:** `/var/ossec/etc/ossec.conf`

#### Format Plain (Texte lisible)
```xml
<logging>
  <log_format>plain</log_format>
</logging>
```

#### Format JSON (Machine-readable)
```xml
<logging>
  <log_format>json</log_format>
</logging>
```

#### Les Deux Formats
```xml
<logging>
  <log_format>plain</log_format>
  <log_format>json</log_format>
</logging>
```

### Explication des Paramètres

| Paramètre | Valeur | Description |
|-----------|--------|-------------|
| `log_format` | `plain` | Format texte lisible par humain |
| `log_format` | `json` | Format JSON pour traitement automatisé |

### Configuration Standard Recommandée

Pour une configuration standard:
```xml
<logging>
  <log_format>plain</log_format>
</logging>
```

Pour traitement automatisé:
```xml
<logging>
  <log_format>json</log_format>
</logging>
```

---

## Types d'Événements Enregistrés

### Événements du Manager

#### Démarrage et Arrêt
```
2024/03/22 14:00:00 1234567890.1234 [ossec-remoted] Starting listener on 192.168.1.100:1514
2024/03/22 14:00:05 1234567890.1235 [ossec-authd] Started listening on port 1515
```

#### Connexion d'Agents
```
2024/03/22 14:01:00 1234567890.1240 [ossec-remoted] New agent connected: debian-agent (id 001) from 192.168.1.101
2024/03/22 14:01:05 1234567890.1241 [ossec-remoted] Agent id '002' windows-agent from 192.168.1.102 is now connected
```

#### Déconnexion d'Agents
```
2024/03/22 14:35:00 1234567890.2000 [ossec-remoted] Agent id '001' debian-agent: Connection lost. Last activity: 2024/03/22 14:34:55
```

#### Traitement d'Événements
```
2024/03/22 14:35:00 1234567890.1234 [ossec-analysisd] Rule ID: '550' level '3' -> 'File size changed'
Agent: (001) debian-agent->>/etc/hosts
```

#### Erreurs
```
2024/03/22 14:40:00 1234567890.3000 [ossec-remoted] ERROR: Invalid agent ID
2024/03/22 14:40:05 1234567890.3001 [ossec-analysisd] ERROR: Could not open rule file
```

#### Alertes
```
2024/03/22 14:35:00 1234567890.1234 [ossec-analysisd] Rule ID: '5710' level '7' -> 'SSH Brute Force'
Agent: (001) debian-agent->>/var/log/auth.log
Full log: 'Failed password for root from 192.168.1.50'
```

---

## Fichiers de Log

### Fichier Principal

| Propriété | Valeur |
|-----------|--------|
| **Chemin** | `/var/ossec/logs/ossec.log` |
| **Contenu** | Tous les événements enregistrés par le Manager |
| **Format** | Selon la configuration (plain ou json) |
| **Taille** | Peut devenir très volumineux (plusieurs gigabytes) |
| **Rotation** | Avant rotation (quelques jours) |

### Fichiers Archives

| Propriété | Valeur |
|-----------|--------|
| **Chemin** | `/var/ossec/logs/archives/` |
| **Contenu** | Logs anciens compressés |
| **Format** | `.gz` (gzip compressed) |
| **Nommage** | `archives.2024.03.22` (par date) |
| **Conservation** | Selon la politique logrotate |

### Fichiers d'Alertes

| Propriété | Valeur |
|-----------|--------|
| **Chemin** | `/var/ossec/logs/alerts/alerts.json` et `alerts.log` |
| **Contenu** | Seulement les alertes (niveau 3+) |
| **Format** | JSON et texte |
| **Rotation** | Avant rotation (quelques jours) |

### Autres Fichiers

```
/var/ossec/var/wazuh.log
  - Log interne du Manager
  - Moins détaillé qu'ossec.log

/var/log/filebeat/filebeat
  - Log de Filebeat
  - Enregistre transmission vers Indexer
```

---

## Format des Logs

### Format Plain (Texte)

**Structure d'une ligne:**
```
DATE TIME TIMESTAMP [PROCESS] MESSAGE
```

**Exemple complet:**
```
2024/03/22 14:35:00 1234567890.1234 [ossec-analysisd] Rule ID: '5710' level '7' -> 'SSH Brute Force'
Agent: (001) debian-agent->>/var/log/auth.log
Full log: 'Failed password for root from 192.168.1.50 port 2222 ssh2'
```

**Explication:**
- `DATE`: 2024/03/22 (date de l'enregistrement)
- `TIME`: 14:35:00 (heure)
- `TIMESTAMP`: 1234567890.1234 (timestamp Unix avec fractions)
- `PROCESS`: [ossec-analysisd] (processus qui a généré le log)
- `MESSAGE`: Reste de l'information

### Format JSON

**Structure:**
```json
{
  "timestamp": "2024-03-22T14:35:00.000Z",
  "level": 7,
  "rule_id": "5710",
  "description": "SSH Brute Force",
  "agent_id": "001",
  "agent_name": "debian-agent",
  "agent_ip": "192.168.1.101",
  "log_file": "/var/log/auth.log",
  "full_log": "Failed password for root from 192.168.1.50 port 2222 ssh2",
  "decoded": {
    "srcip": "192.168.1.50",
    "user": "root",
    "protocol": "ssh2"
  }
}
```

**Avantages du JSON:**
- Facilement parsable par des scripts
- Tous les champs extraits séparément
- Importable dans bases de données
- Logique structurée

---

## Rotation et Maintenance des Logs

### Configuration Logrotate

**Fichier:** `/etc/logrotate.d/wazuh`

```conf
/var/ossec/logs/ossec.log {
    daily
    rotate 30
    delaycompress
    compress
    notifempty
    create 0640 ossec ossec
    sharedscripts
    postrotate
        /var/ossec/bin/wazuh-control restart > /dev/null 2>&1
    endscript
}
```

**Paramètres expliqués:**

| Paramètre | Signification |
|-----------|--------------|
| `daily` | Rotation chaque jour |
| `rotate 30` | Garder 30 versions antérieures |
| `delaycompress` | Ne pas compresser immédiatement |
| `compress` | Compresser les anciennes versions (.gz) |
| `notifempty` | Ne pas rotationner fichiers vides |
| `create 0640 ossec ossec` | Recréer le fichier avec ces permissions |
| `postrotate` | Commande exécutée après rotation |

### Processus de Rotation

```
Jour 1:  ossec.log (fichier courant)
Jour 2:  ossec.log → ossec.log.1 (rotation), nouveau ossec.log créé
Jour 3:  ossec.log.1 → ossec.log.2.gz (compression), ossec.log.1 nouveau
Jour 30: Anciennes versions conservées
Jour 31: Les versions les plus anciennes sont supprimées
```

### Estimation de la Taille

**Pour 10 agents:**
- Événements par agent par jour: 1000 (en moyenne)
- Total: 10 x 1000 = 10,000 événements par jour
- Format plain: ~200 bytes par événement = 2 MB par jour
- Avec rotation 30 jours: **60 MB total**

**Pour 100 agents:**
- 100,000 événements par jour
- Format plain: ~20 MB par jour
- Avec rotation 30 jours: **600 MB total**

---

## Analyse des Logs

### Consulter les Logs en Temps Réel

**Voir les 50 derniers enregistrements:**
```bash
sudo tail -50 /var/ossec/logs/ossec.log
```

**Voir les logs en continu:**
```bash
sudo tail -f /var/ossec/logs/ossec.log
```
*(Ctrl+C pour arrêter)*

### Chercher des Enregistrements Spécifiques

**Chercher tous les erreurs:**
```bash
sudo grep "ERROR" /var/ossec/logs/ossec.log
```

**Chercher les connexions d'agents:**
```bash
sudo grep "New agent connected" /var/ossec/logs/ossec.log
```

**Chercher les alertes d'un agent spécifique:**
```bash
sudo grep "Agent: (001)" /var/ossec/logs/ossec.log
```

**Chercher les alertes d'une sévérité spécifique:**
```bash
sudo grep "level '7'" /var/ossec/logs/ossec.log
```

**Chercher les déconnexions d'agents:**
```bash
sudo grep "Connection lost" /var/ossec/logs/ossec.log
```

### Compter les Occurrences

**Compter le nombre d'alertes:**
```bash
sudo grep -c "Rule ID:" /var/ossec/logs/ossec.log
```

**Compter les erreurs:**
```bash
sudo grep -c "ERROR" /var/ossec/logs/ossec.log
```

**Compter par type d'alerte:**
```bash
sudo grep "Rule ID:" /var/ossec/logs/ossec.log | grep "SSH Brute Force" | wc -l
```

### Analyser les Tendances

**Alertes par heure:**
```bash
sudo grep "14:" /var/ossec/logs/ossec.log | grep "Rule ID:" | wc -l
```

**Les alertes les plus fréquentes:**
```bash
sudo grep "Rule ID:" /var/ossec/logs/ossec.log | grep -o "'[^']*'" | sort | uniq -c | sort -rn | head -10
```

### Voir les Logs Archives

**Lister les archives:**
```bash
sudo ls -la /var/ossec/logs/archives/
```

**Consulter une archive:**
```bash
sudo gunzip -c /var/ossec/logs/archives/archives.2024.03.20.gz | grep "SSH Brute Force"
```

**Compter les enregistrements dans une archive:**
```bash
sudo gunzip -c /var/ossec/logs/archives/archives.2024.03.20.gz | wc -l
```

---

## Bonnes Pratiques

### Monitoring du Volume de Logs

**Vérifier la taille des logs:**
```bash
du -sh /var/ossec/logs/
```

**Vérifier la taille du fichier courant:**
```bash
ls -lh /var/ossec/logs/ossec.log
```

Si la taille dépasse plusieurs gigabytes:
- Vérifier que `log_alert_level` n'est pas trop bas
- Vérifier que la rotation des logs fonctionne correctement
- Considérer archiver les vieux logs sur un système externe

### Sauvegarde des Logs

**Copier les archives:**
```bash
sudo cp /var/ossec/logs/archives/* /backup/wazuh-logs/
```

**Ou avec compression:**
```bash
sudo tar -czf /backup/wazuh-logs-$(date +%Y%m%d).tar.gz /var/ossec/logs/archives/
```

### Sécurité des Logs

Les fichiers de logs contiennent des informations sensibles.

**Vérifier les permissions:**
```bash
ls -la /var/ossec/logs/
```

Doit afficher:
```
drwxr-x--- 4 ossec ossec 4096 /var/ossec/logs/
```

Les permissions par défaut sont correctes (readable seulement par owner et group).

**Si nécessaire, restreindre l'accès:**
```bash
sudo chmod 750 /var/ossec/logs/
```

### Analyse Périodique

**Script de monitoring quotidien:**
```bash
#!/bin/bash
DATE=$(date +%Y%m%d)
LOGFILE="/var/ossec/logs/analysis-${DATE}.txt"

echo "=== WAZUH LOG ANALYSIS ===" > $LOGFILE
echo "Date: $(date)" >> $LOGFILE
echo "" >> $LOGFILE

echo "Total events:" >> $LOGFILE
grep -c "" /var/ossec/logs/ossec.log >> $LOGFILE

echo "Total alerts (level 7+):" >> $LOGFILE
grep "level '7'" /var/ossec/logs/ossec.log | wc -l >> $LOGFILE

echo "Top 5 alerts:" >> $LOGFILE
grep "Rule ID:" /var/ossec/logs/ossec.log | grep -o "'[^']*'" | sort | uniq -c | sort -rn | head -5 >> $LOGFILE

echo "Disconnected agents:" >> $LOGFILE
grep "Connection lost" /var/ossec/logs/ossec.log >> $LOGFILE

mail -s "Wazuh Log Analysis ${DATE}" admin@company.fr < $LOGFILE
```

**Exécuter quotidiennement via cron:**
```bash
0 0 * * * /usr/local/bin/wazuh-log-analysis.sh
```

### Archivage à Long Terme

**Archiver et supprimer:**
```bash
sudo tar -czf /archive/wazuh-logs-2024-03.tar.gz /var/ossec/logs/archives/archives.2024.03*
sudo rm /var/ossec/logs/archives/archives.2024.03*
```

**Ou utiliser rsync vers un serveur distant:**
```bash
sudo rsync -av /var/ossec/logs/archives/ backup-server:/archive/wazuh-logs/
```

---

## Résumé

| Aspect | Event Logging | Alert Management |
|--------|---------------|------------------|
| **Enregistrement** | TOUS les événements | Seulement alertes niveau 3+ |
| **Fichier** | `/var/ossec/logs/ossec.log` | `/var/ossec/logs/alerts/alerts.json` |
| **Filtrage** | Aucun | Par sévérité |
| **Volume** | Très important | Modéré |
| **Utilité** | Debugging, audit complet | Alerting opérationnel |
| **Conservation** | Configurable (jours à mois) | Configurable |

**Les deux systèmes coexistent** et se complètent:
- **Event Logging** enregistre dans `/var/ossec/logs/ossec.log`
- **Alert Management** enregistre dans `/var/ossec/logs/alerts/alerts.json`
- Event Logging est **PLUS complet**
- Alert Management est **PLUS utile** opérationnellement

---

## Configuration Recommandée

### Installation Standard

```xml
<logging>
  <log_format>plain</log_format>
</logging>
```

### Si Volume de Données est un Problème

```xml
<logging>
  <log_format>json</log_format>
</logging>
```

Format JSON utilise moins d'espace (plus compact).

### Si Vous Avez Besoin des Deux Formats

```xml
<logging>
  <log_format>plain</log_format>
  <log_format>json</log_format>
</logging>
```

---

