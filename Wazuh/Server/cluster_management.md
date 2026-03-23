# GUIDE PRATIQUE - Configuration Wazuh Cluster

## Configuration Simple: 1 Master + 2 Workers

### Infrastructure a Preparer

Vous avez besoin de 3 machines Linux:

1. Master: 192.168.1.100 (hostname: master1)
2. Worker 1: 192.168.1.101 (hostname: worker1)
3. Worker 2: 192.168.1.102 (hostname: worker2)

Chacune avec:
- Ubuntu 20.04+ ou CentOS 7+
- Minimum 2 GB RAM, 10 GB disk
- Connectivite reseau entre les machines
- Root access ou sudo

---

## ETAPE 1: Installation de Wazuh sur les 3 Machines

### Sur Master (192.168.1.100)

Installer le package:
```bash
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | apt-key add -
echo "deb https://packages.wazuh.com/4.x/apt/ stable main" > /etc/apt/sources.list.d/wazuh.list
apt-get update
apt-get install wazuh-manager
```

Arreter temporairement:
```bash
sudo systemctl stop wazuh-manager
```

### Sur Worker1 et Worker2

Memes etapes que le Master (installez sur les deux):
```bash
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | apt-key add -
echo "deb https://packages.wazuh.com/4.x/apt/ stable main" > /etc/apt/sources.list.d/wazuh.list
apt-get update
apt-get install wazuh-manager
sudo systemctl stop wazuh-manager
```

Arreter Wazuh sur tous les nodes avant configuration du cluster.

---

## ETAPE 2: Configurer le Cluster sur le MASTER

Editer /var/ossec/etc/ossec.conf sur le Master:

```bash
sudo nano /var/ossec/etc/ossec.conf
```

Trouver la section `<cluster>` (remplacer si existe, sinon ajouter):

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

Points importants:
- name: Meme sur tous les nodes
- node_name: UNIQUE pour chaque node
- node_type: "master" pour le master
- key: Meme sur TOUS les nodes (minimum 32 caracteres)

---

## ETAPE 3: Configurer le Cluster sur WORKER1

Editer /var/ossec/etc/ossec.conf sur Worker1:

```bash
sudo nano /var/ossec/etc/ossec.conf
```

Ajouter la section:

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

Important: Lister TOUS les nodes dans la section <nodes>

---

## ETAPE 4: Configurer le Cluster sur WORKER2

Editer /var/ossec/etc/ossec.conf sur Worker2:

```bash
sudo nano /var/ossec/etc/ossec.conf
```

Ajouter la section:

