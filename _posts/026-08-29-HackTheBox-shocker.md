---
title: "Hack The Box — Shocker : Writeup"
date: 2026-08-29 00:00:00 +0200
categories: [Box, HackTheBox]
tags: [web, content-discovery, cgi, bash, cve-2014-6271, reverse-shell, privesc, sudo, linux, HackTheBox]
description: "Résolution de la machine Shocker sur Hack The Box : reconnaissance des services, découverte du répertoire CGI, exploitation de Shellshock pour obtenir un reverse shell et élévation de privilèges via Perl."
image:
  path: /assets/img/htb/shocker/shocker-banner.png
  alt: shocker Banner
  
---

# Hack The Box — Shocker

## Informations sur la machine

* **Cible** : `10.129.74.122`

* **Système d'exploitation** : Linux

* **Difficulté** : Facile

## 1. Reconnaissance avec Nmap

La première étape consiste à identifier les services accessibles sur la machine cible et à déterminer les différentes surfaces d'attaque disponibles.

J'utilise Nmap avec les options `-sC` (scripts par défaut) et `-sV` (détection de version) afin d'obtenir davantage d'informations sur les services accessibles.

```bash
nmap -sC -sV 10.129.74.122
```

**Résultat :**

![nmap resultat](/assets/img/htb/shocker/nmap_resultat.png)


Le scan révèle deux ports ouverts :

| Port | Service |
| ---- | ------- |
| 80   | http    |
| 2222 | ssh     |

Le port 80 héberge un serveur web. Je vais donc commencer par explorer cette surface d'attaque.

---

## 2. Découverte de contenu Web

Je me rends sur la page web à l'adresse :

`http://10.129.74.122/`

Je tombe alors sur un troll :

![home page](/assets/img/htb/shocker/home_page.png)

En explorant l'interface, je ne constate rien de particulièrement intéressant.

Je passe donc à une phase de **content discovery** afin de rechercher d'éventuels fichiers ou répertoires cachés.

---

## 3. Énumération avec Gobuster

J'utilise Gobuster avec une wordlist commune ainsi que plusieurs extensions afin de rechercher des fichiers et répertoires potentiellement intéressants.

```bash
gobuster dir -u http://10.129.74.122/ -w /usr/share/wordlists/dirb/common.txt -x php,txt,html,js 2>/dev/null
```

**Résultat :**

![gobuster 1](/assets/img/htb/shocker/gobuster_1.png)

Le scan permet notamment d'identifier le répertoire :

`/cgi-bin/`

Il s'agit d'un répertoire connu pour l'exécution de scripts via l'interface **CGI (Common Gateway Interface)**.

Je me rends donc à l'adresse :

`http://10.129.74.122/cgi-bin/`

Cependant, je ne peux pas accéder directement à cette page.

![forbidden](/assets/img/htb/shocker/forbidden.png)

Je décide alors de relancer Gobuster directement sur ce répertoire en recherchant cette fois des extensions correspondant à des scripts CGI.

```bash
gobuster dir -u http://10.129.74.122/cgi-bin/ -w /usr/share/wordlists/dirb/common.txt -x cgi,sh,pl 2>/dev/null
```

**Résultat :**

![gobuster 2](/assets/img/htb/shocker/gobuster_2.png)

Le scan permet de découvrir un script nommé :

`user.sh`

Je me rends alors à l'adresse :

`http://10.129.74.122/cgi-bin/user.sh`

J'obtiens alors le résultat suivant :

![user.sh](/assets/img/htb/shocker/user_sh.png)

Le contenu de cette page correspond à la sortie de la commande `uptime` sous Linux, qui permet notamment d'afficher depuis combien de temps le système fonctionne.

La présence d'un script CGI exécutant des commandes côté serveur constitue une piste intéressante pour la suite de l'exploitation.

---

## 4. Recherche et découverte de la vulnérabilité

En recherchant des informations sur les vulnérabilités liées à **Apache**, **CGI** et **Bash**, je découvre le **CVE-2014-6271**, également connu sous le nom de **Shellshock**.

Cette vulnérabilité permet notamment une exécution de commandes à distance lorsqu'un serveur CGI utilise une version vulnérable de Bash.

L'exploit utilisé est disponible sur Exploit-DB :

https://www.exploit-db.com/exploits/34900

---

## 5. Exploitation de Shellshock

J'adapte l'exploit afin de pouvoir l'utiliser avec Python 3.

Après avoir exécuté l'exploit, j'obtiens un **reverse shell** sur ma machine d'attaque.

![reverse shell](/assets/img/htb/shocker/reverse_shell.png)

Je souhaite ensuite identifier l'utilisateur sous lequel les commandes sont exécutées.

Cependant, lorsque je tente d'utiliser `whoami`, la commande ne fonctionne pas correctement car le shell obtenu n'est pas interactif.

