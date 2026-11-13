# Compte rendu de TP : compiler un noyau Linux dans une VM avec un disque de build séparé

Ce TP s'est déroulé en trois séances. L'objectif est de compiler un noyau Linux 6.17 dans une VM Debian 13 sous VirtualBox, en mettant tout ce qui concerne la compilation sur un deuxième disque monté sur `/build`, pour ne pas remplir le disque système. La première séance a porté sur l'installation de Debian, la deuxième sur la préparation de la VM (étapes 1 à 5 du sujet) et la troisième sur la configuration, la compilation, l'installation du noyau et la vérification au redémarrage (étapes 5 à 8).

## 1. Préparer la machine virtuelle

### 1.1 Installation de Debian

Pour cette installation : utilisateur eleve, mot de passe azerty, nom de la machine master-tp et nom de domaine esiee.lan.

Le but de cette première séance est d'installer Debian nous-mêmes en comprenant ce que fait chaque étape de l'installation. Le choix d'un OS dépend de l'usage qu'on veut faire de la machine, ici c'est Debian.

#### Démarrage sur l'ISO

On a allumé la machine et démarré le BIOS. C'est lui qui décide sur quoi la machine démarre : un CD, un disque dur, une clé USB ou même le réseau. On lui a dit de démarrer sur le lecteur CD.

Dans le lecteur, on a mis le fichier `.iso` téléchargé. Un `.iso` est une copie de tout ce qu'on trouve sur un CD, donc on peut s'en servir à la place d'un vrai CD. Quand le CD a démarré, on a booté le noyau du CD (pas encore celui de notre disque).

Dans l'ISO, on retrouve Debian avec des aides pour l'installation. Tout ce que ces aides font, on aurait pu le faire à la main, mais là un menu nous assiste.

#### Configuration de départ

- L'heure : c'est important pour que les logs soient datés correctement.
- Le réseau : on a choisi la configuration DHCP, c'est-à-dire que la machine demande son adresse IP à un serveur, on n'a pas eu à la rentrer à la main.

#### Le disque brut

Le disque s'appelle `/dev/sda`. Les fichiers de `/dev/` représentent les périphériques, et ici le disque est vu en mode bloc : un accès brut, on lit et on écrit des blocs directement. À ce stade on ne peut rien en faire : il n'y a pas d'« intelligence », donc pas de métadonnées (noms de fichiers, dates...), et donc pas de droits ni d'ACL. Pour que le disque soit utilisable, il faut le structurer.

#### Table des partitions et MBR

On a commencé par créer la table des partitions, ce qui a initialisé le MBR (Master Boot Record). La table dit à quelle adresse chaque partition démarre et où elle finit.

Le MBR fait 512 octets, tout au début du disque :

- 446 octets pour le code d'amorçage (c'est là que GRUB s'installe à la fin de l'installation) ;
- 64 octets pour la table des partitions ;
- 2 octets de signature de fin (`0x55AA`).

#### Nos partitions

On a découpé le disque en 3 partitions : `/` (la racine) de 28 GB, une swap de 2.2 GB et `/tmp` de 2 GB.

- `/` : la partition principale, elle contient tout le reste (programmes, configuration, fichiers des utilisateurs). Tous les dossiers partent de là, donc c'est elle qui a le plus d'espace.
- swap : de l'espace disque utilisé comme extension de la RAM. Quand la RAM est pleine, le système y déplace des données qui ne servent pas tout de suite. On ne la consulte pas nous-mêmes, d'où sa petite taille.
- `/tmp` : l'endroit où les programmes mettent leurs fichiers temporaires. En la mettant sur une partition à part, on la limite à 2 GB : si un programme la remplit, ça ne bloque pas le reste du système qui est sur `/`.

#### Pourquoi un disque de 1 To affiche 931 Go ?

C'est une question qui est revenue en cours. Le fabricant compte en décimal (1 To = 10¹² octets) alors que l'ordinateur compte en puissances de 2 : 1 kio = 2¹⁰ = 1024 octets (le « i » vient de *binary*). Donc 1 Gio = 1024³ = 1 073 741 824 octets, et :

