# Compte rendu TP 1 : Installation de Debian 

utilisateur : eleve
Mot de passe : azerty
Nom de la machine : master-tp
Nom de domaine : esiee.lan

On a allumé la machine, on a démarré le BIOS à qui on a dit de démarrer sur le lecteur CD. 
Quand le CD a démarré, on a booté le noyau du CD. 
On a choisi la configuration DHCP.
Le périphérique en mode bloc s'appelle /dev/sda (accès brut au périphérique).
Pour le moment, on ne peut rien faire du périph en mode bloc. La solution est d'avoir des ACLs étendus qui vont permettre d'agir sur le périph (ex: droits).

Ensuite, on a créé la table des partitions, et initialisé le mbr(master boot record). Cela définit à quelle adresse et à quel moment, quel partition intervient.
On a créé nos récipients via le partitionnement suivant :
Disque dur partitionné en 3 :
  1) Disque / de 28 GB
  2) Disque swap de 2.2 GB
  3) Disque /tmp de 2 GB

On a configuré après, un gestionnaire de paquets.
Le périphérique a ensuite installé le base system (présent sur le CD) qui est l'ensemble des packages de base nécessaires.
Une fois cette partie terminée, vient la configuration du miroir (serveur internet sur lequel sont présents tous les paquets du gestionnaire Debian).

apt get update : télécharge sur le miroir tous les emplacements de tous les packages disponibles
apt get install : va chercher le paquet et l'installe 
apt get upgrade : regarde les packages installés (sauf le noyau) et met à jour 
apt get distupgrade : met tout à jour, y compris le noyau

On a configuré le système de base : SSH et les utilitaires de base
Ce qui nous a amené à configurer GRUB. On a choisi /dev/sda et GRUB s'est installé sur le mbr (master boot record).
Pour finir, on a rebooté.

Il manque system d (sert à démarrer les services).
Quand notre système démarre, ça suit le processus suivant : GRUB → kernel + initramfs → system d
