# Déploiement d’une infrastructure de supervision réseau avec Zabbix

## Table des matières

1. [Introduction](#introduction)
2. [Objectif du projet](#objectif-du-projet)
3. [Architecture mise en place](#architecture-mise-en-place)
4. [Environnement utilisé](#environnement-utilisé)
5. [Préparation des serveurs](#préparation-des-serveurs)
6. [Installation du serveur Zabbix](#installation-du-serveur-zabbix)
7. [Configuration des agents Zabbix](#configuration-des-agents-zabbix)
8. [Ajout des hôtes supervisés](#ajout-des-hôtes-supervisés)
9. [Mise en place d'une alerte de sécurité](#mise-en-place-dune-alerte-de-sécurité)
10. [Résultats obtenus](#résultats-obtenus)
11. [Analyse et pistes d'amélioration](#analyse-et-pistes-damélioration)
12. [Conclusion](#conclusion)

---

## Introduction

Ce projet présente la mise en place d'une solution de supervision réseau avec **Zabbix**, dans le but de surveiller plusieurs machines Linux depuis un serveur central.

L'objectif était de disposer d'une interface unique permettant de suivre l'état des machines, de collecter des métriques système et de détecter automatiquement certains comportements anormaux, comme une charge CPU trop élevée.

Zabbix a été choisi car c'est une solution open source complète, adaptée à la supervision de serveurs, de services réseau, d'applications et d'équipements d'infrastructure.

---

## Objectif du projet

L'objectif principal était de construire une petite infrastructure supervisée composée :

* d'un serveur Zabbix central ;
* de deux machines clientes supervisées ;
* d'agents Zabbix installés sur chaque hôte ;
* d'une alerte simple permettant de détecter une surcharge CPU.

Cette mise en place permet de mieux comprendre le fonctionnement d'une supervision centralisée dans un contexte réseau, avec une approche orientée sécurité et disponibilité.

Les points travaillés sont les suivants :

* installation et configuration d'un serveur Zabbix ;
* configuration d'agents Zabbix sur des machines clientes ;
* ajout d'hôtes supervisés dans l'interface web ;
* collecte de métriques système ;
* création et test d'une alerte de sécurité ;
* vérification du bon fonctionnement via les tableaux de bord Zabbix.

---

## Architecture mise en place

L'architecture repose sur un serveur Zabbix qui centralise les informations envoyées par deux machines clientes.

Les clients disposent chacun d'un agent Zabbix configuré pour communiquer avec le serveur principal. Le serveur collecte ensuite les métriques système comme l'utilisation CPU, la mémoire, le réseau et l'état général de l'agent.

| Nom d'hôte     | Adresse IP       | Masque |
| -------------- | ---------------- | ------ |
| Serveur-Zabbix | `178.62.219.43`  | `/18`  |
| Client-1       | `134.209.203.49` | `/20`  |
| Client-2       | `178.62.235.81`  | `/18`  |

L'accès aux machines a été sécurisé par authentification SSH à l'aide de paires de clés, afin d'éviter l'utilisation d'une authentification classique par mot de passe.

---

## Environnement utilisé

L'environnement de test repose sur plusieurs serveurs Linux sous Ubuntu Server.

Les composants utilisés sont :

* **Ubuntu Server** pour les machines ;
* **Zabbix 7.0** pour la supervision ;
* **Apache** pour l'interface web ;
* **MySQL** pour la base de données ;
* **PHP** pour le fonctionnement du frontend Zabbix ;
* **Zabbix Agent** pour la remontée des métriques depuis les clients ;
* **SSH avec clés** pour l'administration distante des machines.

---

## Préparation des serveurs

Avant l'installation de Zabbix, les machines ont été mises à jour afin de partir sur un environnement propre.

```bash
sudo apt update && sudo apt upgrade -y
```

La connectivité entre les machines a ensuite été vérifiée avec des tests de ping croisés.

Exemple depuis `Client-1` :

```bash
ping 178.62.219.43
ping 178.62.235.81
```

Cette étape permet de confirmer que les machines peuvent communiquer entre elles avant de configurer la supervision.

---

## Installation du serveur Zabbix

### 1. Installation des prérequis

Le serveur Zabbix nécessite plusieurs composants pour fonctionner correctement : un serveur web, une base de données et PHP.

```bash
sudo apt install apache2 mysql-server php libapache2-mod-php php-mysql -y
```

---

### 2. Ajout du dépôt officiel Zabbix

Le dépôt officiel Zabbix 7.0 a été ajouté afin d'installer les paquets nécessaires depuis une source fiable.

```bash
wget https://repo.zabbix.com/zabbix/7.0/ubuntu/pool/main/z/zabbix-release/zabbix-release_7.0-1+ubuntu22.04_all.deb
sudo dpkg -i zabbix-release_7.0-1+ubuntu22.04_all.deb
sudo apt update
```

Installation des paquets Zabbix :

```bash
sudo apt install zabbix-server-mysql zabbix-frontend-php zabbix-apache-conf zabbix-sql-scripts zabbix-agent -y
```

---

### 3. Configuration de la base de données

Une base de données dédiée à Zabbix a été créée dans MySQL.

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

L'importation du schéma Zabbix a ensuite permis d'initialiser la base de données avec les tables nécessaires au fonctionnement de la plateforme.

L'importation a créé environ **204 tables**, ce qui confirme que la base Zabbix 7.0 a bien été initialisée.

---

### 4. Configuration du serveur Zabbix

Le fichier de configuration principal du serveur Zabbix a été modifié pour renseigner le mot de passe de la base de données.

Fichier concerné :

```bash
/etc/zabbix/zabbix_server.conf
```

Ligne modifiée :

```ini
DBPassword=votre_mot_de_passe
```

Le fuseau horaire a également été configuré côté PHP afin d'utiliser l'heure de Paris dans l'interface web.

```ini
php_value date.timezone Europe/Paris
```

---

### 5. Démarrage des services

Une fois la configuration terminée, les services Zabbix, Apache et l'agent local ont été redémarrés puis activés au démarrage.

```bash
sudo systemctl restart zabbix-server zabbix-agent apache2
sudo systemctl enable zabbix-server zabbix-agent apache2
```

---

### 6. Accès à l'interface web

L'interface web Zabbix est accessible depuis un navigateur à l'adresse suivante :

```text
http://178.62.219.43/zabbix
```

La première connexion s'effectue avec le compte par défaut :

```text
Admin / zabbix
```

Après connexion, le mot de passe par défaut a été modifié afin de sécuriser l'accès à l'interface d'administration.

---

## Configuration des agents Zabbix

Les agents Zabbix ont été installés sur les deux machines clientes afin qu'elles puissent transmettre leurs métriques au serveur principal.

### Installation de l'agent

Sur chaque client :

```bash
sudo apt install zabbix-agent -y
```

---

### Configuration de l'agent sur Client-1

Fichier modifié :

```bash
/etc/zabbix/zabbix_agentd.conf
```

Configuration utilisée :

```ini
Server=178.62.219.43
ServerActive=178.62.219.43
Hostname=Client-1
```

---

### Configuration de l'agent sur Client-2

Fichier modifié :

```bash
/etc/zabbix/zabbix_agentd.conf
```

Configuration utilisée :

```ini
Server=178.62.219.43
ServerActive=178.62.219.43
Hostname=Client-2
```

---

### Redémarrage des agents

Après modification de la configuration, les agents ont été redémarrés et activés au démarrage.

```bash
sudo systemctl restart zabbix-agent
sudo systemctl enable zabbix-agent
```

---

## Ajout des hôtes supervisés

Les deux machines clientes ont ensuite été ajoutées dans l'interface web Zabbix.

Étapes réalisées :

1. Accès au menu **Configuration → Hosts**
2. Création d'un nouvel hôte avec **Create Host**
3. Renseignement du nom d'hôte et de l'adresse IP de l'agent
4. Association au groupe correspondant
5. Application du template `Linux by Zabbix agent`
6. Validation avec **Add** puis **Apply**

Après l'ajout des hôtes, le bon fonctionnement des agents a été vérifié avec l'indicateur **Zabbix agent ping**.

Le retour `Up (1)` confirme que le serveur Zabbix communique correctement avec les agents installés sur les clients.

Des premières métriques système ont également été visibles dans l'interface, notamment l'utilisation CPU, la mémoire et les informations réseau.

---

## Mise en place d'une alerte de sécurité

Une alerte a été configurée afin de détecter une charge CPU élevée sur `Client-1`.

### Création du déclencheur

Dans l'interface Zabbix :

1. Accès à **Data collection → Hosts**
2. Sélection de `Client-1`
3. Ouverture de la section **Triggers**
4. Création d'un nouveau trigger

Configuration utilisée :

| Paramètre  | Valeur                                |
| ---------- | ------------------------------------- |
| Nom        | `Charge CPU élevée`                   |
| Sévérité   | `Warning`                             |
| Expression | `last(/Client-1/system.cpu.util)>=80` |

Cette règle déclenche une alerte lorsque la dernière valeur de charge CPU atteint ou dépasse **80 %**.

---

## Test de l'alerte

Pour vérifier le bon fonctionnement du déclencheur, une surcharge CPU volontaire a été générée sur `Client-1` avec l'outil `stress`.

Installation de l'outil :

```bash
sudo apt install stress -y
```

Lancement du test :

```bash
stress --cpu 4 --timeout 60
```

Pendant le test, Zabbix a détecté la charge CPU élevée et a affiché un problème dans la section **Monitoring → Problems**.

Résultat observé :

* pendant la surcharge : statut `PROBLEM` ;
* après la fin du test : statut `RESOLVED`.

Ce comportement confirme que l'alerte est fonctionnelle et que Zabbix détecte correctement une anomalie temporaire sur une machine supervisée.

---

## Résultats obtenus

La mise en place a permis d'obtenir une infrastructure de supervision fonctionnelle.

Résultats validés :

* serveur Zabbix installé et accessible depuis l'interface web ;
* base de données MySQL correctement initialisée ;
* agents Zabbix installés sur les deux machines clientes ;
* communication fonctionnelle entre le serveur et les agents ;
* collecte des métriques système opérationnelle ;
* hôtes visibles dans l'interface Zabbix ;
* alerte CPU configurée et testée avec succès ;
* passage automatique de l'état `PROBLEM` à l'état `RESOLVED` après la fin de l'incident simulé.

---

## Analyse et pistes d'amélioration

Cette installation montre l'intérêt d'une supervision centralisée pour suivre l'état d'un parc de machines. Même avec une infrastructure réduite, Zabbix permet d'avoir une visibilité claire sur les ressources système et de détecter rapidement certains comportements inhabituels.

Dans un contexte plus avancé, plusieurs éléments pourraient être supervisés :

* l'intégrité de fichiers sensibles comme `/etc/passwd` ou `/etc/shadow` ;
* les tentatives de connexion SSH échouées ;
* l'ouverture de ports inhabituels ;
* l'état des services critiques ;
* l'espace disque disponible ;
* les pics de trafic réseau ;
* les changements de configuration système.

Zabbix peut aussi être utilisé pour améliorer la réponse aux incidents grâce aux alertes en temps réel. En production, il serait possible de compléter cette configuration avec des notifications par mail, webhook ou messagerie interne.

Il faudrait cependant faire attention au réglage des seuils d'alerte. Une supervision mal configurée peut générer trop de bruit, ce qui rend les vraies alertes plus difficiles à identifier.

---

## Comparaison rapide avec d'autres solutions

| Critère             | Zabbix       | Nagios                           | PRTG / SolarWinds      |
| ------------------- | ------------ | -------------------------------- | ---------------------- |
| Licence             | Open source  | Open source                      | Commercial             |
| Interface web       | Complète     | Plus limitée selon configuration | Très accessible        |
| Flexibilité         | Élevée       | Élevée mais plus manuelle        | Bonne                  |
| Complexité initiale | Moyenne      | Moyenne à élevée                 | Plus faible            |
| Personnalisation    | Très poussée | Bonne                            | Variable selon l'offre |
| Coût                | Gratuit      | Gratuit                          | Payant                 |

Zabbix se distingue surtout par sa flexibilité, son modèle open source et sa capacité à superviser aussi bien des serveurs que des équipements réseau ou des services applicatifs.

---

## Conclusion

Ce projet a permis de mettre en place une supervision centralisée avec Zabbix sur plusieurs machines Linux.

L'installation du serveur, la configuration des agents, l'ajout des hôtes et la création d'une alerte CPU ont permis de valider le fonctionnement complet de la chaîne de supervision.

L'intérêt principal de cette solution est de rendre l'état du réseau plus visible. Sans supervision, un incident peut rester invisible jusqu'à ce qu'il provoque une panne ou une dégradation importante du service. Avec Zabbix, les anomalies peuvent être détectées plus tôt, analysées plus facilement et traitées plus rapidement.

Cette mise en place constitue une base solide pour aller plus loin vers une supervision orientée sécurité, avec des alertes sur les connexions SSH, les fichiers sensibles, les services critiques et les comportements réseau inhabituels.
