#  WAZUH DETECTION STRATEGY - Implémentation Complète

**Stratégie de détection multi-couches pour environnement Linux/Windows**

---

##  Table des Matières

1. [Vue d'Ensemble](#-vue-densemble)
2. [Architecture de Détection](#-architecture-de-détection)
3. [Sources de Données](#-sources-de-données)
4. [Parsing & Normalisation](#-parsing--normalisation)
5. [Règles de Détection](#-règles-de-détection)
6. [Contextualisation](#-contextualisation)
7. [Performance & Tuning](#-performance--tuning)
8. [Menaces Couvertes](#-menaces-couvertes)
9. [Kill Chain Detection](#-kill-chain-detection)
10. [Métriques & Monitoring](#-métriques--monitoring)
11. [Guide de Déploiement](#-guide-de-déploiement)
12. [Tests & Validation](#-tests--validation)
13. [Troubleshooting](#-troubleshooting)

---

##  Vue d'Ensemble

### Objectif

Cette implémentation fournit une **stratégie de détection complète** basée sur Wazuh pour identifier et alerter sur :

- ✅ Brute force SSH / RDP
- ✅ Privilege escalation (sudo abuse)
- ✅ Insider threats (accès anormaux)
- ✅ Exfiltration de données
- ✅ Web attacks (scans, injections)
- ✅ Kill chains multi-étapes
- ✅ Persistence mechanisms
- ✅ Process injection
- ✅ Suspicious program execution

### Principes de Design

| Principe | Implémentation |
|----------|----------------|
| **Multi-layered** | Decoders → Rules → Correlation → Kill Chain |
| **MITRE Mapped** | Chaque règle = technique ATT&CK |
| **Low False Positive** | Corrélation + whitelists + seuils calibrés |
| **Performance First** | Threads optimisés, buffers dimensionnés |
| **Actionable Alerts** | Context inclus (user, srcip, action) |
| **Forensic Ready** | Archives complètes pour investigation |

---

##  Architecture de Détection

```
┌───────────────────────────────────────────────────────────────┐
│                   AGENTS (Linux/Windows)                      │
│  • Logs: auth, syslog, audit, eventlogs                       │
│  • FIM: /etc, /bin, registre Windows                          │
│  • Syscollector: processes, ports, packages                   │
└────────────────────┬──────────────────────────────────────────┘
                     │
                     ↓
┌───────────────────────────────────────────────────────
│              MANAGER - PIPELINE DE DÉTECTION                  
│                                                              
│  1️⃣ RÉCEPTION                                                 
│     └─ remoted (port 1514)                                    
│                                                               
│  2️⃣ PARSING (Decoders)                                        
│     └─ local_decoder.xml                                     
│     └─ Extraction: user, srcip, action, resource            
│                                                              
│  3️⃣ ENRICHISSEMENT (Lists)                                    
│     └─ Whitelists, Threat Intel, Critical Assets             
│                                                               
│  4️⃣ DÉTECTION (Rules)                                         
│     └─ local_rules.xml                                       
│     └─ Simple → Corrélation → Kill Chain                    
│                                                              
│  5️⃣ ALERTING                                                 
│     └─ archives.json (tous événements)                        
│                                                           
│  6️⃣ INDEXATION                                              
│     └─ Filebeat → Opensearch/Elastic                          
└────────────────────────────────────────────────────
                     │
                     ↓
┌───────────────────────────────────────────────────────────────┐
│                  SOC / INCIDENT RESPONSE                      │
│  • Dashboard temps réel                                       │
│  • Investigation forensic (archives)                          │
│  • Actions automatisées (active-response)                     │
└───────────────────────────────────────────────────────────────┘
```

---

##  Sources de Données

### Configuration: `agent.conf`

####  Agents Linux

##### Logs Collectés

| Source | Format | Objectif |
|--------|--------|----------|
| `/var/log/auth.log` | syslog | Authentification SSH, sudo, su |
| `/var/log/secure` | syslog | Authentification RHEL/CentOS |
| `/var/log/syslog` | syslog | Événements système |
| `/var/log/messages` | syslog | Kernel messages |
| `/var/log/dpkg.log` | syslog | Installations packages (insider) |
| `/var/log/audit/audit.log` | audit | **CRITIQUE** - Exécutions, accès fichiers |
| `/var/log/apache2/*.log` | apache | Web attacks |
| `/var/log/nginx/*.log` | syslog | Web attacks |

##### File Integrity Monitoring (FIM)

```xml
<!-- Répertoires critiques - Real-time -->
<directories realtime="yes">/etc</directories>
<directories realtime="yes">/usr/bin</directories>
<directories realtime="yes">/usr/sbin</directories>
<directories realtime="yes">/bin</directories>
<directories realtime="yes">/sbin</directories>

<!-- Insider threat detection -->
<directories realtime="yes" report_changes="yes">/home</directories>
<directories realtime="yes" report_changes="yes">/root</directories>

<!-- Persistence mechanisms -->
<directories realtime="yes">/etc/init.d</directories>
<directories realtime="yes">/etc/cron.d</directories>
<directories realtime="yes">/var/spool/cron</directories>
```

**Whodata activé** → Identifie QUEL utilisateur a modifié un fichier (nécessite auditd)

##### Syscollector

- **Hardware, OS, Network**: Inventaire complet
- **Packages**: Détection vulnérabilités
- **Ports**: Connections actives
- **Processes**: Surveillance exécutions
- **Fréquence**: 1h

---

####  Agents Windows

##### Event Channels Collectés

| Channel | EventIDs | Objectif |
|---------|----------|----------|
| **Security** | 4624, 4625, 4648 | Logon/Logoff, failed auth |
| | 4688 | Process creation |
| | 4698, 4702 | Scheduled tasks |
| | 4720, 4722, 4724 | Account management |
| | 4732, 4728 | Group membership |
| | 4768, 4771, 4776 | Kerberos/NTLM |
| | 1102 | Log clearing (tampering) |
| **System** | 7045, 7036, 7040 | Service creation/changes |
| | 104 | Log cleared |
| **Sysmon** | Tous | **FORTEMENT RECOMMANDÉ** |
| **PowerShell** | 4103, 4104 | Script execution |
| **Defender** | 1116, 1117 | Malware detection |

##### File Integrity Monitoring (FIM)

```xml
<!-- Répertoires système -->
<directories>%WINDIR%\System32\drivers\etc</directories>
<directories>%WINDIR%\System32\WindowsPowerShell</directories>
<directories>%PROGRAMDATA%\Microsoft\Windows\Start Menu\Programs\StartUp</directories>

<!-- Registre - Persistence (MITRE T1547) -->
<windows_registry>HKLM\Software\Microsoft\Windows\CurrentVersion\Run</windows_registry>
<windows_registry>HKLM\Software\Microsoft\Windows\CurrentVersion\RunOnce</windows_registry>
<windows_registry>HKLM\System\CurrentControlSet\Services</windows_registry>
```

---

####  Configuration Spécialisée

##### Serveurs Web (`^(srv-web|web-|nginx-)`)

```xml
<!-- Logs spécifiques -->
<localfile>
  <log_format>apache</log_format>
  <location>/var/log/apache2/access.log</location>
</localfile>

<!-- Monitoring répertoire uploads -->
<localfile>
  <log_format>command</log_format>
  <command>du -sh /var/www/html/uploads</command>
  <frequency>3600</frequency>
</localfile>
```

---

##  Parsing & Normalisation

### Configuration: `local_decoder.xml`

Les decoders transforment les logs bruts en événements structurés analysables.

#### 1️ SSH Authentication

```xml
<!-- Failed password -->
<decoder name="custom_ssh_failed">
  <parent>sshd</parent>
  <prematch>^Failed password</prematch>
  <regex>^Failed password for( invalid user)? (\S+) from (\d+\.\d+\.\d+\.\d+)</regex>
  <order>extra_data, user, srcip</order>
  <fts>user, srcip</fts>
</decoder>
```

**Extraction:**
- `user`: Compte ciblé
- `srcip`: IP attaquant
- `fts` (First Time Seen): Track new users/IPs

**Exemple:**
```
Log: "Failed password for root from 192.168.1.50 port 22 ssh2"
→ {user: "root", srcip: "192.168.1.50"}
```

#### 2️ Sudo Commands

```xml
<decoder name="custom_sudo_command">
  <program_name>sudo</program_name>
  <prematch>TTY=</prematch>
  <regex>^(\S+) : .* USER=(\S+) ; COMMAND=(.+)$</regex>
  <order>user, dstuser, extra_data</order>
</decoder>
```

**Extraction:**
- `user`: Qui exécute sudo
- `dstuser`: Target user (souvent `root`)
- `extra_data`: Commande exacte

#### 3️ Auditd Process Execution

```xml
<decoder name="custom_auditd_execve">
  <parent>custom_auditd_base</parent>
  <prematch>type=EXECVE</prematch>
  <regex>a0="(.+)"</regex>
  <order>extra_data</order>
</decoder>
```

**Objectif:** Capturer les exécutions de programmes suspects (netcat, nmap, etc.)

#### 4️ Web Access Logs

```xml
<decoder name="custom_web_access">
  <prematch>^\d+\.\d+\.\d+\.\d+</prematch>
  <regex>^(\d+\.\d+\.\d+\.\d+) .* "(\w+) (\S+) HTTP.*" (\d+)</regex>
  <order>srcip, method, extra_data, id</order>
</decoder>
```

**Extraction:**
- `srcip`: IP client
- `method`: GET/POST/etc
- `id`: HTTP status code (404 = scan)

---

##  Règles de Détection

### Configuration: `local_rules.xml`

#### Architecture des Règles

```
Level 0 → Grouping (pas d'alerte)
Level 1-3 → Information (logs)
Level 4-6 → Low priority
Level 7-9 → Medium priority (alerte SOC)
Level 10-12 → High priority (incident response)
Level 13-16 → Critical (escalation immédiate)
```

---

### 1️ SSH Brute Force Detection

#### Règle Baseline (Level 3)

```xml
<rule id="100002" level="3">
  <if_sid>100001</if_sid>
  <description>SSH failed login $(user) from $(srcip)</description>
  <mitre><id>T1110</id></mitre>
</rule>
```

**Décision:** Enregistré mais pas d'alerte (single failure = normal)

#### Règle Corrélation (Level 10)

```xml
<rule id="100004" level="10" frequency="10" timeframe="600">
  <if_matched_sid>100002</if_matched_sid>
  <same_srcip/>
  <description>SSH brute force from $(srcip)</description>
  <mitre><id>T1110.001</id></mitre>
</rule>
```

**Logic:**
- 10 échecs en 10 minutes
- Même IP source
- → **Brute force confirmé**

#### Règle Distribuée (Level 12)

```xml
<rule id="100005" level="12" frequency="20" timeframe="300">
  <if_matched_sid>100002</if_matched_sid>
  <description>Distributed SSH brute force</description>
  <mitre><id>T1110.003</id></mitre>
</rule>
```

**Logic:**
- 20 échecs en 5 minutes
- IPs différentes
- → **Attaque coordonnée**

#### Règle Compromise (Level 13)

```xml
<rule id="100006" level="13" timeframe="300">
  <if_sid>100004, 100002</if_sid>
  <same_srcip/>
  <description>SSH compromise suspected $(srcip)</description>
  <mitre><id>T1110</id><id>T1078</id></mitre>
</rule>
```

**Logic:**
- Brute force détecté (100004)
- Suivi d'une tentative dans les 5 minutes
- → **Possibilité de succès après brute force**

---

### 2️ Privilege Escalation (Sudo)

#### Sudo to Root (Level 4)

```xml
<rule id="100021" level="4">
  <if_sid>100020</if_sid>
  <field name="dstuser">root</field>
  <description>Sudo to root by $(user)</description>
  <mitre><id>T1548.003</id></mitre>
</rule>
```

**Décision:** Information (sudo root = commun)

#### Dangerous Commands (Level 10)

```xml
<rule id="100022" level="10">
  <if_sid>100020</if_sid>
  <field name="dstuser">root</field>
  <match>bash|chmod|chown|dd |mkfs|rm -rf|passwd|shadow</match>
  <description>Dangerous sudo command $(extra_data)</description>
  <mitre><id>T1548.003</id></mitre>
</rule>
```

**Commandes surveillées:**
- `bash` → Shell root (persistence)
- `chmod/chown` → Manipulation permissions
- `passwd/shadow` → Account tampering
- `dd` → Disk manipulation
- `rm -rf` → Destruction données

#### After-Hours Sudo (Level 9)

```xml
<rule id="100024" level="9">
  <if_sid>100021</if_sid>
  <time>00:00-06:00</time>
  <description>After-hours sudo usage $(user)</description>
  <mitre><id>T1548.003</id></mitre>
</rule>
```

**Raison:** Sudo 2h du matin = suspect (insider threat)

---

### 3️ Data Exfiltration

#### Tool Detection (Level 7)

```xml
<rule id="100040" level="7">
  <decoded_as>custom_exfil_tool</decoded_as>
  <description>Exfiltration tool used $(extra_data)</description>
  <mitre><id>T1048</id></mitre>
</rule>
```

**Tools:** curl, wget, nc, netcat, scp, rsync

#### High Volume Transfer (Level 10)

```xml
<rule id="100042" level="10" frequency="50" timeframe="60">
  <if_matched_sid>100041</if_matched_sid>
  <description>High volume data transfer detected</description>
  <mitre><id>T1041</id><id>T1567</id></mitre>
</rule>
```

**Logic:** 50 transferts en 1 minute = exfiltration massive

---

### 4️ Web Attacks

#### Directory Scanning (Level 6)

```xml
<rule id="100101" level="6" frequency="20" timeframe="60">
  <if_matched_sid>100100</if_matched_sid>
  <field name="id">404</field>
  <same_srcip/>
  <description>Web directory scan (Multiple 404s)</description>
  <mitre><id>T1595.003</id></mitre>
</rule>
```

**Pattern:** 20 erreurs 404 en 1 minute = scan automatisé

---

### 5️ Kill Chain Detection (Multi-Stage)

#### SSH Brute Force → Execution (Level 14)

```xml
<rule id="100120" level="14" timeframe="600">
  <if_matched_sid>100004</if_matched_sid>
  <if_matched_sid>100061</if_matched_sid>
  <description>Kill chain detected (SSH Brute Force + Temp Execution)</description>
</rule>
```

**Scénario:**
1. Brute force SSH réussi
2. Exécution depuis `/tmp` dans les 10 minutes
3. → **Compromise confirmé**

#### Privilege Escalation → Exfiltration (Level 15)

```xml
<rule id="100121" level="15" timeframe="600">
  <if_matched_sid>100022</if_matched_sid>
  <if_matched_sid>100041</if_matched_sid>
  <description>Privilege escalation + exfiltration</description>
</rule>
```

**Scénario:**
1. Sudo dangereux exécuté
2. Transfert de données dans les 10 minutes
3. → **Insider threat / APT**

---

##  Contextualisation

### Audit Keys (audit.rules)

Fichier: `audit-keys.txt`

Ces clés permettent de **taguer** les événements auditd pour détection rapide:

```bash
# Accès fichiers sensibles
-w /etc/passwd -p wa -k passwd_access
-w /etc/shadow -p rwa -k shadow_access
-w /etc/sudoers -p wa -k sudoers_modification

# SSH configuration
-w /etc/ssh/sshd_config -p wa -k sshd_config
-w /root/.ssh -p wa -k root_ssh

# Privilege escalation
-a always,exit -F arch=b64 -S setuid -k privilege_su
-a always,exit -F path=/usr/bin/sudo -k privilege_sudo

# Persistence
-w /etc/crontab -p wa -k cron_modification
-w /etc/init.d -p wa -k init_modification

# Process execution from temp
-a always,exit -F dir=/tmp -F perm=x -k exec_from_tmp
-a always,exit -F dir=/dev/shm -F perm=x -k exec_from_shm

# Network connections (root only)
-a always,exit -F arch=b64 -S socket -F a0=2 -k ipv4_socket
-a always,exit -F arch=b64 -S connect -k network_connect
```

**Intégration:**
```bash
# Sur chaque agent Linux
sudo auditctl -l   # Vérifier règles actuelles
sudo cp audit-keys.txt /etc/audit/rules.d/wazuh.rules
sudo service auditd restart
```

---

### Suspicious Programs List

Fichier: `suspicious-programs.txt`

Classification des programmes à surveiller:

####  RED (Alerte immédiate)

```
nc, ncat, netcat, socat          # Reverse shells
nmap, masscan, rustscan          # Network scanners
msfconsole, sqlmap, hydra        # Exploitation tools
linpeas, winpeas                 # Privilege escalation
mimikatz, bloodhound             # Credential dumping
chisel, ligolo, dnscat2          # Tunneling
```

####  ORANGE (Suspect si contexte anormal)

```
nikto, gobuster, ffuf            # Web scanners
john, hashcat                    # Password cracking
plink, ngrok, frp                # Legitimate tunneling
proxychains                      # Proxy chains
```

####  YELLOW (Légitime mais surveillé)

```
bash, python, perl, ruby         # Scripting (depends usage)
tcpdump, wireshark, gdb          # Debugging tools
strace, ltrace                   # System tracing
```

**Intégration dans Rules:**

```xml
<rule id="100050" level="12">
  <if_sid>audit_execve</if_sid>
  <list field="program_name" lookup="match_key_value" check_value="red">etc/lists/suspicious_programs</list>
  <description>RED-level suspicious program executed: $(program_name)</description>
  <mitre><id>T1059</id></mitre>
</rule>
```

---

##  Performance & Tuning

### Configuration: `local_internal_options.conf`

#### Thread Allocation

```conf
# Event processing threads
analysisd.event_threads=4

# Type-specific threads
analysisd.syscheck_threads=2
analysisd.rootcheck_threads=1
analysisd.winevt_threads=2
```

**Dimensionnement:**

| EPS | event_threads | Total CPU |
|-----|---------------|-----------|
| < 1K | 4 | 4 cores |
| 1K-5K | 8 | 8 cores |
| 5K-10K | 12 | 12 cores |
| > 10K | 16+ | 16+ cores |

**Formule:**
```
event_threads = (EPS × avg_parse_time_ms) / 1000
```

#### Queue Sizing

```conf
# Input queues
analysisd.decode_event_queue_size=32768
analysisd.decode_syscheck_queue_size=16384
analysisd.decode_rootcheck_queue_size=4096

# Output queues
analysisd.alerts_queue_size=16384
analysisd.archives_queue_size=16384
```

**Principe:**
- Queue trop petite → `events_dropped > 0`
- Queue trop grande → RAM consommée

**Monitoring:**
```bash
cat /var/ossec/var/run/wazuh-analysisd.state | grep queue
```

#### First Time Seen (FTS)

```conf
analysisd.fts_list_size=32
analysisd.fts_min_size_for_str=14
```

**Usage:** Track new users, IPs, domains (anomaly detection)

---

##  Menaces Couvertes

### MITRE ATT&CK Mapping

| Tactic | Technique | Rule IDs | Description |
|--------|-----------|----------|-------------|
| **Initial Access** | T1078 | 100006 | Valid Accounts (after brute force) |
| **Credential Access** | T1110.001 | 100004 | Password Guessing |
| | T1110.003 | 100005 | Password Spraying |
| **Privilege Escalation** | T1548.003 | 100021-24 | Sudo/Su abuse |
| **Defense Evasion** | T1070 | FIM | Indicator Removal |
| **Persistence** | T1547 | FIM | Registry Run Keys (Windows) |
| | | FIM | Cron/Init modifications |
| **Execution** | T1059 | 100050 | Command & Scripting |
| **Exfiltration** | T1041 | 100040-42 | Exfiltration over C2 |
| | T1567 | 100042 | Exfiltration to Cloud |
| **Discovery** | T1595.003 | 100101 | Active Scanning |

---

##  Kill Chain Detection

### Multi-Stage Attack Scenarios

#### Scenario 1: External Compromise

```
1. SSH Brute Force (Rule 100004) [Level 10]
   └─ 10 failed attempts in 10 min

2. Successful Login
   └─ After pattern 100004

3. Execution from /tmp (Rule 100061) [Level 9]
   └─ Within 10 minutes

→ KILL CHAIN RULE 100120 [Level 14]
→ Action: Block IP + Disable account + IR escalation
```

#### Scenario 2: Insider Threat

```
1. After-hours sudo (Rule 100024) [Level 9]
   └─ Sudo at 2:00 AM

2. Dangerous command (Rule 100022) [Level 10]
   └─ chmod 777 /etc/shadow

3. Data exfiltration (Rule 100041) [Level 9]
   └─ scp to external IP

→ KILL CHAIN RULE 100121 [Level 15]
→ Action: Immediate lockout + forensic acquisition
```

---

##  Métriques & Monitoring

### Key Performance Indicators

#### Detection Quality

```bash
# False Positive Rate
FPR = (False Positives / Total Alerts) × 100
Target: < 15%

# Detection Rate
DR = (Detected Incidents / Total Incidents) × 100
Target: > 80%

# Time to Detection
TTD = Alert Timestamp - Event Timestamp
Target: < 60 seconds
```

#### System Health

```bash
# Check events dropped (DOIT être 0)
cat /var/ossec/var/run/wazuh-analysisd.state | grep events_dropped

# Check queue saturation
cat /var/ossec/var/run/wazuh-analysisd.state | grep queue_size

# Agent connectivity
/var/ossec/bin/agent_control -l
```

#### Alert Volume

```bash
# Alerts per day
grep "Alert" /var/ossec/logs/alerts/alerts.log | wc -l

# Alerts by level
grep "Rule: " /var/ossec/logs/alerts/alerts.log | \
  awk -F"level " '{print $2}' | awk '{print $1}' | sort | uniq -c
```

---

##  Guide de Déploiement

### Phase 1: Prérequis

#### Sur Manager

```bash
# Vérifier version Wazuh
/var/ossec/bin/wazuh-control info

# Vérifier espace disque (archives = volumineux)
df -h /var/ossec

# Installer jq (parsing JSON)
apt-get install jq -y
```

#### Sur Agents Linux

```bash
# Installer auditd (REQUIS)
apt-get install auditd -y

# Configurer audit rules
cp audit-keys.txt /etc/audit/rules.d/wazuh.rules
systemctl restart auditd

# Vérifier
auditctl -l | grep wazuh
```

---

### Phase 2: Déploiement Configuration

#### Backup Configuration Existante

```bash
# Sur Manager
cd /var/ossec/etc
cp -r decoders decoders.backup.$(date +%Y%m%d)
cp -r rules rules.backup.$(date +%Y%m%d)
cp local_internal_options.conf local_internal_options.conf.backup
```

#### Déploiement Decoders

```bash
# Copier local_decoder.xml
cp local_decoder.xml /var/ossec/etc/decoders/

# Tester syntax
/var/ossec/bin/wazuh-logtest -t < test_logs.txt

# Si OK, redémarrer
systemctl restart wazuh-manager
```

#### Déploiement Rules

```bash
# Copier local_rules.xml
cp local_rules.xml /var/ossec/etc/rules/

# Vérifier syntax
/var/ossec/bin/wazuh-logtest -t

# Redémarrer
systemctl restart wazuh-manager
```

#### Déploiement Configuration Agents

```bash
# Option 1: Centralisée (tous agents)
cp agent.conf /var/ossec/etc/shared/default/

# Option 2: Par groupe
cp agent.conf /var/ossec/etc/shared/linux_servers/

# Forcer mise à jour agents
/var/ossec/bin/agent_groups -s -a
```

#### Déploiement Performance Tuning

```bash
# Copier local_internal_options.conf
cp local_internal_options.conf /var/ossec/etc/

# Redémarrer
systemctl restart wazuh-manager

# Vérifier threads actifs
ps aux | grep wazuh-analysisd | wc -l
```

---

### Phase 3: Validation

#### Test 1: Decoder Parsing

```bash
# Test SSH failed login
echo 'Mar 23 15:10:00 server sshd[1234]: Failed password for root from 192.168.1.50 port 22 ssh2' | \
  /var/ossec/bin/wazuh-logtest

# Résultat attendu:
# **Phase 2: Completed decoding.
#    name: 'custom_ssh_failed'
#    user: 'root'
#    srcip: '192.168.1.50'
```

#### Test 2: Rule Triggering

```bash
# Vérifier règle existe
grep "id=\"100004\"" /var/ossec/etc/rules/local_rules.xml

# Simuler brute force (10 tentatives)
for i in {1..10}; do
  echo 'Failed password for root from 192.168.1.50' | \
    /var/ossec/bin/wazuh-logtest
  sleep 1
done

# Vérifier alerte générée
tail -f /var/ossec/logs/alerts/alerts.log | grep "100004"
```

#### Test 3: Performance Monitoring

```bash
# Baseline 24h
cat /var/ossec/var/run/wazuh-analysisd.state

# Vérifier:
# events_dropped: 0
# events_received: [nombre]
# syscheck_queue_usage: < 80%
```

---

### Phase 4: Déploiement Progressif

```
Jour 1: 10% agents (pilote)
  └─ Monitoring intensif 48h
  └─ Ajustement thresholds si FPR > 15%

Jour 3: 25% agents
  └─ Vérification events_dropped = 0
  └─ Tuning queues si nécessaire

Jour 5: 50% agents
  └─ Validation corrélations kill chain

Jour 7: 100% agents
  └─ Monitoring continu
  └─ Documentation incidents détectés
```

---

##  Tests & Validation

### Test Suite

#### 1. SSH Brute Force

```bash
# Depuis machine externe
for i in {1..15}; do
  ssh root@target_server
  # Taper mauvais password
done

# Attendu:
# - Rule 100002 (single failure) × 15
# - Rule 100004 (brute force) × 1
# - Active Response: IP bloquée (si configuré)
```

#### 2. Sudo Dangerous Command

```bash
# Sur agent Linux
sudo bash -c "echo test > /tmp/test.txt"

# Attendu:
# - Rule 100022 (dangerous sudo command)
# - Level 10
# - MITRE: T1548.003
```

#### 3. File Integrity

```bash
# Modifier fichier critique
echo "test" >> /etc/passwd

# Attendu:
# - FIM alert
# - Whodata: utilisateur identifié
# - Hash before/after
```

#### 4. Web Scan Simulation

```bash
# Depuis machine externe
for i in {1..25}; do
  curl http://target/nonexistent$i
done

# Attendu:
# - Rule 100101 (directory scan)
# - 25 × 404 errors
```

---

### Validation Logs

```bash
# Phase 1: Pre-decoding
grep "Phase 1:" /var/ossec/logs/ossec.log

# Phase 2: Decoding
grep "Decoded as:" /var/ossec/logs/ossec.log

# Phase 3: Rules
grep "Rule id:" /var/ossec/logs/alerts/alerts.log

# Alertes générées
tail -f /var/ossec/logs/alerts/alerts.json | jq .
```

---

## 🔧 Troubleshooting

### Problème 1: Decoder ne parse pas

**Symptôme:**
```
**Phase 1: Completed pre-decoding.
**Phase 2: Completed decoding. (No decoder matched)
```

**Diagnostic:**
```bash
# Afficher log brut
echo "log_line" | /var/ossec/bin/wazuh-logtest -v

# Vérifier regex
echo "log_line" | grep -P "regex_pattern"
```

**Solution:**
- Vérifier `<prematch>` match début de ligne
- Tester regex sur https://regex101.com
- Vérifier `<parent>` si decoder child

---

### Problème 2: Rule ne trigger pas

**Symptôme:** Event décodé mais pas d'alerte

**Diagnostic:**
```bash
# Vérifier rule existe
grep "id=\"100XXX\"" /var/ossec/etc/rules/local_rules.xml

# Tester avec logtest
echo "log" | /var/ossec/bin/wazuh-logtest -v

# Vérifier level minimum alerting
grep "<alerts>" /var/ossec/etc/ossec.conf
```

**Solution:**
- Vérifier `<if_sid>` ou `<decoded_as>` correct
- Vérifier `level` ≥ seuil alerting (défaut: 3)
- Tester sans `<field>` conditions

---

### Problème 3: Events Dropped

**Symptôme:**
```bash
cat /var/ossec/var/run/wazuh-analysisd.state | grep events_dropped
events_dropped: 1523
```

**Diagnostic:**
```bash
# Vérifier CPU usage
top -p $(pgrep wazuh-analysisd)

# Vérifier queue saturation
grep "queue" /var/ossec/var/run/wazuh-analysisd.state
```

**Solution:**
```conf
# Augmenter threads
analysisd.event_threads=8

# Augmenter queues
analysisd.decode_event_queue_size=65536

# Redémarrer
systemctl restart wazuh-manager
```

---

### Problème 4: High False Positive Rate

**Diagnostic:**
```bash
# Top alertes
grep "Rule:" /var/ossec/logs/alerts/alerts.log | \
  awk '{print $4}' | sort | uniq -c | sort -nr | head -10
```

**Solution:**

1. **Augmenter thresholds**
```xml
<!-- Avant -->
<rule frequency="5" timeframe="60">

<!-- Après -->
<rule frequency="10" timeframe="120">
```

2. **Ajouter whitelists**
```xml
<rule id="100004">
  <list field="srcip" lookup="address_match_key">etc/lists/whitelist_ips</list>
  <description>Brute force</description>
</rule>
```

3. **Ajouter conditions**
```xml
<!-- Ajouter time constraint -->
<time>00:00-06:00</time>

<!-- Ajouter user filter -->
<field name="user">!admin</field>
```

---

### Problème 5: Agent Non Connecté

**Diagnostic:**
```bash
# Sur Manager
/var/ossec/bin/agent_control -l

# Output:
# Agent ID 001 - Never connected

# Sur Agent
systemctl status wazuh-agent
tail /var/ossec/logs/ossec.log
```

**Solution:**
```bash
# Vérifier connectivité
telnet manager_ip 1514

# Vérifier clés
cat /var/ossec/etc/client.keys

# Re-authentifier
/var/ossec/bin/manage_agents -r 001
```

---

##  Références

### Documentation Officielle

- [Wazuh Documentation](https://documentation.wazuh.com/)
- [MITRE ATT&CK Framework](https://attack.mitre.org/)
- [Auditd Rules](https://github.com/Neo23x0/auditd)

### Configuration Files

| Fichier | Emplacement Manager | Emplacement Agent |
|---------|---------------------|-------------------|
| `agent.conf` | `/var/ossec/etc/shared/default/` | Auto-pushed |
| `local_decoder.xml` | `/var/ossec/etc/decoders/` | N/A |
| `local_rules.xml` | `/var/ossec/etc/rules/` | N/A |
| `local_internal_options.conf` | `/var/ossec/etc/` | `/var/ossec/etc/` |
| `audit.rules` | N/A | `/etc/audit/rules.d/` |

### Logs & Monitoring

| Log | Path | Objectif |
|-----|------|----------|
| Alerts | `/var/ossec/logs/alerts/alerts.json` | Alertes actionnables |
| Archives | `/var/ossec/logs/archives/archives.json` | Tous événements (forensic) |
| Ossec | `/var/ossec/logs/ossec.log` | Diagnostics Wazuh |
| Analysisd State | `/var/ossec/var/run/wazuh-analysisd.state` | Métriques performance |

---

##  Résultats Attendus

### Avec Cette Configuration

✅ **Détection:** 80%+ des attaques SSH brute force  
✅ **Time to Detection:** < 60 secondes  
✅ **False Positive Rate:** < 15%  
✅ **Couverture MITRE:** 12+ techniques  
✅ **Forensic Ready:** Archives complètes 14 jours  
✅ **Performance:** 0 events dropped jusqu'à 5K EPS  
✅ **Kill Chain:** Détection multi-stages attacks  

---

---

