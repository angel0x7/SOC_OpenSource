#  pfSense Firewall Rules — SOC Open Source

> **Firewall :** pfSense Community Edition + Suricata IDS/IPS  
> **Politique :** `DENY ALL` par défaut — Whitelist explicite uniquement  
> **Version :** 1.0 — Mars 2026

---

##  Architecture réseau

```
                        INTERNET
                            │
                       ┌────┴────┐
                       │  pfSense │  + Suricata IDS/IPS
                       │  + pfBlockerNG              │
                       └────┬────┘
              ┌─────────────┼──────────────┬──────────────┐
              │             │              │              │
         ┌────┴────┐   ┌────┴────┐   ┌────┴────┐   ┌────┴────┐
         │   WAN   │   │   LAN   │   │   DMZ   │   │ INTNET  │
         │ em0     │   │ em1     │   │ em2     │   │ em3     │
         │ DHCP    │   │100.1/24 │   │  2.1/24 │   │  3.1/24 │
         └─────────┘   └────┬────┘   └────┬────┘   └────┬────┘
                            │             │              │
                       VM Debian     VM Wazuh      VM Windows 10 RH
                       (Admin/SOC)   (SIEM)        + VM Kali Linux
```

| Interface | Alias   | Réseau              | VM / Rôle SOC                        |
|-----------|---------|---------------------|--------------------------------------|
| `em0`     | WAN     | `192.168.1.24/24`   | Accès Internet + OpenVPN             |
| `em1`     | LAN     | `192.168.100.0/24`  | VM Debian — Admin / Analyste SOC     |
| `em2`     | DMZ     | `192.168.2.0/24`    | VM Wazuh — SIEM central              |
| `em3`     | INTNET  | `192.168.3.0/24`    | VM Windows 10 RH (cible) + VM Kali (Red Team) |

>  **Prérequis critique :** Kali Linux et Windows 10 doivent avoir des **IPs statiques** (ou réservations DHCP par MAC) avant de déployer ces règles.

---

##  Principe de filtrage — Zero Trust

```
Tout trafic est BLOQUÉ par défaut.
Seuls les flux strictement nécessaires au SOC sont autorisés via règles explicites.
```

| Principe              | Application                                      | Bénéfice SOC                    |
|-----------------------|--------------------------------------------------|---------------------------------|
| Deny All par défaut   | Chaque interface termine par une règle `BLOCK *` | Limite la surface d'attaque     |
| Whitelist explicite   | Seuls les ports/IPs nécessaires sont ouverts     | Contrôle précis des flux        |
| Isolation des segments| Kali ne peut pas atteindre LAN ni DMZ            | Confinement Red Team            |
| Flux SOC obligatoires | Agents Wazuh INTNET → DMZ autorisés              | Visibilité SIEM complète        |

---

## WAN — `em0` | `192.168.1.24/24`

> Frontière Internet. Politique : **bloquer tout sauf OpenVPN.**

| # | Action    | Proto    | Source           | Port src | Destination   | Port dst       | Description                          |
|---|-----------|----------|------------------|----------|---------------|----------------|--------------------------------------|
| 1 | ❌ BLOCK  | `*`      | RFC 1918         | `*`      | `*`           | `*`            | Bloquer réseaux privés entrants      |
| 2 | ❌ BLOCK  | `*`      | Bogons           | `*`      | `*`           | `*`            | Bloquer IPs non assignées IANA       |
| 3 | ❌ BLOCK  | `IPv4 *` | `pfB_PRI1_v4`    | `*`      | `*`           | `*`            | pfBlockerNG — IPs malveillantes      |
| 4 | ❌ BLOCK  | `IPv4 *` | `pfB_TOR_v4`     | `*`      | `*`           | `*`            | pfBlockerNG — nœuds TOR              |
| 5 | ✅ PASS   | `UDP`    | `*`              | `*`      | WAN address   | `1194`         | OpenVPN — accès admin distant        |
| 6 | ❌ BLOCK  | `*`      | `*`              | `*`      | `*`           | `*`            | **DEFAULT DENY WAN**                 |

