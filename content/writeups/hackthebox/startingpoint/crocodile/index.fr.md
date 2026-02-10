---
title: "Crocodile"
date: 2026-02-08
description: "Quand des identifiants fuitent via un FTP mal configuré."
draft: false

# Tes taxonomies (correspondant à ton hugo.toml)
tags: ["Misconfiguration", "Passwords", "Web", "FTP", "Gobuster", "Tier 1"]
categories: ["Writeups", "Hack The Box", "Starting Point"]
---

# Introduction

Pour cette machine **Crocodile**, nous avons identifié une mauvaise configuration du service **FTP**. Celle-ci nous a permis de récupérer des identifiants pour nous connecter à une **page web** d'administration. Il s'agit d'un scénario classique où un service "oublié" donne les clés pour un autre.

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

## Reconaissance

### Scan NMAP

On commance par faire un scan nmap, directement avec le flag ``-sV`` pour avoir les versions des services, et ``-sC`` pour voir les scripts de détection par défaut.

````bash
sudo nmap -sV -sC 10.129.1.15
````

<img src="image1.png"/>

Le scan nous révèle deux ports ouverts :
- 21 (FTP) : vsftpd 3.0.3
- 80 (HTTP) : Apache httpd 2.4.41

Regardez bien la sortie du script Nmap sur le port 21 ! Il indique : "Anonymous FTP login allowed" ET on a acces a deux fichier interessants, "allowed.userlist" et "allowed.userlist.passwd"

C'est notre point d'entrée. Le protocole FTP (File Transfer Protocol) est fait pour transférer des fichiers, et ici, l'administrateur a laissé l'accès "invité" ouvert.

### Récupération des fichiers
Puisque la connexion anonyme est autorisée, connectons-nous au serveur pour récupérer les deux fichiers que l'on a vu.

````bash
ftp 10.129.1.15
````
<img src="image2.png"/>

Lorsque le prompt demande le nom d'utilisateur, on entre : ``anonymous`` . N'importe quel mot de passe (ou vide) fera l'affaire.

<img src="image3.png"/>

Le code ``230 Login successful`` confirme que nous sommes à l'intérieur.

Du coup, il ne reste plus qu'a faire ``dir`` pour afficher les fichier disponible et confirmer le scan ``nmap``, puis on peut utiliser la commande get pour récuperer les fichiers 

<img src="image4.png"/>

### Analyse des fichiers
De retour sur notre terminal local, affichons le contenu de ces fichiers avec ``cat``.

<img src="image5.png"/>

On y trouve une liste d'utilisateurs (admin, aron, pwnmeow, etc.) et une liste de mots de passe. C'est une mine d'or !

### Analyse page web
Maintenant que nous avons des identifiants potentiels, que faire ? Le FTP ne semble servir qu'au stockage. Allons voir ce qui se passe sur le port 80 (le site web).

En entrant l'IP dans le navigateur, on tombe sur une page basic pour un business.

<img src="image6.png"/>

Un coup d'œil avec l'extension **Wappalyzer** nous indique qu'il s'agit d'une stack PHP classique, mais rien de très vulnérable à première vue sur la page d'accueil.

### Fuzzing des répertoire avec Gobuster
Puisque nous avons des identifiants, nous cherchons probablement une page de connexion. Pour la trouver, on utilise ``Gobuster`` pour scanner l'arborescence du site.

````bash
gobuster dir --url http://10.129.1.15/ --wordlist /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt -x php,html
````
- ``dir`` : Mode énumération de répertoires.
- ``-x php,html`` : On demande à chercher spécifiquement les fichiers avec ces extensions. 

<img src="image7.png"/>

Yes ! Gobuster trouve une page intéressante : ``login.php``.

--- 

## Exploitation
Rendons-nous sur http://10.129.1.15/login.php. Nous sommes face à un formulaire d'authentification.

<img src="image8.png"/>

Nous avons récupéré une liste d'utilisateurs et de mots de passe via le FTP. La liste est courte, nous pouvons tenter une approche manuelle. Essayons le compte le plus évident qui est **admin**.

On teste l'utilisateur **admin** avec le premier mot de passe de la liste (ou le mot de passe qui semble correspondre dans le fichier passwd).

<img src="image9.png"/>

La connexion fonctionne ! Nous sommes redirigés vers le tableau de bord d'administration. Le flag est affiché bien en évidence sur l'écran.

La machine est ***pwned*** !


