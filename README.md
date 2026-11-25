# SIEVE / SPICE — reproduction et extension (classification de logs SIEM)

Projet notebook basé sur : Artioli et al., *SIEVE: Generating a cybersecurity log dataset collection for SIEM event
classification*, Computer Networks 266 (2025) 111330.

## Contenu
```
sieve_project/
├── SIEVE_Reproduction_Extension.ipynb   # phase 1 : reproduction + extensions (79 cellules)
├── SIEVE_Phase2_OpenWorld_Robustness.ipynb  # phase 2 : open-world (6 plis) + robustesse (47 cellules)
├── requirements.txt
├── README.md
├── data/                                # <- place tes fichiers de logs ici
└── outputs/                             # créé à l'exécution (résultats CSV, figures, config.json)
```

## Installation
```bash
python -m venv .venv
.venv\Scripts\activate            # Windows  (macOS/Linux : source .venv/bin/activate)
pip install -r requirements.txt   # puis installe torch selon ton GPU (voir requirements.txt)
jupyter lab
```

## Exécution
1. Copie tes logs dans `./data/`.
2. Ouvre le notebook, mets `QUICK_MODE = True` (cellule §0.1) et exécute **tout** pour vérifier que le format est bien détecté (quelques minutes).
3. Contrôle les sorties de §1 (colonnes détectées, datasets, rôle actif/réel) puis de l'EDA (§2).
4. Repasse `QUICK_MODE = False` et relance l'ensemble.

## Modèles
Dummy, Naive Bayes (baselines) · Logistic Regression · SVM (LinearSVC + SVC RBF) · RNN · LSTM · BERT Base · DistilBERT.

## Protocoles d'évaluation
- `paper_random` : split aléatoire 75/25 (protocole de l'article)
- `grouped` : split par groupes de logs quasi identiques (limite la fuite de gabarit)
- `real_test` : entraînement sur le synthétique, test sur les logs réels (dataset dont le nom contient `cds|real|combined`)

## Réglages utiles (cellule `CFG`)
| Situation | Réglage |
|---|---|
| Colonnes mal devinées | `TEXT_COL`, `LABEL_COL` |
| Fichiers `.log` : label mal déduit | `LABEL_FROM = "filename"` ou `"parent_folder"` |
| Lignes `label<SEP>log` | `LINE_SEPARATOR = "\t"` (ou `","`) |
| GPU 4 Go : erreur *out of memory* | `TF_BATCH = 8`, `TF_ACCUM = 4`, ou réduire `TF_MAX_TRAIN` |
| Trop long | réduire `TF_MAX_TRAIN`, `RNN_MAX_TRAIN`, `SVM_RBF_MAX_TRAIN` |
| Réglage de C (LR, LinearSVC) | `TUNE = True` |

## Limites connues
- La clé de groupe du protocole `grouped` est une **approximation** du gabarit d'origine.
- SVC RBF et Transformers sont entraînés sur des **sous-échantillons** par défaut (coût).
- Une seule graine (`SEED = 42`) ; pour le mémoire, répéter sur plusieurs graines.
- Le téléchargement de `bert-base-uncased` / `distilbert-base-uncased` exige Internet à la première exécution.

## Phase 2 — open-world et robustesse (un seul projet, même `data/`)
Le second notebook réutilise le chargeur et le prétraitement de la phase 1 (cellules identiques).

**Protocole open-world** : 30 classes → 6 groupes disjoints de 5 classes. À chaque pli, 25 classes connues (train/val/test) et
5 classes inconnues vues uniquement au test ; on recommence 6 fois (chaque classe est inconnue une fois). Deux façons de former
les plis : `random` (tirage aléatoire) et `family` (familles `file-*`, `process-*`… retirées ensemble : mode difficile).

| Volet | Contenu |
|---|---|
| A | détection de l'inconnu : MSP, MaxLogit, Energy, Mahalanobis, kNN ; AUROC, FPR@95 %TPR, open-set F1 ; analyse par classe |
| B | deux solutions de robustesse à budget constant : **S1** multi-entropie + bruit, **S2** mots+caractères + calibration ; évaluées en closed-set et en open-world (dans la distribution, entropie élevée, bruit) |
| C (optionnel) | LSTM en monde ouvert (nécessite PyTorch) |

Réglages utiles (cellule `CFG` de la phase 2) : `MAX_PER_CLASS`, `MAX_PER_CLASS_ROBUST` (taille), `B_FOLDS=[1,2]` (aller plus vite),
`FOLD_MODES=["random"]`, `RUN_DEEP=False`. `QUICK_MODE=True` valide tout en environ une minute.
Sorties : `outputs/ow_*.csv` et `outputs/figures/ow_*.png`.