### Justifications WAN

- **Règles #1 et #2** — Standard pfSense. Empêche l'usurpation d'adresses privées ou réservées depuis Internet. À toujours activer en premier.
- **Règles #3 et #4** — pfBlockerNG bloque automatiquement les IPs à réputation malveillante et les sorties TOR. Réduit le bruit et bloque les C2 connus, essentiel pour un SOC.
- **Règle #5** — L'admin accède au pfSense et à Wazuh depuis l'extérieur via VPN uniquement. Aucun accès direct SSH/HTTPS depuis le WAN.
- **Règle #6** — Tout ce qui n'est pas explicitement autorisé est rejeté.

>  **À supprimer :** La règle existante `IPv4 TCP / WAN address → LAN address` est dangereuse — elle expose potentiellement le LAN depuis Internet.

---

##  LAN — `em1` | `192.168.100.0/24`

> Segment de l'admin/analyste SOC (VM Debian). Accès aux dashboards Wazuh et à l'interface pfSense. **Isolé d'INTNET.**

| # | Action    | Proto      | Source      | Port src | Destination          | Port dst                    | Description                                   |
|---|-----------|------------|-------------|----------|----------------------|-----------------------------|-----------------------------------------------|
| 1 | ✅ PASS   | `*`        | LAN address | `*`      | LAN address          | `8000, 4444`                | Anti-Lockout Rule pfSense (ne pas modifier)   |
| 2 | ❌ BLOCK  | `IPv4 *`   | `*`         | `*`      | `pfB_PRI1_v4`        | `*`                         | pfBlockerNG — destinations malveillantes      |
| 3 | ❌ BLOCK  | `IPv4 *`   | `*`         | `*`      | `pfB_TOR_v4`         | `*`                         | pfBlockerNG — destinations TOR               |
| 4 | ✅ PASS   | `TCP`      | LAN net     | `*`      | `192.168.2.0/24`     | `22`                        | SSH admin vers VM Wazuh (DMZ)                 |
| 5 | ✅ PASS   | `TCP`      | LAN net     | `*`      | `192.168.2.0/24`     | `443, 5601, 9200, 55000`    | Dashboards Wazuh (HTTPS, Kibana, Elastic, API)|
| 6 | ✅ PASS   | `TCP`      | LAN net     | `*`      | LAN address          | `80, 443`                   | Interface web pfSense                         |
| 7 | ✅ PASS   | `UDP`      | LAN net     | `*`      | LAN address          | `53`                        | Résolution DNS via pfSense                    |
| 8 | ✅ PASS   | `TCP`      | LAN net     | `*`      | `*`                  | `80, 443`                   | Mises à jour OS Debian + Threat Intel         |
| 9 | ❌ BLOCK  | `*`        | LAN net     | `*`      | `192.168.3.0/24`     | `*`                         | **ISOLER LAN de INTNET**                      |
|10 | ❌ BLOCK  | `*`        | LAN net     | `*`      | `*`                  | `*`                         | **DEFAULT DENY LAN**                          |

### Justifications LAN

- **Règle #4** — L'admin Debian administre la VM Wazuh via SSH. Port 22 uniquement.
- **Règle #5** — Accès aux interfaces de monitoring : Kibana (`5601`), Elasticsearch (`9200`), API Wazuh REST (`55000`), HTTPS (`443`). Flux strictement LAN → DMZ.
- **Règle #6** — Administration de l'interface web pfSense depuis le LAN uniquement.
- **Règle #8** — Mises à jour de l'OS Debian et consultation des flux de Threat Intelligence. Limité à 80/443.
- **Règle #9** — L'analyste SOC ne doit **jamais** initier de connexion vers le segment Red Team. Si le poste admin est compromis, il ne peut pas pivoter vers Kali ou Windows.

