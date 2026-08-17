---
title: "Matryoshka Bits : Forensic & Automatisation"
date: 2026-08-17 15:00:00 +0100
categories: [CTF, Created Challenges]
tags: [forensic, python, bash, scripting, my-challenges]
description: "Matryoshka Bits : automatisation de l'extraction de 150 archives imbriquées en Python et reconstruction d'une image JPEG à partir de bits ASCII."
---

## Contexte narratif
Les secrets les mieux gardés sont ceux qu'on cache à la vue de tous."

**Fichier fourni :** `challenge.zip`

---

### Étape 1 — Première reconnaissance

On commence toujours par identifier ce qu'on a entre les mains.

```bash
file challenge.zip
# challenge.zip: Zip archive data, at least v1.0 to extract, compression method=store
```

Un fichier ZIP classique. On extrait.

```bash
unzip challenge.zip
# → layer_001.zip
```

On retombe sur un ZIP. On extrait encore.

```bash
unzip layer_001.zip
# → layer_002.zip
```

Et encore... À ce stade, le participant comprend qu'il va falloir **automatiser**.

---

### Étape 2 — Automatiser l'extraction des 150 couches

Extraire 150 ZIP à la main c'est hors de question. On écrit un script Python.

```python
# solve_matryoshka.py
import os, subprocess, shutil, glob

WORK = "solve_work"
os.makedirs(WORK, exist_ok=True)
shutil.copy("challenge.zip", f"{WORK}/challenge.zip")

current = f"{WORK}/challenge.zip"

for i in range(1, 151):
    print(f"[{i:03d}/150] Extraction de {os.path.basename(current)}...")
    subprocess.run(
        ["unzip", "-o", "-d", WORK, current],
        capture_output=True
    )
    os.remove(current)

    files = glob.glob(f"{WORK}/*")
    if not files:
        print("❌ Rien trouvé, arrêt.")
        break
    current = files[0]

print(f"\n✅ Fichier final : {current}")
```

```bash
python3 solve_matryoshka.py
# ✅ Fichier final : solve_work/digits.bin
```

---

### Étape 3 — Analyser le fichier final

On identifie ce qu'on vient d'extraire.

```bash
file solve_work/digits.bin
# digits.bin: ASCII text, with very long lines (65536), with no line terminators
```

Intéressant — un fichier `.bin` qui est en réalité du texte ASCII. On jette un œil au contenu.

```bash
head -c 64 solve_work/digits.bin
# 1111111111011000111111111110000000000000000100000100101001000110
```

Une longue suite de `0` et de `1`. Ce sont des **bits représentés en ASCII**.

---

### Étape 4 — Décoder la chaîne binaire

On analyse la structure et on identifie le type de fichier caché via les **magic bytes**.

```python
# decode.py
with open("solve_work/digits.bin", "r") as f:
    bits = f.read().strip()

print(f"Longueur totale  : {len(bits)} bits")
print(f"Divisible par 8  : {len(bits) % 8 == 0}")

data = bytearray()
for i in range(0, len(bits), 8):
    data.append(int(bits[i:i+8], 2))

print(f"Magic bytes (hex) : {data[:4].hex()}")

with open("output.jpg", "wb") as f:
    f.write(data)

print("Image extraite : output.jpg")
```

```
Longueur totale  : 70960 bits
Divisible par 8  : True
Magic bytes (hex) : ffd8ffe0
Image extraite   : output.jpg
```

`FFD8FF` = **signature JPEG**. On a reconstruit une image.

---

### Étape 5 — Récupérer le flag

```bash
xdg-open output.jpg
```

L'image affiche le flag en clair.

```
CYB3R{b1t5_4r3_3v3rywh3r3}
```

---

## Concepts testés

|Concept|Description|
|---|---|
|Automatisation|Savoir scripter face à une tâche répétitive|
|Identification de fichier|Utiliser `file` et lire les magic bytes|
|Représentation binaire|Comprendre bits vs bytes|
|Reconstruction de fichier|Convertir une chaîne de bits en fichier binaire|
|Persévérance|150 couches = challenge de mindset autant que technique|

---
