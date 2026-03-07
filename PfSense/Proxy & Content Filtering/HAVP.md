# HAVP – HTTP Antivirus Proxy 

## 1. Présentation

**HAVP (HTTP Antivirus Proxy)** permet d’ajouter une **analyse antivirus du trafic HTTP**.

HAVP agit comme un **proxy intermédiaire** capable d’intercepter les fichiers téléchargés depuis Internet et de les analyser avec un moteur antivirus avant de les transmettre aux utilisateurs.

Objectifs principaux :

- détecter les **malwares**
- empêcher le téléchargement de **fichiers infectés**
- protéger les postes clients du réseau

---

# 2. Principe de fonctionnement

Le fonctionnement de HAVP repose sur l’analyse du trafic HTTP entrant.

Processus :

1. Un utilisateur télécharge un fichier depuis Internet
2. La requête passe par le proxy HAVP
3. Le fichier est temporairement intercepté
4. Le moteur antivirus analyse le contenu
5. Si le fichier est propre → téléchargement autorisé  
6. Si le fichier est infecté → téléchargement bloqué

Ce mécanisme permet de bloquer les menaces **avant qu’elles n’atteignent les machines internes**.

---

# 3. Moteur antivirus

HAVP utilise généralement le moteur antivirus **ClamAV** pour analyser les fichiers.

Fonctionnalités :

- analyse des fichiers téléchargés
- détection des virus connus
- mise à jour régulière des signatures
- détection des fichiers malveillants dans les archives

Les bases de signatures sont mises à jour automatiquement afin de garantir une protection efficace.

---

# 4. Intégration avec pfSense

Dans pfSense, HAVP peut être utilisé avec :

- **Squid Proxy**
- des règles de **pare-feu**
- des politiques de sécurité réseau

Cette intégration permet :

- l’inspection du trafic HTTP
- la détection des téléchargements suspects
- la protection centralisée du réseau.

---

# 5. Avantages

L’utilisation de HAVP apporte plusieurs bénéfices :

- protection contre les **malwares téléchargés**
- sécurité supplémentaire pour les utilisateurs
- inspection centralisée du trafic HTTP
- réduction du risque d’infection des postes clients

---

# 6. Limites

Certaines limites doivent être prises en compte :

- analyse uniquement du **trafic HTTP**
- impossibilité d’analyser le **HTTPS sans déchiffrement**
- consommation de ressources selon le volume de trafic

---

Utilisé en complément d’autres outils de sécurité tels que les pare-feu, les systèmes IDS/IPS ou les filtres web, HAVP participe à la mise en place d’une architecture de sécurité réseau plus robuste et mieux adaptée aux menaces.



