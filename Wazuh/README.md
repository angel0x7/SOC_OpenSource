#  Wazuh — Vue d'ensemble approfondie de la plateforme

> **Wazuh** est aujourd'hui la plateforme de cybersécurité open-source la plus adoptée au monde, avec une communauté active de plusieurs dizaines de milliers d'organisations. Elle unifie dans une seule solution deux paradigmes historiquement distincts : le **SIEM** (Security Information and Event Management) et le **XDR** (Extended Detection and Response).

---

##  Positionnement & Contexte

Avant de plonger dans les capacités techniques de Wazuh, il est essentiel de comprendre pourquoi cette plateforme s'est imposée aussi rapidement dans les SOCs modernes. Pendant des années, les équipes de sécurité ont dû jongler entre plusieurs outils spécialisés — un SIEM pour la corrélation de logs, un EDR pour les endpoints, un outil de conformité pour les audits — générant des coûts de licence élevés, des silos d'information et une complexité opérationnelle difficile à absorber pour les équipes de taille moyenne.

Wazuh répond à ce problème en proposant une **plateforme unifiée à coût zéro de licence**, capable de couvrir simultanément la détection de menaces, la réponse aux incidents, la gestion de la conformité réglementaire et la protection des workloads cloud. <br>En 2025, l'intérêt pour les solutions SIEM/XDR open-source a fortement progressé, porté par trois dynamiques majeures : la pression budgétaire sur les DSI, l'exigence croissante de conformité réglementaire (NIS2, DORA, PCI DSS), et la migration massive vers des architectures cloud et hybrides. Dans ce contexte, Wazuh s'est imposé comme **la référence open-source** face à des acteurs commerciaux comme Splunk, IBM QRadar ou CrowdStrike Falcon.

