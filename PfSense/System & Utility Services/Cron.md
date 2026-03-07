# Package Cron pour pfSense

Le package **Cron** est l'interface de gestion de l'ordonnanceur de tâches standard UNIX (`crond`) intégrée au WebGUI de pfSense. Bien que pfSense utilise nativement Cron pour ses opérations internes, ce package expose cette couche système pour permettre une personnalisation avancée.

---

##  Pertinence et Utilité Stratégique

L'ajout de ce package répond à des besoins que l'interface standard de pfSense ne couvre pas nativement :

* **Maîtrise du Cycle de Vie des Services** : Utile pour forcer le redémarrage périodique de services instables en mémoire sans intervention humaine.
* **Maintenance Prédictive** : Permet de planifier des purges de répertoires temporaires (`/tmp`, `/var/tmp`) ou la rotation de logs personnalisés avant saturation du disque.
* **Synchronisation de Données** : Indispensable pour l'exécution de scripts de sauvegarde externes (Rsync, SCP) ou la mise à jour de listes de blocages (IP/DNS) provenant de sources tierces non supportées par les packages officiels.
* **Économie d'Énergie / Sécurité** : Capacité à activer ou désactiver des interfaces ou des règles de pare-feu à des heures précises (ex: couper le Wi-Fi des invités la nuit via un script shell).

---

##  Mécanique de Fonctionnement

Le fonctionnement du package repose sur l'interaction entre l'interface PHP de pfSense et le fichier système `/etc/crontab`.

### 1. La Structure de l'Automate
Chaque instruction enregistrée via le package suit la nomenclature standard de l'ordonnanceur, décomposée en six segments temporels :



* **Minute / Hour** : Précision du déclenchement.
* **mday / month / wday** : Calendrier d'exécution (jour du mois, mois, jour de la semaine).
* **Who** : Définition du contexte de privilèges (généralement `root` pour les actions système).
* **Command** : L'appel binaire ou le script à exécuter.

### 2. Persistance et Priorité
Contrairement à une édition manuelle du fichier `/etc/crontab` via SSH — qui risque d'être écrasée lors d'une mise à jour du système ou d'une modification de la configuration XML — le package Cron injecte les données directement dans le fichier de configuration `config.xml` de pfSense. 
Cela garantit que les tâches :
* Survivent aux mises à jour du firmware.
* Sont incluses dans les sauvegardes de configuration standard.

### 3. Environnement d'Exécution
Il est crucial de noter que Cron s'exécute dans un environnement "restreint". Contrairement à une session shell interactive, les variables d'environnement (comme `$PATH`) sont minimales. Par conséquent, le fonctionnement repose sur l'utilisation de **chemins absolus** (ex: `/usr/local/bin/php` au lieu de `php`) pour garantir que l'exécutable est localisé par le démon.

---

## 4. Étude de cas : Automatisation d'une sauvegarde locale avec alerte mail

L'implémentation d'une tâche planifiée permet de sécuriser la configuration du pare-feu de manière autonome. Ce scénario repose sur un script shell exécuté quotidiennement à 2h00 du matin, incluant une vérification d'erreur.

### Script de sauvegarde :

```bash 
#!/bin/sh


BACKUP_DIR="/cf/conf/backup"
TIMESTAMP=$(date +"%Y%m%d_%H%M%S")

/usr/local/bin/pfSense-backup.sh -d "$BACKUP_DIR" -f "config_$TIMESTAMP.xml"

if [ $? -ne 0 ]; then
  echo "La sauvegarde a échoué !" | mail -s "Alerte : Échec de sauvegarde pfSense" admin@domain.com
fi

```


