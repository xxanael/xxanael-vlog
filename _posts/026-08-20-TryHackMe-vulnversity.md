---
title: "TryHackMe — Vulnversity : Writeup"
date: 2026-08-20 00:22:00 +0100
categories: [Room, TryHackMe]
tags: [web, file-upload, privesc, suid, linux, systemctl]
description: "Résolution de la room RootMe sur TryHackMe : contournement de blacklist d'upload, reverse shell et privilège escalation via un binaire SUID systemctl."
image:
  path: /assets/img/tryhackme/vulnversity/vulnversity-banner.webp
  alt: vulnersity Banner
---

# TryHackMe — Vulnversity

## Informations sur la machine

**Cible :** `10.128.153.32`

---

# 1. Connexion à la machine

La première étape consiste à identifier les services accessibles sur la machine cible et à déterminer les différentes surfaces d'attaque disponibles.

---

# 2. Reconnaissance avec Nmap

Comme toujours lors d'un pentest, je commence par effectuer une reconnaissance des ports et des services exposés.

J'utilise Nmap avec les options `-sC` et `-sV` afin d'exécuter les scripts de reconnaissance par défaut et d'identifier les versions des services.

```bash
nmap -sC -sV 10.128.153.32
```

**Résultat :**

![nmap result](/assets/img/tryhackme/vulnversity/nmap_result.png)

Le scan révèle **6 ports ouverts** :

|Port|Service|
|---|---|
|21|FTP|
|22|SSH|
|139|NetBIOS|
|445|NetBIOS/SMB|
|3128|HTTP Proxy|
|3333|HTTP|

Ces informations constituent notre première cartographie de la surface d'attaque.

### Réponses aux questions

**How many ports are open?**

```text
6
```

**What version of the Squid proxy is running on the machine?**

```text
3.5.12
```

**What is the most likely operating system this machine is running?**

```text
Ubuntu
```

**What port is the web server running on?**

```text
3333
```

---

# 3. Découverte de contenu avec Gobuster

Le port `3333` héberge un serveur web. Je commence donc par consulter directement la page d'accueil :

```text
http://10.128.153.32:3333/
```

![home page](/assets/img/tryhackme/vulnversity/home_page.png)

La page d'accueil ne révèle rien d'intéressant dans un premier temps. Je passe donc à une phase de **content discovery** afin de rechercher des répertoires et fichiers cachés.

Pour cela, j'utilise Gobuster :

```bash
gobuster dir -u http://10.128.153.32:3333/ \
-w /usr/share/wordlists/dirb/common.txt \
-x php,txt,html,js \
2>/dev/null
```

**Résultat :**

![gobuster result](/assets/img/tryhackme/vulnversity/gobuster_result.png)

Le scan permet notamment de découvrir le répertoire :

```text
/internal/
```

Ce répertoire semble particulièrement intéressant puisqu'il peut contenir des fonctionnalités qui ne sont pas accessibles depuis la page principale.

### Question TryHackMe

**What is the directory that has an upload form page?**

```text
/internal/
```

---

# 4. Compromission du serveur

## 4.1 Découverte du formulaire d'upload

Je me rends sur :

```text
http://10.128.153.32:3333/internal/
```

J'y découvre un formulaire permettant d'uploader un fichier.

![upload page](/assets/img/tryhackme/vulnversity/upload_page.png)

Un formulaire d'upload représente une surface d'attaque intéressante, notamment lorsqu'il est possible d'envoyer un fichier contenant du code exécutable.

Je tente donc dans un premier temps d'uploader un payload PHP.

Cependant, l'upload échoue.

Cela laisse penser que l'application applique un filtrage sur les extensions de fichiers autorisées.

---

## 4.2 Contournement du filtrage d'extension

Plutôt que de tester manuellement toutes les extensions possibles, j'utilise **Burp Suite Intruder** afin d'automatiser le test.

Je commence par créer une liste contenant différentes extensions susceptibles d'être acceptées par le serveur.

![create list](/assets/img/tryhackme/vulnversity/create_list.png)

Je démarre ensuite Burp Suite et capture la requête d'upload à l'aide du module **Proxy**.

![proxy](/assets/img/tryhackme/vulnversity/capture_proxy.png)

La requête est ensuite envoyée vers **Intruder**.

![intruder](/assets/img/tryhackme/vulnversity/intruder.png)

Je configure Intruder afin de tester les différentes extensions présentes dans ma liste.

![list](/assets/img/tryhackme/vulnversity/config_list.png)
  
![attack result](/assets/img/tryhackme/vulnversity/attack_result.png)

Après avoir lancé l'attaque, je remarque notamment une réponse différente pour l'extension `phtml`.

La réponse correspondante possède une longueur de **759 octets**, ce qui indique que le serveur a probablement accepté cette extension alors que l'extension `.php` était filtrée.

Je peux donc utiliser `.phtml` pour contourner le filtre.

---

# 5. Obtention d'un Reverse Shell

Je modifie localement l'extension de mon payload PHP afin d'utiliser l'extension :

```text
.phtml
```

Avant d'uploader le fichier, je lance un listener Netcat sur ma machine :

```bash
nc -lnvp 4444
```

![listener](/assets/img/tryhackme/vulnversity/netcat.png)

J'upload ensuite le fichier `.phtml`.

Le fichier passe correctement le filtre.

![success](/assets/img/tryhackme/vulnversity/success.png)

Je peux alors accéder au répertoire dans lequel les fichiers uploadés sont stockés :

```text
http://10.128.153.32:3333/internal/uploads/
```

