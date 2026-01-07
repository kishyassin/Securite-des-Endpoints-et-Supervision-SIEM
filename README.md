# Sécurité des Endpoints et Supervision SIEM - Projet Atelier

## Description
Ce projet illustre l'implémentation d'une plateforme de supervision et de protection de la sécurité à travers l’utilisation de Wazuh, combinant les approches SIEM (Security Information and Event Management) et EDR (Endpoint Detection and Response). Il est déployé sur un environnement AWS avec une architecture multi-OS (Linux & Windows).

## Composants du Projet
- **Wazuh Server (Ubuntu)** : Collecte, analyse et visualisation des événements de sécurité.
- **Linux Client (Ubuntu)** : Agent Wazuh pour supervision des événements de sécurité.
- **Windows Client (Windows Server)** : Agent Wazuh pour supervision des événements de sécurité.

### VPC Configuration
- Trois instances EC2 dans le même VPC : Wazuh Server, Client Linux, et Client Windows.
- Security Groups configurés pour permettre les communications entre les instances.

### Déploiement
1. **Création des instances EC2** (Wazuh Server, Linux Client, Windows Client).
2. **Installation de Wazuh Server** sur Ubuntu.
3. **Enrôlement des agents** pour Linux et Windows via le Wazuh Dashboard.

### Étapes de Sécurisation et Détection
1. Démonstration de détection des événements de sécurité sur Linux (SSH Failures, Privilege Escalation).
2. Démonstration sur Windows (Failed Logins, User Creation).
3. Configuration des alertes dans le Dashboard Wazuh.

