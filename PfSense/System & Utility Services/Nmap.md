# Nmap sur pfSense

Le package **Nmap** (Network Mapper) pour pfSense intègre l'un des outils de diagnostic réseau les plus célèbres au monde directement dans votre pare-feu. Il permet d'effectuer des audits de sécurité et de découvrir les équipements actifs sur vos réseaux locaux ou distants.

---

##  Présentation du Package

Une fois installé, Nmap sur pfSense remplit trois rôles principaux :
1.  **Scanner de Ports** : Identifie les services ouverts sur une machine.
2.  **Identification d'OS** : Tente de deviner le système d'exploitation des cibles.
3.  **Base de données OUI** : Améliore l'affichage des adresses MAC dans pfSense (ARP Table, DHCP Leases) en affichant le nom du constructeur.

---

##  Utilisation dans l'interface Web

Après l'installation via le *Package Manager*, l'outil est accessible via :  
**Diagnostics > nmap**

### Configuration d'un scan :

* **Host(s)** : L'adresse IP, le nom d'hôte ou le sous-réseau (ex: `192.168.1.0/24`).
* **Scan Method** : 
    * `SYN Scan` (Défaut) : Rapide et discret (ne complète pas la connexion TCP).
    * `Connect Scan` : Utilise l'appel système standard (plus lent, plus visible).
    * `UDP Scan` : Pour tester les services comme DNS ou DHCP.
* **Port Range** : Les ports à scanner (ex: `80,443,22` ou `1-1024`).
* **Service & OS Detection** : Active les options `-sV` (version) et `-O` (OS).

---

##  Utilisation via le Shell (CLI)

L'un des grands avantages de ce package est qu'il rend la commande `nmap` disponible en ligne de commande via SSH ou la console.

### Exemples de commandes utiles :

| Commande | Action |
| :--- | :--- |
| `nmap -sn 192.168.1.0/24` | **Ping Sweep** : Liste uniquement les machines allumées. |
| `nmap -A 192.168.1.50` | **Scan Agressif** : Détecte OS, versions et lance des scripts. |
| `nmap -p- 192.168.1.1` | **Full Scan** : Scanne les 65535 ports TCP. |
| `nmap -sU -p 53,67 192.168.1.1` | **UDP Scan** : Vérifie les ports DNS et DHCP. |

---

##  Interprétation des résultats

Nmap classe les ports dans différents états :

1.  **Open** : Une application accepte les connexions sur ce port.
2.  **Closed** : Le port est accessible mais aucune application n'écoute.
3.  **Filtered** : Un pare-feu bloque le scan (impossible de savoir si le port est ouvert).
4.  **Open|Filtered** : Nmap ne peut pas déterminer si le port est ouvert ou filtré (souvent en UDP).

---