>  **À supprimer :** La règle existante `LAN subnets → * / *` est beaucoup trop permissive. La remplacer intégralement par les règles ci-dessus.

---

##  DMZ — `em2` | `192.168.2.0/24`

> Segment hébergeant uniquement la VM Wazuh (SIEM). Reçoit les logs des agents, est administrable depuis le LAN, peut contacter Internet pour ses mises à jour. **N'initie jamais de connexion vers LAN ou INTNET.**

| # | Action    | Proto      | Source               | Port src | Destination            | Port dst   | Description                                      |
|---|-----------|------------|----------------------|----------|------------------------|------------|--------------------------------------------------|
| 1 | ❌ BLOCK  | `IPv4 *`   | `*`                  | `*`      | `pfB_PRI1_v4`          | `*`        | pfBlockerNG — destinations malveillantes         |
| 2 | ❌ BLOCK  | `IPv4 *`   | `*`                  | `*`      | `pfB_TOR_v4`           | `*`        | pfBlockerNG — destinations TOR                  |
| 3 | ✅ PASS   | `TCP/UDP`  | `192.168.3.0/24`     | `*`      | `192.168.2.x` (Wazuh)  | `1514`     | Agents Wazuh — réception logs (INTNET → DMZ)    |
| 4 | ✅ PASS   | `TCP`      | `192.168.3.0/24`     | `*`      | `192.168.2.x` (Wazuh)  | `1515`     | Agents Wazuh — enregistrement agent             |
| 5 | ✅ PASS   | `UDP`      | DMZ net              | `*`      | DMZ address            | `53`       | DNS interne via pfSense                          |
| 6 | ✅ PASS   | `TCP`      | DMZ net              | `*`      | `*`                    | `443`      | HTTPS — updates OS + règles Suricata + feeds TI  |
| 7 | ✅ PASS   | `TCP`      | DMZ net              | `*`      | `*`                    | `80`       | HTTP — flux Threat Intelligence                  |
| 8 | ❌ BLOCK  | `*`        | DMZ net              | `*`      | `192.168.100.0/24`     | `*`        | **BLOQUER DMZ → LAN**                            |
| 9 | ❌ BLOCK  | `*`        | DMZ net              | `*`      | `192.168.3.0/24`       | `*`        | **BLOQUER DMZ → INTNET**                         |
|10 | ❌ BLOCK  | `*`        | DMZ net              | `*`      | `*`                    | `*`        | **DEFAULT DENY DMZ**                             |

### Justifications DMZ

- **Règles #3 et #4** — Ports essentiels Wazuh. `1514` = collecte des logs agents (syslog/JSON). `1515` = enregistrement initial des agents. Sans ces règles, **aucun log Windows RH ne remonte dans le SIEM**.
- **Règles #6 et #7** — Wazuh a besoin d'Internet pour : mises à jour de signatures Suricata, règles de détection, flux OSINT. Limité à 80/443 uniquement.
- **Règle #8** — Le SIEM ne doit jamais initier de connexion vers le poste admin. Principe de moindre privilège : Wazuh écoute et répond, il n'initie pas.
- **Règle #9** — Wazuh ne doit jamais envoyer de trafic vers le segment Red Team. Les logs arrivent en **push** depuis les agents, pas en pull depuis le SIEM.

>  **À remplacer :** La règle existante `DMZ subnets → WAN *` (all ports) est trop large. Si Wazuh est compromis, il peut exfiltrer des données. La remplacer par les règles #6 et #7 (80/443 uniquement).

---

##  INTNET — `em3` | `192.168.3.0/24`

> Segment le plus sensible. Contient la cible Red Team (Windows 10 RH) et l'attaquant (Kali Linux). Kali attaque Windows librement sur ce segment mais est totalement isolé du reste de l'infrastructure.

