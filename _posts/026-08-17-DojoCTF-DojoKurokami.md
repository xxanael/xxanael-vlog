---
title: "Dojo Kurokami : OSINT & Web Traps"
date: 2026-08-17 14:00:00 +0100
categories: [CTF, Created Challenges]
tags: [my-challenges, osint, web, tumblr, github, exiftool]
description: "Dojo Kurokami : analyse de code source web, évitement de faux flags sur Tumblr et extraction EXIF sur GitHub."
---

## Contexte narratif

Le Dojo Kurokami était une école secrète de samouraï fondée dans les montagnes du Kyushu. Son maître, connu sous le pseudonyme SenseiKenzan, a disparu après avoir laissé des indices dispersés sur internet. Votre mission : retrouver le nom du sanctuaire où il s'est retiré.

**Lien :** https://dojo-archives.vercel.app/

---

## Informations générales

| Champ | Valeur |
|---|---|
| Catégorie | OSINT |
| Difficulté | Avancée |
| Format des flags | `CYB3R{...}` |
| Nombre de flags | 3 (2 faux, 1 vrai) |
| Point d'entrée | Site vitrine du Dojo (HTML statique) |

---

## Tableau des flags

| # | Flag | Type | Emplacement |
|---|---|---|---|
| F1 | `CYB3R{honneur_du_samourai}` | ❌ Faux | Blog caché, visible |
| F2 | `CYB3R{voie_des_anciens_1337}` | ❌ Faux | Commentaire Tumblr |
| ✅ | `CYB3R{katana_no_michi_4782}` | ✅ Vrai | EXIF image GitHub |

---

## Outils recommandés

- `exiftool`
- `strings`
- Inspect Element (DevTools navigateur)

---

# 🏁 Writeup du Challenge (Solution)

## Étape 1 — Analyser le code source du site

On commence par ouvrir le site vitrine du Dojo Kurokami. Au premier regard, rien de particulier. On inspecte le code source (`Ctrl+U` ou clic droit > Afficher le source) et on découvre un commentaire HTML dissimulé :

```html
<!-- Archive des enseignements : voir /dojo-archives/bushido.html -->
```

---

## Étape 2 — Explorer le blog caché

On navigue vers `/dojo-archives/bushido.html`. La page présente trois articles sur le Bushido. On remarque immédiatement un encadré avec :

```
CYB3R{honneur_du_samourai}
```

> ⚠️ **Faux flag** — placé intentionnellement pour piéger les joueurs pressés.

On continue la lecture et on repère une référence à `senseikenzan.tumblr.com`. En inspectant le source, on trouve également un commentaire caché :

```html
<!-- contact_node: senseikenzan.tumblr.com -->
```

---

## Étape 3 — Investiguer le profil Tumblr

On visite `senseikenzan.tumblr.com`. Le profil présente 5 posts thématiques sur les arts martiaux et le Japon. En parcourant les commentaires, on trouve :

```
CYB3R{voie_des_anciens_1337}
```

> ⚠️ **Faux flag** — encore un piège.

Dans le post sur les archives photographiques, on trouve un lien direct vers une image hébergée sur GitHub.

---

## Étape 4 — Extraire les métadonnées EXIF

On télécharge l'image depuis GitHub et on lance `exiftool` :

```bash
exiftool kurokami_dojo.jpg
```

Parmi les métadonnées, on trouve le champ `Comment` :

```
Comment : Le sanctuaire se trouve la ou le soleil se leve sur le mont Kurokami. CYB3R{katana_no_michi_4782}
```

On note également des coordonnées GPS : `33°4'59.88"N, 130°28'0.12"E` — pointant vers la région de Kyushu au Japon, confirmant la cohérence du scénario.

---

## Flag final

```
CYB3R{katana_no_michi_4782}
```

---

## Résumé des étapes

| Étape | Action                        | Résultat                                 |
| ----- | ----------------------------- | ---------------------------------------- |
| 1     | Inspecter le code source HTML | Lien vers `/dojo-archives/bushido.html`  |
| 2     | Lire le blog caché            | Faux flag + lien Tumblr                  |
| 3     | Explorer le profil Tumblr     | Faux flag + lien image GitHub            |
| 4     | `exiftool` sur l'image        | Flag final `CYB3R{katana_no_michi_4782}` |

