# Projet "Test de Peter" : Exploration OverlayFS & LVM Migration

## Introduction : Inspiration Scénaristique et Exploration Technique

Ce dépôt contient des scripts de preuve de concept (Proof of Concept - PoC) inspirés par une analyse technique détaillée, similaire à celle décrite dans le document "Test de Peter". Cette analyse portait sur la faisabilité d'un scénario de piratage informatique vu dans un épisode d'une série télévisée américaine (probablement NCIS), où un agent compromet l'ordinateur d'un collègue en le démarrant sur une clé USB modifiée.

Le scénario fictif soulevait plusieurs questions techniques fascinantes :

1. Comment un système d'exploitation pourrait-il être modifié via une clé USB sans que l'utilisateur final ne s'en aperçoive immédiatement ?
2. Comment la clé USB pourrait-elle être retirée "à chaud" alors que le système d'exploitation modifié continue de fonctionner, après l'affichage d'une mystérieuse barre de progression ?

L'analyse technique explorée dans "Test de Peter" suggère une combinaison plausible (bien que complexe) de technologies Linux existantes :

* **Système sur NFS :** L'OS de la cible résiderait sur un partage réseau centralisé (NFS), typique des grandes organisations. La clé USB ne modifierait donc pas directement cet OS distant.
* **Initramfs modifié :** Le processus de démarrage initial (initramfs) serait altéré pour intercepter le montage NFS.
* **OverlayFS :** Un système de fichiers en superposition serait utilisé pour "empiler" une couche de modifications ("hacks") par-dessus le système NFS de base. L'OS démarré verrait alors l'union de ces deux couches, incluant les modifications malveillantes sans altérer le NFS original. Les "hacks" seraient initialement sur la clé USB.
* **Migration LVM à chaud :** Pour permettre le retrait de la clé USB, la couche de "hacks" (et potentiellement l'initramfs modifié lui-même) serait migrée depuis un volume logique LVM sur la clé USB vers un volume logique LVM sur le disque interne de la machine cible *pendant que le système tourne*, en utilisant la commande `pvmove` de LVM. Cette migration correspondrait à la barre de progression observée.

**Le but de ce dépôt N'EST PAS de fournir un outil de piratage fonctionnel**, mais plutôt d'offrir des scripts éducatifs qui **démontrent certains des concepts clés** évoqués :

* Le fonctionnement d'**OverlayFS** pour combiner des systèmes de fichiers.
* La simulation d'un **chroot** pour changer de contexte racine.
* La **logique d'appel** des commandes **LVM** nécessaires à une migration à chaud.

## ⚠️ Avertissements et Précautions d'Usage ⚠️

**TRÈS IMPORTANT :** Ces scripts sont fournis **UNIQUEMENT à des fins éducatives et de démonstration**. Ils illustrent des concepts techniques liés au fonctionnement interne de Linux.

* **CE NE SONT PAS DES OUTILS DE PIRATAGE.** Toute utilisation à des fins malveillantes, illégales ou non éthiques est strictement interdite et relève de votre seule responsabilité.
* **RISQUES SYSTÈME :** Le script Shell (`peter_test.sh`) utilise la commande `sudo` pour effectuer des opérations système réelles (montage/démontage de systèmes de fichiers, chargement de modules noyau, `chroot`). **Une mauvaise utilisation ou une exécution dans un environnement inapproprié peut potentiellement endommager votre système.**
* **ENVIRONNEMENT SÉCURISÉ REQUIS :** Exécutez le script Shell **UNIQUEMENT** dans un environnement contrôlé et non critique :
  * Idéalement une **machine virtuelle** Linux dédiée aux tests.
  * Sinon, dans un **répertoire temporaire** (`/tmp/peter_demo` comme suggéré) sur un système de test dont vous comprenez les risques.
  * **NE JAMAIS exécuter sur un système de production ou contenant des données importantes.**
* **SCRIPT PYTHON PLUS SÛR :** Le script Python (`peter_test.py`) est intrinsèquement plus sûr car il **simule** la logique d'OverlayFS via des opérations de fichiers standard et **n'exécute pas** de commandes LVM dangereuses par défaut (il se contente de les afficher). La simulation d'`exec` remplace le processus Python, ce qui est sans danger. Seule l'option `execute_check=True` (désactivée par défaut) tente une commande `sudo lvm version` inoffensive.
* **SIMULATION PARTIELLE :** Ces scripts ne répliquent **PAS** l'intégralité du scénario complexe :
  * Ils ne configurent pas de serveur ou client NFS.
  * Ils ne modifient pas un véritable initramfs.
  * Ils ne configurent pas de volumes LVM et n'effectuent pas de migration `pvmove` réelle.
  * La simulation `chroot` est très basique.

**En utilisant ces scripts, vous reconnaissez avoir lu, compris et accepté ces avertissements et vous assumez l'entière responsabilité de vos actions.**

## Aperçu des Scripts

1. **`peter_test.sh` (Script Shell)**
    * Démontre la création de répertoires simulant les couches `lowerdir` (base) et `upperdir` (hacks).
    * Charge le module noyau `overlay` si nécessaire.
    * Effectue un montage `overlayfs` réel combinant les deux couches.
    * Teste la lecture et l'écriture sur le point de montage overlay.
    * Simule un `chroot` basique dans l'environnement overlay pour exécuter un script.
    * Démonte proprement l'overlayfs et nettoie les répertoires temporaires.
    * **Nécessite `sudo` et manipule le système.**

2. **`peter_test.py` (Script Python)**
    * Simule la logique d'accès aux fichiers d'un overlayfs (priorité à `upperdir`) sans montage réel.
    * Implémente des fonctions `read_overlay_file`, `write_overlay_file`, `list_overlay_dir` basées sur cette logique.
    * Simule l'effet de `exec` avec `os.execvp` pour remplacer le processus courant.
    * Affiche les commandes LVM (`pvcreate`, `vgextend`, `pvmove`, etc.) qui *seraient* utilisées pour une migration, sans les exécuter.
    * Option pour vérifier l'accès à LVM via `sudo lvm version`.
    * **Généralement sûr, axé sur la logique.**

## Prérequis

* **Général :** Un environnement Linux (machine virtuelle recommandée).
* **Pour `peter_test.sh` :**
  * Accès `sudo`.
  * Interpréteur `bash`.
  * Commandes standard : `mkdir`, `echo`, `chmod`, `ls`, `cat`, `rm`, `rmdir`, `id`, `grep`.
  * Commandes système : `mount`, `umount`, `lsmod`, `modprobe`, `chroot`.
  * Module noyau `overlay` disponible et chargeable.
* **Pour `peter_test.py` :**
  * Interpréteur Python 3.
  * (Optionnel, pour `execute_check=True`) : Paquet `lvm2` installé et accès `sudo` fonctionnel pour exécuter `lvm version`.

## Installation

1. Clonez ce dépôt GitHub :

    ```bash
    git clone <URL_DU_DEPOT>
    cd <NOM_DU_REPERTOIRE>
    ```

2. Rendez le script Shell exécutable :

    ```bash
    chmod +x peter_test.sh
    ```

    Le script Python n'a généralement pas besoin d'être rendu exécutable si vous l'appelez avec `python3`.

## Procédure d'Exécution

### Script Shell (`peter_test.sh`) - **AVEC PRÉCAUTION !**

1. **CRÉEZ UN RÉPERTOIRE TEMPORAIRE ISOLÉ :**

    ```bash
    mkdir /tmp/peter_demo
    cd /tmp/peter_demo
    ```

2. Copiez le script dans ce répertoire (s'il n'y est pas déjà) :

    ```bash
    cp /chemin/vers/peter_test.sh .
    ```

3. **EXÉCUTEZ AVEC SUDO :**

    ```bash
    sudo ./peter_test.sh
    ```

4. Le script vous demandera une confirmation avant de continuer. Appuyez sur `Entrée`.
5. **Observez la sortie :** Le script affichera les étapes qu'il réalise :
    * Création des répertoires `base_system` et `hacks`.
    * Création de `mnt_union` et `work_dir`.
    * Vérification/chargement du module `overlay`.
    * Montage de l'overlayfs sur `mnt_union`.
    * Affichage du contenu de l'union et tests de lecture/écriture.
    * Simulation du `chroot` (copie de bash/sh, exécution de `chroot`).
    * Démontage et nettoyage.
6. **En cas de problème au démontage :** Si le script se termine avec un avertissement indiquant que le démontage a échoué (rare, mais possible si un processus utilise encore `mnt_union`), vous pouvez essayer de forcer le démontage :

    ```bash
    sudo umount -l /tmp/peter_demo/mnt_union
    ```

    Ensuite, vous pouvez supprimer manuellement le répertoire temporaire :

    ```bash
    cd /tmp && rm -rf /tmp/peter_demo
    ```

### Script Python (`peter_test.py`)

1. Naviguez jusqu'au répertoire contenant le script.
2. Exécutez avec Python 3 :

    ```bash
    python3 peter_test.py
    ```

3. **Observez la sortie :** Le script affichera les étapes de la simulation :
    * Création des répertoires `py_base_system` et `py_hacks`.
    * Tests de lecture via la logique overlay simulée.
    * Tests de listage de répertoires combinés.
    * Tests d'écriture (qui affectent `py_hacks`).
    * Affichage des commandes LVM simulées.
    * Tentative de simulation d'`exec` (si elle réussit, le script s'arrêtera là après avoir lancé le script `original_script.py` version hackée).
    * Nettoyage des répertoires (si `exec` échoue ou n'est pas atteint).
4. **(Optionnel) Tester la vérification LVM :** Modifiez la ligne `execute_check=False` en `execute_check=True` dans la fonction `simulate_lvm_migration_commands` et réexécutez. Si LVM est installé et `sudo` configuré, vous devriez voir la sortie de `lvm version`.

## Lien avec le Scénario "Test de Peter"

* **OverlayFS (`peter_test.sh`) :** Montre concrètement comment une couche de "hacks" (`hacks`) peut être superposée à un système de base (`base_system`) de manière transparente pour l'utilisateur final (via `mnt_union`). C'est le mécanisme clé pour injecter les modifications sans toucher au NFS original dans le scénario.
* **Chroot (`peter_test.sh`) :** La simulation (bien que basique) illustre l'étape où l'initramfs modifié, après avoir monté l'overlay, changerait la racine du système de fichiers pour pointer vers cet overlay avant de lancer le système d'init final.
* **Logique LVM (`peter_test.py`) :** La fonction `simulate_lvm_migration_commands` montre précisément le type de commandes (`pvcreate`, `vgextend`, `pvmove`, `vgreduce`) qui seraient nécessaires pour implémenter la migration à chaud des données (la couche `hacks`) depuis la clé USB vers le disque interne, expliquant ainsi la barre de progression et la possibilité de retirer la clé.

## Limitations Connues

* Simulation partielle du scénario global.
* Pas de gestion NFS.
* Pas de modification d'initramfs réel.
* Pas de configuration LVM ni de migration `pvmove` réelle.
* Simulation `chroot` très simplifiée.

## Licence

Ce projet est fourni sous la licence MIT. Voir le fichier `LICENSE` pour plus de détails. (Note: Vous devriez ajouter un fichier LICENSE contenant le texte de la licence MIT si vous souhaitez utiliser cette licence).
