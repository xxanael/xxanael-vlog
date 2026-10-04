---
title: "Hack The Box — Orion : Writeup"
date: 2026-10-04 00:00:00 +0200
categories: [Box, HackTheBox]
tags: [web, craftcms, content-discovery, metasploit, rce, mysql, hash-cracking, telnet, privesc, cve-2025-32432, cve-2026-24061, linux, HackTheBox]
description: "Résolution de la machine Orion sur Hack The Box : reconnaissance des services, exploitation de Craft CMS, récupération des identifiants MySQL, accès utilisateur et élévation de privilèges via Telnet."
image:
  path: /assets/img/htb/orion/orion-banner.png
  alt: Orion Banner
---

# Hack The Box — Orion

## Informations sur la machine

* **Cible** : `10.129.153.100`
* **Système d'exploitation** : Linux
* **Difficulté** : À compléter

## 1. Reconnaissance avec Nmap

La première étape consiste à identifier les services accessibles sur la machine cible afin de déterminer les différentes surfaces d'attaque disponibles.

J'utilise Nmap avec les options `-sC` (scripts par défaut) et `-sV` (détection de version) afin d'obtenir davantage d'informations sur les services accessibles.

```bash
nmap -sC -sV 10.129.153.100
```

**Résultat :**

![nmap resultat](/assets/img/htb/orion/nmap_result.png)

Le scan révèle deux ports ouverts :

| Port | Service |
| ---- | ------- |
| 22   | ssh     |
| 80   | http    |

Le port 80 héberge un serveur web. Je vais donc commencer par explorer cette surface d'attaque.

---

## 2. Découverte de contenu Web

Je me rends sur la page web à l'adresse :

`http://10.129.153.100/`

Je tombe alors sur un site vitrine.

**Page d'accueil :**

![home page](/assets/img/htb/orion/home_page.png)

En explorant la section **À propos**, je découvre que le site utilise **Craft CMS**.

N'ayant pas identifié d'informations particulièrement exploitables à ce stade, je décide de passer à une phase de **content discovery** afin de rechercher d'éventuels fichiers ou répertoires intéressants.

---

## 3. Énumération avec Gobuster

J'utilise Gobuster afin de rechercher des répertoires et des fichiers potentiellement intéressants sur le serveur web.

**Résultat :**

![gobuster](/assets/img/htb/orion/gobuster.png)

Le scan permet notamment d'identifier deux pages intéressantes.

Je décide alors de me rendre sur la page d'administration :

`http://orion.htb/admin/login/`

En explorant cette interface, je constate que la version de Craft CMS utilisée est la **5.6.16**.

Cette information va me permettre d'orienter mes recherches vers d'éventuelles vulnérabilités connues.

---

## 4. Recherche et découverte de la vulnérabilité

Après avoir identifié la version de Craft CMS, je lance une recherche sur Exploit Database afin de trouver des vulnérabilités associées à cette version.

Je découvre alors le **CVE-2025-32432**, une vulnérabilité permettant notamment une exécution de commandes à distance (RCE).

Un exploit écrit en Python est disponible pour exploiter cette vulnérabilité.

Cependant, après plusieurs tentatives, je ne parviens pas à obtenir le résultat attendu avec cet exploit.

Je décide donc d'utiliser une autre approche en passant par **Metasploit Framework**.

---

## 5. Exploitation de Craft CMS avec Metasploit

Je lance Metasploit Framework et recherche le module correspondant à la vulnérabilité identifiée.

**Interface Metasploit :**

![metasploit](/assets/img/htb/orion/metasploit.png)

Après avoir sélectionné le module d'exploitation, je procède à sa configuration en renseignant les paramètres nécessaires à la cible.

Je lance ensuite l'exploit.

**Configuration et exécution :**

![metasploit config](/assets/img/htb/orion/metasploit_config.png)

L'exploitation fonctionne et me permet d'obtenir un shell sur la machine distante.

**Shell obtenu :**

![metasploit result](/assets/img/htb/orion/metasploit_result.png)

Je souhaite maintenant identifier l'utilisateur sous lequel les commandes sont exécutées.

Pour cela, j'utilise :

```bash
whoami
```

Le résultat indique :

```text
www-data
```

Cela signifie que j'ai obtenu un accès au serveur distant avec les privilèges de l'utilisateur `www-data`.