![uploads](/assets/img/tryhackme/vulnversity/uploads.png)

En cliquant sur le fichier uploadé, le serveur exécute le payload et celui-ci déclenche une connexion vers mon listener.

J'obtiens alors un **reverse shell** sur ma machine.

![reverse shell](/assets/img/tryhackme/vulnversity/reverse_shell.png)

---

# 6. Stabilisation du shell et récupération du flag utilisateur

Une fois le reverse shell obtenu, je le stabilise afin de pouvoir travailler plus confortablement. Je peux ensuite explorer le système et me rendre dans le répertoire personnel de l'utilisateur :

```bash
cd /home/bill
```

Je découvre notamment le fichier :

```text
user.txt
```

Ce fichier contient le premier flag.

### Question TryHackMe

**What is the name of the user who manages the webserver?**

```text
bill
```

**What is the user flag?**

```text
8bd**************************edb
```

---

# 7. Escalade de privilèges

Après avoir obtenu un accès utilisateur, l'objectif suivant est de devenir `root`.

Je commence par rechercher les fichiers possédant le bit **SUID**.

Pour cela, j'utilise :

```bash
find / -perm -4000 -type f 2>/dev/null
```

Cette commande permet de rechercher les fichiers exécutables disposant de la permission SUID.

Parmi les résultats, un binaire attire immédiatement mon attention :

```text
/bin/systemctl
```

![result SUID](/assets/img/tryhackme/vulnversity/suid_result.png)

Le fait que `systemctl` possède le bit SUID est particulièrement intéressant car il peut permettre une élévation de privilèges.

### Question TryHackMe

**On the system, search for all SUID files. What file stands out?**

```text
/bin/systemctl
```

---

# 8. Exploitation de systemctl

Afin de déterminer comment exploiter ce binaire, je recherche une méthode d'exploitation connue sur **GTFOBins**.

Je trouve une technique permettant d'utiliser `systemctl` afin de créer et démarrer un service.

La méthode trouvée est la suivante :

```bash
echo '[Service]
Type=oneshot
ExecStart=/path/to/command
[Install]
WantedBy=multi-user.target' >/path/to/temp-file.service

systemctl link /path/to/temp-file.service

systemctl enable --now /path/to/temp-file.service
```

L'idée est de créer un fichier de service temporaire contenant la commande que l'on souhaite exécuter, puis de demander à `systemctl` de l'activer.

J'utilise donc un fichier temporaire correspondant à la variable `$TF`.

La commande suivante permet d'activer le service :

```bash
/bin/systemctl enable --now $TF
```

Le système confirme alors que le service a été créé et lié à la cible `multi-user.target` :

```text
Created symlink /etc/systemd/system/multi-user.target.wants/tmp.xn9ps63VM9.service → /tmp/tmp.xn9ps63VM9.service.
```

![process](/assets/img/tryhackme/vulnversity/process.png)


---

# 9. Récupération du flag root

Après l'exécution du service, je retourne dans `/tmp` afin de vérifier les fichiers générés :

```bash
cd /tmp/
ls
```

Je trouve notamment :

```text
output
root.service
root_proof.txt
tmp.xn9ps63VM9
tmp.xn9ps63VM9.service
```

Le fichier `output` semble particulièrement intéressant. Je l'affiche avec `cat` :

```bash
cat output
```

Le contenu du fichier est :

```text
a58**************************fd5
```

J'ai ainsi récupéré le **root flag**.


### Question TryHackMe

**Become root and get the last flag (/root/root.txt)**

```text
a58**************************fd5
```

---

# 10. Chaîne d'exploitation

La compromission de la machine peut être résumée ainsi :

```text
Nmap
  │
  ▼
Port 3333
  │
  ▼
Gobuster
  │
  ▼
/internal/
  │
  ▼
Formulaire d'upload
  │
  ▼
Filtrage de .php
  │
  ▼
Burp Suite Intruder
  │
  ▼
Découverte de .phtml
  │
  ▼
Upload du payload
  │
  ▼
Reverse Shell
  │
  ▼
Utilisateur www-data
  │
  ▼
/home/bill
  │
  ▼
user.txt
  │
  ▼
Recherche des fichiers SUID
  │
  ▼
/bin/systemctl
  │
  ▼
Création / activation d'un service
  │
  ▼
Exécution avec privilèges élevés
  │
  ▼
/tmp/output
  │
  ▼
Root Flag
```

---

# 11. Ce que j'ai appris

Cette room m'a principalement permis de renforcer plusieurs notions :

- Utilisation de **Nmap** pour la reconnaissance des services et des versions.
    
- Utilisation de **Gobuster** pour la découverte de contenu web.
    
- Identification et exploitation d'un **formulaire d'upload vulnérable**.
    
- Utilisation de **Burp Suite Intruder** pour tester automatiquement différentes extensions.
    
- Compréhension du contournement d'un filtre d'extension avec `.phtml`.
    
- Obtention d'un **reverse shell**.
    
- Stabilisation d'un shell distant.
    
- Recherche de fichiers possédant le bit **SUID** avec `find`.
    
- Identification de `/bin/systemctl` comme vecteur d'escalade de privilèges.
    
- Utilisation de **GTFOBins** pour rechercher des techniques d'exploitation de binaires Linux.
    

Cette machine m'a surtout permis de comprendre comment plusieurs petites étapes d'énumération et d'exploitation peuvent être enchaînées pour passer progressivement de **l'accès au serveur web → reverse shell → utilisateur → root**.
