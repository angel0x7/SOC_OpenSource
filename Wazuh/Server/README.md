# Wazuh Server Management & Operations Guide

## Table des Matières
- [Introduction](#introduction)
- [Architecture de Déploiement](#architecture-de-déploiement)
- [Installation et Configuration Initiale](#installation-et-configuration-initiale)
- [Alert Management & Event Logging](#alert-management--event-logging)
- [Cluster Management](#cluster-management)
- [External API & Integrations](#external-api--integrations)
- [Monitoring et Maintenance](#monitoring-et-maintenance)
- [Troubleshooting](#troubleshooting)
- [Best Practices](#best-practices)

---

## Introduction

Ce guide couvre la **gestion complète** d'un déploiement Wazuh en production, incluant :

- **Configuration du Manager** (serveur central)
- **Event Logging & Alerting** (enregistrement des événements)
- **Clustering** (haute disponibilité avec multiple managers)
- **External API & Integrations** (intégration avec systèmes externes)
- **Maintenance & Troubleshooting** (opérations quotidiennes)

**Prérequis:**
- Linux (Ubuntu 20.04+ ou CentOS 7+)
- 2+ GB RAM, 10+ GB disque
- Accès root ou sudo
- Réseau configuré

---

## Architecture de Déploiement

### Déploiement Simple (Développement/Test)

```
┌─────────────────────────────────────┐
│   Single Wazuh Manager              │
│   192.168.1.100                     │
│  ┌─────────────────────────────┐    │
│  │ - Manager                   │    │
│  │ - Indexer (Elasticsearch)   │    │
│  │ - Dashboard                 │    │
│  └─────────────────────────────┘    │
└─────────────────────────────────────┘
         ↑
         │ Agents (1514)
         ↓
   ┌──────────────────┐
   │ Agents           │
   │ - Windows        │
   │ - Linux          │
   │ - macOS          │
   └──────────────────┘
```

### Déploiement Cluster (Production)

```
┌─────────────────────────────────────────────────────────────┐
│                     Wazuh Cluster                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  MASTER (192.168.1.100)                                     │
│  ┌──────────────────────────────────────┐                   │
│  │ - Manager (Master)                   │                   │
│  │ - API Server                         │                   │
│  │ - Cluster Communication Port 1516    │                   │
│  └──────────────────────────────────────┘                   │
│                    ↓                                        │
│  ┌─────────────────────────┐  ┌─────────────────────────┐   │
│  │ WORKER 1 (192.168.1.101)│  │ WORKER 2 (192.168.1.102)│   │
│  │ - Cluster Communication │  │ - Cluster Communication │   │
│  └─────────────────────────┘  └─────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
         ↓
  ┌─────────────────────────┐
  │  Load Balancer (Optional)
  │  HAProxy / Nginx        │
  └─────────────────────────┘
         ↓
   ┌──────────────────┐
   │ Agents           │
   │ - 100+ endpoints │
   └──────────────────┘
```

### Déploiement Complet (Enterprise)

```
┌────────────────────────────────────────────────────┐
│          WAZUH COMPLETE ARCHITECTURE               │
├────────────────────────────────────────────────────┤
│                                                    │
│  MANAGER LAYER                                     │
│  ┌──────────────────────────────────────────────┐  │
│  │ Master Manager + 2 Worker Managers (Cluster) │  │
│  │ Port 1514: Agent Communication               │  │
│  │ Port 1515: Agent Registration                │  │
│  │ Port 1516: Cluster Communication             │  │
│  │ Port 55000: API Server                       │  │
│  └──────────────────────────────────────────────┘  │
│                      ↓                             │
│  INDEXING LAYER                                    │
│  ┌──────────────────────────────────────────────┐  │
│  │ Elasticsearch (Indexing)                     │  │
│  │ Port 9200: REST API                          │  │
│  │ Port 9300: Node Communication                │  │
│  │ Filebeat: Data Collection from Manager       │  │
│  └──────────────────────────────────────────────┘  │
│                      ↓                             │
│  VISUALIZATION LAYER                               │
│  ┌──────────────────────────────────────────────┐  │
│  │ Kibana Dashboard                             │  │
│  │ Port 5601: Web UI                            │  │
│  │ Real-time Alerts & Analytics                 │  │
│  └──────────────────────────────────────────────┘  │
│                                                    │
│  AGENT LAYER                                       │
│  Thousands of Agents across Infrastructure         │
│                                                    │
└────────────────────────────────────────────────────┘
```

---

## Installation et Configuration Initiale

### Étape 1 : Installation du Manager

#### Sur Ubuntu/Debian

```bash
# Ajouter la clé GPG
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | apt-key add -

# Ajouter le repository
echo "deb https://packages.wazuh.com/4.x/apt/ stable main" > /etc/apt/sources.list.d/wazuh.list

# Installer le package
apt-get update
apt-get install wazuh-manager

# Démarrer le service
sudo systemctl start wazuh-manager
sudo systemctl enable wazuh-manager

# Vérifier le statut
sudo systemctl status wazuh-manager
```

#### Sur CentOS/RHEL

```bash
# Ajouter le repository
rpm --import https://packages.wazuh.com/key/GPG-KEY-WAZUH
echo -e '[wazuh_repo]\nname=Wazuh repository\nbaseurl=https://packages.wazuh.com/4.x/yum/\nenabled=1\ngpgcheck=1\ngpgkey=https://packages.wazuh.com/key/GPG-KEY-WAZUH' > /etc/yum.repos.d/wazuh.repo

# Installer le package
yum install wazuh-manager

# Démarrer le service
sudo systemctl start wazuh-manager
sudo systemctl enable wazuh-manager
```

### Étape 2 : Configuration Principale

**Fichier:** `/var/ossec/etc/ossec.conf`

#### Configuration d'Alert Management

```xml
<alerts>
  <log_alert_level>3</log_alert_level>
  <email_alert_level>7</email_alert_level>
</alerts>
```

**Explication:**
- `log_alert_level=3` : Enregistre les événements de niveau 3+
- `email_alert_level=7` : Envoie emails pour niveau 7+ uniquement

#### Configuration du Remote (Connexion Agents)

```xml
<remote>
  <connection>secure</connection>
  <protocol>tcp</protocol>
  <port>1514</port>
  <queue_size>131072</queue_size>
</remote>
```

#### Configuration d'Authentification Agents

```xml
<auth>
  <disabled>no</disabled>
  <port>1515</port>
  <use_source_ip>yes</use_source_ip>
  <limit_maxagents>300</limit_maxagents>
  <ciphers>HIGH:!aNULL:!MD5</ciphers>
  <ssl_verify_mode>verify_peer</ssl_verify_mode>
  <ssl_certificate>/etc/filebeat/certs/server.crt</ssl_certificate>
  <ssl_key>/etc/filebeat/certs/server.key</ssl_key>
  <ssl_auto_negotiate>yes</ssl_auto_negotiate>
</auth>
```

#### Configuration Globale

```xml
<global>
  <email_notification>no</email_notification>
  <email_from>wazuh-alerts@company.fr</email_from>
  <smtp_server>localhost</smtp_server>
  <agent_disconnection_time>10m</agent_disconnection_time>
  <status_file>/var/ossec/var/wazuh-status.json</status_file>
</global>
```

#### Configuration des Emails (Optionnel)

```xml
<email_alerts>
  <email_notification>yes</email_notification>
  <smtp_server>smtp.gmail.com</smtp_server>
  <smtp_port>587</smtp_port>
  <from>wazuh-alerts@company.fr</from>
  <to>security-team@company.fr</to>
</email_alerts>
```

#### Configuration d'Indexation (Elasticsearch)

```xml
<indexer>
  <enabled>yes</enabled>
  <hosts>
    <host>https://192.168.1.100:9200</host>
  </hosts>
  <ssl>
    <certificate_authorities>
      <ca>/etc/filebeat/certs/ca.pem</ca>
    </certificate_authorities>
    <certificate>/etc/filebeat/certs/filebeat.pem</certificate>
    <key>/etc/filebeat/certs/filebeat-key.pem</key>
  </ssl>
</indexer>
```

### Étape 3 : Validation et Redémarrage

```bash
# Valider la syntaxe
sudo /var/ossec/bin/wazuh-control -t
# Résultat attendu: "OK: ossec.conf syntax is valid"

# Redémarrer le Manager
sudo systemctl restart wazuh-manager

# Vérifier le statut
sudo systemctl status wazuh-manager

# Voir les logs
sudo tail -20 /var/ossec/logs/ossec.log
```

---

## Alert Management & Event Logging

### Concepts Fondamentaux

#### Event Logging
- **Enregistre TOUS les événements** sans filtrage
- **Fichier:** `/var/ossec/logs/ossec.log`
- **Volume:** Très important (peut faire plusieurs GB/jour)
- **Utilité:** Debugging, audit complet, conformité

#### Alert Management
- **Enregistre seulement les alertes importantes** (niveau 3+)
- **Fichier:** `/var/ossec/logs/alerts/alerts.json`
- **Volume:** Modéré (seulement alertes filtrées)
- **Utilité:** Alerting opérationnel, notifications

### Configuration d'Event Logging

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

### Gestion des Seuils d'Alerte

| Niveau | Sévérité | Action | Exemple |
|--------|----------|--------|---------|
| 1-2 | Bruit | Ignoré | Temp files |
| 3-4 | Information | Enregistré | File change |
| 5-6 | Avertissement | Enregistré + Review | Port scan |
| 7-8 | Alerte | Enregistré + Email | SSH Brute Force |
| 9-15 | Critique | Escalade immédiate | Malware detected |
| 16 | Danger extrême | Action d'urgence | System compromise |

### Configuration Recommandée par Environnement

#### Développement/Test

```xml
<alerts>
  <log_alert_level>2</log_alert_level>
  <email_alert_level>9</email_alert_level>
</alerts>
```

#### Production Standard

```xml
<alerts>
  <log_alert_level>3</log_alert_level>
  <email_alert_level>7</email_alert_level>
</alerts>
```

#### Machines Critiques (Database, FileServer)

```xml
<alerts>
  <log_alert_level>2</log_alert_level>
  <email_alert_level>5</email_alert_level>
</alerts>
```

### Consultation des Alertes et Logs

#### Voir les alertes en temps réel

```bash
# Voir les 50 derniers
sudo tail -50 /var/ossec/logs/alerts/alerts.json

# Voir en continu
sudo tail -f /var/ossec/logs/alerts/alerts.json
```

#### Chercher des alertes spécifiques

```bash
# Alertes d'un agent spécifique (ID 001)
sudo grep '"id":"001"' /var/ossec/logs/alerts/alerts.json

# Alertes d'une règle spécifique (SSH Brute Force)
sudo grep -i "ssh.*brute" /var/ossec/logs/alerts/alerts.json

# Alertes par niveau de sévérité
sudo grep '"level":7' /var/ossec/logs/alerts/alerts.json

# Alertes entre deux dates
sudo sed -n '/2024-03-22T10:00/,/2024-03-22T11:00/p' /var/ossec/logs/alerts/alerts.json
```

#### Analyser les logs d'événements

```bash
# Compter les événements totaux
sudo grep -c "" /var/ossec/logs/ossec.log

# Voir les erreurs
sudo grep "ERROR" /var/ossec/logs/ossec.log

# Connexions/déconnexions d'agents
sudo grep "New agent connected\|Connection lost" /var/ossec/logs/ossec.log

# Alertes par règle
sudo grep "Rule ID:" /var/ossec/logs/ossec.log | cut -d"'" -f2 | sort | uniq -c | sort -rn | head -10
```

### Rotation des Logs

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

**Paramètres:**
- `daily` : Rotation chaque jour
- `rotate 30` : Garder 30 versions antérieures
- `compress` : Compresser les anciennes versions (.gz)
- `postrotate` : Redémarrer Wazuh après rotation

---

## Cluster Management

### Architecture Cluster Simple

**Composants:**
- 1 Master (192.168.1.100)
- 2 Workers (192.168.1.101, 192.168.1.102)

**Communication:**
- Agents → Port 1514 (données)
- Cluster Sync → Port 1516 (synchronisation)

### Étape 1 : Installation sur les 3 Machines

#### Sur le Master

```bash
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | apt-key add -
echo "deb https://packages.wazuh.com/4.x/apt/ stable main" > /etc/apt/sources.list.d/wazuh.list
apt-get update
apt-get install wazuh-manager

sudo systemctl stop wazuh-manager
```

#### Sur Worker1 et Worker2 (même installation)

```bash
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | apt-key add -
echo "deb https://packages.wazuh.com/4.x/apt/ stable main" > /etc/apt/sources.list.d/wazuh.list
apt-get update
apt-get install wazuh-manager

sudo systemctl stop wazuh-manager
```

### Étape 2 : Configuration du Master

**Fichier:** `/var/ossec/etc/ossec.conf`

```xml
<cluster>
  <name>production_cluster</name>
  <node_name>master1</node_name>
  <node_type>master</node_type>
  <key>MySecureClusterKeyHere123456789012345</key>
  <port>1516</port>
  <bind_addr>0.0.0.0</bind_addr>
  <nodes>
    <node>master1</node>
  </nodes>
  <hidden>no</hidden>
  <disabled>no</disabled>
</cluster>
```

**Points importants:**
- `name` : Identique sur TOUS les nodes
- `node_name` : UNIQUE pour chaque node
- `node_type` : "master" sur le master, "worker" sur workers
- `key` : Identique sur TOUS (minimum 32 caractères)

### Étape 3 : Configuration des Workers

**Fichier:** `/var/ossec/etc/ossec.conf` (sur Worker1 et Worker2)

```xml
<cluster>
  <name>production_cluster</name>
  <node_name>worker1</node_name>
  <node_type>worker</node_type>
  <key>MySecureClusterKeyHere123456789012345</key>
  <port>1516</port>
  <bind_addr>0.0.0.0</bind_addr>
  <nodes>
    <node>master1</node>
    <node>worker1</node>
    <node>worker2</node>
  </nodes>
  <hidden>no</hidden>
  <disabled>no</disabled>
</cluster>
```

**Important:** Lister TOUS les nodes dans la section `<nodes>`.

### Étape 4 : Validation et Déploiement des Certificats

```bash
# Sur CHAQUE machine
sudo /var/ossec/bin/wazuh-control -t
# Résultat: "OK: ossec.conf syntax is valid"

# Copier les certificats du Master vers les Workers
sudo scp -r /var/ossec/etc/ssl/certs/* worker1:/var/ossec/etc/ssl/certs/
sudo scp -r /var/ossec/etc/ssl/keys/* worker1:/var/ossec/etc/ssl/keys/

# Vérifier les permissions
sudo chown -R root:ossec /var/ossec/etc/ssl/
sudo chmod 640 /var/ossec/etc/ssl/keys/*
sudo chmod 644 /var/ossec/etc/ssl/certs/*
```

### Étape 5 : Démarrage du Cluster

```bash
# Sur le MASTER
sudo systemctl restart wazuh-manager
sudo systemctl status wazuh-manager

# Sur les WORKERS
sudo systemctl start wazuh-manager
sudo systemctl status wazuh-manager
```

### Étape 6 : Vérification du Cluster

```bash
# Sur LE MASTER
sudo /var/ossec/bin/cluster_control -i

# Résultat attendu:
# Name            Address             Type       Status
# ================================================
# master1         192.168.1.100       master     active
# worker1         192.168.1.101       worker     active
# worker2         192.168.1.102       worker     active
```

### Commandes de Gestion du Cluster

```bash
# État complet du cluster
sudo /var/ossec/bin/cluster_control -i

# Voir les logs du cluster
sudo tail -50 /var/ossec/logs/cluster.log

# Redémarrer tout le cluster
sudo /var/ossec/bin/wazuh-control -r

# Lister les agents connectés
sudo /var/ossec/bin/agent_control -l

# Forcer la synchronisation
sudo /var/ossec/bin/cluster_control -i
```

### Configuration du Load Balancer (Optionnel mais Recommandé)

**Configuration HAProxy (/etc/haproxy/haproxy.cfg):**

```
frontend wazuh_agents
  mode tcp
  bind *:1514
  default_backend wazuh_backends
  
backend wazuh_backends
  mode tcp
  balance leastconn
  server master 192.168.1.100:1514 check
  server worker1 192.168.1.101:1514 check
  server worker2 192.168.1.102:1514 check
```

**Démarrer HAProxy:**

```bash
sudo systemctl restart haproxy
sudo systemctl status haproxy
```

### Configuration des Agents pour le Cluster

**Configuration Agent (/var/ossec/etc/ossec.conf):**

#### OPTION A : Avec Load Balancer (RECOMMANDÉ)

```xml
<client>
  <server>
    <address>loadbalancer.company.com</address>
    <port>1514</port>
    <protocol>tcp</protocol>
  </server>
</client>
```

#### OPTION B : Sans Load Balancer (avec failover)

```xml
<client>
  <server>
    <address>master1.company.com</address>
    <port>1514</port>
  </server>
  <server>
    <address>worker1.company.com</address>
    <port>1514</port>
  </server>
  <server>
    <address>worker2.company.com</address>
    <port>1514</port>
  </server>
</client>
```

---

## External API & Integrations

### Wazuh API

#### Activation de l'API

**Fichier:** `/var/ossec/etc/ossec.conf`

```xml
<api>
  <enabled>yes</enabled>
  <bind_addr>0.0.0.0</bind_addr>
  <port>55000</port>
  <https>yes</https>
  <log_path>/var/ossec/logs/api.log</log_path>
  <log_level>info</log_level>
  <cors>yes</cors>
  <cache>yes</cache>
  <access_max_login_attempts>5</access_max_login_attempts>
  <access_login_block_time>300</access_login_block_time>
  <ssl_certificate>/etc/filebeat/certs/server.crt</ssl_certificate>
  <ssl_key>/etc/filebeat/certs/server.key</ssl_key>
  <ssl_cipher>HIGH:!aNULL:!MD5</ssl_cipher>
  <ssl_protocol>TLSv1.2</ssl_protocol>
  <ssl_verify>yes</ssl_verify>
</api>
```

#### Redémarrer après modification

```bash
sudo systemctl restart wazuh-manager
```

#### Accéder à l'API

```bash
# Via curl
curl -k -X GET https://localhost:55000/api/info -H 'Authorization: Bearer YOUR_TOKEN'

# Via Python
import requests
url = "https://localhost:55000/api/info"
headers = {"Authorization": "Bearer YOUR_TOKEN"}
response = requests.get(url, headers=headers, verify=False)
print(response.json())
```

### Principales Endpoints API

| Endpoint | Méthode | Description |
|----------|---------|-------------|
| `/api/info` | GET | Informations du serveur |
| `/api/agents` | GET | Liste des agents |
| `/api/agents/{id}` | GET | Détails d'un agent |
| `/api/agents/{id}/stats` | GET | Statistiques d'un agent |
| `/api/rules` | GET | Liste des règles |
| `/api/rules/{id}` | GET | Détails d'une règle |
| `/api/alerts` | GET | Alertes récentes |
| `/api/groups` | GET | Groupes d'agents |
| `/api/manager/logs` | GET | Logs du manager |

### Intégrations Slack

**Déclaration des webhooks Slack dans ossec.conf:**

```xml
<integration>
  <name>slack</name>
  <hook_url>https://hooks.slack.com/services/YOUR/WEBHOOK/URL</hook_url>
  <alert_format>json</alert_format>
  <rule_id>5710,5720,6000</rule_id>
  <group>syscheck,rootcheck</group>
</integration>
```

**Résultat:** Les alertes sont envoyées en direct à Slack.

### Intégrations PagerDuty

```xml
<integration>
  <name>pagerduty</name>
  <api_key>PAGERDUTY_KEY</api_key>
  <alert_format>json</alert_format>
  <rule_id>5710,5720</rule_id>
</integration>
```

### Intégrations Email Avancée

```xml
<email_alerts>
  <email_notification>yes</email_notification>
  <smtp_server>smtp.company.fr</smtp_server>
  <smtp_port>587</smtp_port>
  <smtp_user>alerts@company.fr</smtp_user>
  <smtp_password>Password</smtp_password>
  <from>wazuh-alerts@company.fr</from>
  <to>security-team@company.fr</to>
  <subject>Wazuh Alert: $RULE_NAME</subject>
</email_alerts>
```

### Intégrations SIEM (Splunk, ELK)

#### Vers Splunk HTTP Event Collector (HEC)

```xml
<integration>
  <name>splunk</name>
  <hook_url>https://splunk-server:8088/services/collector</hook_url>
  <hec_token>YOUR_HEC_TOKEN</hec_token>
  <alert_format>json</alert_format>
</integration>
```

#### Via Syslog vers ELK

**Configuration Wazuh:**
```bash
# Envoyer les logs via syslog
sudo /var/ossec/bin/wazuh-control -e
```

**Configuration Logstash:**
```
input {
  syslog {
    port => 5000
    codec => json
  }
}

output {
  elasticsearch {
    hosts => ["localhost:9200"]
    index => "wazuh-%{+YYYY.MM.dd}"
  }
}
```

---

## Monitoring et Maintenance

### Santé du Manager

```bash
# Statut général
sudo /var/ossec/bin/wazuh-control -i

# Résultat attendu:
# ossec-init        is running...
# ossec-monitord    is running...
# ossec-logcollector is running...
# ossec-remoted     is running...
# ossec-analysisd   is running...
# ossec-authd       is running...
# ossec-execd       is running...
```

### Agents Connectés

```bash
# Lister tous les agents
sudo /var/ossec/bin/agent_control -l

# Voir les détails d'un agent spécifique (ID 001)
sudo /var/ossec/bin/agent_control -i 001

# Voir les statistiques
sudo /var/ossec/bin/agent_control -s
```

### Espace Disque

```bash
# Vérifier l'utilisation disque global
df -h /var/ossec/

# Vérifier la taille des logs
du -sh /var/ossec/logs/

# Vérifier la taille des données Elasticsearch
curl -s localhost:9200/_cat/indices?v | head -20
```

### Performance des Processus

```bash
# Voir la consommation CPU et mémoire
ps aux | grep ossec

# Ou avec top
top -u ossec
```

### Statistiques Journalières

**Script de reporting:**

```bash
#!/bin/bash
DATE=$(date +%Y-%m-%d)
LOGFILE="/tmp/wazuh-stats-${DATE}.txt"

echo "=== WAZUH STATISTICS ===" > $LOGFILE
echo "Date: $(date)" >> $LOGFILE
echo "" >> $LOGFILE

echo "Connected Agents:" >> $LOGFILE
sudo /var/ossec/bin/agent_control -l | grep -c "Active" >> $LOGFILE

echo "Total Alerts (24h):" >> $LOGFILE
grep "$DATE" /var/ossec/logs/alerts/alerts.json | wc -l >> $LOGFILE

echo "Errors (24h):" >> $LOGFILE
grep "$DATE" /var/ossec/logs/ossec.log | grep "ERROR" | wc -l >> $LOGFILE

echo "Disconnected Agents:" >> $LOGFILE
grep "$DATE" /var/ossec/logs/ossec.log | grep "Connection lost" | wc -l >> $LOGFILE

echo "Top 5 Alert Rules:" >> $LOGFILE
grep "$DATE" /var/ossec/logs/alerts/alerts.json | jq -r '.rule.description' | sort | uniq -c | sort -rn | head -5 >> $LOGFILE

mail -s "Wazuh Report ${DATE}" admin@company.fr < $LOGFILE
```

### Mise à Jour

```bash
# Vérifier la version actuelle
sudo /var/ossec/bin/wazuh-control -v

# Mettre à jour sur Ubuntu/Debian
sudo apt-get update
sudo apt-get upgrade wazuh-manager

# Redémarrer après mise à jour
sudo systemctl restart wazuh-manager

# Sur CentOS/RHEL
sudo yum update wazuh-manager
sudo systemctl restart wazuh-manager
```

---

## Troubleshooting

### Les agents ne se connectent pas

**Diagnostic:**

```bash
# Vérifier que le port 1514 écoute
sudo netstat -tlnp | grep 1514

# Tester la connectivité
telnet agent_ip 1514

# Vérifier les logs du manager
sudo tail -50 /var/ossec/logs/ossec.log | grep "remoted"

# Vérifier les logs de l'agent
sudo tail -50 /var/ossec/logs/ossec.log
```

**Solutions:**

```bash
# Sur le Manager:
# 1. Vérifier que le service remoted s'est démarré
sudo /var/ossec/bin/wazuh-control -i | grep remoted

# 2. Vérifier la configuration
grep -A5 "<remote>" /var/ossec/etc/ossec.conf

# 3. Redémarrer le service
sudo systemctl restart wazuh-manager

# Sur l'Agent:
# 1. Vérifier la configuration
grep -A5 "<server>" /var/ossec/etc/ossec.conf

# 2. Vérifier la connectivité réseau
ping manager_ip

# 3. Redémarrer l'agent
sudo systemctl restart wazuh-agent
```

### Le cluster n'est pas synchronisé

**Diagnostic:**

```bash
# Voir l'état du cluster
sudo /var/ossec/bin/cluster_control -i

# Voir les logs du cluster
sudo tail -50 /var/ossec/logs/cluster.log

# Tester la connectivité entre nodes (port 1516)
telnet worker_ip 1516
```

**Solutions:**

```bash
# 1. Vérifier la clé est identique partout
grep "<key>" /var/ossec/etc/ossec.conf

# 2. Vérifier les certificats existent
ls -la /var/ossec/etc/ssl/

# 3. Redémarrer le node
sudo systemctl restart wazuh-manager

# 4. Attendre 30-60 secondes et revérifier
sudo /var/ossec/bin/cluster_control -i
```

### Les alertes ne s'enregistrent pas

**Diagnostic:**

```bash
# Vérifier que des événements arrivent
sudo tail -50 /var/ossec/logs/ossec.log

# Vérifier la configuration des alertes
grep -A5 "<alerts>" /var/ossec/etc/ossec.conf

# Vérifier les permissions du répertoire
ls -la /var/ossec/logs/alerts/
```

**Solutions:**

```bash
# 1. Vérifier que log_alert_level n'est pas trop élevé
# (valeur basse = plus d'alertes, ex: 1=tout, 7=critique)

# 2. Redémarrer
sudo systemctl restart wazuh-manager

# 3. Générer une alerte de test (connexion SSH)
ssh root@localhost

# 4. Vérifier si l'alerte apparaît
grep "SSH" /var/ossec/logs/alerts/alerts.json
```

### Performance lente / CPU élevé

**Diagnostic:**

```bash
# Voir les processus les plus gourmands
ps aux | grep ossec | sort -k3 -rn

# Voir la charge système
uptime

# Voir les I/O disque
iostat -x 1 5
```

**Solutions:**

```bash
# 1. Vérifier le nombre d'agents connectés
sudo /var/ossec/bin/agent_control -l | wc -l

# 2. Vérifier la taille du fichier de log
ls -lh /var/ossec/logs/ossec.log

# 3. Archiver les vieux logs
sudo gzip /var/ossec/logs/archives/*.log

# 4. Augmenter les ressources (RAM, CPU)
# Éditer configuration agent syscheck:
# Réduire la fréquence de scan
# Réduire les répertoires monitorés
```

---

## Best Practices

### 1. Configuration et Sécurité

```
✅ Toujours utiliser des certificats SSL/TLS
✅ Changer la clé du cluster (ne pas utiliser l'exemple)
✅ Utiliser des mots de passe forts pour les emails/SMTP
✅ Restreindre l'accès au port 55000 (API) via firewall
✅ Garder les fichiers de config avec permissions strictes
```

### 2. Gestion des Agents

```
✅ Utiliser des noms d'agents significatifs
✅ Grouper les agents par département/fonction
✅ Réviser les agents inactifs mensuellement
✅ Supprimer les agents désactivés
✅ Documenter chaque enregistrement d'agent
```

### 3. Alerting

```
✅ Ne pas envoyer des emails pour chaque événement
✅ Utiliser alert_level=3 en production standard
✅ Adapter alert_level par criticité du serveur
✅ Tester les intégrations (Slack, PagerDuty, etc.)
✅ Maintenir une liste de destinataires à jour
```

### 4. Monitoring et Maintenance

```
✅ Monitorer l'espace disque (alerter à 80%, 90%, 95%)
✅ Archiver les logs anciens régulièrement
✅ Mettre à jour Wazuh mensuellement
✅ Tester les procédures de récupération
✅ Documenter tous les changements de configuration
✅ Faire des sauvegardes régulières du config
```

### 5. Clustering

```
✅ Toujours utiliser un cluster pour la production
✅ Utiliser un load balancer devant le cluster
✅ Tester le failover régulièrement
✅ Monitorer la synchronisation entre nodes
✅ Garder tous les nodes à jour (même version)
```

### 6. Sauvegardes

```bash
# Sauvegarder la configuration
sudo cp /var/ossec/etc/ossec.conf /backup/ossec.conf.$(date +%Y%m%d)

# Sauvegarder les rules personnalisées
sudo cp /var/ossec/etc/rules/local_rules.xml /backup/

# Sauvegarder les decoders personnalisés
sudo cp /var/ossec/etc/decoders/local_decoders.xml /backup/

# Sauvegarder la base de données des agents
sudo cp -r /var/ossec/queue/agent-info /backup/

# Sauvegarder les logs (archivés)
sudo tar -czf /backup/wazuh-logs-$(date +%Y%m%d).tar.gz /var/ossec/logs/archives/
```

### 7. Documentation

```
Documenter:
- Configuration des alerts par serveur
- Processus d'ajout d'agents
- Processus de creation de règles
- Processus d'escalade d'incidents
- Contacts d'urgence
- Plan de récupération
```

---

## Commandes Rapides de Référence

### Manager

```bash
# Status
sudo systemctl status wazuh-manager
sudo /var/ossec/bin/wazuh-control -i

# Redémarrer
sudo systemctl restart wazuh-manager
sudo /var/ossec/bin/wazuh-control -r

# Arrêter
sudo systemctl stop wazuh-manager

# Logs
sudo tail -f /var/ossec/logs/ossec.log
sudo grep "ERROR" /var/ossec/logs/ossec.log
```

### Agents

```bash
# Lister agents
sudo /var/ossec/bin/agent_control -l

# Détails agent
sudo /var/ossec/bin/agent_control -i 001

# Extraire clé d'agent
sudo /var/ossec/bin/manage_agents -e 001

# Supprimer agent
sudo /var/ossec/bin/manage_agents -r 001
```

### Cluster

```bash
# État
sudo /var/ossec/bin/cluster_control -i

# Logs
sudo tail -f /var/ossec/logs/cluster.log

# Redémarrer
sudo /var/ossec/bin/wazuh-control -r
```

### Alertes

```bash
# Voir alertes
sudo tail -50 /var/ossec/logs/alerts/alerts.json

# Compter alertes
sudo grep -c "id" /var/ossec/logs/alerts/alerts.json

# Filtrer par niveau
sudo grep '"level":7' /var/ossec/logs/alerts/alerts.json
```

---

## Ressources Additionnelles

### Documentation Officielle
- [Wazuh Official Documentation](https://documentation.wazuh.com)
- [Wazuh API Reference](https://documentation.wazuh.com/current/api/index.html)
- [Wazuh Ruleset](https://documentation.wazuh.com/current/user-manual/ruleset/)

### Support Communautaire
- Wazuh Community Forum
- GitHub Issues
- Slack Community

### Certifications et Formation
- Wazuh Certified Administrator
- Wazuh Training Programs
- Online Courses

---