Je dispose désormais d'un premier accès à la machine. Je vais donc poursuivre l'énumération afin de rechercher des informations permettant d'obtenir un accès à un utilisateur du système.

---

## 6. Récupération des identifiants de la base de données

Après quelques recherches, j'apprends que les informations de connexion à une base de données peuvent être stockées dans le fichier `.env`, notamment dans les applications utilisant certains frameworks web.

Je décide donc de rechercher ce fichier sur le serveur.

Une fois sa localisation identifiée, je l'ouvre afin d'examiner son contenu.

**Résultat :**

![Recherche du mot de passe](/assets/img/htb/orion/recherche.png)

Je découvre ainsi le mot de passe utilisé pour accéder à la base de données MySQL.

![Mot de passe de la base de donnée](/assets/img/htb/orion/mdp_db.png)

Cette information constitue une nouvelle piste pour poursuivre l'exploitation.

---

## 7. Énumération de la base de données MySQL

À l'aide du mot de passe récupéré dans le fichier `.env`, je me connecte à la base de données MySQL.

Après avoir exploré les différentes tables, je découvre notamment une table nommée `users`.

Cette table contient des informations relatives aux utilisateurs de l'application, dont le compte administrateur.

**Résultat :**

![Connexion à la base de donnée](/assets/img/htb/orion/connexion_db.png)

![Show database](/assets/img/htb/orion/show_db.png)

![Show table](/assets/img/htb/orion/show_table1.png)

![Show table suite](/assets/img/htb/orion/show_table2.png)

![afficher users](/assets/img/htb/orion/afficher_users.png)


En examinant les données, je remarque une chaîne qui semble correspondre à un hash :

```text
2y$13e9zuohgFZzGtbQalcn9Mz.5PJbjxobO0GMbXo8NHp3P/B42LUg0lS
```

Il s'agit vraisemblablement d'un mot de passe haché. Je décide donc d'utiliser **John the Ripper** afin de tenter de retrouver le mot de passe en clair.

---

## 8. Crack du hash avec John the Ripper

Je récupère le hash et le soumets à John the Ripper afin de rechercher une correspondance dans sa wordlist.

**Résultat :**

![John The Ripper](/assets/img/htb/orion/john.png)

Le mot de passe retrouvé est :

```text
darkangel
```

Je dispose désormais d'un mot de passe qui me permet de poursuivre l'énumération des utilisateurs présents sur la machine.

---

## 9. Connexion en tant qu'utilisateur adam

Lors de l'énumération du système, j'identifie un utilisateur nommé `adam`.

Je décide alors de tenter une connexion avec cet utilisateur en utilisant le mot de passe récupéré précédemment.

Pour cela, j'utilise la commande :

```bash
su - adam
```

Après avoir renseigné le mot de passe, je parviens à basculer vers l'utilisateur `adam`.

Je vérifie ensuite les fichiers présents dans son répertoire personnel afin de rechercher le flag utilisateur.

Je peux afficher son contenu avec :

```bash
cat user.txt
```

**Flag utilisateur :**

```text
5927c386f289b1262abc50c21857720c
```

L'accès utilisateur est donc obtenu. Je peux maintenant poursuivre l'énumération afin de rechercher une éventuelle possibilité d'escalade de privilèges.

---

## 10. Énumération des services actifs

Après avoir obtenu un accès en tant que `adam`, je cherche à identifier les services actifs sur la machine.

Pour cela, j'utilise deux commandes complémentaires.

La première permet de lister les services systemd actuellement en cours d'exécution :

```bash
systemctl list-units --type=service --state=running
```

La seconde permet d'identifier les ports en écoute sur la machine :

```bash
ss -tulpen | grep LISTEN
```

**Résultat :**

![Découverte des services](/assets/img/htb/orion/services.png)

L'énumération révèle notamment la présence d'un service **Telnet** actif localement sur le port 23.

Cette découverte constitue une piste intéressante pour la suite de l'exploitation.

Je décide alors d'examiner la version du client Telnet présent sur la machine.

```bash
telnet --version
```

**Résultat :**

![Telnet version](/assets/img/htb/orion/telnet.png)

La version identifiée est la **2.7**.

---

## 11. Recherche et découverte de la vulnérabilité Telnet

Après quelques recherches, je découvre le **CVE-2026-24061**, une vulnérabilité associée à Telnet.

Cette découverte me permet d'orienter mes recherches vers une possible élévation de privilèges.

