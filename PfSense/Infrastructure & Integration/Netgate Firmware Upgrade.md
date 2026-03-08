# Netgate Firmware Upgrade

## Introduction

La mise à jour du firmware **Netgate / pfSense** est une étape importante pour maintenir la sécurité, la stabilité et les performances de l’infrastructure réseau. Les mises à jour permettent de corriger des vulnérabilités, d’améliorer les fonctionnalités existantes et d’ajouter de nouveaux outils de gestion réseau.

## Procédure de mise à jour

La mise à jour du firmware se réalise depuis l’interface d’administration pfSense.

Étapes générales :

1. Accéder à l’interface Web pfSense.
2. Aller dans **System → Update**.
3. Vérifier la version disponible.
4. Lancer la mise à jour.
5. Redémarrer le système si nécessaire.

Il est recommandé d’effectuer une **sauvegarde de la configuration** avant toute mise à jour afin d’éviter toute perte de configuration.

## Bonnes pratiques

- Planifier la mise à jour pendant une période de faible trafic
- Sauvegarder la configuration du firewall
- Vérifier la compatibilité des packages installés
- Tester les services réseau après la mise à jour

La mise à jour régulière du firmware garantit une infrastructure réseau plus sécurisée et plus fiable.