# 🔍 TP Zabbix — Supervision et Sécurité Réseau LAN
**Module S4.01 — Sécurité des Réseaux | HAMEG Sami**

---

## Table des matières

1. [Introduction](#introduction)
2. [Contexte](#contexte)
3. [Objectifs pédagogiques](#objectifs-pédagogiques)
4. [Prérequis](#prérequis)
5. [Environnement de test](#environnement-de-test)
6. [Préparation des VMs](#préparation-des-vms)
7. [Installation du serveur Zabbix](#installation-du-serveur-zabbix)
8. [Configuration des agents Zabbix](#configuration-des-agents-zabbix)
9. [Ajout des hôtes supervisés](#ajout-des-hôtes-supervisés)
10. [Mise en place d'une alerte de sécurité](#mise-en-place-dune-alerte-de-sécurité)
11. [Résultats](#résultats)
12. [Questions & Réponses](#questions--réponses)
13. [Conclusion](#conclusion)

---

## Introduction

Dans le paysage numérique actuel, la sécurité des réseaux locaux (LAN) est devenue une priorité vitale. Ce projet repose sur la mise en œuvre d'une **surveillance proactive** à l'aide de **Zabbix**, une plateforme de supervision open source reconnue pour sa flexibilité et sa puissance.

> Zabbix permet de surveiller l'état et les performances des serveurs, applications et services réseau en centralisant la gestion de l'ensemble de l'architecture informatique sur un **point de contrôle unique**.

---

## Contexte

Face à la sophistication croissante des cyberattaques, une approche proactive de supervision est indispensable. L'absence d'une telle solution expose le Système d'Information (SI) à des risques majeurs :

* Vulnérabilités et intrusions passant inaperçues pendant de longues périodes
* Allongement du délai de réaction moyen aux incidents (MTRS)
* Diagnostic des problèmes complexifié par l'absence de données historiques
* Non-respect potentiel des obligations réglementaires

---

## Objectifs pédagogiques

* Installer et configurer un serveur Zabbix et ses agents
* Superviser efficacement des serveurs et services réseau
* Mettre en place des alertes de sécurité en temps réel
* Visualiser les données via l'interface web (dashboards, graphiques)
* Comprendre l'importance de la supervision dans la sécurité réseau

**Durée totale :** 3 heures

| Étape | Description | Durée estimée |
|-------|-------------|---------------|
| 1 | Installation et configuration du serveur Zabbix | 90 min |
| 2 | Configuration des agents Zabbix sur les clients | 60 min |
| 3 | Ajout des hôtes à superviser | 45 min |
| 4 | Mise en place d'une alerte de sécurité simple | 15 min |

---

## Prérequis

* Notions de base en réseaux **TCP/IP**
* Connaissances fondamentales sur l'environnement **Linux** (gestion des paquets, édition de fichiers de configuration)
* Notions sur la **sécurité des systèmes et des réseaux**

---

## Environnement de test

L'environnement repose sur une **topologie en étoile** centrée sur le Serveur Zabbix, qui collecte en temps réel les métriques de performance (CPU, RAM, réseau) depuis deux hôtes clients via des agents Zabbix.

| Nom d'hôte | Adresse IP | Masque |
|------------|------------|--------|
| Serveur-Zabbix | `178.62.219.43` | `/18` |
| Client-1 | `134.209.203.49` | `/20` |
| Client-2 | `178.62.235.81` | `/18` |

> **Infrastructure :** Déployé sur des instances **DigitalOcean** (Droplets), avec authentification par **paires de clés SSH** pour sécuriser les accès.

---

## Préparation des VMs

1. Installer **Ubuntu Server** sur chaque VM
2. Configurer une adresse IP fixe pour chaque VM (voir tableau ci-dessus)
3. Mettre à jour le système :
```bash
sudo apt update && sudo apt upgrade -y
```
4. Vérifier la connectivité entre les VMs avec des pings croisés :
```bash
# Exemple depuis Client-1
ping 178.62.219.43   # vers Serveur-Zabbix
ping 178.62.235.81   # vers Client-2
```

---

## Installation du serveur Zabbix

### 1. Installation des prérequis (Apache, MySQL, PHP)

```bash
sudo apt install apache2 mysql-server php libapache2-mod-php php-mysql -y
```

### 2. Ajout du dépôt et installation des paquets Zabbix

```bash
# Ajout du dépôt officiel Zabbix 7.0
wget https://repo.zabbix.com/zabbix/7.0/ubuntu/pool/main/z/zabbix-release/zabbix-release_7.0-1+ubuntu22.04_all.deb
sudo dpkg -i zabbix-release_7.0-1+ubuntu22.04_all.deb
sudo apt update
sudo apt install zabbix-server-mysql zabbix-frontend-php zabbix-apache-conf zabbix-sql-scripts zabbix-agent -y
```

### 3. Configuration de la base de données MySQL

```bash
sudo mysql -u root -p
```

```sql
CREATE DATABASE zabbix CHARACTER SET utf8mb4 COLLATE utf8mb4_bin;
CREATE USER 'zabbix'@'localhost' IDENTIFIED BY 'votre_mot_de_passe';
GRANT ALL PRIVILEGES ON zabbix.* TO 'zabbix'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

> ✅ L'importation du schéma (204 tables) confirme le succès de l'initialisation de la base Zabbix 7.0.

### 4. Configuration du serveur Zabbix

Modifier `/etc/zabbix/zabbix_server.conf` pour renseigner le mot de passe de la base :

```ini
DBPassword=votre_mot_de_passe
```

Définir le fuseau horaire dans la configuration PHP :

```ini
php_value date.timezone Europe/Paris
```

### 5. Démarrage des services

```bash
sudo systemctl restart zabbix-server zabbix-agent apache2
sudo systemctl enable zabbix-server zabbix-agent apache2
```

### 6. Accès à l'interface web

Ouvrir un navigateur et accéder à : [http://178.62.219.43/zabbix](http://178.62.219.43/zabbix)

Se connecter avec le compte par défaut `Admin / zabbix`, puis modifier le mot de passe immédiatement.

---

## Configuration des agents Zabbix

### Installation sur les clients (Client-1 & Client-2)

```bash
sudo apt install zabbix-agent -y
```

### Configuration de l'agent

Modifier `/etc/zabbix/zabbix_agentd.conf` sur chaque client :

```ini
# Sur Client-1
Server=178.62.219.43
ServerActive=178.62.219.43
Hostname=Client-1

# Sur Client-2
Server=178.62.219.43
ServerActive=178.62.219.43
Hostname=Client-2
```

### Redémarrage de l'agent

```bash
sudo systemctl restart zabbix-agent
sudo systemctl enable zabbix-agent
```

---

## Ajout des hôtes supervisés

1. Dans l'interface Zabbix, aller dans **Configuration → Hosts**
2. Cliquer sur **Create Host**
3. Renseigner le nom d'hôte, le groupe, et l'adresse IP de l'agent
4. Appliquer le template `Linux by Zabbix agent`
5. Cliquer sur **Add** puis **Apply**

> ✅ **Vérification :** Le *Zabbix agent ping* répond `Up (1)`. Le CPU idle time est supérieur à **99.7%**, confirmant que l'agent fonctionne correctement.

---

## Mise en place d'une alerte de sécurité

### Création d'un déclencheur (Trigger)

1. Aller dans **Data collection → Hosts**
2. Sur la ligne de **Client-1**, cliquer sur **Triggers**
3. Cliquer sur **Create trigger**

| Paramètre | Valeur |
|-----------|--------|
| Nom | `Charge CPU élevée` |
| Sévérité | `Warning` |
| Expression | `last(/Client-1/system.cpu.util)>=80` |

> Si le dernier relevé indique que le CPU tourne à **≥ 80%** de ses capacités, l'alarme **Warning** (jaune) se déclenche.

### Test de l'alerte avec `stress`

```bash
# Installation de l'outil stress sur Client-1
sudo apt install stress -y

# Lancement d'une surcharge CPU à 100% pendant 60 secondes
stress --cpu 4 --timeout 60
```

Vérifier dans **Monitoring → Problems** :

* Durant les 60 secondes : statut **`PROBLEM`** ⚠️
* Après la surcharge : statut **`RESOLVED`** ✅

---

## Résultats

* ✅ Serveur Zabbix entièrement fonctionnel et accessible via l'interface web
* ✅ Agents Zabbix installés et configurés sur les deux machines clientes
* ✅ Collecte et affichage des données de supervision opérationnels (CPU, RAM, réseau)
* ✅ Alerte de sécurité configurée, testée et validée avec succès

---

## Questions & Réponses

### Quels autres éléments de sécurité superviser avec Zabbix ?

* **Intégrité des fichiers** critiques (ex: `/etc/shadow`) via des sommes de contrôle
* **Tentatives de connexion SSH infructueuses** pour détecter des attaques par brute force
* **État des ports ouverts** sur les machines pour repérer des services illégitimes

### Comment Zabbix améliore-t-il la réponse aux incidents ?

Zabbix accélère la réponse grâce aux **alertes en temps réel** et à ses capacités d'**automatisation** (redémarrage de service, exécution de scripts de blocage) avant même toute intervention humaine.

### Défis en environnement de production réel

* Trouver le bon équilibre entre alertes pertinentes et **réduction du bruit**
* Gérer la montée en charge face à des **milliers d'équipements** supervisés
* Sécuriser les **flux de supervision eux-mêmes** pour éviter qu'ils deviennent un vecteur d'attaque

### Zabbix vs autres solutions de supervision

| Critère | Zabbix | Nagios | SolarWinds / PRTG |
|---------|--------|--------|-------------------|
| Licence | Open source (gratuit) | Open source | Commercial |
| Flexibilité | Très élevée (templates) | Modérée | Élevée |
| Complexité initiale | Moyenne à élevée | Élevée | Faible |
| Personnalisation | Très poussée | Bonne | Limitée |

### Compétences nécessaires pour administrer Zabbix

* Maîtrise des **systèmes Linux** (gestion des paquets, fichiers de configuration)
* Connaissance des **protocoles réseaux** (TCP/IP)
* Compréhension solide de la **sécurité informatique** pour interpréter les métriques

---

## Conclusion

Ce projet démontre concrètement l'importance vitale d'une solution de supervision pour la sécurité des réseaux locaux. En déployant l'architecture Zabbix sur DigitalOcean, nous avons transformé un réseau de machines passives en un **environnement surveillé et auto-évaluant**.

> *"Sans données de supervision, il est impossible de diagnostiquer les causes d'un incident ou de réagir efficacement aux menaces."*

Les compétences acquises — installation d'agents, configuration de triggers, interprétation de dashboards — sont **directement transférables dans le monde professionnel de la cybersécurité**.

---

*TP réalisé dans le cadre du module S4.01 — Sécurité des Réseaux | BUT R&T Cybersécurité — Villetaneuse*
