<!--
SPDX-FileCopyrightText: 2026 Guillaume Ponçon <guillaume@poncon.fr>

SPDX-License-Identifier: MIT
-->

# Documentation Technique de Sécurix

Cf. [documentation de Sécurix](https://github.com/cloud-gouv/securix/blob/main/docs/manual/src/SUMMARY.md).

## 1. Vue d'Ensemble

**Sécurix** est une distribution NixOS durcie pour postes de travail sécurisés, construite selon une approche modulaire et déclarative. Développé par la **DINUM** (Direction du Numérique - Ministère français), ce projet vise à créer des postes de travail conformes aux recommandations de l'ANSSI.

* **Version actuelle :** `v0.11-beta2`
* **Approche :** Configuration "Infrastructure as Code" intégrale.
* **Cible :** Postes de travail nécessitant un haut niveau de sécurité et de reproductibilité.

### Architecture Globale

```text
    +------------------------------------------------------------+
    |                    Architecture Sécurix                    |
    |                                                            |
    |  +-------------+    +-------------+    +-------------+     |
    |  |  Inventory  |    |   VPN       |    |   Modules   |     |
    |  | (machines/  |    | (Strongswan)|    |   Sécurix   |     |
    |  |   users/)   |    |             |    |             |     |
    |  +------+------+    +------+------+    +------+------+     |
    |         |                  |                  |            |
    |         +------------------+------------------+            |
    |                            |                               |
    |                   +--------v--------+                      |
    |                   |   mkTerminal    |                      |
    |                   | (lib/default)   |                      |
    |                   +--------v--------+                      |
    |                            |                               |
    |         +------------------+-------------------+           |
    |         |                  |                   |           |
    |  +------+--------+  +------+-------+  +--------+------+    |
    |  |  Installateur |  |  Netboot     |  |   Image       |    |
    |  |  (USB/ISO)    |  | Installateur |  |   Terminal    |    |
    |  +---------------+  +--------------+  +---------------+    |
    +------------------------------------------------------------+
```

---

## 2. Structure du Projet

### Arborescence Racine (`/`)

| Fichier/Dossier | Rôle |
|:---|:---|
| `default.nix` | Point d'entrée principal (définit lib, pkgs, modules, tests). |
| `shell.nix` | Environnement de développement (utilisé via `nix-shell`). |
| `lib/` | Bibliothèque Nix (fonctions de construction des images). |
| `modules/` | Cœur du système (~25 modules NixOS durcis). |
| `hardware/` | Configurations spécifiques aux modèles de machines supportés. |
| `pkgs/` | Overlay Nixpkgs et paquets personnalisés. |
| `npins/` | Gestion des dépendances externes (via `sources.json`). |
| `community/` | Modules et configurations partagés par la communauté. |
| `workflows/` | Définition des pipelines CI/CD (GitHub Actions). |
| `tests/` | Tests d'intégration NixOS. |

### Gestion des Dépendances (`npins`)

Contrairement à d'autres projets utilisant des fichiers JSON multiples, Sécurix utilise `npins` avec un fichier centralisé :
* `npins/sources.json` : Contient toutes les révisions verrouillées de `nixpkgs`, `lanzaboote`, `disko`, etc.

* **Avantage :** Garantit la reproductibilité parfaite du build sans utiliser les Flakes (choix architectural).

---

## 3. Configuration Système (`securix.self`)

Le module `modules/self.nix` définit l'identité unique de chaque poste. 

```nix
{
  securix.self = {

    # Type de description (user/machine/both)
    selfDescriptionType = "both";
    
    # Disque principal cible
    mainDisk = "/dev/nvme0n1";
    
    # Édition personnalisée (ex: "ministere-interieur")
    edition = "acme-corp";
    
    # Configuration utilisateur
    user = {
      email = "prenom.nom@domain.fr";
      username = "pnom";           # Dérivé automatiquement
      hashedPassword = "...";      # Hash bcrypt ($2b$...)
      u2f_keys = [ "..." ];        # Identifiants de clés U2F/FIDO2
      bit = 1;                     # Identifiant IP pour le VPN
      allowedVPNs = [ "vpn-id-1" ];
      teams = [ "devops" ];
    };
    
    # Configuration machine
    machine = {
      serialNumber = "SN123456789";
      inventoryId = 101;
      hardwareSKU = "t14g6";       # Référence au modèle dans hardware/
      users = [ "pnom" ];          # Liste des utilisateurs autorisés sur ce poste
    };
  };
}
```

---

## 4. Architecture de Sécurité

### Couches de Défense

| Niveau | Composant | Description |
|:---|:---|:---|
| **Matériel** | TPM 2.0 / FIDO2 | Stockage des clés et authentification forte. |
| **Boot** | Lanzaboote | Gestion du Secure Boot (clés PK/KEK/db) sans GRUB. |
| **Noyau** | Linux Hardened | Configuration conforme aux recommandations ANSSI. |
| **Réseau** | **Strongswan** | VPN IPSec robuste pour les flux nomades. |
| **Secrets** | Agenix | Chiffrement des secrets système via `age` et clés SSH. |
| **Audit** | Auditd | Traçabilité complète des actions privilégiées. |

### Flux d'Installation Automatique (`autoinstall`)

1.  **Wipe & Partitionnement :** Effacement sécurisé et application du schéma via `disko`.
2.  **Génération des clés :** Création locale des clés Secure Boot, clés d'hôte SSH et identifiants `age`.
3.  **Installation NixOS :** Déploiement du système dans le `chroot` cible.
4.  **Enrollment Secure Boot :** Enregistrement des clés dans le firmware UEFI via `sbctl`.
5.  **Provisionnement Strongswan :** Configuration des profils VPN IPSec.

---

## 5. Guide de Développement

### Entrer dans l'environnement

> Note : le projet n'utilise pas de Flakes pour le développement mais un shell nix.

Utilisez :

```bash
# Entrer dans le shell de développement
# Une fois dans le shell, les outils suivants sont disponibles :
# npins, agenix, sbctl, mdbook
nix-shell
```

### Construction des cibles

Les cibles sont définies dans le `default.nix` à la racine.

```bash
# Lancer les tests d'intégration en machine virtuelle
nix-build -A tests
```

---

## 6. Dépannage et FAQ

### Erreurs de permissions lors de l'édition des secrets

Les secrets sont gérés par `agenix`. Pour les modifier, votre clé SSH doit être déclarée dans le fichier `secrets.nix` du projet.

```bash
# Commande correcte
agenix -e secrets/vpn-password.age
```

### Échec de l'enrôlement Secure Boot

Si `sbctl enroll-keys` échoue, vérifiez que le BIOS est bien en "Setup Mode" (les clés d'usine doivent être effacées au préalable).

```bash
sbctl status  # Vérifiez 'Setup Mode: Enabled'
```

### Problèmes de connexion VPN IPSec

Sécurix utilise **Strongswan**. Vérifiez les journaux du service :

```bash
journalctl -u strongswan-swanctl -f
```

### Clé d'hôte SSH pour le chiffrement age

Si vous devez régénérer une identité pour `age` basée sur la clé de l'hôte :

```bash
# Ne pas utiliser age-keygen directement sur une clé privée SSH existante sans précaution.
# Utilisez la clé publique pour chiffrer et la clé privée machine pour déchiffrer via agenix.
ssh-keyscan localhost > /etc/ssh/ssh_host_ed25519_key.pub
```

---

## 7. Matériel Supporté

Le dossier `hardware/` contient des optimisations spécifiques (firmwares, kernel modules) pour :

* **Lenovo :** X280, T14 Gen 6, E14 Gen 7.
* **HP :** EliteBook 645 G11.
* **Dell :** Latitude 5340.
* **Fujitsu :** Lifebook U9-15.
* **Dynabook :** Portega X30L-G.

---

## 8. Liste des Modules

Le cœur de Sécurix repose sur des modules NixOS spécialisés. Ils sont organisés pour appliquer une politique de sécurité de type "défense en profondeur", inspirée des standards de l'ANSSI.

### Modules de Sécurité & Conformité

**`anssi/`** -> Durcissement général et conformité.

Ce module applique les recommandations de sécurité de l'ANSSI pour les systèmes GNU/Linux. Il configure les paramètres du noyau (sysctl), restreint certains accès système et désactive les protocoles ou fonctionnalités jugés peu sûrs.

**`auditd/`** -> Traçabilité et journalisation des événements système.

Gère la configuration du démon d'audit Linux (`auditd`). Il permet de tracer de manière granulaire les accès aux fichiers sensibles, les appels système critiques et les élévations de privilèges.

**`pam/`** -> Authentification renforcée (Pluggable Authentication Modules).

Configure la brique d'authentification Linux pour forcer l'usage du MFA (authentification multifacteur). Il s'interface notamment avec les clés matérielles Yubikey pour interdire l'accès par simple mot de passe.

**`security-keys/`** -> Support des clés FIDO2 / U2F.

Fournit les règles système (comme les règles `udev`) nécessaires pour reconnaître et interagir nativement avec des clés de sécurité matérielles.

**`ssh-tpm-agent/`** -> Agent SSH adossé à la puce TPM 2.0.

Gère l'agent SSH capable de générer et d'utiliser des clés d'authentification scellées dans le composant matériel TPM 2.0 de la machine. La clé privée ne quitte ainsi jamais le matériel.

---

### Gestion des Utilisateurs & Accès

**`admins/`** -> Gestion des comptes administrateurs locaux.

Définit la liste et les droits des utilisateurs locaux ayant l'autorisation d'élever leurs privilèges (via `sudo`) pour effectuer de la maintenance.

**`authorized-users/`** -> Gestion des utilisateurs réguliers autorisés.

Déclare les utilisateurs légitimes du poste de travail et lie leurs identités aux clés de chiffrement et d'authentification configurées.

**`superadmins/`** -> Comptes de secours ou d'administration globale.

Réserve des accès spécifiques et hautement surveillés pour les administrateurs centraux du parc informatique en cas de besoin de remédiation lourde.

---

### Système de Fichiers & Amorçage

**`bootloader/`** -> Gestion du démarrage sécurisé.

Configure l'amorçage de la machine. C'est ici que l'on retrouve l'intégration de `lanzaboote` pour l'implémentation d'un véritable UEFI Secure Boot.

**`disko/`** -> Partitionnement déclaratif.

Utilise l'outil `disko` pour formater et partitionner le disque. Il s'assure que le chiffrement au repos (LUKS) est correctement mis en place sur les partitions de données.

**`filesystems/`** -> Montage et sécurité des partitions.

Définit les points de montage et applique des options de sécurité strictes sur ces derniers (comme l'interdiction d'exécuter des binaires sur certaines partitions avec `noexec`).

---

### Réseau & Connectivité

**`bastion/`** -> Configuration des accès via rebond.

Paramètre les accès SSH à travers un bastion d'administration pour respecter les schémas d'architecture réseau sécurisés.

**`http-proxy/`** -> Sortie réseau via un mandataire.

Configure les variables d'environnement globales nécessaires pour forcer le trafic HTTP/HTTPS à transiter par un proxy d'entreprise.

**`known-hosts/`** -> Gestion des identités de serveurs SSH.

Pré-remplit les clés publiques des serveurs distants de confiance pour éviter les attaques de type *Man-in-the-Middle* lors des connexions des utilisateurs.

**`networking/`** -> Pile réseau et pare-feu.

Gère le nom de la machine, les DNS, et définit des règles de pare-feu restrictives pour n'autoriser que les flux strictement nécessaires.

**`vpn/`** -> Tunnelisation sécurisée.

Met en place les tunnels VPN d'entreprise en s'appuyant sur Strongswan pour assurer des liaisons IPsec chiffrées aux utilisateurs nomades.

---

### Administration & Identité

**`self.nix`** -> Carte d'identité de la machine (Module crucial).

Ce module fait la liaison entre l'inventaire global et la machine locale. Il récupère les spécificités de l'utilisateur (e-mail, hash de mot de passe, clé U2F) et de la machine (numéro de série, modèle matériel) pour forger la configuration finale.

**`distribution/`** -> Paramètres fondamentaux de l'OS.

Règle les paramètres globaux propres à NixOS (comme la version de l'état du système pour assurer la compatibilité ascendante).

**`package-manager/`** -> Configuration du binaire `nix`.

Restreint l'utilisation du gestionnaire de paquets Nix pour s'assurer que seuls les dépôts et caches binaires autorisés par l'organisation sont utilisés.

**`pki/`** -> Gestion des certificats d'autorité (CA).

Permet d'injecter des certificats racines d'entreprise dans le magasin de confiance de l'OS pour déchiffrer les flux (si nécessaire) ou s'authentifier auprès de briques internes.

**`spiffe/`** -> Identité applicative dynamique.

Apporte le support du framework SPIFFE pour délivrer des identités cryptographiques automatiques à courte durée de vie aux services tournant sur le poste.

**`updates/`** -> Mises à jour automatiques.

Planifie et automatise la récupération et l'application des nouvelles versions de la configuration système sans action requise de l'utilisateur.

---

### Interface & Ergonomie

**`console/`** -> Configuration du terminal TTY natif.

Gère la langue du clavier virtuel (AZERTY par défaut) et les polices d'affichage de la console hors interface graphique.

**`graphical-interface/`** -> Bureau et affichage.

Lance et sécurise le serveur graphique ainsi que le gestionnaire de fenêtres du poste de travail.

**`journal/`** -> Gestion des logs systemd.

Configure la taille maximale et la persistance des journaux locaux pour éviter les dénis de service par saturation de disque.

**`power-saving/`** -> Optimisation de la batterie.

Applique des profils d'économie d'énergie pour maximiser l'autonomie des ordinateurs portables sans trop dégrader les performances.

**`shells/`** -> Configuration des interpréteurs de commandes.

Définit les interpréteurs autorisés (comme Zsh) et les munit d'un prompt ainsi que de raccourcis sécurisés pour l'utilisateur.

**`tools/`** -> Utilitaires d'administration.

Embarque des petits utilitaires indispensables à la vie d'un administrateur système ou d'un utilisateur au quotidien (curls, htop, etc.).

---
*Dernière mise à jour : Avril 2026 - Version v0.11-beta2*
