<!--
SPDX-FileCopyrightText: 2026 Guillaume Ponçon <guillaume@poncon.fr>

SPDX-License-Identifier: MIT
-->

# Documentation Technique de Sécurix

- [Documentation officielle de Sécurix](https://github.com/cloud-gouv/securix/blob/main/docs/manual/src/SUMMARY.md).
- [Recommandations de configuration d'un système GNU/Linux](https://messervices.cyber.gouv.fr/guides/recommandations-de-securite-relatives-un-systeme-gnulinux)

---

## 1. Vue d'Ensemble

**Sécurix** est une distribution NixOS durcie pour postes de travail sécurisés, construite selon une approche modulaire et déclarative. Développé au sein du département de l'opérateur (OPI) de la **DINUM** (Direction du Numérique - Ministère français), ce projet vise à créer des postes de travail conformes aux recommandations de l'ANSSI, utilisables aussi bien pour l'accès aux environnements de production que pour d'autres usages critiques.

Grâce à NixOS, chaque poste est entièrement ré-instantiable et adaptable : configuration multi-agent, multi-niveaux, intranet uniquement, avec des équipes et des profils VPN différents.

* **Approche :** Configuration "Infrastructure as Code" intégrale, sans Flakes (choix architectural).
* **Cible :** Postes de travail nécessitant un haut niveau de sécurité et de reproductibilité, pour des équipes de taille petite à moyenne.

### Architecture Globale

```text
    +------------------------------------------------------------+
    |                    Architecture Sécurix                    |
    |                                                            |
    |  +-------------+    +-------------+    +-------------+     |
    |  |  Inventory  |    |  VPN        |    |   Modules   |     |
    |  | (machines/  |    | (Strongswan |    |   Sécurix   |     |
    |  |   users/)   |    |  IPSec)     |    |             |     |
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
    |  |  Installateur |  |  Netboot     |  |   Système     |    |
    |  |  (USB/ISO)    |  | Installateur |  |   NixOS       |    |
    |  +---------------+  +--------------+  +---------------+    |
    +------------------------------------------------------------+
```

---

## 2. Structure du Projet

### Arborescence Racine (`/`)

| Fichier/Dossier | Rôle |
|:---|:---|
| `default.nix` | Point d'entrée principal (définit lib, pkgs, modules, tests). |
| `shell.nix` | Environnement de développement (utilisé via `nix-shell` ou `direnv`). |
| `.envrc` | Intégration `direnv` pour activer automatiquement le shell Nix. |
| `lib/` | Bibliothèque Nix : fonctions de construction des images et de lecture de l'inventaire. |
| `modules/` | Cœur du système (~25 modules NixOS durcis). |
| `hardware/` | Configurations spécifiques aux modèles de machines supportés. |
| `pkgs/` | Overlay Nixpkgs et paquets personnalisés (`overlay.nix`). |
| `npins/` | Gestion des dépendances externes verrouillées (`sources.json`). |
| `docs/manual/` | Source de la documentation utilisateur (générée via `mdbook`). |
| `examples/basic/` | Exemple de projet Sécurix minimal pour démarrer. |
| `community/` | Scripts partagés par la communauté (ex. : inscription sur le registre Grist). |
| `workflows/` | Scripts de workflow complémentaires (séparés des pipelines `.github/workflows/`). |
| `tests/` | Tests d'intégration NixOS (exécution en machine virtuelle). |

### Gestion des Dépendances (`npins`)

Sécurix utilise `npins` avec un fichier centralisé (`npins/sources.json`) plutôt que des Flakes. Ce fichier verrouille les révisions exactes de toutes les dépendances externes, garantissant ainsi une reproductibilité parfaite du build.

| Dépendance | Version | Rôle |
|:---|:---|:---|
| `nixpkgs` | `nixos-25.11` | Base du système d'exploitation. |
| `lanzaboote` | `v0.4.2` | Implémentation du Secure Boot pour NixOS. |
| `disko` | `v1.9.0` | Partitionnement déclaratif des disques. |
| `agenix` | `main` (fcdea22) | Intégration `age` pour le chiffrement des secrets. |
| `git-hooks.nix` | `master` (9364dc0) | Pre-commit hooks (`statix`, `nixfmt-rfc-style`, `reuse`). |

---

## 3. Bibliothèque (`lib/`)

La bibliothèque est le cœur programmatique de Sécurix. Elle expose les fonctions de construction et de lecture de l'inventaire, utilisées dans le `default.nix` d'un projet consommateur.

```nix
lib = {
  # Lecture de l'inventaire v1 (répertoire plat de fichiers .nix)
  readInventory             # readInventory dir

  # Lecture de l'inventaire v2 (répertoires machines/ et users/ séparés)
  readInventory2            # readInventory2 { dir }

  # Construction du système installateur de bas niveau
  buildInstallerSystem

  # Construction d'une image USB bootable (ISO)
  buildUSBInstaller         # → ISO flashable sur clé USB
  buildUSBInstallerISO      # → retourne directement le chemin de l'image ISO

  # Construction d'un installateur via le réseau (iPXE/netboot)
  buildNetbootInstaller

  # Construction d'une image terminal unique (un poste)
  mkTerminal                # → { modules, partitioningModules, installer, system }

  # Construction de plusieurs images terminaux depuis un inventaire
  mkTerminals               # mkTerminals { users, vpn-profiles, edition } baseSystem

  # Génération de la documentation
  mkDocs                    # mkDocs { users, vpn-profiles, terminals } → { bastions, inventory }
}
```

La fonction centrale est **`mkTerminal`** : elle agrège les modules globaux, la configuration matérielle, le module utilisateur spécifique, les profils VPN et les outils de chiffrement (`lanzaboote`, `disko`, `agenix`) pour produire quatre artefacts :

| Clé | Description |
|:---|:---|
| `modules` | Liste complète des modules NixOS du terminal (utilisable dans d'autres contextes). |
| `partitioningModules` | Sous-ensemble de modules pour le partitionnement seul (disko + filesystems + self + pam). |
| `installer` | Image ISO bootable générée par `buildUSBInstallerISO`. |
| `system` | Système NixOS évalué (`pkgs.nixos allModules`), donnant accès à `system.config`. |

---

## 4. Configuration Système (`securix.self`)

Le module `modules/self.nix` est central : il définit l'identité unique de chaque poste en faisant le lien entre l'inventaire global et la machine locale.

```nix
{
  securix.self = {

    # Type de description du fichier (user / machine / both)
    selfDescriptionType = "both";

    # Disque principal cible pour l'installation
    mainDisk = "/dev/nvme0n1";

    # Édition personnalisée (ex: "ministere-interieur", "equipe-ops")
    edition = "acme-corp";

    # Configuration de l'utilisateur du poste
    user = {
      email = "prenom.nom@domain.fr";
      username = "pnom";           # Dérivé automatiquement depuis l'e-mail
      hashedPassword = "...";      # Hash bcrypt ($2b$...)
      u2f_keys = [ "..." ];        # Identifiants des clés U2F/FIDO2 enregistrées
      bit = 1;                     # Identifiant numérique pour l'adressage VPN
      allowedVPNs = [ "vpn-id-1" ];
      teams = [ "devops" ];
    };

    # Configuration de la machine physique
    machine = {
      serialNumber = "SN123456789";
      inventoryId = 101;
      hardwareSKU = "t14g6";       # Référence au profil dans hardware/
      users = [ "pnom" ];          # Liste des utilisateurs autorisés sur ce poste
    };
  };
}
```

---

## 5. Architecture de Sécurité

### Couches de Défense en Profondeur

| Niveau | Composant | Description |
|:---|:---|:---|
| **Matériel** | TPM 2.0 / FIDO2 | Stockage des clés et authentification forte, la clé privée SSH ne quitte jamais le matériel. |
| **Boot** | Lanzaboote | Gestion du Secure Boot (clés PK/KEK/db) sans GRUB, enrollment via `sbctl`. |
| **Chiffrement au repos** | LUKS (via `disko`) | Déchiffrement du disque possible via une clé FIDO2 ; une clé de secours est générée à l'installation. |
| **Noyau** | Linux Hardened | Configuration conforme aux recommandations ANSSI (sysctl, désactivation de protocoles non sûrs). |
| **Authentification** | PAM + U2F/Yubikey | Connexion au poste en FIDO2 ; le mot de passe n'est qu'un mode de secours. |
| **Réseau** | Strongswan | VPN IPSec/IKEv2 robuste pour les accès nomades aux ressources internes. |
| **Secrets** | Agenix (`age`) | Chiffrement des secrets système avec les clés SSH de l'hôte pour la gestion décentralisée. |
| **Audit** | Auditd | Traçabilité complète et granulaire des actions privilégiées et des accès sensibles. |
| **Observabilité** | `o11y` | Envoi des traces et métriques vers un puits de logs centralisé. |

### Flux d'Installation Automatique (`autoinstall`)

```text
autoinstall-terminal
  +-> 1. Effacement sécurisé du disque (wipefs + dd zero)
  +-> 2. Partitionnement et chiffrement (disko / LUKS)
  +-> 3. Montage et vérification de /mnt (doit être sur disque persistant)
  +-> 4. Génération des clés Secure Boot (sbctl create-keys)
  +-> 5. Installation NixOS (nixos-install)
  +-> 6. Enrollment Secure Boot (sbctl enroll-keys)
  +-> 7. Provisionnement clés TPM2 (si /dev/tpm0 disponible, via ssh-tpm-keygen -A)
  +-> 8. Provisionnement identité age (clé d'hôte SSH → déchiffrement des secrets)
  +-> 9. Script post-installation personnalisé (optionnel)
```

Voir la [section 8](#8-installation-de-nixos-avec-sécurix) pour le détail du processus interactif.

---

## 6. Guide de Développement

### Prérequis

* Nix installé sur le poste de développement (Linux ou macOS).
* `direnv` recommandé pour activer automatiquement le shell au `cd` dans le projet.

### Entrer dans l'environnement

Le projet n'utilise pas de Flakes. L'environnement de développement est défini dans `shell.nix`.

```bash
# Option 1 : activation manuelle
nix-shell

# Option 2 : activation automatique via direnv (recommandé)
# À configurer une seule fois après le clonage du dépôt :
direnv allow
```

Une fois dans le shell, les outils suivants sont disponibles : `npins`, `agenix`, `mdbook`, `statix`, `nixfmt-rfc-style`, `reuse`.

### Tester les Modifications

Pour valider vos changements, utilisez les tests d'intégration NixOS :

```bash
# Lancer les tests d'intégration en machine virtuelle
nix-build -A tests
```

### Workflow de Développement

Le dépôt Sécurix est une bibliothèque Nix (toolkit) destinée à être importée par un projet consommateur. Pour développer sur Sécurix lui-même :

1. **Modifier les modules** : les modules NixOS sont dans `modules/`. Chaque module est indépendant et configure un aspect spécifique du système.

2. **Tester les modifications** : utilisez les tests d'intégration pour valider vos changements :
   ```bash
   # Lancer tous les tests en VM
   nix-build -A tests

   # Tester une configuration minimale
   nix-build -A tests.minimal
   ```

3. **Vérifier le style de code** : les hooks pre-commit (statix, nixfmt-rfc-style, reuse) sont exécutés avant chaque push.

### Structure d'un Projet Consommateur

Pour utiliser Sécurix, créez un projet avec la structure suivante :

```
mon-projet/
├── default.nix      # Point d'entrée qui importe securix
├── inventory/       # Inventaire des machines et utilisateurs
│   ├── machines/   # Configurations des postes
│   └── users/      # Configurations des utilisateurs
├── vpn-profiles.nix # Profils VPN
└── npins/          # Dépendances (incluant securix)
```

Exemple de `default.nix` consommateur (inspiré de `examples/basic/`) :

```nix
{
  sources ? import ./npins,
  pkgs ? import sources.nixpkgs { },
  securixSrc ? sources.securix,
}:
let
  securix = import securixSrc {
    edition = "mon-equipe";
    defaultTags = [ "mon-equipe" ];
    inherit pkgs;
  };
  inherit (pkgs) lib;
in
rec {
  users = securix.lib.readInventory ./inventory;
  vpn-profiles = import ./vpn-profiles { inherit lib; };
  terminals = securix.lib.mkTerminals {
    inherit users vpn-profiles;
    edition = "mon-equipe";
  } (
    { lib, ... }: {
      securix = {
        users.allowAnyOperator = true;
        graphical-interface.enable = true;
        vpn.enable = true;
        ssh.tpm-agent.hostKeys = true;
      };
    }
  );
  docs = securix.lib.mkDocs { inherit users terminals vpn-profiles; };
}
```

> `mkTerminals` est une fonction curryfiée : elle prend d'abord un ensemble `{users, vpn-profiles, edition}`, puis un module NixOS de base (`baseSystem`) appliqué à tous les terminaux.
> `mkDocs` retourne `{ bastions, inventory }` : deux fichiers texte dans le store Nix contenant les flux réseau et l'inventaire des postes au format Markdown.

### Mettre à Jour une Dépendance

Pour mettre à jour une dépendance verrouillée dans `npins/sources.json` :

```bash
# Mettre à jour une dépendance spécifique
npins update nixpkgs

# Mettre à jour toutes les dépendances
npins update
```

---

## 7. Création d'une Image

### Avec un Projet Consommateur

La méthode recommandée est d'utiliser un projet consommateur avec un inventaire (voir [Structure d'un Projet Consommateur](#structure-dun-projet-consommateur)).

```bash
# Depuis le projet consommateur, construire toutes les images terminaux
nix-build -A terminals

# Construire l'image ISO pour un terminal spécifique
nix-build -A terminals.nom-du-terminal.installer
```

Le résultat est une image ISO (`result/iso/*.iso`) que vous pouvez flasher sur une clé USB.

### Création Manuelle d'une Image (Test/Sans Inventaire)

Pour tester rapidement sans inventaire complet, utilisez `mkTerminal` directement :

```nix
# test-image.nix
{ pkgs, lib }:
let
  securix = import ./. {
    edition = "test";
    pkgs = import <nixpkgs> { };
  };
  terminal = securix.lib.mkTerminal {
    name = "test";
    userSpecificModule = { };
    vpnProfiles = { };
    modules = [
      {
        securix = {
          graphical-interface.variant = "sway";
          self = {
            mainDisk = "/dev/nvme0n1";
            machine = {
              hardwareSKU = "x280";
              serialNumber = "TEST001";
            };
          };
        };
      }
    ];
  };
in
terminal.installer
```

Puis construisez avec :
```bash
nix-build test-image.nix
```

### Flashage de l'Image sur une Clé USB

```bash
# Identifier la clé USB (ex: /dev/sda)
lsblk

# Flasher l'ISO (remplacez le chemin et le périphérique)
sudo dd if=./result/iso/securix-*.iso of=/dev/sda bs=4M status=progress oflag=sync
```

---

## 8. Installation de NixOS avec Sécurix

### Préparation

1. **Démarrer sur la clé USB** contenant l'image Sécurix.
2. Le système démarre en mode "live installer" et se connecte automatiquement en root.

### Processus d'Installation Automatisé

Lancez le script d'installation automatique :

```bash
autoinstall-terminal
```

Ce script effectue les étapes suivantes :

```text
autoinstall-terminal
  +-> 1. Effacement sécurisé du disque (wipefs + dd zero)
  +-> 2. Partitionnement et chiffrement (disko / LUKS)
  +-> 3. Montage et vérification de /mnt (disque persistant requis)
  +-> 4. Génération des clés Secure Boot (sbctl create-keys)
  +-> 5. Installation NixOS (nixos-install)
  +-> 6. Enrollment Secure Boot (sbctl enroll-keys)
  +-> 7. Provisionnement clés TPM2 (si /dev/tpm0 présent)
  +-> 8. Provisionnement identité age (clé d'hôte SSH)
```

### Post-Installation

Après le redémarrage :

1. **Connexion** : Utilisez votre clé FIDO2/U2F ou le mot de passe de secours.
2. **Configuration VPN** : Les profils Strongswan sont pré-configurés selon l'inventaire.
3. **Vérification** :
   ```bash
   # Vérifier le statut Secure Boot
   sbctl status

   # Vérifier les logs VPN
   journalctl -u strongswan-swanctl -f
   ```

---

## 9. Fonctionnalités en Développement

Ces fonctionnalités sont planifiées et listées par priorité dans le README officiel.

**Renforcement de la sécurité**

Application complémentaire des recommandations ANSSI pour un durcissement plus poussé du système GNU/Linux. Ajout d'une configuration de puits de traces pour l'envoi centralisé des activités Sécurix.

**Onboarding automatisé ("Phone Home")**

Mise en place d'un serveur de provisionnement permettant, lors du premier démarrage d'un nouveau poste, d'enregistrer automatiquement sa clé SSH TPM2 dans le dépôt d'infrastructure et de lui accorder l'autorisation de déchiffrer ses secrets via `age` (ou via Vault dans une intégration future).

**Gestion et rotation des clés Secure Boot**

Support de la rotation des clés Secure Boot avec TPM2 pour garantir la continuité d'un niveau de sécurité élevé sur le long terme.

---

## 10. Dépannage et FAQ

### Erreurs de permissions lors de l'édition des secrets

Les secrets sont gérés par `agenix`. Pour les modifier, votre clé SSH doit être déclarée dans le fichier `secrets.nix` du projet.

```bash
# Syntaxe correcte pour éditer un secret
agenix -e secrets/vpn-password.age
```

### Échec de l'enrôlement Secure Boot

Si `sbctl enroll-keys` échoue, vérifiez que le firmware UEFI est bien en mode "Setup Mode" (les clés d'usine constructeur doivent être effacées au préalable depuis le BIOS).

```bash
# Vérifier l'état du Secure Boot
sbctl status
# Le champ 'Setup Mode' doit afficher : Enabled
```

### Problèmes de connexion VPN IPSec

Sécurix utilise **Strongswan** pour les tunnels IPSec/IKEv2. Pour diagnostiquer une connexion défaillante :

```bash
# Suivre les journaux du service Strongswan en temps réel
journalctl -u strongswan-swanctl -f
```

### Clé d'hôte SSH et chiffrement `age`

`agenix` utilise la clé d'hôte SSH de la machine (`/etc/ssh/ssh_host_ed25519_key`) comme identité `age` pour déchiffrer les secrets. La clé publique correspondante doit être déclarée dans `secrets.nix`. Ne pas régénérer manuellement cette clé sans mettre à jour les secrets chiffrés avec l'ancienne, au risque de rendre le système inaccessible.

```bash
# Afficher la clé publique de l'hôte (à référencer dans secrets.nix)
cat /etc/ssh/ssh_host_ed25519_key.pub
```

### Utilisation du cache binaire (build trop long)

Pour éviter de recompiler l'ensemble du système depuis les sources :

```bash
# Ajouter le cache de la communauté NixOS
cachix use nix-community
```

---

## 11. Matériel Supporté

Le dossier `hardware/` contient des profils d'optimisation spécifiques (firmwares, modules noyau, pilotes) pour les modèles suivants :

| SKU | Modèle |
|:---|:---|
| `x280` | Lenovo ThinkPad X280 |
| `t14g6` | Lenovo ThinkPad T14 Gen 6 |
| `e14-g7` | Lenovo ThinkPad E14 Gen 7 |
| `elitebook645g11` | HP EliteBook 645 G11 |
| `elitebook850g8` | HP EliteBook 850 G8 |
| `latitude5340` | Dell Latitude 5340 |
| `x9-15` | Fujitsu Lifebook U9-15 |
| `x13-20ug` | Dynabook Portégé X30L-G |

---

## 12. Liste des Modules

Le cœur de Sécurix repose sur des modules NixOS spécialisés, organisés selon une politique de "défense en profondeur" inspirée des standards de l'ANSSI.

### Sécurité & Conformité

**`anssi/`** — Durcissement général et conformité ANSSI.

Ce module applique les recommandations de sécurité de l'ANSSI pour les systèmes GNU/Linux. Il configure les paramètres du noyau via `sysctl`, restreint certains appels système et désactive les protocoles ou fonctionnalités jugés non sûrs. Certains paramètres peuvent être désactivés selon les besoins opérationnels.

**`auditd/`** — Traçabilité et journalisation des événements système.

Configure le démon d'audit Linux (`auditd`). Permet de tracer de manière granulaire les accès aux fichiers sensibles, les appels système critiques et les élévations de privilèges.

**`pam/`** — Authentification renforcée (Pluggable Authentication Modules).

Configure la brique d'authentification Linux pour imposer le MFA (authentification multifacteur). S'interface avec les clés matérielles Yubikey pour interdire l'accès par simple mot de passe.

**`security-keys/`** — Support des clés FIDO2 / U2F.

Fournit les règles `udev` nécessaires pour reconnaître et interagir nativement avec les clés de sécurité matérielles.

**`ssh-tpm-agent/`** — Agent SSH adossé à la puce TPM 2.0.

Gère l'agent SSH capable de générer et d'utiliser des clés d'authentification scellées dans le composant matériel TPM 2.0. La clé privée ne quitte ainsi jamais la machine.

---

### Gestion des Utilisateurs & Accès

**`admins/`** — Comptes administrateurs locaux.

Définit la liste et les droits des utilisateurs locaux autorisés à élever leurs privilèges via `sudo` pour effectuer de la maintenance.

**`authorized-users/`** — Utilisateurs réguliers autorisés.

Déclare les utilisateurs légitimes du poste et lie leurs identités aux clés de chiffrement et d'authentification configurées.

**`superadmins/`** — Comptes de secours ou d'administration centrale.

Réserve des accès spécifiques et hautement surveillés pour les administrateurs du parc informatique, à utiliser en cas de remédiation lourde.

---

### Système de Fichiers & Amorçage

**`bootloader/`** — Démarrage sécurisé.

Configure l'amorçage de la machine. Intègre `lanzaboote` pour un UEFI Secure Boot véritable, sans GRUB.

**`disko/`** — Partitionnement déclaratif.

Utilise l'outil `disko` pour formater et partitionner le disque. S'assure que le chiffrement au repos (LUKS) est correctement mis en place sur les partitions de données.

**`filesystems/`** — Montage et sécurité des partitions.

Définit les points de montage et applique des options de sécurité strictes (comme l'option `noexec` pour interdire l'exécution de binaires sur certaines partitions).

---

### Réseau & Connectivité

**`bastion/`** — Accès via rebond SSH.

Paramètre les accès SSH à travers un bastion d'administration, conformément aux schémas d'architecture réseau sécurisés préconisés par l'ANSSI.

**`http-proxy/`** — Sortie réseau via mandataire.

Configure les variables d'environnement globales pour forcer le trafic HTTP/HTTPS à transiter par un proxy d'entreprise.

**`known-hosts/`** — Identités de serveurs SSH de confiance.

Pré-remplit les clés publiques des serveurs distants connus pour prévenir les attaques de type *man-in-the-middle* lors des connexions.

**`networking/`** — Pile réseau et pare-feu.

Gère le nom de la machine, les DNS et définit des règles de pare-feu restrictives pour n'autoriser que les flux strictement nécessaires.

**`vpn/`** — Tunnelisation sécurisée.

Le module VPN est composite et inclut trois sous-protocoles configurables indépendamment :

| Sous-module | Technologie | Usage |
|:---|:---|:---|
| `vpn/ipsec/` | Strongswan IKEv2 | Tunnels IPSec d'entreprise (mode nomade). |
| `vpn/netbird/` | Netbird (WireGuard-based) | VPN mesh pair-à-pair. |
| `vpn/wireguard/` | WireGuard | Tunnels légers point-à-point. |

---

### Administration & Identité

**`self.nix`** — Identité de la machine *(module central)*.

Fait le lien entre l'inventaire global et la machine locale. Récupère les spécificités de l'utilisateur (e-mail, hash de mot de passe, clé U2F) et de la machine (numéro de série, modèle matériel) pour forger la configuration finale.

**`distribution/`** — Paramètres fondamentaux de l'OS.

Règle les paramètres globaux NixOS, notamment la version de l'état du système (`stateVersion`) pour assurer la compatibilité ascendante lors des mises à jour.

**`package-manager/`** — Configuration du gestionnaire de paquets Nix.

Restreint l'utilisation de Nix aux seuls dépôts et caches binaires autorisés par l'organisation.

**`pki/`** — Certificats d'autorité (CA).

Injecte les certificats racines d'entreprise dans le magasin de confiance de l'OS, nécessaires pour l'inspection TLS ou l'authentification auprès des briques internes.

**`spiffe/`** — Identités applicatives dynamiques.

Apporte le support du framework SPIFFE pour délivrer des identités cryptographiques à courte durée de vie aux services tournant sur le poste.

**`o11y/`** — Observabilité.

Configure l'envoi des traces et métriques du poste vers un puits de logs centralisé, permettant une supervision homogène du parc.

**`updates/`** — Mises à jour automatiques.

Planifie et automatise la récupération et l'application des nouvelles versions de la configuration système.

---

### Interface & Ergonomie

**`console/`** — Terminal TTY natif.

Gère la langue du clavier (AZERTY par défaut) et les polices d'affichage de la console hors interface graphique.

**`graphical-interface/`** — Bureau et affichage.

Lance et sécurise le serveur graphique ainsi que le gestionnaire de fenêtres du poste. Inclut un thème KDE de base.

**`journal/`** — Logs systemd.

Configure la taille maximale et la persistance des journaux locaux pour éviter les dénis de service par saturation du disque.

**`power-saving/`** — Gestion de l'énergie.

Applique des profils d'économie d'énergie pour maximiser l'autonomie des ordinateurs portables.

**`shells/`** — Interpréteurs de commandes.

Définit les interpréteurs autorisés (Zsh par défaut) et les configure avec un prompt et des raccourcis adaptés.

**`tools/`** — Utilitaires d'administration.

Embarque les utilitaires courants pour les administrateurs et les utilisateurs, dont `qrencode` pour générer des QR codes (utile pour l'enrôlement FIDO2).

---

## 13. Références

* [Recommandations ANSSI — Systèmes GNU/Linux](https://cyber.gouv.fr/publications/recommandations-de-securite-relatives-un-systeme-gnulinux)
* [Recommandations ANSSI — Administration sécurisée des SI](https://cyber.gouv.fr/publications/recommandations-relatives-ladministration-securisee-des-si)
* [Dépôt GitHub — cloud-gouv/securix](https://github.com/cloud-gouv/securix)

---
*Dernière mise à jour : Avril 2026 — Version `v0.11-beta2`*
