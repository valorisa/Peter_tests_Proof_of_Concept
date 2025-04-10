# Projet

Le texte fourni relate une conversation entre le narrateur et son ami Peter, un passionné de tests en tout genre. La discussion se concentre sur la faisabilité technique d'une scène de piratage informatique tirée d'une série américaine (probablement NCIS, mentionnant les personnages Gibbs, McGee, DiNozzo). Dans cette scène, l'agent McGee compromet l'ordinateur de l'agent Devon (soupçonné d'être une taupe) en le redémarrant sur une clé USB spécialement préparée, puis en retirant la clé "à chaud" une fois une barre de progression terminée, sans que Devon ne remarque de changement sur son système par la suite.

Peter et le narrateur décortiquent les aspects techniques qui pourraient rendre ce scénario, bien qu'invraisemblable à première vue, potentiellement réalisable sous certaines conditions spécifiques. Voici les points techniques clés abordés :

Le Problème Initial :

Réinstaller un OS sans que l'utilisateur ne s'en rende compte semble impossible.

Un redémarrage standard après installation n'a pas lieu dans la scène ; la clé est retirée et le système continue de tourner.

Hypothèse 1 : Système sur NFS (Network File System)

Peter suggère que l'OS de Devon pourrait être hébergé sur un serveur NFS centralisé, typique des grandes organisations.

Dans ce cas, le démarrage temporaire sur une clé USB locale ne modifierait pas les données du système d'exploitation distant.

Le piratage ne consisterait pas à modifier l'OS sur le serveur (trop sécurisé), mais à altérer la manière dont le poste local accède à cet OS distant.

Le Processus de Démarrage GNU/Linux :

Classique : Firmware (BIOS/UEFI) -> Bootloader (GRUB) -> Noyau Linux -> Montage racine -> Système d'init (PID 1).

Complexité : Gérer des configurations complexes (RAID logiciel, Btrfs, LVM, chiffrement, NFS) directement depuis le noyau initial ou le bootloader est difficile.

Solution : Initramfs (Initial RAM File System)

Une archive CPIO compressée (initrd.img-<version>) chargée en RAM par le bootloader.

Contient un mini-système de fichiers avec les outils et modules noyau nécessaires (ex: btrfs.ko, nfs, lvm) pour monter le vrai système de fichiers racine.

Le noyau exécute le script /init de l'initramfs.

Ce script prépare l'environnement (charge les modules, assemble le RAID/LVM, monte le NFS, etc.).

L'initramfs est souvent généré/mis à jour automatiquement (update-initramfs) pour s'adapter à la configuration système.

Passage de l'Initramfs au Système Final :

Une fois le système de fichiers racine final monté (par exemple, le partage NFS dans mnt/os-devon), le script /init de l'initramfs doit lancer le système d'init final (ex: /sbin/init qui est souvent un lien vers systemd).

Problème : Le système final s'attend à être à la racine (/), mais il est actuellement monté dans un sous-répertoire de l'initramfs.

Solution : chroot (Change Root)

chroot /chemin/vers/nouvelle/racine /chemin/vers/commande change le répertoire racine perçu par le processus lancé et ses descendants.

Solution : exec

exec commande remplace le processus courant (le script initramfs) par la commande spécifiée.

Pourquoi ?

Économie de ressources : Pas besoin de garder le processus initramfs parent en attente.

Crucial : Conserver le PID 1. Le noyau lance le premier /init (celui de l'initramfs) avec le PID 1. Le vrai système d'init s'attend aussi à avoir le PID 1 pour fonctionner correctement (il utilise son PID pour distinguer s'il doit démarrer le système ou contrôler une instance existante, comme avec init 6). exec permet cette transition de PID 1.

La commande finale est donc typiquement : exec chroot ./sbin/init (après un cd dans la nouvelle racine).

Hypothèse 2 : Introduction de Hacks via OverlayFS

Pour pirater le système NFS sans le modifier directement, McGee pourrait modifier l'initramfs de Devon.

L'initramfs modifié monterait le NFS (couche inférieure), puis monterait par-dessus un système de fichiers overlay (overlayfs).

OverlayFS : Permet de fusionner plusieurs répertoires (couches) en un seul point de montage virtuel.

mount -t overlay -o lowerdir=...,upperdir=...,workdir=... none /point/de/montage

lowerdir : Le système de base (lecture seule ou lecture/écriture). Ici, le NFS monté.

upperdir : La couche supérieure où les modifications sont écrites. Ici, un répertoire contenant les "hacks" (programmes modifiés, scripts espions), potentiellement stocké sur la clé USB initialement.

