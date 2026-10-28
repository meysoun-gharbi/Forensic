# Forensic
**Université Bretagne Sud — Master Cybersécurité des Systèmes Embarqués**  
**Auteure :** Meysoun GHARBI  
**Année universitaire :** 2026–2027  
**Date :** Octobre 2026

## 1. Présentation du projet

Ce dépôt regroupe les éléments d'un travail pratique d'investigation numérique portant sur l'analyse d'une image disque de clé USB : `forensic_key_1.img`.

L'objectif est d'examiner le support afin d'identifier sa structure, de rechercher des traces d'activité, de récupérer des éléments supprimés et d'étudier d'éventuelles données chiffrées.

L'approche repose sur l'analyse des traces techniques et sur la distinction entre les faits établis, les interprétations vraisemblables et les hypothèses non vérifiées.


## 2. Organisation du dépôt

```text
Forensic/
├── forensic_ubs/
│   └── forensic_key_1.zip
├── README.md
├── rapport_forensic.pdf
├── .gitattributes
└── .gitignore
```

| Fichier | Description |
|---|---|
| `forensic_ubs/forensic_key_1.zip` | Archive contenant l'image disque analysée (`forensic_key_1.img`), stockée avec Git LFS |
| `rapport_forensic.pdf` | Rapport d'investigation numérique détaillé, comprenant les analyses, les résultats et les captures d'écran |
| `README.md` | Documentation du projet, méthodologie et synthèse des résultats |
| `.gitattributes` | Configuration du suivi des fichiers volumineux avec Git LFS |
| `.gitignore` | Configuration des fichiers exclus du suivi Git |
```


## 3. Environnement et outils

L'analyse a été réalisée principalement dans une machine virtuelle Debian 13 sous VirtualBox, avec certaines opérations effectuées sur Windows.

| Outil | Utilisation |
|---|---|
| The Sleuth Kit | Analyse des partitions et récupération de fichiers |
| TestDisk 7.2 | Recherche de structures de systèmes de fichiers |
| Autopsy 4.23.1 | Exploration et recherche par mots-clés |
| Binwalk | Analyse d'entropie |
| Python 3 | Traitement de données et interrogation SQLite |
| Cryptsetup | Ouverture d'un volume de type TrueCrypt |
| SHA-256 | Contrôle d'intégrité de l'image |

## 4. Méthodologie d'investigation

### 4.1. Vérification de l'intégrité

La première étape consiste à comparer l'empreinte SHA-256 de l'image à celle fournie dans le fichier de référence.

```bash
sha256sum forensic_key_1.img
```

La comparaison effectuée pendant le TP a indiqué une correspondance avec l'empreinte fournie.

Cette vérification contrôle l'intégrité par rapport à la référence, sans attester les conditions d'acquisition initiale du support.

### 4.2. Analyse des partitions

Les commandes suivantes permettent d'examiner la structure du disque :

```bash
file forensic_key_1.img
fdisk -l forensic_key_1.img
mmls forensic_key_1.img
```

L'image mesure **3 995 074 560 octets**, soit **7 802 880 secteurs de 512 octets**.

La table MBR déclare une seule partition FAT32, débutant au secteur `5304`.

TestDisk a également détecté des structures supplémentaires, notamment :

- Un système de fichiers ext2 associé à `casper-rw`.
- Deux structures NTFS, dont une identifiée comme `WINSHARE`.

Ces détections ne prouvent pas, à elles seules, l'existence de partitions autonomes.

### 4.3. Recherche et récupération de fichiers supprimés

L'analyse du système FAT32 a permis d'identifier un fichier de persistance Ubuntu nommé `casper-rw`, marqué comme supprimé.

Son contenu a pu être extrait avec :

```bash
icat -o 5304 forensic_key_1.img 22 > casper-rw-extrait.img
```

Le fichier extrait contient un système de fichiers ext2 lisible, avec notamment des répertoires utilisateur et des artefacts applicatifs.

### 4.4. Analyse des artefacts Linux et Firefox

L'exploration de la persistance a permis d'identifier :

- Un répertoire utilisateur `/home/ubuntu`.
- Des traces liées à l'installation de TrueCrypt.
- Une base d'activité Zeitgeist (`activity.sqlite`).
- Un lien symbolique vers un profil Firefox situé sous `/media/WINSHARE/`.

TestDisk a permis de repérer plusieurs fichiers du profil Firefox, notamment :

- `places.sqlite`
- `cookies.sqlite`
- `formhistory.sqlite`
- `downloads.sqlite`
- `key3.db`
- `prefs.js`

**Limite :** les bases Firefox n'ont pas été extraites et analysées en détail. Aucun historique précis de navigation ne peut donc être établi à partir des seuls fichiers identifiés.

### 4.5. Analyse d'entropie

Une analyse d'entropie a été réalisée pour identifier les zones présentant une distribution d'octets proche de données aléatoires.

```bash
binwalk -E -N forensic_key_1.img
```

Une mesure complémentaire a été effectuée avec Python.

Plusieurs régions de forte entropie ont été observées, notamment dans une partie de l'espace brut du support.

Une forte entropie peut être compatible avec du chiffrement, de la compression ou du remplissage pseudo-aléatoire. Elle ne constitue donc pas, à elle seule, une preuve de chiffrement.

### 4.6. Étude du volume TrueCrypt

Un volume de type TrueCrypt a été identifié à partir du secteur `7000000`, correspondant à l'offset :

```text
7000000 × 512 = 3584000000 octets
```

L'ouverture en lecture seule a été réalisée à l'aide d'un périphérique loop et de Cryptsetup.

```bash
sudo losetup -r -f --show -o $((7000000*512)) forensic_key_1.img
```

La commande retourne un périphérique, par exemple `/dev/loop2`.

```bash
sudo cryptsetup open --type tcrypt --readonly /dev/loop2 tcvol
sudo cryptsetup status tcvol
```

Le périphérique `/dev/loop2` doit être remplacé par celui réellement retourné par `losetup`.

L'ouverture a réussi avec une phrase de passe communiquée dans le cadre du TP. Son origine n'a pas été vérifiée indépendamment.

**Limite :** le contenu déchiffré du volume n'a pas été inventorié. Une incohérence entre la taille annoncée du volume et certaines observations d'entropie reste également à vérifier.

### 4.7. Recherche par mots-clés

Des recherches ont été effectuées avec Autopsy et sur des données textuelles extraites.

La recherche brute a produit **1 520 lignes** contenant certains termes liés au terrorisme.

Cependant, les éléments examinés comprennent notamment des commentaires de pages web et des entrées de dictionnaires.

Ce nombre ne correspond donc pas à 1 520 recherches effectuées par un utilisateur et ne permet pas d'attribuer une intention particulière.

## 5. Synthèse des résultats

| Élément | Résultat |
|---|---|
| Intégrité de l'image | Correspondance avec le SHA-256 de référence |
| Système principal | Support Ubuntu 12.04 Live |
| Partition déclarée | FAT32 |
| Fichier supprimé | `casper-rw`, récupéré et lisible |
| Structures supplémentaires | Détectées par TestDisk, à corroborer |
| Artefacts Firefox | Profil localisé sur WINSHARE |
| TrueCrypt | Logiciel identifié et volume ouvert |
| Contenu déchiffré | Non inventorié |
| Attribution à un utilisateur | Non établie |

## 6. Limites de l'investigation

Cette investigation ne permet pas de conclure sur l'identité de l'utilisateur, ses intentions ou la nature exacte de l'ensemble des données présentes.

Certaines analyses nécessiteraient des investigations complémentaires :

- Extraction et interrogation des bases SQLite du profil Firefox.
- Analyse du contenu du volume TrueCrypt.
- Vérification des structures détectées par TestDisk.
- Reconstitution chronologique des événements.
- Vérification de certaines limites et mesures d'entropie.

Les conclusions doivent être interprétées dans le périmètre des opérations réellement réalisées.

## 7. Rapport détaillé

Le document [`rapport_forensic.pdf`](rapport_forensic.pdf) contient les observations détaillées, les tableaux de résultats, les captures d'écran, les interprétations et les commandes utilisées pendant le TP.

## 8. Précautions

L'image disque est une pièce d'étude susceptible de contenir des données personnelles ou sensibles.

L'analyse doit être réalisée sur une copie de travail, en privilégiant les accès en lecture seule et la vérification des empreintes.

La diffusion de l'image doit rester conforme aux autorisations données dans le cadre pédagogique.

---

**Projet académique — Université Bretagne Sud — 2026/2027**
