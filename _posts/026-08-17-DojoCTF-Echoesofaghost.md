---
title: "Echoes of a Ghost : GEOINT & OSINT"
date: 2026-08-17 15:00:00 +0100
categories: [CTF, Created Challenges]
tags: [my-challenges, osint, geoint, what3words, mastodon, base64]
description: "Traque d'un cyber-espion via la méthode What3Words, pivot sur les réseaux décentralisés (Mastodon) et décodage de preuve d'attaque."
---
## 📑 Fiche du Challenge : Echoes of a Ghost

- **Catégorie :** OSINT
    
- **Difficulté :** Avancée (L3 SSI)
    
- **Points :** 250 pts
    
- **Fichier fourni :** `Journal.pdf`
    
- **Format du Flag :** `CYB3R{...}`
   

### 📝 Synopsis

La multinationale _BeninHydro_ a subi un vol de données massives concernant son réseau énergétique régional. Entre la revendication opportuniste du collectif `#ShadowVanguard` et l'incrimination technique de l'ingénieur `Real_Alpha_Cyber`, l'ANSSI soupçonne une manipulation de haut vol. L'attaquant semble avoir dissimulé sa véritable position ainsi que sa preuve de réussite (PoC) sur un réseau alternatif.

### 🎯 Objectif

Analysez l'article de presse fourni, localisez l'origine du signal et menez l'enquête en source ouverte pour identifier le véritable pseudonyme du cyber-espion afin d'extraire la signature.

# 🏁 Writeup du Challenge (Solution)

## Étape 1 : Analyse sémantique de la pièce à conviction (`Journal.pdf`)

En lisant attentivement l'article condensé, un élément textuel étrange retient l'attention dans le second paragraphe :

> _"...l'auteur a utilisé des scripts d'attaque nommés **"saucepans"** pour faire **"disappear"** les logs d'accès, avant d'effacer ses traces en **"immigrating"** vers un autre serveur."_

Les mots `"saucepans"`, `"disappear"`, et `"immigrating"` sont placés entre guillemets de manière artificielle au milieu du jargon cyber. Un analyste OSINT doit reconnaître le système de géolocalisation universel **What3Words (W3W)**, qui fragmente le monde en carrés de 3x3 mètres à l'aide de 3 mots anglais (ou français).

En assemblant ces mots sous la forme d'une adresse W3W : `///saucepans.disappear.immigrating`.

## Étape 2 : Pivot géospatial (Localisation du point d'impact)

L'introduction de ces trois mots sur le site officiel de **What3Words** ou l'utilisation de son API mène directement à une coordonnée géographique précise :

- **Lieu trouvé :** Le restaurant **"Lion Bar"** situé à **Grand-Popo** au Bénin.
    

L'article précisait que l'attaquant était _"très actif sur le réseau décentralisé Mastodon"_ et qu'il y avait laissé une _"critique publique liée à cet emplacement géographique"_.

## Étape 3 : Investigation en Source Ouverte (OSINT)

La recherche doit maintenant se focaliser sur la plateforme décentralisée **Mastodon** (historiquement connue pour ses instances comme `mastodon.social`).

Plusieurs méthodologies de recherche mènent au post (visible sur l'image `image_9e97e9.jpg`) :

1. **Google Dorking :** L'utilisation de requêtes ciblées sur les moteurs de recherche indexant les instances Mastodon : `site:mastodon.social "Lion Bar" "Grand-Popo"`
    
2. **Recherche interne Mastodon :** L'utilisation de la barre de recherche native de l'instance avec les mots-clés du lieu.
    

On découvre le compte de l'attaquant dont le pseudonyme est **@K0n0_Sec** avec le message suivant :

> _"Opération réussie. Les serveurs de R&D sont nettoyés. Pour mon commanditaire, la preuve d'authenticité depuis mon repaire temporaire au Lion Bar de Grand-Popo est scellée ici : Q1lCM1J7RnIwbV9HcjRuZF9QMHAwX1cxdGhfTDB2M30K"_

## Étape 4 : Décodage du Flag

Le message contient une chaîne de caractères robuste encodée en **Base64** : `Q1lCM1J7RnIwbV9HcjRuZF9QMHAwX1cxdGhfTDB2M30K`.

Il suffit d'utiliser un terminal Linux (via l'utilitaire `base64`) ou un outil comme CyberChef pour décoder la charge utile :

Bash

```
echo "Q1lCM1J7RnIwbV9HcjRuZF9QMHAwX1cxdGhfTDB2M30K" | base64 -d
```

**Résultat :** `CYB3R{Fr0m_Gr4nd_P0p0_W1th_L0v3}`

### 🚩 Flag Final :

`CYB3R{Fr0m_Gr4nd_P0p0_W1th_L0v3}`