workdir : Un répertoire de travail vide sur le même système de fichiers que upperdir.

Les fichiers de upperdir masquent ceux de lowerdir en cas de conflit. Les écritures/suppressions affectent upperdir.

L'initramfs modifié ferait ensuite exec chroot sur le point de montage de l'overlay, et non directement sur le NFS. L'OS démarré verrait donc la version "hackée".

Origine d'OverlayFS : Utilisé historiquement pour les Live CDs (CD lecture seule + RAMDisk lecture/écriture) et toujours pour les Live USB (SquashFS compressé lecture seule + partition lecture/écriture).

Hypothèse 3 : Migration "à Chaud" avec LVM (Logical Volume Management)

Problème : Si upperdir (les hacks et les nouvelles données écrites par l'utilisateur) est sur la clé USB, retirer la clé planterait le système. La barre de progression tardive suggère une copie de données après le démarrage du bureau.

Solution : LVM

Concepts LVM :

Volume Physique (PV - Physical Volume) : Un disque ou une partition préparé(e) pour LVM (pvcreate).

Groupe de Volumes (VG - Volume Group) : Un pool de un ou plusieurs PVs (vgcreate, vgextend).

Volume Logique (LV - Logical Volume) : Un volume "virtuel" créé à partir de l'espace d'un VG (lvcreate), sur lequel on peut mettre un système de fichiers.

Flexibilité LVM : Redimensionnement facile, extension sur plusieurs disques.

Migration LVM (pvmove) : Permet de déplacer les données d'un PV vers un autre PV au sein du même VG, pendant que le système tourne.

Exemple : Remplacer un disque défaillant (/dev/sdc) par un nouveau (/dev/sdf) dans VG_DATA.

Ajouter le nouveau disque comme PV au VG (pvcreate /dev/sdf, vgextend VG_DATA /dev/sdf).

Interdire les nouvelles allocations sur l'ancien disque (pvchange -x n /dev/sdc).

Migrer les données (pvmove /dev/sdc).

Retirer l'ancien disque du VG (vgreduce VG_DATA /dev/sdc).

Application au Scénario (inspiré de debootstick) :

La clé USB pirate contient l'OS (ou au moins la couche upperdir de l'overlay) sur un LV, lui-même dans un VG qui utilise initialement un PV situé sur une partition de la clé USB (${ORIGIN}3).

L'initramfs modifié (ou un script lancé plus tard) prépare une partition sur le disque interne comme PV (pvcreate ${TARGET}3).

Il ajoute ce PV au VG existant (vgextend $LVM_VG ${TARGET}3).

Il lance la migration des données du PV de la clé USB vers le PV du disque interne (pvmove ${ORIGIN}3). C'est cette opération qui correspondrait à la barre de progression.

Une fois la migration terminée, le PV de la clé USB peut être retiré du VG (vgreduce $LVM_VG ${ORIGIN}3) et la clé peut être physiquement enlevée. Le système continue de tourner en utilisant les données maintenant sur le disque interne.

Déroulé Final du Piratage :

La clé USB boote, l'initramfs modifié est chargé.

Il monte le partage NFS de l'OS original.

Il crée un répertoire hacks (initialement sur la clé USB).

Il monte un overlayfs combinant le NFS (lowerdir) et hacks (upperdir).

Il configure LVM pour que le volume logique contenant hacks (ou l'ensemble du système modifié) utilise un PV sur la clé USB.

Il lance en arrière-plan le processus de migration LVM (pvmove) vers un PV préparé sur le disque interne.

Il effectue exec chroot sur le point de montage de l'overlayfs pour démarrer le système "hacké".

Le bureau s'affiche, l'utilisateur travaille. Pendant ce temps, pvmove copie les données (barre de progression).

Une fois pvmove terminé, l'initramfs (ou le script de migration) finalise en retirant le PV de la clé USB du VG.

McGee retire physiquement la clé. Le système continue sur les données migrées sur le disque interne. L'initramfs modifié et la configuration LVM doivent aussi être rendus persistants sur le disque interne pour survivre au prochain redémarrage.

Conclusion du Texte : Le scénario, bien que complexe et nécessitant des conditions précises (NFS, LVM pré-existant ou mis en place, initramfs modifiable), n'est pas techniquement "impossible", mais plutôt "invraisemblable" dans un contexte non préparé. L'analyse détaillée montre comment des technologies Linux existantes pourraient être combinées pour atteindre le résultat vu dans la série.
