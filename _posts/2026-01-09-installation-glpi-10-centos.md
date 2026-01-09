---
title: "Installation et configuration de GLPI 10 sur CentOS Stream 9"
date: 2026-01-09
author: Aymane Aboukhaira
status: Completed
---
---

# Installation et configuration de GLPI 10 sur CentOS Stream 9

**Mise en place d’un environnement LAMP sécurisé**

## Introduction

Dans le cadre de notre formation en **Infrastructure Digitale / Administration Systèmes**, ce travail a pour objectif de mettre en pratique les compétences liées au déploiement d’une application web professionnelle sous Linux.

Le projet consiste à installer et configurer **GLPI 10**, une solution open-source de gestion des services informatiques (**ITSM**), sur un serveur **CentOS Stream 9**, en utilisant l’architecture **LAMP (Linux, Apache, MariaDB, PHP)**.

Ce rapport détaille toutes les étapes techniques, les commandes utilisées, ainsi que leur **explication**, afin de démontrer la compréhension réelle du fonctionnement du système.

---

## 1. Présentation de GLPI

**GLPI (Gestionnaire Libre de Parc Informatique)** est une application web permettant :

* La gestion des tickets d’assistance
* L’inventaire du matériel et des logiciels
* La gestion des utilisateurs et des droits
* L’intégration avec Active Directory (LDAP)

**GLPI** est largement utilisé dans les entreprises, administrations et établissements scolaires.

---

## 2. Environnement de travail

| Composant              | Technologie     |
| ---------------------- | --------------- |
| Système d’exploitation | CentOS Stream 9 |
| Serveur Web            | Apache (httpd)  |
| Base de données        | MariaDB         |
| Langage serveur        | PHP 8.x         |
| Architecture           | LAMP            |

---

## 3. Préparation du système

### 3.1 Mise à jour du système

```bash
sudo dnf update -y
```

**Explication :**

* `dnf` : gestionnaire de paquets de CentOS
* `update` : recherche et installe les mises à jour
* `-y` : accepte automatiquement toutes les confirmations

  Cette étape permet d’éviter les failles de sécurité et les incompatibilités logicielles.

---

### 3.2 Installation des outils de base

```bash
sudo dnf install epel-release wget tar nano policycoreutils-python-utils -y
```

**Explication :**

* `epel-release` : ajoute le dépôt EPEL (paquets supplémentaires)
* `wget` : téléchargement de fichiers depuis Internet
* `tar` : extraction d’archives
* `nano` : éditeur de texte
* `policycoreutils-python-utils` : gestion avancée de SELinux

  Ces outils sont indispensables pour l’administration du système.

---

## 4. Installation du serveur Web Apache

### 4.1 Installation d’Apache

```bash
sudo dnf install httpd -y
```

**Explication :**
Installe le serveur web Apache, qui permettra aux utilisateurs d’accéder à GLPI via un navigateur.

---

### 4.2 Démarrage et activation du service

```bash
sudo systemctl enable --now httpd
```

**Explication :**

* `systemctl` : gestion des services
* `enable` : démarre Apache automatiquement au démarrage
* `--now` : démarre le service immédiatement

  Apache devient opérationnel et persistant.

---

## 5. Installation et configuration de MariaDB

### 5.1 Installation de MariaDB

```bash
sudo dnf install mariadb-server mariadb -y
sudo systemctl enable --now mariadb
```

**Explication :**
MariaDB est le serveur de base de données qui stockera toutes les informations de GLPI.

---

### 5.2 Sécurisation de MariaDB

```bash
sudo mysql_secure_installation
```

**Explication :**
Cette commande lance un script interactif permettant de :

* Définir un mot de passe root
* Supprimer les utilisateurs anonymes
* Désactiver l’accès root distant
* Supprimer les bases de test

  Étape essentielle pour la sécurité du serveur.

---

### 5.3 Création de la base de données GLPI

```bash
mysql -u root -p
```

```sql
CREATE DATABASE glpidb CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'glpiuser'@'localhost' IDENTIFIED BY '123456';
GRANT ALL PRIVILEGES ON glpidb.* TO 'glpiuser'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

**Explication :**

* `CREATE DATABASE` : crée une base dédiée à GLPI
* `CREATE USER` : crée un utilisateur spécifique
* `GRANT` : attribue les droits uniquement sur cette base
* `FLUSH PRIVILEGES` : applique les changements

  Application du principe du moindre privilège.

---

## 6. Installation et configuration de PHP

### 6.1 Installation de PHP et extensions

```bash
sudo dnf install php php-cli php-fpm php-mysqlnd php-gd php-mbstring php-curl php-xml php-intl php-ldap php-soap php-bcmath php-zip -y
```

**Explication :**

* `php-mysqlnd` : connexion à MariaDB
* `php-ldap` : intégration Active Directory
* `php-gd` : gestion des images
* `php-mbstring` : gestion des caractères spéciaux
* `php-curl` : requêtes réseau

  Sans ces extensions, GLPI ne peut pas fonctionner correctement.

---

### 6.2 Configuration de PHP

```bash
sudo nano /etc/php.ini
```

Modifier ou ajouter les paramètres suivants :

```ini
memory_limit = 512M
upload_max_filesize = 20M
post_max_size = 20M
max_execution_time = 300
date.timezone = Africa/Casablanca
```

**Explication :**

* Augmentation de la mémoire PHP
* Autorisation de l’upload de fichiers volumineux
* Éviter les erreurs de timeout
* Configuration correcte du fuseau horaire

---

## 7. Installation de GLPI

### 7.1 Téléchargement

```bash
cd /tmp
wget https://github.com/glpi-project/glpi/releases/download/10.0.10/glpi-10.0.10.tgz
```

**Explication :**
Téléchargement de la version stable de GLPI depuis GitHub.

---

### 7.2 Extraction et déplacement

```bash
tar -xvf glpi-10.0.10.tgz
sudo mv glpi /var/www/html/
```

  `/var/www/html` est le répertoire web par défaut d’Apache.

---

### 7.3 Permissions

```bash
sudo chown -R apache:apache /var/www/html/glpi
sudo chmod -R 755 /var/www/html/glpi
```

**Explication :**
Apache doit pouvoir écrire dans les dossiers GLPI.
Sans ces droits, GLPI générera des erreurs.

---

## 8. Sécurité : Pare-feu et SELinux

### 8.1 Pare-feu

```bash
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --reload
```

  Autorise l’accès web au serveur.

---

### 8.2 SELinux

```bash
sudo setsebool -P httpd_can_network_connect 1
sudo setsebool -P httpd_can_network_connect_db 1
sudo semanage fcontext -a -t httpd_sys_rw_content_t "/var/www/html/glpi(/.*)?"
sudo restorecon -Rv /var/www/html/glpi
```

**Explication :**

* Autorise Apache à communiquer avec la base de données et le réseau
* Applique les contextes de sécurité corrects aux fichiers GLPI

---

## 9. Finalisation

```bash
sudo systemctl restart httpd
```

**Accès via navigateur :**

```text
http://adresse_ip/glpi
```

---

## 10. Post-installation

```bash
sudo rm /var/www/html/glpi/install/install.php
```

  Empêche toute réinstallation malveillante.

---

## Conclusion

Ce projet a permis de :

* Comprendre l’architecture **LAMP**
* Déployer une application professionnelle
* Sécuriser un serveur Linux
* Appliquer les bonnes pratiques d’administration systèmes

---

🎉 **GLPI 10 est maintenant opérationnel et sécurisé sur CentOS Stream 9.**

```

