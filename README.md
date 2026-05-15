# Oscar Predictions - Applied ML for Business

Projet de Machine Learning applique : prediction des nominees et gagnants aux Oscars a partir de donnees IMDb, scraping Wikipedia des ceremonies, et enrichissement via l'API TMDb.

## Equipe

- Anna
- Keira
- Robin
- Jonathan

## Objectif

Predire les gagnants des Oscars a partir de donnees combinees issues de trois sources :

1. **Scraping Wikipedia** (`Data/Raw/Scraping/`) — nominations et winners de toutes les ceremonies (1929-2026), scrapees depuis les pages "Nth Academy Awards" via [`Notebooks/scraping.ipynb`](Notebooks/scraping.ipynb).
2. **IMDb Non-Commercial Datasets** (`Data/Raw/IMDb/`) — ratings, casting, equipe technique, metadata des films (~10 Go de TSV).
3. **TMDb API** (cache `Data/Processed/tmdb_cache.parquet`) — synopsis, keywords thematiques, budget, recette, langue originale, pays de production.

Le pipeline complet est dans [`Notebooks/EDA_merge.ipynb`](Notebooks/EDA_merge.ipynb) et produit `Data/Processed/oscar_imdb_merged.parquet` (+ `.csv` pour inspection).

## Donnees

### Sources
- **IMDb Non-Commercial Datasets** : [developer.imdb.com/non-commercial-datasets](https://developer.imdb.com/non-commercial-datasets/)
- **Oscar Nominations** : scraping Wikipedia (notebook fourni)
- **TMDb API** : [themoviedb.org/settings/api](https://www.themoviedb.org/settings/api)

### Perimetre
Films nomines aux Oscars entre 2000 et 2025 (~1100 films uniques, ~2400 nominations apres matching).

### Colonnes du dataset final
- **Identifiants** : `tconst` (IMDb), `nconst` (IMDb personne), `tmdb_id` (TMDb)
- **Oscar** : `year`, `ceremony_number`, `category`, `nominee`, `nominee_type` (film/person), `winner`
- **Film (IMDb)** : `film_title`, `film_original_title`, `film_year`, `runtime_minutes`, `genres`, `n_genres`
- **Popularite (IMDb)** : `imdb_rating`, `imdb_votes`
- **Equipe (IMDb)** : `directors`, `writers`, `n_cast`
- **Enrichissement (TMDb)** : `overview`, `tagline`, `release_date`, `original_language`, `keywords` (list), `production_countries` (list), `budget`, `revenue`, `tmdb_vote_average`, `tmdb_vote_count`

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

Les fichiers IMDb ne sont pas inclus dans le repo (trop volumineux). Telecharger les datasets depuis [developer.imdb.com/non-commercial-datasets](https://developer.imdb.com/non-commercial-datasets/) et les placer dans `Data/Raw/IMDb/`.

### Donnees Scraping

Les CSV/JSON Oscar sont commitees dans `Data/Raw/Scraping/` (fichiers legers). Pour les regenerer, executer [`Notebooks/scraping.ipynb`](Notebooks/scraping.ipynb).

### Donnees TMDb

Le cache TMDb (`Data/Processed/tmdb_cache.parquet`) est genere automatiquement au premier run de la section 6 de [`Notebooks/EDA_merge.ipynb`](Notebooks/EDA_merge.ipynb). Premier run : ~2 minutes pour ~1100 films. Reruns instantanes grace au cache. Reprise automatique en cas d'interruption.

## Structure du projet

```
├── .env                    # Cle API TMDb (gitignored)
├── Data/
│   ├── Raw/
│   │   ├── IMDb/           # TSV IMDb (gitignored, ~10 Go)
│   │   └── Scraping/       # CSV/JSON Oscar scrapes
│   └── Processed/          # Datasets generes (gitignored)
│       ├── tmdb_cache.parquet
│       └── oscar_imdb_merged.{parquet,csv}
├── Notebooks/
│   ├── scraping.ipynb      # Scraping des ceremonies Oscar
│   └── EDA_merge.ipynb     # Merge IMDb + Oscar + enrichissement TMDb
└── app/                    # Interface web Streamlit (optionnel)
```

## Licence

Projet academique. Les donnees IMDb sont utilisees dans le cadre d'un usage non-commercial conformement aux [conditions IMDb](https://developer.imdb.com/non-commercial-datasets/). Les donnees TMDb sont utilisees conformement aux [conditions TMDb](https://www.themoviedb.org/terms-of-use) — attribution requise en cas de redistribution.
