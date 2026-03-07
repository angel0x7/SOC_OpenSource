# SquidGuard – Filtrage web

## 1. Présentation

**SquidGuard** est un package utilisé avec le proxy **Squid** dans pfSense afin de fournir un système de **filtrage d’URL et de contrôle d’accès web**.  
Il permet de contrôler et restreindre l’accès aux sites Internet selon des règles définies par l’administrateur réseau.

Les principales fonctionnalités incluent :

- filtrage des sites web par **catégories**
- blocage d’**URL spécifiques**
- contrôle d’accès par **utilisateur ou groupe**
- redirection vers une **page de blocage personnalisée**
- intégration avec des **bases de données de filtrage**

---

# 2. Fonctionnement

SquidGuard fonctionne comme un **redirecteur pour le proxy Squid**.

Processus :

1. Un utilisateur effectue une requête HTTP via le proxy Squid
2. Squid transmet l’URL demandée à SquidGuard
3. SquidGuard compare l’URL avec les **listes de filtrage**
4. Selon la règle configurée :
   - l’accès est **autorisé**
   - l’accès est **bloqué**
   - l’utilisateur est **redirigé**

---

# 3. Listes de filtrage (Blacklist)

SquidGuard utilise des **blacklists** contenant des bases de données d’URL classées par catégories.

Exemples de catégories :

- Adult
- Gambling
- Social Networks
- Malware
- Ads
- Phishing

Ces listes peuvent être :

- téléchargées automatiquement
- mises à jour régulièrement
- activées ou désactivées selon la politique de sécurité.

---

# 4. Politiques de filtrage

SquidGuard permet de définir différentes politiques selon :

- les **utilisateurs**
- les **groupes**
- les **adresses IP**
- les **plages réseau**

Exemples de politiques :

- bloquer les réseaux sociaux pendant les heures de travail
- autoriser certains sites pour des groupes spécifiques
- bloquer les catégories dangereuses (malware, phishing)


---

# 5. Avantages

L’utilisation de SquidGuard dans pfSense offre plusieurs avantages :

- amélioration du **contrôle d’accès Internet**
- protection contre les **sites malveillants**
- gestion centralisée des politiques web
- réduction des risques liés à l’utilisation d’Internet

---

SquidGuard constitue une solution pour mettre en place un **système de filtrage web** dans pfSense.  
Associé au proxy Squid, il permet d’appliquer des politiques de sécurité et de contrôler l’utilisation d’Internet au sein d’un réseau.