![Wazuh Security Platform — Vue globale](https://wazuh.com/wp-content/themes/wazuh-v3/assets/images/platform/security-platform-diagram.gif)

---

##  Architecture — Une conception modulaire et scalable

L'architecture de Wazuh repose sur quatre composants fondamentaux qui peuvent être déployés sur un nœud unique (pour les environnements de lab ou de petite taille) ou distribués sur un cluster multi-nœuds pour répondre aux exigences de haute disponibilité et de performance en production. Il est important de noter que **pour tout déploiement dépassant 500 agents**, la mise en cluster devient nécessaire pour maintenir des latences de traitement acceptables et garantir la résilience de la plateforme.

### 1.  Wazuh Indexer — Le moteur de stockage

Basé sur **OpenSearch** (le fork open-source d'Elasticsearch), le Wazuh Indexer est responsable de l'indexation et du stockage de toutes les alertes générées. Son architecture interne repose sur un mécanisme de **sharding et de réplication** : les données sont distribuées en fragments (shards) sur différents nœuds du cluster, ce qui garantit à la fois la tolérance aux pannes matérielles et la capacité à absorber de grands volumes d'événements en parallèle. Au-delà du stockage brut, l'indexer intègre nativement des fonctions de **data roll-up**, de gestion du cycle de vie des indices et de détection d'anomalies — des fonctionnalités qui font souvent l'objet de modules additionnels coûteux dans les solutions commerciales. Les données sont stockées au format JSON, ce qui simplifie leur exploitation via des requêtes DQL ou des intégrations tierces.

### 2.  Wazuh Server — Le cerveau analytique

Le serveur Wazuh est le composant central qui orchestre l'ensemble de la logique de détection. Il reçoit les données des agents, les fait transiter par une chaîne de **décodeurs** (qui normalisent les logs en données structurées) puis par un moteur de **règles de corrélation** qui les évalue en temps réel. Un seul serveur Wazuh peut analyser les données provenant de **centaines, voire de milliers d'agents simultanément**, avec 95% des événements traités en moins de 500 millisecondes selon les benchmarks publiés. Le serveur expose également une **API RESTful** complète qui permet l'automatisation, l'intégration avec des outils SOAR, et la gestion à distance des agents. Il embarque Filebeat pour la transmission sécurisée des données vers l'indexer.

### 3.  Wazuh Dashboard — La surface de travail de l'analyste SOC

Le dashboard est une interface web construite sur OpenSearch Dashboards, offrant des capacités avancées de data mining, de visualisation et de gestion de la configuration. Pour un analyste SOC, c'est le point d'entrée quotidien : investigation d'alertes, threat hunting, consultation des rapports de conformité, pilotage des agents. L'interface communique avec le serveur via l'API RESTful et supporte le monitoring temps réel ainsi que la corrélation avec le framework MITRE ATT&CK directement depuis les dashboards natifs.

### 4.  Wazuh Agent — Le capteur endpoint

L'agent Wazuh est un composant léger déployé directement sur les endpoints à surveiller. Il assure les fonctions de collecte, de prévention et de réponse active. Sa compatibilité multi-plateforme est l'une de ses forces distinctives :

| OS | Support |
|----|---------|
| Windows | ✅ |
| macOS | ✅ |
| Linux (toutes distributions majeures) | ✅ |
| Solaris, HP-UX, AIX | ✅ |

Pour les équipements sans agent possible (firewalls, routeurs, appliances réseau), Wazuh supporte l'ingestion de logs via **Syslog, SSH ou API**, garantissant une couverture complète même dans les infrastructures hétérogènes.

> 💡 **Wazuh 5.0 — Ce qui change :** La version 5.0 a introduit une intégration **eBPF (Extended Berkeley Packet Filter)** pour le File Integrity Monitoring sur Linux, permettant une surveillance du système de fichiers directement au niveau du noyau OS. Cela élimine les délais liés aux méthodes traditionnelles de type inotify et fournit en temps réel le contexte utilisateur/processus (UID, PID, nom du processus) associé à chaque modification — une capacité forensique majeure pour les investigations post-incident.

---

##  Options de déploiement

Wazuh s'adapte aux environnements modernes via plusieurs modes de déploiement. Les quatre orchestrateurs pris en charge nativement sont **Kubernetes**, **Docker**, **Ansible** et **Puppet**, ce qui permet une intégration dans les pipelines CI/CD existants et facilite les déploiements as-code dans les grandes organisations.

| Modèle | Adapté pour | Remarque |
|--------|-------------|----------|
| All-in-one | Labs, POC, < 50 agents | Simple, pas de HA |
| Single-node | Environnements moyens | Bonne performance |
| Multi-node Cluster | Enterprise, > 500 agents | HA, scalabilité totale |
| Wazuh Cloud (SaaS) | Toute taille | Managed, 0 ops infra |

---

##  Cas d'usage SOC — Analyse approfondie

###  1. Security Configuration Assessment (SCA)

La SCA est l'un des cas d'usage les plus directement valorisables dans un SOC, car elle répond à une réalité statistique bien documentée : **une proportion élevée des incidents de sécurité exploite des mauvaises configurations plutôt que des failles zero-day**. Les agents Wazuh effectuent des scans périodiques en comparant l'état réel des systèmes avec des benchmarks de référence — notamment les **CIS Benchmarks** — et génèrent des alertes enrichies incluant des recommandations de remediation et un mapping réglementaire. Cette approche proactive permet de réduire la surface d'attaque avant qu'un acteur malveillant puisse l'exploiter.

![Security Configuration Assessment Dashboard](https://wazuh.com/wp-content/themes/wazuh-v3/assets/images/platform/sca-dashboard.png)

---

###  2. Détection de Malwares

La détection de malwares dans Wazuh s'appuie sur une combinaison de mécanismes complémentaires. Le module **Rootcheck** inspecte le système à la recherche de processus masqués, de fichiers cachés ou d'écouteurs réseau non répertoriés — autant de signaux caractéristiques d'une infection par rootkit. Le ruleset natif de Wazuh, régulièrement mis à jour, permet de corréler les comportements observés avec des indicateurs de compromission (IoC) connus. Enfin, l'intégration avec des plateformes de threat intelligence tierces comme **MISP** ou **VirusTotal** permet d'enrichir automatiquement les alertes avec du contexte sur les familles de malwares identifiées, réduisant ainsi le temps d'investigation pour les analystes L1/L2.

![Malware Detection Dashboard](https://wazuh.com/wp-content/themes/wazuh-v3/assets/images/platform/malware-detection-dashboard.png)

---

###  3. File Integrity Monitoring (FIM)

Le FIM est souvent considéré comme la fonction la plus mature de Wazuh, héritée directement de son ancêtre **OSSEC**. Il surveille en temps réel les modifications de contenu, de permissions, de propriétaire et d'attributs sur les fichiers et répertoires critiques. Avec l'introduction d'**eBPF dans Wazuh 5.0**, chaque événement FIM est désormais enrichi du contexte utilisateur et processus en quasi-temps réel, ce qui transforme une simple alerte de modification en une piste forensique complète. Par comparaison, des outils comme IBM QRadar ou ArcSight nécessitent des **modules additionnels ou des intégrations tierces** pour atteindre un niveau de granularité équivalent — ce qui représente un avantage concurrentiel significatif de Wazuh sur ce cas d'usage spécifique. Le FIM est également un prérequis explicite de plusieurs référentiels réglementaires (**PCI DSS**, **NIST 800-53**, **HIPAA**).

![File Integrity Monitoring Dashboard](https://wazuh.com/wp-content/themes/wazuh-v3/assets/images/platform/fim-dashboard.png)

---

###  4. Threat Hunting & MITRE ATT&CK

C'est l'un des domaines où Wazuh apporte une valeur ajoutée réelle pour les analystes SOC expérimentés. Le module MITRE ATT&CK intégré nativement dans le dashboard permet de **mapper chaque alerte à une tactique et une technique précise** du framework, offrant ainsi une lecture stratégique de la menace plutôt qu'une simple liste d'événements bruts. Concrètement, cela signifie qu'au lieu d'observer des alertes isolées — une tentative de connexion échouée ici, une modification de fichier là — l'analyste peut reconstituer la **chaîne d'attaque complète** : Initial Access → Execution → Persistence, et comprendre l'intention de l'attaquant à chaque étape.

Le framework MITRE ATT&CK décrit 14 tactiques et plusieurs centaines de techniques. Wazuh enrichit ses règles avec les identifiants ATT&CK correspondants (ex : T1110 pour le brute-force, T1059 pour l'exécution de commandes). Cela standardise la communication des incidents au sein des équipes et avec le management, et permet de **mesurer objectivement la couverture de détection** du SOC via une heatmap tactiques/techniques.

![Threat Hunting Dashboard](https://wazuh.com/wp-content/themes/wazuh-v3/assets/images/platform/threat-hunting-dashboard.png)

---

###  5. Log Data Analysis

Les agents Wazuh collectent les logs systèmes et applicatifs, puis les transmettent de manière chiffrée au serveur central pour analyse. Ce qui distingue Wazuh des solutions de log management génériques comme Logstash ou Fluentd, c'est son approche **security-first** : la chaîne de traitement en deux étapes (décodeurs → règles) est spécifiquement conçue pour la détection de menaces, avec des parsers embarqués pour des centaines de sources de logs (Apache, Nginx, Windows EventLog, syslog, auditd, etc.). Les règles permettent de détecter des erreurs applicatives, des violations de politiques, des tentatives d'accès non autorisées ou des comportements anormaux, et génèrent des alertes enrichies avec un niveau de sévérité, un groupe de règle et un mapping réglementaire.

![Log Data Analysis Dashboard](https://wazuh.com/wp-content/themes/wazuh-v3/assets/images/platform/log-data-analysis-dashboard.png)

---

###  6. Vulnerability Detection

Le module de détection de vulnérabilités de Wazuh fonctionne par corrélation automatique entre l'inventaire logiciel collecté sur chaque endpoint (applications installées, versions, patchs appliqués) et les bases de données **CVE** maintenues à jour en temps réel. Depuis Wazuh 5.0, cette corrélation s'appuie sur la plateforme centrale **Wazuh CTI** (Cyber Threat Intelligence), qui agrège les données de vulnérabilités issues des fournisseurs OS (RHEL, Ubuntu, AlmaLinux), de la **NVD** (National Vulnerability Database), des bulletins de sécurité Microsoft et des alertes **CISA**. Ce mécanisme permet non seulement d'identifier les logiciels vulnérables, mais aussi de vérifier si les patchs correctifs ont bien été appliqués — réduisant significativement les faux positifs dans les environnements bien maintenus.

![Vulnerability Detection Dashboard](https://wazuh.com/wp-content/themes/wazuh-v3/assets/images/platform/vulnerability-detection-dashboard.png)

---

###  7. Incident Response

La capacité de réponse active est l'une des dimensions XDR de Wazuh. Lorsqu'une règle de détection est déclenchée et que certains critères de sévérité ou de type sont satisfaits, Wazuh peut automatiquement exécuter des **scripts de réponse** sur l'endpoint concerné ou sur d'autres composants de l'infrastructure. Les actions typiques incluent le blocage d'une adresse IP source via le firewall local, la mise en quarantaine d'un processus ou la désactivation d'un compte utilisateur compromis. Il est également possible d'exécuter des requêtes osquery sur les agents pour collecter des informations forensiques en temps réel, sans avoir à se connecter manuellement à la machine — ce qui accélère considérablement les investigations en cas d'incident actif.

![Incident Response Dashboard](https://wazuh.com/wp-content/themes/wazuh-v3/assets/images/platform/incident-response-dashboard.png)

---

###  8. Regulatory Compliance

La conformité réglementaire est souvent le premier argument qui convainc les RSSI d'adopter Wazuh. La plateforme intègre nativement des dashboards et des rapports pré-construits pour les principaux référentiels : **PCI DSS**, **NIST 800-53**, **HIPAA**, **TSC (SOC 2)**, **GDPR** et les **CIS Benchmarks**. Chaque alerte générée est automatiquement mappée aux contrôles correspondants dans ces référentiels, ce qui simplifie considérablement la préparation aux audits. Les fonctions de FIM, SCA et de détection de vulnérabilités — qui sont des exigences techniques explicites de ces standards — sont toutes couvertes nativement, sans surcoût de licence.

![Regulatory Compliance Dashboard](https://wazuh.com/wp-content/themes/wazuh-v3/assets/images/platform/regulatory-compliance-dashboard.png)

---

###  9. IT Hygiene & Inventaire

Wazuh maintient en permanence un inventaire système à jour de tous les endpoints supervisés : applications installées, processus actifs, ports ouverts, informations matérielles et OS. Cette visibilité est le socle sur lequel repose l'ensemble des autres capacités — on ne peut pas protéger ce qu'on ne connaît pas. Pour les équipes SOC, cet inventaire est directement exploitable pour la gestion des actifs, la détection de logiciels non autorisés (shadow IT) et la priorisation des efforts de remediation en cas de vulnérabilité critique.

![IT Hygiene Dashboard](https://wazuh.com/wp-content/themes/wazuh-v3/assets/images/platform/it-hygiene-dashboard.png)

---

###  10. Container Security

Avec la généralisation des architectures microservices et Kubernetes, la sécurité des conteneurs est devenue un enjeu majeur. Wazuh propose une intégration native avec le **Docker Engine** qui va au-delà de la simple collecte de logs : la plateforme surveille en temps réel les images Docker, les configurations réseau, les volumes persistants et le comportement des conteneurs en cours d'exécution. Elle génère des alertes spécifiques lorsqu'un conteneur s'exécute en **mode privilégié**, qu'une application vulnérable est détectée dans une image, ou qu'un shell interactif est lancé à l'intérieur d'un conteneur — autant de comportements atypiques qui peuvent signaler une compromission ou une erreur de configuration critique.

![Container Security Dashboard](https://wazuh.com/wp-content/themes/wazuh-v3/assets/images/platform/container-security-dashboard.png)

---

###  11. Cloud Posture Management (CSPM)

Le Cloud Security Posture Management permet à Wazuh d'intégrer directement les plateformes cloud pour collecter et analyser les données de configuration et de sécurité. Les risques identifiés — buckets S3 publics, règles de firewall trop permissives, comptes sans MFA — sont remontés comme des alertes actionnables avec des recommandations de remediation, contribuant à maintenir une posture de sécurité conforme dans les environnements multi-cloud.

![Posture Management Dashboard](https://wazuh.com/wp-content/themes/wazuh-v3/assets/images/platform/posture-management-dashboard.png)

---

###  12. Cloud Workload Protection

Wazuh s'intègre avec les principaux cloud providers et services SaaS pour étendre la surveillance au-delà des endpoints traditionnels. Les intégrations disponibles couvrent **AWS**, **Microsoft Azure**, **Google Cloud Platform (GCP)**, **Microsoft 365** et **GitHub**, permettant une centralisation des événements de sécurité dans un seul point d'analyse. Le log management centralisé facilite également le respect des obligations de rétention imposées par les référentiels réglementaires, qui exigent souvent plusieurs années d'archivage des journaux d'événements.

![Cloud Workload Protection Dashboard](https://wazuh.com/wp-content/themes/wazuh-v3/assets/images/platform/cloud-workload-protection-dashboard.png)

---

##  Wazuh vs. Solutions commerciales 

Wazuh n'est pas exempt de compromis, et il est important pour un responsable SOC de les connaître avant d'engager un déploiement à grande échelle.

**Points forts objectifs :**
- **Coût de licence nul** — avantage TCO décisif face à Splunk (facturation à la volumétrie) ou QRadar (facturation à l'EPS). Le FIM et la SCA natifs évitent l'achat de modules additionnels souvent nécessaires chez les concurrents commerciaux.
- **Facilité d'évaluation** — Wazuh est systématiquement mieux noté que Splunk Enterprise et QRadar pour la rapidité de déploiement initial et d'intégration.
- **Personnalisation** — les règles de détection sont entièrement modifiables et extensibles, sans dépendance à un éditeur.
- **Scalabilité démontrée** — la plateforme maintient des latences de traitement < 500ms à 500 EPS en conditions de charge normales.

**Limites à connaître :**
- **Courbe d'apprentissage opérationnelle** — si la mise en route est rapide, la maîtrise avancée (tuning des règles, gestion du cluster, optimisation des performances) requiert une expertise spécialisée non négligeable.
- **TCO réel** — le coût zéro de licence ne signifie pas coût zéro d'exploitation. Les organisations qui s'appuient sur du support externe déclarent en médiane des dépenses annuelles autour de **16 000 USD** pour le support spécialisé seul.
- **Efficacité de détection brute** — sur les menaces avancées de type zero-day, les plateformes XDR propriétaires comme **CrowdStrike Falcon** (score de 9.6/10 vs 8.5/10 pour Wazuh sur la détection temps réel selon les benchmarks PeerSpot) bénéficient de modèles ML/IA entraînés sur des données propriétaires massives. Wazuh compense par la personnalisation de ses règles, mais requiert plus d'effort d'ingénierie.
- **Pas de SOAR intégré** — contrairement à Splunk ES (qui intègre nativement des fonctions SOAR), Wazuh nécessite une intégration externe avec des outils comme **TheHive**, **Cortex**, ou **Shuffle** pour couvrir l'orchestration et l'automatisation des réponses de bout en bout.

| Critère | Wazuh | Splunk ES | IBM QRadar | CrowdStrike |
|---------|-------|-----------|------------|-------------|
| Licence | Gratuit | Élevé (volumétrie) | Élevé (EPS) | Élevé |
| Déploiement | ★★★★☆ | ★★★☆☆ | ★★★☆☆ | ★★★★★ |
| FIM natif | ✅ | ❌ (module) | ❌ (module) | ✅ |
| MITRE ATT&CK | ✅ natif | ✅ | ✅ | ✅ |
| SOAR intégré | ❌ | ✅ | Partiel | ✅ |
| Détection ML/IA | Partiel | ✅ | ✅ | ✅ avancé |
| Cloud natif | ✅ | ✅ | ✅ | ✅ |
| Open-source | ✅ | ❌ | ❌ | ❌ |

---

##  Tableau récapitulatif des capacités & référentiels

| Domaine | Capacité Wazuh | Référentiel associé |
|---------|---------------|---------------------|
| Détection de menaces | Rules engine, MITRE ATT&CK mapping | MITRE ATT&CK |
| Conformité réglementaire | SCA, FIM, rapports automatisés | PCI DSS, NIST, HIPAA, TSC |
| Réponse aux incidents | Active Response, commandes distantes, osquery | — |
| Vulnérabilités | Corrélation CVE, Wazuh CTI | CVE / NVD / CISA |
| Cloud Security | Intégration AWS, Azure, GCP, M365 | CIS Benchmarks |
| Conteneurs | Docker natif, Kubernetes | — |
| Inventaire & IT Hygiene | Processus, ports, applications, HW | — |

---

##  Ressources utiles

| Ressource | Lien |
|-----------|------|
| Site officiel | [wazuh.com](https://wazuh.com) |
| Documentation | [documentation.wazuh.com](https://documentation.wazuh.com) |
| GitHub | [github.com/wazuh](https://github.com/wazuh) |
| Communauté Discord | [discord.gg/rg9eZTtC7W](https://discord.gg/rg9eZTtC7W) |
| Installation rapide | [Quickstart Guide](https://documentation.wazuh.com/current/quickstart.html) |
| Wazuh Cloud (SaaS) | [cloud.wazuh.com](https://console.cloud.wazuh.com) |

---
