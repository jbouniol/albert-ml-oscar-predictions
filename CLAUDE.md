# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

> Règles d'orchestration multi-agents : voir [`ORCHESTRATOR.md`](ORCHESTRATOR.md)

## Projet

Projet académique ML : **predire les nominees et gagnants aux Oscars** a partir de donnees IMDb et TMDb enrichies. Livrable final : notebooks d'analyse + presentation, et une interface web (Streamlit ou autre).

- **Repo** : `git@github.com:jbouniol/albert-ml-oscar-predictions.git`
- **Scope** : Films nomines aux Oscars (2000-2025)
- **Langue du code** : Python, conventions en anglais (noms de variables, fonctions, commits)
- **Langue des commentaires/docs** : Francais
- **Equipe** : Anna, Keira, Robin, Jonathan

## Donnees

Trois sources sont combinees dans `Notebooks/EDA_merge.ipynb` pour produire le dataset final `Data/Processed/oscar_imdb_merged.parquet` (+ `.csv` pour inspection).

### 1. Scraping Oscar — Wikipedia (`Data/Raw/Scraping/`)

Source : scraping des pages Wikipedia "Nth Academy Awards" (cf. [`Notebooks/scraping.ipynb`](Notebooks/scraping.ipynb)).

| Fichier | Contenu | Lignes |
|---------|---------|--------|
| `all_data_oscars.csv` | Une ligne par categorie × ceremonie : winner + nominees separes par `\|` | 2194 (1929-2026) |
| `all_data_oscars.json` | Meme contenu, format JSON | idem |

Colonnes : `ceremony_number`, `year`, `date`, `url`, `category`, `winner`, `nominees`.

**Important** : 119 categories distinctes historiquement — le notebook les normalise (regroupe les variantes : "Best Actor" ↔ "Best Actor in a Leading Role", "Best Art Direction" ↔ "Best Production Design", etc.) et passe le dataset en format long (une ligne par nominee, colonne `winner` booleenne).

### 2. IMDb Non-Commercial Datasets (`Data/Raw/IMDb/`)

