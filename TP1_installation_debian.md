# Compte rendu TP1 : Installation de Debian

**Utilisateur :** eleve | **Mot de passe :** azerty | **Nom de la machine :** master-tp | **Nom de domaine :** esiee.lan

Le but du TP est d'installer Debian nous-mêmes sur une machine en comprenant ce que fait chaque étape de l'installation. Le choix d'un OS dépend de l'usage qu'on veut faire de la machine, ici c'est Debian.

## 1. Démarrage sur l'ISO

On a allumé la machine et on a démarré le BIOS. C'est lui qui décide sur quoi la machine démarre : un CD, un disque dur, une clé USB ou même le réseau. On lui a dit de démarrer sur le lecteur CD.

Dans le lecteur, on a mis le fichier `.iso` qu'on avait téléchargé. Un `.iso` est une copie de tout ce qu'on trouve sur un CD, donc on peut s'en servir à la place d'un vrai CD. Quand le CD a démarré, on a booté le noyau **du CD** (pas encore celui de notre disque).

Dans l'iso, on retrouve Debian avec des aides pour l'installation. Tout ce que ces aides font, on aurait pu le faire à la main, mais là un menu nous assiste.

## 2. Configuration de départ

- **L'heure** : c'est important pour que les logs soient datés correctement.
- **Le réseau** : on a choisi la configuration **DHCP**, c'est-à-dire que la machine demande son adresse IP à un serveur, on n'a pas eu à la rentrer à la main.

## 3. Le disque et le partitionnement

### Le disque brut

Le disque s'appelle `/dev/sda`. Les fichiers de `/dev/` représentent les périphériques, et ici le disque est vu **en mode bloc** : un accès brut, on lit et on écrit des blocs directement. À ce stade on ne peut rien en faire : il n'y a pas d'« intelligence », donc pas de métadonnées (noms de fichiers, dates...), et donc pas de droits ni d'ACL. Pour que le disque soit utilisable, il faut le structurer.

### Table des partitions et MBR

On a commencé par créer la **table des partitions**, ce qui a initialisé le **MBR** (Master Boot Record). La table dit à quelle adresse chaque partition démarre et où elle finit.

Le MBR fait 512 octets, tout au début du disque :

- 446 octets pour le code d'amorçage (c'est là que GRUB s'installe à la fin du TP) ;
- 64 octets pour la table des partitions ;
- 2 octets de signature de fin (`0x55AA`).

### Nos partitions

On a découpé le disque en 3 partitions :

| Partition | Taille |
|---|---|
| `/` (racine) | 28 GB |
| swap | 2.2 GB |
| `/tmp` | 2 GB |

- **`/`** : la partition principale, elle contient tout le reste (programmes, configuration, fichiers des utilisateurs). Tous les dossiers partent de là, donc c'est elle qui a le plus d'espace.
- **swap** : de l'espace disque utilisé comme extension de la RAM. Quand la RAM est pleine, le système y déplace des données qui ne servent pas tout de suite. On ne la consulte pas nous-mêmes, d'où sa petite taille.
- **`/tmp`** : l'endroit où les programmes mettent leurs fichiers temporaires. En la mettant sur une partition à part, on la limite à 2 GB : si un programme la remplit, ça ne bloque pas le reste du système qui est sur `/`.

### Pourquoi un disque de 1 To affiche 931 Go ?

Une question revenue en cours. Le fabricant compte en décimal (1 To = 10¹² octets) alors que l'ordinateur compte en puissances de 2 : 1 kio = 2¹⁰ = 1024 octets (le « i » vient de *binary*). Donc 1 Gio = 1024³ = 1 073 741 824 octets, et :

```
1 000 000 000 000 / 1 073 741 824 = 931,32 Gio
```

L'écart grandit avec la capacité :

| Décimal | Binaire |
|---|---|
| 1 ko = 1000 octets | 1 kio = 1024 octets |
| 1 Mo = 1 000 000 octets | 1 Mio = 1 048 576 octets |
| 1 Go = 1 000 000 000 octets | 1 Gio = 1 073 741 824 octets |
| 1 To = 10¹² octets | 1 Tio = 1 099 511 627 776 octets |

## 4. Le système de base et le miroir

Pendant l'installation, on n'a besoin d'aucun autre support que le CD, parce qu'il contient le **base system** : l'ensemble des paquets de base, donc tous les outils pour gérer le système. L'installateur a d'abord installé le base system depuis le CD.

Ensuite vient la configuration du **miroir** : un serveur sur internet sur lequel sont présents tous les paquets de Debian. On en a choisi un et on a configuré `apt` pour aller chercher dessus. Les commandes à connaître :

| Commande | Ce qu'elle fait |
|---|---|
| `apt-get update` | télécharge depuis le miroir la liste de tous les paquets disponibles (ne met rien à jour, juste la liste) |
| `apt-get install` | va chercher un paquet et l'installe |
| `apt-get upgrade` | met à jour les paquets installés, **sauf le noyau** |
| `apt-get dist-upgrade` | met tout à jour, **y compris le noyau** |

Une fois le miroir choisi, on a configuré le système de base : SSH (connexion à distance) et les utilitaires de base.

## 5. GRUB et redémarrage

Ça nous a amenés à configurer **GRUB**, le bootloader (le programme qui lance le système au démarrage). On a choisi `/dev/sda` et GRUB s'est installé dans le MBR, dans les 446 octets d'amorçage vus plus haut. Comme c'est très petit, ce premier bout de code sert surtout à charger la suite de GRUB ailleurs sur le disque (un petit programme qui en lance un plus gros). Pour finir, on a rebooté.

## 6. Démarrage du système

Il manque encore **systemd**, qui sert à démarrer les services. Au démarrage, le processus est :

```
GRUB → noyau + initramfs → systemd
```