```xml
<cluster>
  <name>production_cluster</name>
  <node_name>worker2</node_name>
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

---

## ETAPE 5: Valider la Configuration

Sur CHAQUE machine (master, worker1, worker2):

```bash
sudo /var/ossec/bin/wazuh-control -t
```

Resultat attendu:
```
Verifying ossec.conf...
OK: ossec.conf syntax is valid
```

Si erreur, corriger et retenter.

---

## ETAPE 6: Deployer les Certificats SSL/TLS

Les certificats sont generes par Wazuh lors du premier demarrage.

Sur le MASTER, demarrer Wazuh:

```bash
sudo systemctl start wazuh-manager
sleep 10
sudo systemctl status wazuh-manager
```

Attendre quelques secondes. Les certificats sont generes dans:
```bash
ls -la /var/ossec/etc/ssl/
```

Copier les certificats vers les workers:

```bash
# Depuis le Master vers Worker1
sudo scp -r /var/ossec/etc/ssl/certs/* worker1:/var/ossec/etc/ssl/certs/
sudo scp -r /var/ossec/etc/ssl/keys/* worker1:/var/ossec/etc/ssl/keys/

# Depuis le Master vers Worker2
sudo scp -r /var/ossec/etc/ssl/certs/* worker2:/var/ossec/etc/ssl/certs/
sudo scp -r /var/ossec/etc/ssl/keys/* worker2:/var/ossec/etc/ssl/keys/
```

Verifier les permissions sur TOUS les nodes:

```bash
sudo chown -R root:ossec /var/ossec/etc/ssl/
sudo chmod 640 /var/ossec/etc/ssl/keys/*
sudo chmod 644 /var/ossec/etc/ssl/certs/*
```

---

## ETAPE 7: Demarrer le Cluster

Redemarrer le MASTER (il continue):

```bash
sudo systemctl restart wazuh-manager
sudo systemctl status wazuh-manager
```

Demarrer WORKER1:

```bash
sudo systemctl start wazuh-manager
sudo systemctl status wazuh-manager
```

Demarrer WORKER2:

```bash
sudo systemctl start wazuh-manager
sudo systemctl status wazuh-manager
```

---

## ETAPE 8: Verifier que le Cluster Fonctionne

Sur LE MASTER, executer:

```bash
sudo /var/ossec/bin/cluster_control -i
```

Resultat attendu:
```
Name            Address             Type       Status
================================================
master1         192.168.1.100       master     active
worker1         192.168.1.101       worker     active
worker2         192.168.1.102       worker     active
```

Tous les nodes doivent montrer "active".

Si un node montre "failed":
1. Verifier les logs: `sudo tail -50 /var/ossec/logs/cluster.log`
2. Redemarrer: `sudo systemctl restart wazuh-manager`
3. Attendre 30 secondes et re-verifier

---

## ETAPE 9: Configurer le Load Balancer (Optional mais Recommande)

Si vous avez un load balancer separe (ex: HAProxy):

Configuration HAProxy (/etc/haproxy/haproxy.cfg):

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

Demarrer HAProxy:
```bash
sudo systemctl restart haproxy
```

---

## ETAPE 10: Configurer les Agents pour le Cluster

Maintenant que le cluster fonctionne, configurer les agents pour se connecter au cluster.

Configuration Agent (/var/ossec/etc/ossec.conf):

OPTION A: Avec Load Balancer (RECOMMANDE)
```xml
<client>
  <server>
    <address>loadbalancer.company.com</address>
    <port>1514</port>
    <protocol>tcp</protocol>
  </server>
</client>
```

OPTION B: Sans Load Balancer (avec failover)
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

OPTION C: Un seul node (SIMPLE mais sans HA)
```xml
<client>
  <server>
    <address>worker1.company.com</address>
    <port>1514</port>
  </server>
</client>
```

Redemarrer l'agent:
```bash
sudo systemctl restart wazuh-agent
```

---

## Verification que les Agents se Connectent

Sur LE MASTER, voir les agents connectes:

```bash
sudo /var/ossec/bin/agent_control -l
```

Resultat:
```
ID   Name             IP             Status
========================
001  agent-linux      192.168.1.150  Active
002  agent-windows    192.168.1.200  Active
003  agent-macos      192.168.1.220  Active
```

Si un agent ne s'affiche pas:
1. Verifier la connexion reseau
2. Verifier l'IP du serveur dans l'agent
3. Redemarrer l'agent

---

## Commandes Utiles pour Management du Cluster

### Voir l'etat global:
```bash
sudo /var/ossec/bin/cluster_control -i
```

### Voir les logs du cluster:
```bash
sudo tail -50 /var/ossec/logs/cluster.log
```

### Redemarrer tout le cluster:
```bash
sudo /var/ossec/bin/wazuh-control -r
```

### Voir les agents par node:
```bash
sudo /var/ossec/bin/agent_control -l | head -20
```

### Forcer la synchronisation:
```bash
# Sur le master seulement
sudo /var/ossec/bin/cluster_control -i
```

---

## Troubleshooting Rapide

### Les nodes ne se connectent pas:

1. Verifier la connectivite:
```bash
ping 192.168.1.100
ping 192.168.1.101
ping 192.168.1.102
```

2. Verifier le port 1516 (cluster):
```bash
telnet 192.168.1.100 1516
```

3. Verifier la cle est identique partout:
```bash
grep "<key>" /var/ossec/etc/ossec.conf
```

4. Verifier les certificats existent:
```bash
ls -la /var/ossec/etc/ssl/
```

### Cluster montre "degraded":

1. Redemarrer le node problematique:
```bash
sudo systemctl restart wazuh-manager
```

2. Attendre 30-60 secondes

3. Re-verifier:
```bash
sudo /var/ossec/bin/cluster_control -i
```

### Les agents ne se connectent pas:

1. Verifier l'IP/hostname dans la config agent:
```bash
grep -A2 "<server>" /var/ossec/etc/ossec.conf
```

2. Tester la connexion:
```bash
nc -zv loadbalancer.company.com 1514
```

3. Redemarrer l'agent:
```bash
sudo systemctl restart wazuh-agent
```

---

## Prochaines Etapes

Une fois le cluster fonctionne:

1. Configurer tous vos agents a se connecter via le load balancer
2. Configurer Indexer (Elasticsearch) pour stockage
3. Configurer Dashboard pour visualisation
4. Mettre en place le monitoring du cluster
5. Configurer l'Alert Management
6. Configurer Event Logging
7. Ajouter External API Integrations (Slack, PagerDuty, etc.)

---

## Configuration Cluster Complete: Resume

```
MASTER (192.168.1.100)
├── /var/ossec/etc/ossec.conf
│   ├── <cluster>
│   │   ├── name: production_cluster
│   │   ├── node_name: master1
│   │   ├── node_type: master
│   │   └── key: MySecureClusterKeyHere123456789012345
│   └── Certificats: /var/ossec/etc/ssl/

WORKER1 (192.168.1.101)
├── /var/ossec/etc/ossec.conf
│   ├── <cluster>
│   │   ├── name: production_cluster
│   │   ├── node_name: worker1
│   │   ├── node_type: worker
│   │   └── key: MySecureClusterKeyHere123456789012345
│   └── Certificats: /var/ossec/etc/ssl/ (copies du master)

WORKER2 (192.168.1.102)
├── /var/ossec/etc/ossec.conf
│   ├── <cluster>
│   │   ├── name: production_cluster
│   │   ├── node_name: worker2
│   │   ├── node_type: worker
│   │   └── key: MySecureClusterKeyHere123456789012345
│   └── Certificats: /var/ossec/etc/ssl/ (copies du master)

LOAD BALANCER (192.168.1.99) - OPTIONAL
├── Port 1514 --> Master:1514, Worker1:1514, Worker2:1514
└── Balance: Round-robin ou Least connection

AGENTS --> Load Balancer:1514
```

---

C'est tout ce qu'il vous faut pour un cluster Wazuh opérationnel!