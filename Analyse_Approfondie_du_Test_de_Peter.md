# Projet

Le texte fourni relate une conversation entre le narrateur et son ami Peter, un passionné de tests en tout genre. La discussion se concentre sur la faisabilité technique d'une scène de piratage informatique tirée d'une série américaine (probablement *NCIS*, mentionnant les personnages Gibbs, McGee, DiNozzo).

Dans cette scène, l'agent McGee compromet l'ordinateur de l'agent Devon (soupçonné d'être une taupe) en le redémarrant sur une clé USB spécialement préparée, puis en retirant la clé "à chaud" une fois une barre de progression terminée, sans que Devon ne remarque de changement sur son système par la suite.

Peter et le narrateur décortiquent les aspects techniques qui pourraient rendre ce scénario, bien qu'invraisemblable à première vue, potentiellement réalisable sous certaines conditions spécifiques. Voici les points techniques clés abordés :

---

## Le Problème Initial

- **Réinstaller un OS sans que l'utilisateur ne s'en rende compte semble impossible.**
- Un redémarrage standard après installation n'a pas lieu dans la scène ; la clé est retirée et le système continue de tourner.

---

## Hypothèse 1 : Système sur NFS (Network File System)

Peter suggère que l'OS de Devon pourrait être hébergé sur un serveur NFS centralisé, typique des grandes organisations.

### Scénario possible

1. Le démarrage temporaire sur une clé USB locale ne modifierait pas les données du système d'exploitation distant.
2. Le piratage consisterait à altérer la manière dont le poste local accède à cet OS distant, sans modifier l'OS sur le serveur (trop sécurisé).

---

## Le Processus de Démarrage GNU/Linux

### Étapes classiques

1. **Firmware (BIOS/UEFI)** → **Bootloader (GRUB)** → **Noyau Linux** → **Montage racine** → **Système d'init (PID 1)**.

### Complexité

- Gérer des configurations complexes (RAID logiciel, Btrfs, LVM, chiffrement, NFS) directement depuis le noyau initial ou le bootloader est difficile.

### Solution : Initramfs (Initial RAM File System)

- Une archive CPIO compressée (ex. `initrd.img-<version>`) chargée en RAM par le bootloader.
- Contient un mini-système de fichiers avec les outils et modules nécessaires pour monter le vrai système de fichiers racine.
- Le noyau exécute le script `/init` de l'initramfs.

### Transition vers le système final

1. Une fois le système de fichiers racine monté, le script `/init` doit lancer le système d'init final (ex. `/sbin/init`, souvent un lien vers `systemd`).
2. **Problème** : Le système final s'attend à être à la racine (`/`), mais il est monté dans un sous-répertoire de l'initramfs.
3. **Solution** :
   - **chroot** : Change le répertoire racine perçu par le processus lancé.
   - **exec** : Remplace le processus courant par la commande spécifiée, crucial pour conserver le PID 1.

---

## Hypothèse 2 : Introduction de Hacks via OverlayFS

Pour pirater le système NFS sans le modifier directement, McGee pourrait modifier l'initramfs de Devon.

### OverlayFS

- Permet de fusionner plusieurs répertoires (couches) en un seul point de montage virtuel.
- Commande :

  ```bash
  mount -t overlay -o lowerdir=...,upperdir=...,workdir=... none /point/de/montage
  ```

---

## Hypothèse 3 : Migration "à Chaud" avec LVM (Logical Volume Management)

### Problème

- Si upperdir (les hacks et les nouvelles données écrites par l'utilisateur) est sur la clé USB, retirer la clé planterait le système. La barre de progression tardive suggère une copie de données après le démarrage du bureau.

### Solution : LVM

#### Concepts LVM

- **Volume Physique (PV - Physical Volume)** : Un disque ou une partition préparé(e) pour LVM (pvcreate).
- **Groupe de Volumes (VG - Volume Group)** : Un pool de un ou plusieurs PVs (vgcreate, vgextend).
- **Volume Logique (LV - Logical Volume)** : Un volume "virtuel" créé à partir de l'espace d'un VG (lvcreate), sur lequel on peut mettre un système de fichiers.

#### Flexibilité LVM

- Redimensionnement facile, extension sur plusieurs disques.

#### Migration LVM (pvmove)

- Permet de déplacer les données d'un PV vers un autre PV au sein du même VG, pendant que le système tourne.

### Exemple

- Remplacer un disque défaillant (/dev/sdc) par un nouveau (/dev/sdf) dans VG_DATA.
- Ajouter le nouveau disque comme PV au VG (pvcreate /dev/sdf, vgextend VG_DATA /dev/sdf).
- Interdire les nouvelles allocations sur l'ancien disque (pvchange -x n /dev/sdc).
- Migrer les données (pvmove /dev/sdc).
- Retirer l'ancien disque du VG (vgreduce VG_DATA /dev/sdc).

### Application au Scénario (inspiré de debootstick)

- La clé USB pirate contient l'OS (ou au moins la couche upperdir de l'overlay) sur un LV, lui-même dans un VG qui utilise initialement un PV situé sur une partition de la clé USB (${ORIGIN}3).
- L'initramfs modifié (ou un script lancé plus tard) prépare une partition sur le disque interne comme PV (pvcreate ${TARGET}3).
- Il ajoute ce PV au VG existant (vgextend $LVM_VG ${TARGET}3).
- Il lance la migration des données du PV de la clé USB vers le PV du disque interne (pvmove ${ORIGIN}3). C'est cette opération qui correspondrait à la barre de progression.
- Une fois la migration terminée, le PV de la clé USB peut être retiré du VG (vgreduce $LVM_VG ${ORIGIN}3) et la clé peut être physiquement enlevée. Le système continue de tourner en utilisant les données maintenant sur le disque interne.

### Déroulé Final du Piratage

1. La clé USB boote, l'initramfs modifié est chargé.
2. Il monte le partage NFS de l'OS original.
3. Il crée un répertoire hacks (initialement sur la clé USB).
4. Il monte un overlayfs combinant le NFS (lowerdir) et hacks (upperdir).
5. Il configure LVM pour que le volume logique contenant hacks (ou l'ensemble du système modifié) utilise un PV sur la clé USB.
6. Il lance en arrière-plan le processus de migration LVM (pvmove) vers un PV préparé sur le disque interne.
7. Il effectue exec chroot sur le point de montage de l'overlayfs pour démarrer le système "hacké".
8. Le bureau s'affiche, l'utilisateur travaille. Pendant ce temps, pvmove copie les données (barre de progression).
9. Une fois pvmove terminé, l'initramfs (ou le script de migration) finalise en retirant le PV de la clé USB du VG.
10. McGee retire physiquement la clé. Le système continue sur les données migrées sur le disque interne. L'initramfs modifié et la configuration LVM doivent aussi être rendus persistants sur le disque interne pour survivre au prochain redémarrage.

---

## Conclusion du Texte

Le scénario, bien que complexe et nécessitant des conditions précises (NFS, LVM pré-existant ou mis en place, initramfs modifiable), n'est pas techniquement "impossible", mais plutôt "invraisemblable" dans un contexte non préparé. L'analyse détaillée montre comment des technologies Linux existantes pourraient être combinées pour atteindre le résultat vu dans la série.
