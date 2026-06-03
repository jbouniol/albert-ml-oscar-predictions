# Oscar Predictions — Applied ML for Business

Projet de Machine Learning applique : **predire les nominations aux Oscars** (objectif principal) et,
en bonus, **le gagnant parmi les nomines**, a partir de donnees combinees IMDb + scraping Wikipedia
des ceremonies + enrichissement via l'API TMDb. Perimetre : ceremonies **2000-2026**, **7 categories**.

## Presentation

> La presentation complete du projet (slides) est hebergee sur Canva.

[![Voir la presentation sur Canva](https://img.shields.io/badge/%F0%9F%8E%AC%20Voir%20la%20pr%C3%A9sentation-Canva-7D2AE8?style=for-the-badge&logo=canva&logoColor=white)](https://canva.link/je1tv6vtppm0v5v)

<sub>Lien direct : https://canva.link/je1tv6vtppm0v5v</sub>

## Equipe

- Anna
- Keira
- Robin
- Jonathan

## Objectif — une cible en deux etages

| | **Etage 1 — NOMINATION** (principal) | **Etage 2 — GAGNANT** (bonus) |
|---|---|---|
| Question | *Qui sera nomine ?* | *Qui gagnera parmi les nomines ?* |
| Cible | `nominated ∈ {0,1}` | `winner ∈ {0,1}` |
| Base-rate | ~1-3 % (aiguille dans une botte de foin) | 13-26 % |
| Metrique metier | **Precision@K** (K = nb de slots/an) | top-1 accuracy (argmax par groupe) |
| Notebook | `05_Nomination_modeling.ipynb` | `06_Winner_modeling.ipynb` |

**7 categories modelisees** : Best Picture, Best Director, Best Actor, Best Actress, Best Supporting
Actor, Best Supporting Actress, Best Original Screenplay. *(Best Animated Feature et Best Visual
Effects ont ete explores puis exclus : faible valeur marche + aucune spec technique exploitable.)*

## Le pipeline en 7 notebooks

Les notebooks se lisent dans l'ordre — ils racontent l'histoire du projet, de la collecte a la
modelisation :

| # | Notebook | Role |
|---|---|---|
| 00 | [`00_Scraping.ipynb`](Notebooks/00_Scraping.ipynb) | **Collecte** — scraping Wikipedia des ceremonies (1929-2026) |
| 01 | [`01_Merge_datasets.ipynb`](Notebooks/01_Merge_datasets.ipynb) | **Construction du dataset** — merge IMDb + Oscar + enrichissement TMDb → `oscar_imdb_merged.parquet` |
| 02 | [`02_EDA_visualisation.ipynb`](Notebooks/02_EDA_visualisation.ipynb) | **EDA** — visualisations du dataset |
| 03 | [`03_EDA_justification.ipynb`](Notebooks/03_EDA_justification.ipynb) | **EDA justificative** — NA, distributions, drift, set negatif |
| 04 | [`04_Model_justification.ipynb`](Notebooks/04_Model_justification.ipynb) | **Hypotheses de modeles** par categorie (menu enseigne) |
| 05 | [`05_Nomination_modeling.ipynb`](Notebooks/05_Nomination_modeling.ipynb) | **Etage 1 (principal)** — prediction des nominations |
| 06 | [`06_Winner_modeling.ipynb`](Notebooks/06_Winner_modeling.ipynb) | **Etage 2 (bonus)** — prediction du gagnant + interpretabilite |

## Donnees

Trois sources sont combinees dans `01_Merge_datasets.ipynb` pour produire le dataset canonique
`Data/Processed/oscar_imdb_merged.parquet` (+ `.csv` pour inspection) :

1. **Scraping Wikipedia** (`Data/Raw/Scraping/`) — nominations et winners de toutes les ceremonies,
   scrapes depuis les pages "Nth Academy Awards" via [`00_Scraping.ipynb`](Notebooks/00_Scraping.ipynb).
2. **IMDb Non-Commercial Datasets** (`Data/Raw/IMDb/`) — ratings, casting, equipe, metadata (~10 Go de TSV).
3. **TMDb API** (cache `Data/Processed/tmdb_cache.parquet`) — synopsis, keywords, budget, recette,
   langue originale, pays de production.

### Sources
- **IMDb Non-Commercial Datasets** : [developer.imdb.com/non-commercial-datasets](https://developer.imdb.com/non-commercial-datasets/)
- **Oscar Nominations** : scraping Wikipedia ([`00_Scraping.ipynb`](Notebooks/00_Scraping.ipynb))
- **TMDb API** : [themoviedb.org/settings/api](https://www.themoviedb.org/settings/api)

### Datasets generes (`Data/Processed/`)
- `oscar_imdb_merged.{parquet,csv}` — nominations reelles (positifs), ~2 400 lignes, 30 colonnes.
- `oscar_nomination_dataset.parquet` — **etage 1** : positifs + set negatif (stratégie B+), 7 categories, ~63k lignes.
- `benchmark_nomination_models.csv` — resultats du benchmark etage 1 (categorie × modele).

### Colonnes du dataset final
- **Identifiants** : `tconst` (IMDb), `nconst` (IMDb personne), `tmdb_id` (TMDb)
- **Oscar** : `year`, `ceremony_number`, `category`, `nominee`, `nominee_type` (film/person), `winner`
- **Film (IMDb)** : `film_title`, `film_original_title`, `film_year`, `runtime_minutes`, `genres`, `n_genres`
- **Popularite (IMDb)** : `imdb_rating`, `imdb_votes`
- **Equipe (IMDb)** : `directors`, `writers`, `n_cast`
- **Enrichissement (TMDb)** : `overview`, `tagline`, `release_date`, `original_language`, `keywords` (list), `production_countries` (list), `budget`, `revenue`, `tmdb_vote_average`, `tmdb_vote_count`

## Modelisation

### Etage 1 — nomination (notebook 05)
- **Set negatif "stratégie B+"** : univers d'eligibles "in conversation" (movie, 1999-2025, runtime ≥ 60,
  votes ≥ 10 000, rating ≥ 5.0, hors documentaire) ; tous les vrais nomines ré-injectes (couverture 100 %).
- **Anti-leakage** : uniquement des features connues *avant* les nominations ; validation `GroupKFold(5, year)`.
- **Roster = menu enseigne complet** : LogReg, Decision Tree, Random Forest, AdaBoost, XGBoost, LightGBM,
  Stacking + 3 baselines, benchmarkes sur les 7 categories.
- **Resultat** : le ML bat le hasard dans **les 7 categories** (lift ×13 a ×78), aucune baseline ne domine ;
  le meilleur modele varie par categorie (lineaire pour les categories "film", ensembles d'arbres pour
  les "personne").

### Etage 2 — gagnant (notebook 06)
- Parmi les nomines, prediction du `winner` (argmax par groupe `(categorie, annee)`), metrique top-1 accuracy.
- 5 modeles ML + 2 baselines par categorie, metriques par fold (top-1 ± std, PR-AUC, log-loss),
  auto-check anti-leakage, et bloc d'interpretabilite (permutation importance, SHAP).

## Installation

```bash
# Cloner le repo
git clone git@github.com:jbouniol/albert-ml-oscar-predictions.git
cd applied_ml_for_business

# Creer l'environnement virtuel
python -m venv .venv
source .venv/bin/activate

# Installer les dependances
pip install -r requirements.txt

# Configurer la cle TMDb (a creer sur themoviedb.org/settings/api)
echo "TMDB_API_KEY=ta_cle_ici" > .env
```

### Donnees IMDb
Non incluses (trop volumineuses). Telecharger les datasets depuis
[developer.imdb.com/non-commercial-datasets](https://developer.imdb.com/non-commercial-datasets/) et les
placer dans `Data/Raw/IMDb/`.

### Donnees Scraping
Les CSV/JSON Oscar sont commitees dans `Data/Raw/Scraping/` (fichiers legers). Pour les regenerer,
executer [`00_Scraping.ipynb`](Notebooks/00_Scraping.ipynb).

### Donnees TMDb
Le cache TMDb (`Data/Processed/tmdb_cache.parquet`) est genere automatiquement au premier run de la
section 6 de [`01_Merge_datasets.ipynb`](Notebooks/01_Merge_datasets.ipynb). Premier run : ~2 minutes
pour ~1100 films. Reruns instantanes grace au cache, reprise automatique en cas d'interruption.

## Structure du projet

```
├── .env                    # Cle API TMDb (gitignored)
├── Data/
│   ├── Raw/
│   │   ├── IMDb/           # TSV IMDb (gitignored, ~10 Go)
│   │   └── Scraping/       # CSV/JSON Oscar scrapes
│   └── Processed/          # Datasets generes (gitignored)
│       ├── tmdb_cache.parquet
│       ├── oscar_imdb_merged.{parquet,csv}
│       ├── oscar_nomination_dataset.parquet
│       └── benchmark_nomination_models.csv
├── Notebooks/
│   ├── 00_Scraping.ipynb              # Collecte : scraping des ceremonies Oscar
│   ├── 01_Merge_datasets.ipynb        # Merge IMDb + Oscar + enrichissement TMDb
│   ├── 02_EDA_visualisation.ipynb     # EDA / visualisations
│   ├── 03_EDA_justification.ipynb     # EDA justifiant les choix de modelisation
│   ├── 04_Model_justification.ipynb   # Hypotheses de modeles par categorie
│   ├── 05_Nomination_modeling.ipynb   # Etage 1 : prediction des nominations
│   └── 06_Winner_modeling.ipynb       # Etage 2 : prediction du gagnant
└── app/                    # Interface web Streamlit (optionnel)
```

## Licence

Projet academique. Les donnees IMDb sont utilisees dans le cadre d'un usage non-commercial conformement
aux [conditions IMDb](https://developer.imdb.com/non-commercial-datasets/). Les donnees TMDb sont
utilisees conformement aux [conditions TMDb](https://www.themoviedb.org/terms-of-use) — attribution
requise en cas de redistribution.