Source : [IMDb developer datasets](https://developer.imdb.com/non-commercial-datasets/). Fichiers TSV, separateur tabulation, `\N` pour les valeurs manquantes.

| Fichier | Colonnes cles | Taille |
|---------|--------------|--------|
| `title.basics.tsv` | tconst, titleType, primaryTitle, startYear, runtimeMinutes, genres | ~1 GB |
| `title.ratings.tsv` | tconst, averageRating, numVotes | ~29 MB |
| `title.principals.tsv` | tconst, nconst, category, characters | ~4.2 GB |
| `title.crew.tsv` | tconst, directors, writers | ~410 MB |
| `name.basics.tsv` | nconst, primaryName, birthYear, primaryProfession, knownForTitles | ~940 MB |
| `title.akas.tsv` | tconst, title, region, language | ~2.8 GB |
| `title.episode.tsv` | tconst, parentTconst, seasonNumber, episodeNumber | ~250 MB |

**Important** : Ces fichiers font ~10 Go au total. Toujours filtrer tot (ex: `titleType == 'movie'`, `startYear >= 2000`) et utiliser la lecture par chunks ou l'optimisation des dtypes avec pandas. Le matching Oscar ↔ IMDb se fait par titre de film (categories "film") et par nom de personne via `title.principals` (categories "personne").

### 3. TMDb API (`Data/Processed/tmdb_cache.parquet`)

Source : [The Movie Database API](https://developer.themoviedb.org/). Enrichissement par appel API (lookup par `tconst` IMDb, 2 calls par film : `/find/{tconst}` puis `/movie/{tmdb_id}?append_to_response=keywords`).

**Cle API** : stockee dans `.env` a la racine (gitignored). Variable : `TMDB_API_KEY`. Quota journalier supprime par TMDb en 2023, seul le rate limit (~50 req/s) subsiste.

**Volumes** : ~1097 films uniques × 2 calls = ~2200 requetes, ~2 min au premier run. Cache parquet ecrit incrementalement (toutes les 50 iterations) → reprise automatique apres interruption, reruns instantanes.

| Colonne | Type | Contenu |
|---------|------|---------|
| `tmdb_id` | int | Identifiant TMDb (pivot avec `/movie/{id}`) |
| `overview` | str | Synopsis court (1-3 phrases) |
| `tagline` | str | Phrase marketing du film |
| `release_date` | str (YYYY-MM-DD) | Date de sortie complete (vs `film_year` IMDb seul) |
| `original_language` | str (ISO 639-1) | Langue originale ("en", "ko", "fr"...) |
| `keywords` | list[str] | Mots-cles thematiques (10-30 par film, ex: "dystopia", "father son relationship") |
| `production_countries` | list[str] (ISO 3166-1) | Pays de production |
| `budget` | int (USD) | Budget de production (0 si inconnu) |
| `revenue` | int (USD) | Recette mondiale (0 si inconnu) |
| `tmdb_vote_average` | float | Note TMDb (0-10) |
| `tmdb_vote_count` | int | Nombre de votes TMDb |
| `error` | str / null | `null` si OK, sinon raison (`not_found_on_tmdb`, message HTTP, etc.) |

### Convention des repertoires Data

- `Data/Raw/` — fichiers sources non modifies (non commites, trop volumineux)
- `Data/Processed/` — datasets nettoyes, merges, feature-engineeres (generes par les notebooks)

## Structure du projet

```
applied_ml_for_business/
├── .env                   # Cle API TMDb (gitignored)
├── Data/
│   ├── Raw/
│   │   ├── IMDb/          # Fichiers TSV sources (gitignored, ~10 Go)
│   │   └── Scraping/      # CSV/JSON scrapes depuis Wikipedia
│   └── Processed/         # Datasets generes (gitignored)
│       ├── tmdb_cache.parquet      # Cache des appels API TMDb
│       └── oscar_imdb_merged.*     # Dataset final (parquet + csv)
├── Notebooks/
│   ├── scraping.ipynb     # Scraping Wikipedia des nominations Oscar
│   └── EDA_merge.ipynb    # Pipeline complet : merge IMDb + Oscar + enrichissement TMDb
└── app/                   # Interface web Streamlit (optionnel)
```

## Workflow

### 1. Planifier d'abord
- Passer en mode plan pour toute tache non triviale (3+ etapes)
- Ecrire le plan dans `tasks/todo.md` avant d'implementer
- Si quelque chose ne va pas, STOP et re-planifier — ne jamais forcer

### 2. Strategie sous-agents
- Utiliser des sous-agents pour garder le contexte principal propre
- Une tache par sous-agent
- Investir plus de compute sur les problemes difficiles

### 3. Boucle d'auto-amelioration
- Apres toute correction : mettre a jour `tasks/lessons.md`
- Format : `[date] | ce qui a mal tourne | regle pour l'eviter`
- Relire les lecons a chaque demarrage de session

### 4. Standard de verification
- Se demander : "Est-ce qu'un staff engineer validerait ca ?"

### 5. Exiger l'elegance
- Pour les changements non triviaux : existe-t-il une solution plus elegante ?
- Si un fix semble bricole : le reconstruire proprement
- Ne pas sur-ingenieriser les choses simples

### 6. Correction de bugs autonome
- Quand on recoit un bug : le corriger directement
- Aller dans les logs, trouver la cause racine, resoudre
- Pas besoin d'etre guide etape par etape

## Principes fondamentaux

- Simplicite d'abord — toucher un minimum de code
- Pas de paresse — causes racines uniquement, pas de fixes temporaires
- Ne jamais supposer — verifier chemins, APIs, variables avant utilisation
- Demander une seule fois — une question en amont si necessaire, ne jamais interrompre en cours de tache

## Gestion des taches

1. Planifier → `tasks/todo.md`
2. Verifier → confirmer avant d'implementer
3. Suivre → marquer comme termine au fur et a mesure
4. Expliquer → resume de haut niveau a chaque etape
5. Apprendre → `tasks/lessons.md` apres corrections

## Developpement

```bash
# Creer l'environnement virtuel
python -m venv .venv && source .venv/bin/activate

# Installer les dependances
pip install -r requirements.txt

# Configurer la cle TMDb (a creer sur themoviedb.org/settings/api)
echo "TMDB_API_KEY=ta_cle_ici" > .env

# Lancer les notebooks
jupyter lab

# Lancer l'app Streamlit
streamlit run app/main.py
```

## Contraintes du projet

- **Principe DRY** : Extraire la logique repetee en fonctions reutilisables (helpers definis dans le notebook). Exigence explicite du professeur.
- **Feature engineering obligatoire** : Ne pas supposer que les donnees brutes sont pretes pour le ML.
- **Caching des appels API** : Toute requete TMDb doit etre cachee sur disque pour resilience et reproductibilite (cf. `tmdb_cache.parquet`).