| # | Action    | Proto      | Source               | Port src | Destination            | Port dst   | Description                                          |
|---|-----------|------------|----------------------|----------|------------------------|------------|------------------------------------------------------|
| 1 | ❌ BLOCK  | `IPv4 *`   | `*`                  | `*`      | `pfB_PRI1_v4`          | `*`        | pfBlockerNG — destinations malveillantes             |
| 2 | ❌ BLOCK  | `IPv4 *`   | `*`                  | `*`      | `pfB_TOR_v4`           | `*`        | pfBlockerNG — destinations TOR                      |
| 3 | ✅ PASS   | `TCP/UDP`  | `Windows_IP`         | `*`      | `192.168.2.x` (Wazuh)  | `1514`     | Agent Wazuh Windows → SIEM (remontée des logs)      |
| 4 | ✅ PASS   | `TCP`      | `Windows_IP`         | `*`      | `192.168.2.x` (Wazuh)  | `1515`     | Agent Wazuh Windows → SIEM (enregistrement)         |
| 5 | ✅ PASS   | `UDP`      | `Windows_IP`         | `*`      | `192.168.3.1`          | `53`       | DNS Windows via pfSense                              |
| 6 | ✅ PASS   | `TCP`      | `Windows_IP`         | `*`      | `*`                    | `80, 443`  | Windows Update + services Microsoft uniquement       |
| 7 | ✅ PASS   | `*`        | `Kali_IP`            | `*`      | `Windows_IP`           | `*`        | 🔴 Red Team — Kali attaque Windows (but du lab)     |
| 8 | ❌ BLOCK  | `*`        | `Kali_IP`            | `*`      | `192.168.2.0/24`       | `*`        | **ISOLER Kali de la DMZ (Wazuh protégé)**           |
| 9 | ❌ BLOCK  | `*`        | `Kali_IP`            | `*`      | `192.168.100.0/24`     | `*`        | **ISOLER Kali du LAN (admin protégé)**              |
|10 | ❌ BLOCK  | `*`        | `Kali_IP`            | `*`      | `*`                    | `80, 443`  | **BLOQUER Internet depuis Kali**                    |
|11 | ❌ BLOCK  | `*`        | `Kali_IP`            | `*`      | `*`                    | `*`        | DEFAULT DENY Kali — tout le reste bloqué             |
|12 | ❌ BLOCK  | `*`        | `192.168.3.0/24`     | `*`      | `192.168.100.0/24`     | `*`        | **BLOQUER INTNET → LAN**                            |
|13 | ❌ BLOCK  | `*`        | `192.168.3.0/24`     | `*`      | `*`                    | `*`        | **DEFAULT DENY INTNET**                             |

### Justifications INTNET

- **Règles #3 et #4** — Flux SOC le plus critique. L'agent Wazuh sur Windows 10 RH remonte ses logs vers le SIEM. Sans ces règles, **les attaques de Kali ne sont pas visibles dans Wazuh**.
- **Règle #6** — Windows 10 a besoin de Windows Update. Limité à la seule IP Windows (pas à Kali).
- **Règle #7** — Cœur pédagogique du lab : Kali peut scanner, exploiter et post-exploiter Windows librement sur ce segment.
- **Règle #8** — Dans un scénario réaliste, un attaquant tenterait de désactiver le SIEM en premier. Ce blocage simule la protection de l'infrastructure SOC contre les mouvements latéraux.
- **Règle #9** — Si Kali compromet Windows et tente un pivot, il ne peut pas atteindre le poste d'administration.
- **Règle #10** — Kali ne doit pas sortir sur Internet : évite les téléchargements d'outils offensifs, la communication avec des C2 réels, ou l'exfiltration accidentelle.

>  **À remplacer :** La règle existante `INTNET subnets → WAN *` autorise Kali à sortir librement sur Internet. Remplacer par les règles granulaires ci-dessus.

---

##  Référence des ports Wazuh