Après avoir étudié le fonctionnement de l'exploitation, je comprends partiellement comment exploiter cette vulnérabilité à l'aide de la variable d'environnement `USER`.

La commande utilisée est :

```bash
USER='-f root' telnet -a 10.129.153.100
```

L'exploitation fonctionne et me permet d'obtenir un shell avec les privilèges de `root`.

**Résultat :**

![Telnet exploitation](/assets/img/htb/orion/telnet_exploit.png)

L'élévation de privilèges est donc réussie.

---

## 12. Récupération du flag root

Maintenant que je dispose des privilèges de `root`, je peux accéder au répertoire personnel de cet utilisateur.

Je recherche alors le fichier `root.txt` et affiche son contenu.

**Flag root :**

```text
938fef186ab969f8756e7b0b0c31b424
```

La compromission complète de la machine est désormais terminée.

---

## 13. Chaîne d'exploitation (Kill Chain)

La compromission de la machine peut être résumée ainsi :

```text
Nmap
   │
   ▼
Ports 22 / 80
   │
   ▼
Serveur Web
   │
   ▼
Identification de Craft CMS
   │
   ▼
Content Discovery avec Gobuster
   │
   ▼
Découverte de l'interface /admin/login/
   │
   ▼
Identification de Craft CMS 5.6.16
   │
   ▼
Recherche du CVE-2025-32432
   │
   ▼
Exploitation avec Metasploit
   │
   ▼
RCE / Shell www-data
   │
   ▼
Recherche du fichier .env
   │
   ▼
Récupération du mot de passe MySQL
   │
   ▼
Énumération de la base de données
   │
   ▼
Récupération du hash utilisateur
   │
   ▼
Crack avec John the Ripper
   │
   ▼
Mot de passe : darkangel
   │
   ▼
Connexion en tant que adam
   │
   ▼
/home/adam/user.txt
   │
   ▼
Flag User
   │
   ▼
Énumération des services actifs
   │
   ▼
Découverte de Telnet sur le port 23
   │
   ▼
Identification du CVE-2026-24061
   │
   ▼
Exploitation de Telnet
   │
   ▼
Shell Root
   │
   ▼
/root/root.txt
   │
   ▼
Flag Root
```

---

## 14. Ce que j'ai appris

Cette machine m'a permis de mettre en pratique plusieurs notions importantes en cybersécurité offensive.

1. **L'importance du content discovery** : l'utilisation de Gobuster permet de découvrir des ressources et des pages qui ne sont pas nécessairement accessibles depuis la page d'accueil. L'identification de l'interface d'administration a notamment permis de découvrir la version de Craft CMS utilisée.

2. **L'identification des vulnérabilités à partir des versions** : connaître la version d'un service ou d'une application permet d'orienter les recherches vers des vulnérabilités connues et des exploits potentiellement exploitables.

3. **L'utilisation de Metasploit Framework** : cette machine m'a permis de découvrir comment rechercher, configurer et exécuter un module d'exploitation à l'aide de Metasploit.

4. **L'importance des fichiers de configuration** : le fichier `.env` peut contenir des informations sensibles, notamment des identifiants de connexion à une base de données. Une mauvaise gestion de ces informations peut faciliter la compromission d'une application.

5. **L'énumération des bases de données** : l'accès à MySQL m'a permis de découvrir les données stockées dans la table `users`, notamment des informations relatives aux comptes utilisateurs.

6. **Le crack de hash avec John the Ripper** : j'ai pu mettre en pratique l'utilisation d'un outil de cracking afin de retrouver un mot de passe à partir d'un hash récupéré dans une base de données.

7. **L'importance de l'énumération locale** : après avoir obtenu un premier accès, il est essentiel de continuer à explorer la machine. Les commandes `systemctl` et `ss` permettent notamment d'identifier les services actifs et les ports en écoute.

8. **L'exploitation des services locaux** : la découverte du service Telnet a permis d'identifier une nouvelle surface d'attaque et de poursuivre l'escalade de privilèges.

9. **L'importance de la veille sur les CVE** : cette machine m'a montré qu'il est nécessaire de savoir rechercher, comprendre et adapter les informations relatives aux vulnérabilités connues.

10. **La progression méthodique dans une machine HTB** : l'obtention d'un premier shell ne signifie pas que l'exploitation est terminée. Chaque accès doit être suivi d'une nouvelle phase d'énumération afin d'identifier les possibilités de progression vers des privilèges plus élevés.

---