```
1 000 000 000 000 / 1 073 741 824 = 931,32 Gio
```

L'écart grandit avec la capacité : 1 ko vaut 1000 octets alors que 1 kio en vaut 1024, 1 Mo vaut 1 000 000 octets contre 1 048 576 pour 1 Mio, 1 Go vaut 1 000 000 000 octets contre 1 073 741 824 pour 1 Gio, et 1 To vaut 10¹² octets contre 1 099 511 627 776 pour 1 Tio.

#### Le système de base et le miroir

Pendant l'installation, on n'a besoin d'aucun autre support que le CD, parce qu'il contient le base system : l'ensemble des paquets de base, donc tous les outils pour gérer le système. L'installateur a d'abord installé le base system depuis le CD.

Ensuite vient la configuration du miroir : un serveur sur Internet sur lequel sont présents tous les paquets de Debian. On en a choisi un et on a configuré `apt` pour aller chercher dessus. Les commandes à connaître :

- `apt-get update` télécharge depuis le miroir la liste de tous les paquets disponibles (il ne met rien à jour, c'est juste la liste) ;
- `apt-get install` va chercher un paquet et l'installe ;
- `apt-get upgrade` met à jour les paquets installés, sauf le noyau ;
- `apt-get dist-upgrade` met tout à jour, y compris le noyau.

Une fois le miroir choisi, on a configuré le système de base : SSH (connexion à distance) et les utilitaires de base.

#### GRUB et redémarrage

On a ensuite configuré GRUB, le bootloader (le programme qui lance le système au démarrage). On a choisi `/dev/sda` et GRUB s'est installé dans le MBR, dans les 446 octets d'amorçage vus plus haut. Comme c'est très petit, ce premier bout de code sert surtout à charger la suite de GRUB ailleurs sur le disque (un petit programme qui en lance un plus gros). Pour finir, on a rebooté.

#### Démarrage du système

Il manque encore systemd, qui sert à démarrer les services. Au démarrage, le processus est :

```
GRUB → noyau + initramfs → systemd
```

### 1.2 Importer la VM et se connecter en SSH

Pour la suite du TP, on a importé `Debian13-base.ova` (environ 970 Mo, fournie par le prof) dans VirtualBox.

La VM est en réseau NAT : elle n'est pas joignable directement depuis notre machine. Pour s'y connecter en SSH, on ajoute une règle de redirection de port (Configuration > Réseau > Avancé > Redirection de ports), du port hôte 2424 vers le port invité 22.

Ça veut dire : quand un paquet arrive sur le port 2424 de notre machine, il est redirigé vers le port 22 (SSH) de la VM. Ensuite, depuis un terminal de notre machine (PowerShell sous Windows, pas dans la fenêtre de la VM) :

```
ssh -p 2424 user@127.0.0.1
```

`127.0.0.1` c'est notre propre machine, et c'est la redirection qui envoie la connexion vers la VM.

On a eu un petit souci au début : on avait noté `eleve` comme nom d'utilisateur (celui de la première séance), mais sur cette VM le compte s'appelle `user` (vérifié avec `whoami` dans la fenêtre de la VM). Le mot de passe est `azerty`.

### 1.3 Ajouter un deuxième disque

Il faut ajouter le disque VM éteinte : on ne peut pas brancher un disque à chaud dans une machine en exécution.

Dans Configuration > Stockage, sur le contrôleur SATA, on a fait *Add Hard Disk* puis *Create* : format VDI, allocation dynamique, 60 Go. Le fichier `.vdi` représente le disque dur, donc la VM a maintenant 2 disques. Pour l'instant il est inutilisable : il faut démarrer la VM, puis le partitionner et le formater.

## 2. Préparer le disque supplémentaire

### 2.1 Repérer le bon disque

C'est l'étape où il ne faut pas se tromper, parce qu'on va partitionner et formater un disque.

```
user@debian:~$ lsblk -o NAME,SIZE,TYPE,MOUNTPOINT
NAME    SIZE TYPE MOUNTPOINT
sda      30G disk
├─sda1 27.9G part /
└─sda2  2.1G part [SWAP]
sdb      60G disk
sr0    1024M rom
```

Le disque système (avec `/` et `[SWAP]`) est `sda`, et le nouveau disque (60G, sans partition ni point de montage) est `sdb`. Le sujet du TP écrit `/dev/sdb` en dur, mais l'ordre des noms dépend de la VM : pendant le cours, sur la VM du prof, c'était l'inverse. Recopier les commandes sans regarder aurait pu formater le disque système. La règle suivie : le disque à formater est celui sans partition et sans point de montage.

### 2.2 Partitionner, formater, monter

On installe `parted`, on crée une table de partitions GPT, puis une partition qui commence à 1 MiB et prend tout le disque :

```
sudo apt-get install -y parted
sudo parted /dev/sdb --script mklabel gpt
sudo parted /dev/sdb --script mkpart build ext4 1MiB 100%
```

Résultat :

```
NAME    SIZE TYPE MOUNTPOINT
sda      30G disk
├─sda1 27.9G part /
└─sda2  2.1G part [SWAP]
sdb      60G disk
└─sdb1   60G part
sr0    1024M rom
```

Ensuite on crée le système de fichiers ext4 (avec le label `KBUILD`), on crée le dossier `/build` et on y monte la partition (le montage raccroche un chemin à la partition) :

```
sudo mkfs.ext4 -L KBUILD /dev/sdb1
sudo mkdir /build
sudo mount /dev/sdb1 /build
df -h /build
```

```
Filesystem      Size  Used Avail Use% Mounted on
/dev/sdb1        59G  2.1M   56G   1% /build
```

On voit bien le point de montage. On a fait à la main ce que fait l'installateur lors de la première séance.

### 2.3 Montage automatique avec /etc/fstab

Pour que `/build` soit monté à chaque démarrage, il faut l'ajouter dans `/etc/fstab` (qui dit quoi monter et où). On récupère l'UUID de la partition :

```
user@debian:~$ sudo blkid /dev/sdb1
/dev/sdb1: LABEL="KBUILD" UUID="86a4816f-e217-4ff0-b368-bd0493fb120c" BLOCK_SIZE="4096" TYPE="ext4" PARTLABEL="build" PARTUUID="b7b7b48b-d8ff-4d35-9719-43f308fce36f"
```

On utilise l'UUID plutôt que `/dev/sdb1` parce que les noms `sda`/`sdb` peuvent changer si on déplace ou ajoute des disques, alors que l'UUID est donné à la création de la partition et ne bouge plus. Ligne ajoutée :

```
echo 'UUID=86a4816f-e217-4ff0-b368-bd0493fb120c  /build  ext4  defaults,noatime  0  2' | sudo tee -a /etc/fstab
sudo systemctl daemon-reload
sudo mount -a
sudo chown -R $USER:$USER /build
```

Le `chown` sert à ce que la compilation se fasse en simple utilisateur, pas en root. Après un `sudo reboot`, `/build` était bien remonté tout seul :

```
user@debian:~$ df -h /build
Filesystem      Size  Used Avail Use% Mounted on
/dev/sdb1        59G  2.1M   56G   1% /build
```

Au premier redémarrage, on a eu un problème : la VM est restée bloquée sur un écran iPXE (démarrage par le réseau), ce qui veut dire qu'elle n'avait pas trouvé de disque sur lequel démarrer. L'état de la machine avait été enregistré, donc elle reprenait au même endroit. On a éteint la VM, vérifié que le disque système était bien sur le port SATA 0 et le nouveau sur le port 1, mis Disque dur en premier dans l'ordre d'amorçage (Système > Carte mère), puis relancé. Ça a démarré normalement.

## 3. Mettre à jour le système et installer les prérequis

La commande d'installation des outils de compilation a d'abord échoué avec des erreurs 404 :

```
Err:5 http://debian.obspm.fr/debian trixie/main amd64 rsync amd64 3.4.1+ds1-5
  404  Not Found [IP: 145.238.186.56 80]
```

La liste des paquets de la VM était périmée : les versions qu'elle connaissait n'existaient plus sur les serveurs. Il fallait d'abord mettre à jour (`update` puis `upgrade`, vus lors de la première séance) :

```
sudo apt-get update
sudo apt-get -y upgrade
```

Pendant l'upgrade, un écran bleu de configuration de GRUB propose une liste de disques. On a coché `/dev/sda` : GRUB s'installe dans le MBR du disque entier du système (pas dans la partition `sda1`), et surtout pas `sdb` qui est le disque de build.

Ensuite, l'installation des prérequis est passée (73 nouveaux paquets, sans erreur) :

```
sudo apt-get -y install bc bison flex libelf-dev libssl-dev libncurses-dev dwarves fakeroot rsync ccache build-essential
```

Puis on active `ccache` (un cache de compilation) en le mettant en tête du `PATH` :

```
echo 'export PATH="/usr/lib/ccache:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

```
user@debian:/build$ echo $PATH
/usr/lib/ccache:/usr/local/bin:/usr/bin:/bin:/usr/local/games:/usr/games
```

## 4. Récupérer les sources du noyau

### 4.1 L'archive linux-build.tar.gz du prof

Compiler tout le noyau prendrait plusieurs heures, trop pour la séance. Le prof nous a donc distribué `linux-build.tar.gz` (environ 7,5 Go), qui contient un dossier de build déjà compilé.

L'archive vient de Linux, donc on ne l'a pas extraite sous Windows (pas de droits ni de liens symboliques Linux). On l'a copiée dans la VM avec `scp`, qui copie des fichiers en s'appuyant sur SSH (`scp source destination`). Depuis PowerShell, dans le dossier de l'archive :

```
scp -P 2424 linux-build.tar.gz user@127.0.0.1:/build/
```

(`-P` en majuscule pour `scp`, alors que `ssh` utilise `-p`.) Dans la VM, on l'a extraite dans `/build`, puis on a supprimé l'archive pour récupérer 7 Go :

```
cd /build
tar -xzf linux-build.tar.gz
rm /build/linux-build.tar.gz
```

Vérification (`tar -tzf` avait montré que tous les chemins commencent par `linux-build/`, donc on obtient bien `/build/linux-build`) :

```
user@debian:/build$ ls /build/linux-build | head
arch
block
built-in.a
certs
crypto
drivers
fs
include
init
io_uring
user@debian:/build$ du -sh /build/linux-build
24G     /build/linux-build
```

### 4.2 Télécharger les sources du noyau

```
mkdir -p ~/src && cd ~/src
wget https://cdn.kernel.org/pub/linux/kernel/v6.x/linux-6.17.tar.xz
tar -xf linux-6.17.tar.xz
cd linux-6.17
```

```
user@debian:~/src/linux-6.17$ ls | head
arch
block
certs
COPYING
CREDITS
crypto
Documentation
drivers
fs
include
user@debian:~/src/linux-6.17$ pwd
/home/user/src/linux-6.17
```

Les sources sont dans `~/src` et les fichiers de compilation dans `/build/linux-build` : c'est la séparation voulue par le TP.

## 5. Configurer le noyau

### 5.1 Préparation et reprise de session

```
mkdir -p /build/linux-build /build/tmp
export TMPDIR=/build/tmp
make O=/build/linux-build menuconfig
```

- `O=...` : tous les fichiers générés vont dans ce dossier, ça évite de polluer le dossier des sources.
- `TMPDIR` : les fichiers temporaires vont aussi sur le disque de build. Cette variable ne dure que le temps de la session, donc il faut la refaire à chaque nouvelle connexion SSH.
- `menuconfig` : une interface dans le terminal pour choisir ce qu'on compile.

Au début de la troisième séance, il a fallu relancer VirtualBox, démarrer la VM, puis se reconnecter depuis un terminal de notre machine et relancer la configuration :

```
ssh -p 2424 user@127.0.0.1
cd ~/src/linux-6.17
export TMPDIR=/build/tmp
make O=/build/linux-build menuconfig
```

### 5.2 Configuration avec menuconfig

`menuconfig` est une interface en ncurses où on choisit ce qu'on compile dans le noyau. Il y a environ 15 000 paramètres (par exemple la pile réseau : sockets Unix, TCP/IP, IPsec...). Avant `menuconfig`, il fallait éditer à la main le fichier `.config` option par option, ce qui était très long. Ici, le point de départ est le `.config` qui se trouve déjà dans `/build/linux-build`.

Le TP demande de vérifier quatre réglages sur les clés et signatures :

- *Additional X.509 keys for default system keyring* : vide ;
- *File name or PKCS#11 URI of module signing key* : vide ;
- *Automatically sign all modules* : décoché ;
- *Digital signature verification using multiple keyrings* : décoché.

Ils étaient déjà bons dans la config du prof, donc on n'a rien changé. Pour le label (*General setup > Local version*), le TP demande `-custom`, mais il était déjà à `e4cyb`. On l'a laissé tel quel : le label est cosmétique, mais le changer aurait forcé une recompilation d'une grande partie du build du prof. Le noyau s'appelle donc `6.17.0e4cyb` (et pas `6.17.0-custom`).

On a quitté `menuconfig` avec Échap sans enregistrer, puisque rien n'avait été modifié.

## 6. Compiler sur le disque séparé

On n'a pas compilé le noyau entièrement nous-mêmes. Le prof nous a donné un dossier `linux-build` déjà compilé (une compilation complète prend des heures). On a vérifié que la compilation démarre, mais on s'est appuyés sur le build fourni.

```
make O=/build/linux-build -j"$(nproc)"
```

Résultat :

```
user@debian:~/src/linux-6.17$ make O=/build/linux-build -j"$(nproc)"
make[1]: Entering directory '/build/linux-build'
  GEN     Makefile
  CALL    /home/user/src/linux-6.17/scripts/checksyscalls.sh
  DESCEND objtool
  INSTALL libsubcmd_headers
  DESCEND bpf/resolve_btfids
  INSTALL libsubcmd_headers
Kernel: arch/x86/boot/bzImage is ready  (#1)
make[1]: Leaving directory '/build/linux-build'
```

Ça s'est terminé presque tout de suite avec `bzImage is ready` : `make` ne recompile que ce qui a changé, et tout était déjà compilé. `-j"$(nproc)"` lance autant de compilations en parallèle que la VM a de cœurs.

## 7. Installer le noyau et les modules

### 7.1 Installation

Avant d'installer, on regarde la place sur le disque système (les modules s'installent sur `/`, pas sur `/build`) et le nom du noyau :

```
user@debian:~/src/linux-6.17$ df -h /
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda1        28G  3.5G   23G  14% /
user@debian:~/src/linux-6.17$ make O=/build/linux-build kernelrelease
make[1]: Entering directory '/build/linux-build'
6.17.0e4cyb
make[1]: Leaving directory '/build/linux-build'
```

Puis les deux commandes d'installation :

```
sudo make O=/build/linux-build modules_install
sudo make O=/build/linux-build install
```

`modules_install` copie les modules dans `/lib/modules/6.17.0e4cyb` (fin de la sortie, avec un `DEPMOD` qui génère les dépendances) :

```
  INSTALL /lib/modules/6.17.0e4cyb/kernel/virt/lib/irqbypass.ko
  XZ      /lib/modules/6.17.0e4cyb/kernel/virt/lib/irqbypass.ko.xz
  DEPMOD  /lib/modules/6.17.0e4cyb
make[1]: Leaving directory '/build/linux-build'
```

`install` copie le noyau dans `/boot`, génère l'initramfs et met à jour GRUB. Pas besoin de lancer `update-grub` à la main : sur Debian, l'installation le fait automatiquement (le prof l'a dit aussi).

### 7.2 Ce que contient /boot

```
user@debian:~$ ls /boot
config-6.12.43+deb13-amd64  initrd.img-6.12.43+deb13-amd64  System.map-6.12.48+deb13-amd64  vmlinuz-6.17.0e4cyb
config-6.12.48+deb13-amd64  initrd.img-6.12.48+deb13-amd64  System.map-6.17.0e4cyb
config-6.17.0e4cyb          initrd.img-6.17.0e4cyb          vmlinuz-6.12.43+deb13-amd64
grub                        System.map-6.12.43+deb13-amd64  vmlinuz-6.12.48+deb13-amd64
user@debian:~$ ls /lib/modules
6.12.43+deb13-amd64  6.12.48+deb13-amd64  6.17.0e4cyb
```

Les noyaux `6.12` sont les anciens, qu'on garde : si le nouveau ne démarre pas, on peut choisir l'ancien dans GRUB. Pour chaque version du noyau, il y a 4 fichiers :

- `vmlinuz-...` : le noyau lui-même, un gros exécutable monolithique. C'est lui que GRUB charge (il ne faut surtout pas faire `cat` dessus, ça casse le terminal).
- `initrd.img-...` (initramfs) : une petite archive d'un système Linux minimal, chargée en RAM. Le noyau tout seul ne peut pas faire grand-chose, il lui faut des outils côté utilisateur pour monter la vraie partition racine (lire `fstab`, déchiffrer le disque, gérer LVM...). Une fois la racine montée, l'initramfs passe la main à systemd et disparaît.
- `System.map-...` : la table qui associe des adresses aux fonctions du noyau (dont les appels système).
- `config-...` : une copie du `.config` utilisé pour compiler.

Le prof nous a aussi montré le menu de GRUB : avec `e` sur une entrée, on voit sa configuration (les `insmod` pour lire la partition, la racine, la ligne `linux` pour le noyau et la ligne `initrd`). Donc GRUB charge noyau + initramfs, l'initramfs monte la racine, puis systemd démarre les services.

## 8. Redémarrer et vérifier

On redémarre depuis la VM :

```
sudo shutdown -r now
```

Au début, on a tapé par erreur cette commande dans PowerShell sous Windows, où `sudo` est désactivé. Elle n'a rien fait, c'est dans la session SSH vers la VM qu'il fallait la lancer.

Après reconnexion en SSH, la bannière affichait déjà le nouveau noyau :

```
Linux debian 6.17.0e4cyb #1 SMP PREEMPT_DYNAMIC Fri Oct 16 09:47:59 CEST 2026 x86_64
```

et la vérification finale :

```
user@debian:~$ uname -r
6.17.0e4cyb
user@debian:~$ df -h /build
Filesystem      Size  Used Avail Use% Mounted on
/dev/sdb1        59G   24G   33G  42% /build
```

La VM tourne bien sur le noyau installé (GRUB démarre par défaut sur le plus récent) et `/build` est remonté automatiquement grâce au `fstab`.

## 9. Paquets .deb (optionnel) : non réalisé

La section 9 du TP (fabriquer des paquets `.deb`) est marquée *optionnelle* et on ne l'a pas faite, le prof s'étant arrêté à « vous avez compilé votre premier noyau ».

## 10. Lien avec le cours sur l'isolation

Pendant la troisième séance, le prof a aussi expliqué les conteneurs :

- Un conteneur, c'est un processus à qui on donne des instances privées d'objets du noyau (les *namespaces* : pid, net, mnt, ipc, user, time, cgroup). Dans le noyau, la `task_struct` d'un processus a un pointeur `nsproxy` qui liste ses namespaces, et l'appel système `unshare` permet d'arrêter de partager un objet avec les autres.
- Les cgroups limitent la consommation de ressources. Namespaces + cgroups = un processus isolé, ce qui est la base de Docker (via `runc`).
- Docker conteneurise un service ; LXC conteneurise tout un espace utilisateur avec systemd. Ce n'est pas de la virtualisation : le noyau reste le même.