| Port    | Proto     | Service             | Flux                        | Description                                      |
|---------|-----------|---------------------|-----------------------------|--------------------------------------------------|
| `1514`  | `TCP/UDP` | Wazuh Agent         | INTNET → DMZ                | Collecte des logs depuis les agents              |
| `1515`  | `TCP`     | Wazuh Agent         | INTNET → DMZ                | Enregistrement initial de l'agent                |
| `55000` | `TCP`     | Wazuh API REST      | LAN → DMZ                   | API d'administration Wazuh                       |
| `9200`  | `TCP`     | Elasticsearch       | LAN → DMZ                   | Base de données des logs                         |
| `5601`  | `TCP`     | Kibana / Dashboard  | LAN → DMZ                   | Interface web de visualisation                   |
| `443`   | `TCP`     | HTTPS Wazuh UI      | LAN → DMZ / DMZ → WAN       | Interface unifiée (versions récentes) + updates  |

---

##  Synthèse des flux autorisés

```
                        INTERNET (WAN)
                             ▲
              ┌──────────────┼──────────────┐
              │ DMZ :443/80  │              │ Windows :443/80
              │ (updates/TI) │              │ (Windows Update)
              │              │              │
         ┌────┴──────┐  ┌────┴──────┐  ┌───┴──────────┐
         │    LAN    │  │    DMZ    │  │    INTNET     │
         │  Debian   │  │  Wazuh    │  │  Win10 + Kali │
         └────┬──────┘  └─────▲─────┘  └──────────────┘
              │               │                 │
              │ SSH :22        │ :1514/:1515     │ Kali → Windows
              │ :443/5601      │ (agents logs)   │ (Red Team ✔)
              │ /9200/55000    │                 │
              └───────────────┘                 │
              LAN → DMZ (admin)     Kali → DMZ ✘ BLOQUÉ
                                    Kali → LAN ✘ BLOQUÉ
                                    Kali → WAN ✘ BLOQUÉ
```

| Source            | Destination       | Ports          | Statut      |
|-------------------|-------------------|----------------|-------------|
| LAN (Debian)      | DMZ (Wazuh)       | 22, 443, 5601, 9200, 55000 | ✅ Autorisé |
| LAN (Debian)      | pfSense           | 80, 443        | ✅ Autorisé |
| LAN (Debian)      | Internet          | 80, 443        | ✅ Autorisé |
| DMZ (Wazuh)       | Internet          | 80, 443        | ✅ Autorisé |
| INTNET (Windows)  | DMZ (Wazuh)       | 1514, 1515     | ✅ Autorisé |
| INTNET (Windows)  | Internet          | 80, 443        | ✅ Autorisé |
| INTNET (Kali)     | INTNET (Windows)  | `*` (ALL)      | ✅ Autorisé |
| INTNET (Kali)     | DMZ / LAN / WAN   | `*`            | ❌ BLOQUÉ  |
| WAN (OpenVPN)     | LAN (via tunnel)  | 1194/UDP       | ✅ Autorisé |

---

##  Actions prioritaires avant déploiement

```
CRITIQUE — À faire avant d'appliquer ces règles
```

- [ ] Assigner une **IP statique** à la VM Kali Linux (ex: `192.168.3.10`)
- [ ] Assigner une **IP statique** à la VM Windows 10 RH (ex: `192.168.3.20`)
- [ ] **Supprimer** la règle WAN `TCP / WAN address → LAN address`
- [ ] **Supprimer** la règle LAN `LAN subnets → * / *` (trop permissive)
- [ ] **Remplacer** la règle INTNET `INTNET subnets → WAN *` par les règles granulaires
- [ ] **Remplacer** la règle DMZ `DMZ subnets → WAN *` par HTTPS/HTTP uniquement
- [ ] Remplacer `Windows_IP` et `Kali_IP` par les vraies IPs statiques dans toutes les règles INTNET

---