Je décide donc de stabiliser le shell avec Python :

```bash
python -c 'import pty; pty.spawn("/bin/bash")'
```

Une fois le shell stabilisé, j'utilise la commande :

```bash
whoami
```

Le résultat indique que je suis connecté en tant que :

`shelly`

---

## 6. Récupération du flag utilisateur

Je me rends dans le répertoire personnel de l'utilisateur `shelly` afin de rechercher le fichier `user.txt`.

```bash
cd /home/shelly
ls
```

Le fichier `user.txt` est présent.

Je peux alors afficher son contenu :

```bash
cat user.txt
```

**Flag utilisateur :**

`42828566dfe0d054c25e07085ea76e8c`

---

## 7. Énumération des privilèges

Après avoir obtenu un accès à la machine en tant que `shelly`, je cherche maintenant à déterminer si cet utilisateur possède des privilèges particuliers.

J'utilise pour cela :

```bash
sudo -l
```

**Résultat :**

![sudo l](/assets/img/htb/shocker/sudo_l.png)

On constate que l'utilisateur `shelly` peut exécuter le binaire **Perl** avec les privilèges de `root` sans avoir besoin de fournir de mot de passe.

Cette configuration constitue une piste intéressante pour l'escalade de privilèges.

---

## 8. Exploitation de la permission sudo

Je recherche alors une méthode permettant d'exploiter les privilèges sudo accordés à Perl.

Pour cela, je consulte **GTFOBins**, qui répertorie notamment différentes techniques permettant d'abuser de binaires disposant de privilèges particuliers.

https://gtfobins.org/

Je trouve une commande permettant d'utiliser Perl afin d'obtenir un shell avec les privilèges de `root`.

Après son exécution, je vérifie mon niveau de privilèges avec :

```bash
whoami
```

**Résultat :**
![root](/assets/img/htb/shocker/root.png)

Le résultat indique que je suis maintenant connecté en tant que :

`root`

L'escalade de privilèges est donc réussie.

---

## 9. Récupération du flag root

Je peux maintenant accéder au répertoire personnel de `root` et rechercher le fichier `root.txt`.

```bash
cd /root
ls
```

Je peux ensuite afficher le contenu du fichier :

```bash
cat root.txt
```

**Flag root :**

`92972780e0d9402ab80dd2ddc403debe`

---

## 10. Chaîne d'exploitation (Kill Chain)

La compromission de la machine peut être résumée ainsi :

```text
Nmap
   │
   ▼
Ports 80 / 2222
   │
   ▼
Serveur Web Apache
   │
   ▼
Content Discovery avec Gobuster
   │
   ▼
Découverte du répertoire /cgi-bin
   │
   ▼
Nouvelle énumération avec Gobuster
   │
   ▼
Découverte de user.sh
   │
   ▼
Identification de Shellshock
   │
   ▼
CVE-2014-6271
   │
   ▼
Exécution de commandes à distance
   │
   ▼
Reverse Shell
   │
   ▼
Shell shelly
   │
   ▼
/home/shelly/user.txt
   │
   ▼
Flag User
   │
   ▼
sudo -l
   │
   ▼
Perl avec privilèges root
   │
   ▼
Exploitation via GTFOBins
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

1. **L'importance du content discovery** : l'utilisation de Gobuster permet de découvrir des répertoires qui ne sont pas forcément visibles depuis la page d'accueil. Dans ce cas, la découverte de `/cgi-bin` a permis d'identifier une nouvelle surface d'attaque.

2. **L'énumération des scripts CGI** : après avoir découvert `/cgi-bin`, il était pertinent de rechercher spécifiquement des extensions telles que `.cgi`, `.sh` ou `.pl` afin d'identifier les scripts exécutables présents dans le répertoire.

3. **L'exploitation de Shellshock** : la découverte du script `user.sh` a permis d'identifier une piste menant au **CVE-2014-6271**, une vulnérabilité historique de Bash permettant l'exécution de commandes à distance dans certaines configurations CGI.

4. **La stabilisation d'un reverse shell** : un reverse shell obtenu à distance n'est pas nécessairement interactif. La stabilisation du shell permet ensuite d'utiliser plus facilement des commandes et d'interagir avec le système.

5. **L'énumération des privilèges sudo** : après avoir obtenu un accès initial, la commande `sudo -l` constitue une étape importante afin d'identifier les binaires pouvant être exécutés avec des privilèges élevés.

6. **L'utilisation de GTFOBins** : lorsqu'un binaire possède des permissions sudo particulières, GTFOBins peut permettre d'identifier rapidement les techniques d'exploitation correspondantes.

7. **L'importance de poursuivre l'énumération après le premier accès** : l'obtention d'un shell utilisateur ne signifie pas que la compromission est terminée. Il faut continuer à analyser les privilèges disponibles afin d'identifier une éventuelle escalade vers `root`.

---
