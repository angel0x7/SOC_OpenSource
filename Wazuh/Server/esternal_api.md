# External API Integration - Wazuh

## Table des Matieres

1. Introduction
2. Concepts Fondamentaux
3. Configuration Base
4. Parametres de Configuration
5. Filtres Optionnels
6. Integration Slack
7. Integration PagerDuty
8. Integration VirusTotal
9. Integration Shuffle
10. Integration Maltiverse
11. Integrations Personnalisees
12. Validation et Tests
13. Troubleshooting
14. Bonnes Pratiques

---

## 1. Introduction

External API Integration est le module Integrator de Wazuh qui permet au Manager de se connecter a des services externes pour envoyer automatiquement les alertes. Ce systeme permet l'orchestration des alertes, l'automation des reponses, et l'integration avec les outils de communication et de gestion d'incidents.

Les services supportes sont:
- Slack (messagerie d'equipe)
- PagerDuty (gestion des incidents)
- VirusTotal (analyse de malware)
- Shuffle (SOAR - orchestration)
- Maltiverse (threat intelligence)
- Custom (services personnalises)

---

## 2. Concepts Fondamentaux

### Qu'est-ce que l'Integrator Module

L'Integrator Module est un daemon Wazuh (wazuh-integratord) qui:
- Lit les alertes generees par le Manager
- Filtre les alertes selon les criteres definis
- Formate les donnees selon le service externe
- Envoie les alertes via API HTTP/HTTPS
- Gere les reponses des services externes

### Flux d'une Alerte vers une Integration

Etape 1: Generation de l'Alerte
- Le Manager detecte un evenement
- L'analyse applique les regles
- Une alerte est generee

Etape 2: Enregistrement de l'Alerte
- L'alerte est stockee dans alerts.json
- L'alerte est enregistree dans alerts.log

Etape 3: Lecture par Integrator
- Le daemon Integrator lit l'alerte
- Il verifie si l'alerte correspond aux filtres

Etape 4: Filtrage
- L'alerte est testee contre tous les filtres definis
- Si elle correspond, elle est traitee
- Si elle ne correspond pas, elle est ignoree

Etape 5: Formatage
- Les donnees de l'alerte sont formatees en JSON
- Les champs personnalises sont ajoutes si specifies

Etape 6: Envoi vers l'API
- L'alerte est envoyee au service externe via HTTP/HTTPS
- L'authentification est effectuee (clés, tokens, webhooks)
- La reponse du service est enregistree

Etape 7: Stockage et Logging
- L'envoi est loggé dans les fichiers de logs
- Les erreurs sont enregistrees pour troubleshooting

### Difference entre les Services

SLACK:
- Type: Messagerie de groupe
- Authentification: Incoming Webhooks
- Format: JSON formaté pour affichage
- Cas d'usage: Notifications en temps reel pour l'equipe

PAGERDUTY:
- Type: Gestion des incidents
- Authentification: API Key
- Format: Events API v2
- Cas d'usage: Escalade automatique des incidents

VIRUSTOTAL:
- Type: Analyse de malware
- Authentification: API Key
- Format: Requetes d'analyse de fichiers/URLs
- Cas d'usage: Verification de fichiers suspects

SHUFFLE:
- Type: SOAR (automation)
- Authentification: Webhooks
- Format: JSON personalise
- Cas d'usage: Workflows d'automation

MALTIVERSE:
- Type: Threat Intelligence
- Authentification: API Key
- Format: API Maltiverse
- Cas d'usage: Enrichissement des alertes

---

## 3. Configuration Base

### Fichier de Configuration

Fichier: /var/ossec/etc/ossec.conf

Structure generale:

```
<ossec_config>
  <integration>
    <n>slack</n>
    <hook_url>https://hooks.slack.com/services/...</hook_url>
    <alert_format>json</alert_format>
    <level>7</level>
  </integration>
</ossec_config>
```

### Sections de Configuration

OBLIGATOIRES:
- <n>: Nom du service (slack, pagerduty, virustotal, shuffle, maltiverse, ou custom-*)
- <alert_format>: Format des alertes (toujours json)
- <hook_url> OU <api_key>: Selon le service

OPTIONNELS:
- <rule_id>: Filtrer par ID de regle
- <level>: Filtrer par severite
- <group>: Filtrer par groupe de regle
- <event_location>: Filtrer par emplacement de l'event
- <options>: Personnalisation JSON

---

## 4. Parametres de Configuration

### Parametre: n

Fonction: Indique le nom du service a integrer

Valeurs possibles:
- slack: Integration Slack
- pagerduty: Integration PagerDuty
- virustotal: Integration VirusTotal
- shuffle: Integration Shuffle
- maltiverse: Integration Maltiverse
- custom-*: Integration personnalisee (exemple: custom-jira, custom-teams)

Exemple:
```
<n>slack</n>
<n>pagerduty</n>
<n>custom-myservice</n>
```

### Parametre: hook_url

Fonction: URL du webhook pour communication avec le service

Requis pour: Slack, Shuffle, Maltiverse

Format: URL HTTPS valide

Exemple:
```
<hook_url>https://hooks.slack.com/services/T00000000/B00000000/XXXXXXXXXXXXXXXXXXXXX</hook_url>
<hook_url>https://api.maltiverse.com/integration/webhook</hook_url>
```

### Parametre: api_key

Fonction: Cle d'authentification API du service

Requis pour: PagerDuty, VirusTotal, Maltiverse

Obtention: Selon le service (voir sections specifiques)

Exemple:
```
<api_key>a1b2c3d4e5f6g7h8i9j0k1l2m3n4</api_key>
```

### Parametre: alert_format

Fonction: Format d'enregistrement des alertes

Valeur acceptee: json (obligatoire)

Fonction: Force le Manager a enregistrer les alertes en format JSON, ce qui permet a l'Integrator de parser les champs

Exemple:
```
<alert_format>json</alert_format>
```

Note: C'est un parametre obligatoire pour toutes les integrations.

---

## 5. Filtres Optionnels

### Filtre: rule_id

Fonction: Envoyer les alertes uniquement si leur ID de regle correspond

Syntaxe: Coma-separated list d'IDs

Exemple:
```
<rule_id>5710,5711,5712</rule_id>
```

Comportement:
- L'alerte est testee si son rule_id est dans la liste
- Si oui, l'alerte est envoyee
- Si non, l'alerte est ignoree

### Filtre: level

Fonction: Envoyer les alertes de severite egale ou superieure

Syntaxe: Niveau entre 0 et 16

Exemple:
```
<level>7</level>
```

Comportement:
- Seules les alertes de niveau 7 ou plus sont envoyees
- Les alertes de niveau 0-6 sont ignorees
- C'est un seuil minimum, pas exact

### Filtre: group

Fonction: Envoyer les alertes uniquement si leur groupe correspond

Syntaxe: Coma-separated list de groupes

Exemple:
```
<group>syscheck,fim,malware</group>
```

Groupes courants:
- syscheck: Modifications de fichiers
- authentication: Authentification
- malware: Detection de malware
- ossec: Alertes du systeme Wazuh
- access_control: Controle d'acces
- fim: File Integrity Monitoring

### Filtre: event_location

Fonction: Envoyer les alertes uniquement si l'emplacement correspond

Syntaxe: Expression sregex (regex Perl)

Exemple:
```
<event_location>/var/log/auth.log</event_location>
<event_location>.*->syslog</event_location>
<event_location>Windows.*</event_location>
```

### Parametre: options

Fonction: Ajouter ou personnaliser des champs JSON dans les alertes

Syntaxe: Objet JSON

Exemple:
```
<options>{"channel":"#security","icon_emoji":":lock:"}</options>
```

Cas d'usage:
- Ajouter des champs personnalises
- Overrider des champs existants
- Parametres specifiques au service

---

## 6. Integration Slack

### Concepts

Slack est une plateforme de messagerie collaborative. L'integration Wazuh utilise les "Incoming Webhooks" pour poster les alertes dans un channel Slack.

Les avantages:
- Communication en temps reel
- Visualisation immediate des alertes
- Integration avec les workflows Slack
- Facile a mettre en place

### Configuration Slack

Etape 1: Creer un Incoming Webhook

1. Aller a https://api.slack.com/apps
2. Creer une nouvelle application ou selectionner une existante
3. Aller a "Incoming Webhooks" dans le menu
4. Activer "Incoming Webhooks"
5. Cliquer "Add New Webhook to Workspace"
6. Selectionner le channel ou envoyer les alertes
7. Copier l'URL du webhook

L'URL ressemble a:
```
https://hooks.slack.com/services/T00000000/B00000000/XXXXXXXXXXXXXXXXXXXXXXXXXXXXX
```

Etape 2: Configurer Wazuh

Ajouter dans /var/ossec/etc/ossec.conf:

```
<ossec_config>
  <integration>
    <n>slack</n>
    <hook_url>https://hooks.slack.com/services/T00000000/B00000000/XXXXXXXXXXXXXXXXXXXXX</hook_url>
    <alert_format>json</alert_format>
    <level>7</level>
  </integration>
</ossec_config>
```

Etape 3: Redemarrer le Manager

```
sudo systemctl restart wazuh-manager
```

### Personnalisation Slack

Utiliser le parametre <options> pour personnaliser:

```
<options>{"channel":"#security-alerts","icon_emoji":":lock:","username":"Wazuh Bot"}</options>
```

Champs personnalisables (voir Slack API):
- channel: Override du channel
- icon_emoji: Emoji pour le message
- icon_url: URL de l'image
- username: Nom du bot
- attachments: Format avance des messages

---

## 7. Integration PagerDuty

### Concepts

PagerDuty est une plateforme de gestion des incidents. L'integration Wazuh envoie les alertes critiques a PagerDuty pour escalade automatique vers les equipes appropriees.

Les avantages:
- Escalade automatique d'incidents
- Assignation selon les schedules
- Notifications mobiles
- Integration avec les on-call rotations

### Configuration PagerDuty

Etape 1: Creer une Service dans PagerDuty

1. Aller a https://www.pagerduty.com/
2. Se connecter a votre compte
3. Aller a "Services"
4. Creer un nouveau service
5. Configurer les regles d'escalade et les contacts
6. Copier l'Integration Key (Events API v2)

L'Integration Key ressemble a:
```
a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6
```

Etape 2: Configurer Wazuh

Ajouter dans /var/ossec/etc/ossec.conf:

```
<ossec_config>
  <integration>
    <n>pagerduty</n>
    <api_key>a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6</api_key>
    <alert_format>json</alert_format>
    <level>10</level>
  </integration>
</ossec_config>
```

Etape 3: Redemarrer le Manager

```
sudo systemctl restart wazuh-manager
```

### Personnalisation PagerDuty

Utiliser le parametre <options> pour personnaliser:

```
<options>{"severity":"critical","service":"web-team"}</options>
```

Parametres personnalisables (voir PagerDuty API):
- severity: critical, error, warning, info
- service: Service PagerDuty cible
- client: Client qui genere l'alerte

---

## 8. Integration VirusTotal

### Concepts

VirusTotal est un service en ligne qui analyse les fichiers et URLs pour detecter des virus, chevaux de Troie et autres malwares. L'integration Wazuh utilise l'API VirusTotal pour analyser les fichiers suspects.

Les avantages:
- Analyse de fichiers contre 90+ antivirus
- Detection de URLs malveillantes
- Enrichissement des alertes de securite
- Verification automatique de fichiers

### Configuration VirusTotal

Etape 1: Obtenir la Cle API

1. Aller a https://www.virustotal.com/gui/my-apikey
2. Se connecter a votre compte (creer un si necessaire)
3. Copier votre API Key

La clé ressemble a:
```
a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6q7r8s9t0
```

Etape 2: Configurer Wazuh

Ajouter dans /var/ossec/etc/ossec.conf:

```
<ossec_config>
  <integration>
    <n>virustotal</n>
    <api_key>a1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6q7r8s9t0</api_key>
    <group>syscheck</group>
    <alert_format>json</alert_format>
  </integration>
</ossec_config>
```

Note: L'integration VirusTotal fonctionne avec le groupe "syscheck" (File Integrity Monitoring).

Etape 3: Redemarrer le Manager

```
sudo systemctl restart wazuh-manager
```

### Quand VirusTotal Intervient

Automatiquement lorsque:
- Un fichier est modifie dans un repertoire surveille (syscheck)
- Une alerte syscheck est generee
- L'alerte match les filtres configures

Le fichier est:
- Upload sur VirusTotal
- Analyse
- Rapport revient dans les alertes Wazuh

---

## 9. Integration Shuffle

### Concepts

Shuffle est une plateforme SOAR (Security Orchestration, Automation and Response) open-source. L'integration Wazuh envoie les alertes vers des workflows Shuffle pour automation des reponses de securite.

Les avantages:
- Automation des workflows de securite
- Orchestration entre outils multiples
- Playbooks de reponse automatique
- Integration plug-and-play

### Configuration Shuffle

Etape 1: Creer un Webhook dans Shuffle

1. Aller a https://shuffler.io/
2. Se connecter a votre instance Shuffle
3. Creer un nouveau workflow
4. Ajouter un trigger "Webhook"
5. Copier l'URL du webhook

L'URL ressemble a:
```
https://shuffler.io/api/v1/hooks/[workflow-id]
```

Etape 2: Configurer Wazuh

Ajouter dans /var/ossec/etc/ossec.conf:

```
<ossec_config>
  <integration>
    <n>shuffle</n>
    <hook_url>https://shuffler.io/api/v1/hooks/workflow-id-here</hook_url>
    <alert_format>json</alert_format>
  </integration>
</ossec_config>
```

Etape 3: Redemarrer le Manager

```
sudo systemctl restart wazuh-manager
```

### Cas d'usage Shuffle

Exemples de workflows:
- Creer automatiquement un ticket JIRA pour les alertes critiques
- Envoyer les details de l'alerte a une equipe Slack
- Bloquer une IP dans le firewall
- Collecter des informations additionnelles
- Lancer une investigation automatique

---

## 10. Integration Maltiverse

### Concepts

Maltiverse est une plateforme de threat intelligence. L'integration Wazuh envoie les alertes pour enrichissement et correlation avec d'autres donnees de menace.

Les avantages:
- Threat intelligence enrichie
- Correlation avec d'autres donnees
- Reputation des IPs et domaines
- Partage d'indicateurs de compromise

### Configuration Maltiverse

Etape 1: Obtenir les Credentials

1. S'inscrire a Maltiverse: https://maltiverse.com/
2. Generer une API Key
3. Recuperer l'Integration Webhook URL

Etape 2: Configurer Wazuh

Ajouter dans /var/ossec/etc/ossec.conf:

```
<ossec_config>
  <integration>
    <n>maltiverse</n>
    <api_key>your_api_key_here</api_key>
    <hook_url>https://api.maltiverse.com/integration/webhook</hook_url>
    <alert_format>json</alert_format>
  </integration>
</ossec_config>
```

Etape 3: Redemarrer le Manager

```
sudo systemctl restart wazuh-manager
```

---

## 11. Integrations Personnalisees

### Concept: Custom Integrations

Vous pouvez creer une integration avec n'importe quel service HTTP/HTTPS en utilisant le format `custom-*`.

Exemples:
- custom-jira: Creer des tickets JIRA
- custom-teams: Envoyer vers Microsoft Teams
- custom-mattermost: Integration Mattermost
- custom-discord: Integration Discord
- custom-email: Envoyer par email via API

### Configuration Custom

Structure generique:

```
<ossec_config>
  <integration>
    <n>custom-myservice</n>
    <hook_url>https://myservice.com/api/alerts</hook_url>
    <alert_format>json</alert_format>
    <api_key>optional_key_if_needed</api_key>
    <level>7</level>
  </integration>
</ossec_config>
```

### Exemple: Microsoft Teams

Configuration pour Teams:

```
<integration>
  <n>custom-teams</n>
  <hook_url>https://outlook.webhook.office.com/webhookb2/...</hook_url>
  <alert_format>json</alert_format>
</integration>
```

### Exemple: Discord

Configuration pour Discord:

```
<integration>
  <n>custom-discord</n>
  <hook_url>https://discordapp.com/api/webhooks/...</hook_url>
  <alert_format>json</alert_format>
</integration>
```

---

## 12. Validation et Tests

### Verifier la Configuration

Etape 1: Valider la syntaxe XML

```
sudo /var/ossec/bin/wazuh-control -t
```

Resultat attendu:
```
Verifying ossec.conf...
OK: ossec.conf syntax is valid
```

Etape 2: Verifier que le Manager tourne

```
sudo systemctl status wazuh-manager
```

Resultat attendu:
```
Active: active (running)
```

Etape 3: Verifier le daemon Integrator

```
sudo ps aux | grep wazuh-integratord
```

Resultat attendu: Voir le processus wazuh-integratord en cours d'execution

### Generer une Alerte de Test

Etape 1: Modifier un fichier surveille

Sur un agent Linux:
```
echo "TEST" >> /etc/hosts
```

Sur un agent Windows:
```
echo "TEST" >> C:\Windows\System32\drivers\etc\hosts
```

Etape 2: Attendre quelques secondes

Le Manager analysera le changement et generera une alerte.

Etape 3: Verifier la reception

Pour Slack: Regarder le channel specifie
Pour PagerDuty: Verifier le dashboard Incidents
Pour VirusTotal: Regarder les alertes avec les resultats
Pour Shuffle: Verifier le webhook trigger

### Consulter les Logs

Voir les logs du daemon Integrator:

```
sudo tail -100 /var/ossec/logs/ossec.log | grep integratord
```

Chercher les erreurs:

```
sudo grep -i "integrator\|error" /var/ossec/logs/ossec.log | tail -50
```

---

## 13. Troubleshooting

### Probleme: Les alertes ne sont pas envoyees

Verification 1: Le daemon Integrator tourne?
```
sudo /var/ossec/bin/wazuh-control -i
```

Doit afficher wazuh-integratord dans la liste.

Verification 2: La configuration est-elle valide?
```
sudo /var/ossec/bin/wazuh-control -t
```

Verification 3: Les filtres sont-ils corrects?
- Verifier <level>, <group>, <rule_id>
- Verifier que l'alerte generee correspond aux filtres

Verification 4: La connexion a l'API externe fonctionne?
```
curl -X POST https://hooks.slack.com/services/... -d '{"text":"test"}'
```

### Probleme: Authentification echouee

Cause: Cles API ou webhooks invalides

Solution:
- Verifier que la cle API est correcte
- Verifier que le webhook URL est correct
- Verifier les permissions dans le service externe
- Verifier la validite des credentials

### Probleme: Alerte envoyee mais sans donnees

Cause: <alert_format>json</alert_format> manquant

Solution: Ajouter le parametre:
```
<alert_format>json</alert_format>
```

### Probleme: Trop d'alertes envoyees

Cause: Pas assez de filtres

Solution: Ajouter des filtres:
```
<level>7</level>
<group>syscheck,fim</group>
```

---

## 14. Bonnes Pratiques

### Securite des Credentials

Ne JAMAIS:
- Partager les API keys publiquement
- Les stocker dans du version control
- Les envoyer par email
- Les ecrire en clair dans les logs

Faire:
- Utiliser des variables d'environnement (non suporte actuellement)
- Restreindre l'acces a ossec.conf
- Rotater les clés regulierement
- Verifier les permissions du fichier

Verifier les permissions:
```
sudo ls -la /var/ossec/etc/ossec.conf
```

Doit afficher:
```
-rw-r----- 1 root ossec
```

### Filtrage des Alertes

Recommandations:
- Ne pas envoyer TOUTES les alertes
- Definir un <level> minimum adapte
- Utiliser des <group> specifiques
- Tester les regexes pour <event_location>

Niveau recommande par service:
- Slack: level >= 5 (notification generale)
- PagerDuty: level >= 8 (vraiment critique)
- VirusTotal: group = syscheck (fichiers)
- Shuffle: level >= 7 (incidents graves)

### Monitoring des Integrations

Verifier regulierement:
- Les logs d'erreur dans ossec.log
- La connectivite a l'API externe
- Le nombre d'alertes envoyees
- Les reponses du service externe

Script de monitoring:
```
#!/bin/bash
DATE=$(date +%Y%m%d)
ERRORS=$(sudo grep -c "integration.*error\|integration.*fail" /var/ossec/logs/ossec.log)
if [ $ERRORS -gt 10 ]; then
  echo "WARNING: $ERRORS errors in integrations"
  mail -s "Wazuh Integration Errors" admin@company.fr
fi
```

### Maintenance des Configurations

Bonnes pratiques:
- Documenter chaque integration
- Tester apres chaque modification
- Garder un backup des configs
- Versioner les changements
- Etablir un calendrier de revue

Sauvegarde:
```
sudo cp /var/ossec/etc/ossec.conf /var/ossec/etc/ossec.conf.backup.$(date +%Y%m%d)
```

---

## Resume

External API Integration permet:
- Communication en temps reel (Slack)
- Gestion d'incidents (PagerDuty)
- Analyse de menaces (VirusTotal, Maltiverse)
- Automation de reponses (Shuffle)
- Integration avec outils custom

Configuration simple: Ajouter bloc <integration> dans ossec.conf
Filtrage: level, group, rule_id, event_location
Authentification: hook_url ou api_key selon le service

Tester apres chaque modification et monitorer les logs.