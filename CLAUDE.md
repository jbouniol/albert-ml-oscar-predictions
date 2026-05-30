# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

> Règles d'orchestration multi-agents : voir [`ORCHESTRATOR.md`](ORCHESTRATOR.md)

## Projet

Projet académique ML : **predire les nominations aux Oscars** a partir de donnees IMDb et TMDb enrichies. Objectif en **deux etages** :
1. **Etage principal — NOMINATION** : predire *qui sera nomine* dans chaque categorie (`nominated ∈ {0,1}`, metrique metier **Precision@K**, K = nb de slots/an).
2. **Etage bonus — GAGNANT** : parmi les nomines, predire *le gagnant* (`winner ∈ {0,1}`, **top-1 accuracy** = argmax par groupe `(categorie, annee)`).

Livrable final : notebooks d'analyse + presentation, et une interface web (Streamlit ou autre).

- **Repo** : `git@github.com:jbouniol/albert-ml-oscar-predictions.git`
- **Scope** : nominations Oscar **2000-2026**, **7 categories** modelisees — Best Picture, Director, Actor, Actress, Supporting Actor, Supporting Actress, Original Screenplay. *Best Animated Feature* et *Best Visual Effects* explores puis **exclus** du livrable (faible valeur marche + aucune spec technique exploitable dans IMDb/TMDb).
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

### 4. Set negatif — dataset de nomination (`Data/Processed/oscar_nomination_dataset.parquet`)

`oscar_imdb_merged.parquet` ne contient **que des positifs** (2 427 nominations). Pour l'**etage 1 (nomination)**, on construit un **set negatif** = candidats *eligibles non nomines* (cf. `EDA_merge.ipynb §8`).

**Stratégie B+** (eligibilite "in conversation", logique cinephile + data-driven) :
- **Films** (Best Picture, Original Screenplay) : `titleType == movie`, `startYear ∈ [1999, 2025]`, `runtimeMinutes ≥ 60`, `numVotes ≥ 10 000`, `averageRating ≥ 5.0`, genre ≠ Documentary → ~290 films/an.
- **Personnes** (Director, Actor/Actress lead & supporting) : acteurs/realisateurs des films eligibles via `title.principals` (lead = `ordering ≤ 2`, supporting = `ordering 3-5`, genre via `category` actor/actress).

**Regles cles** :
- Match positif sur le **triplet** `(nconst, year, tconst)` — le film *precis* de la nomination (sinon les autres roles de la personne la meme annee seraient faussement etiquetes nomines).
- **Re-injection de TOUS les positifs reels** (fallback features depuis `oscar_imdb_merged`) → aucun vrai nomine perdu, meme si son film est sous le seuil d'eligibilite.
- **Anti-leakage** : seules des features connues *avant* les nominations (note, votes, duree, genres, decennie, historique Oscar de la personne `n_prior_noms`, billing). `film_n_total_noms` est **exclu** (post-nomination).

Colonnes : `tconst`, `nconst`, `year`, `nominated ∈ {0,1}`, `category`, `kind` (film/person) + features. ~62 850 lignes, base-rate ~1-3 % selon categorie.

### Convention des repertoires Data

- `Data/Raw/` — fichiers sources non modifies (non commites, trop volumineux)
- `Data/Processed/` — datasets nettoyes, merges, feature-engineeres (generes par les notebooks)
- Caches internes (prefixe `_`, gitignored) : `_elig_films_broad.parquet`, `_principals_elig.parquet` — accelerent les reruns du set negatif sans relire les TSV bruts.

## Modelisation — objectif en 2 etages

| | **Etage 1 — NOMINATION** (principal) | **Etage 2 — GAGNANT** (bonus) |
|---|---|---|
| Notebook | `EDA_merge.ipynb §8` | `Model_experimentation.ipynb` |
| Unite | 1 candidat eligible × (categorie, annee) | 1 nomine × (categorie, annee) |
| Cible | `nominated ∈ {0,1}` | `winner ∈ {0,1}` |
| Negatifs | candidats eligibles non nomines (set negatif) | nomines perdants (deja dans le dataset) |
| Base-rate | ~1-3 % | 13-26 % |
| Metrique | **Precision@K** (K = nb de slots/an) + PR-AUC | **top-1 accuracy** (argmax/groupe) + PR-AUC + log-loss |
| Validation | `GroupKFold(5, groups=year)` — pas de split aleatoire, anti-leakage temporel |

- **1 modele independant par categorie** (chaque categorie a sa propre logique de vote).
- 5 modeles ML candidats + 2 baselines (random, most-nominated) ; justification par categorie dans `Model justification.ipynb`.
- Finding : les **seconds roles** se predisent mieux que les premiers (veteran dans film acclame = signal net) ; pour certaines categories la **baseline reste imbattable** (ex. Best Actor Supporting) — on l'assume.

## Structure du projet

```
applied_ml_for_business/
├── .env                   # Cle API TMDb (gitignored)
├── Data/
│   ├── Raw/
│   │   ├── IMDb/          # Fichiers TSV sources (gitignored, ~10 Go)
│   │   └── Scraping/      # CSV/JSON scrapes depuis Wikipedia
│   └── Processed/         # Datasets generes (gitignored)
│       ├── tmdb_cache.parquet            # Cache des appels API TMDb
│       ├── oscar_imdb_merged.*           # Dataset des nominations (positifs) — parquet + csv
│       ├── oscar_nomination_dataset.parquet  # Etage 1 : positifs + set negatif (7 cats)
│       └── _elig_films_broad / _principals_elig.parquet  # caches internes
├── Notebooks/
│   ├── scraping.ipynb              # Scraping Wikipedia des nominations Oscar
│   ├── EDA_merge.ipynb             # Pipeline merge IMDb+Oscar+TMDb + §8 set negatif (etage 1)
│   ├── 01_EDA_visualisation.ipynb  # EDA / visualisations du dataset
│   ├── EDA Justification.ipynb     # EDA justifiant les choix de modelisation
│   ├── Model justification.ipynb   # Hypotheses de modeles par categorie (7 cats)
│   └── Model_experimentation.ipynb # Etage 2 (bonus) : prediction du gagnant + interpretabilite
├── tasks/                 # todo.md (plan) + lessons.md (boucle d'auto-amelioration)
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
