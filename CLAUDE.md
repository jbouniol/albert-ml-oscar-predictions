# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

> Règles d'orchestration multi-agents : voir [`ORCHESTRATOR.md`](ORCHESTRATOR.md)

## Projet

Projet académique ML : **predire les nominees et gagnants aux Oscars** a partir de donnees IMDb enrichies (box office, analyse sentimentale, bios acteurs, etc.). Livrable final : notebooks d'analyse + presentation, et une interface web (Streamlit ou autre).

- **Repo** : `git@github.com:jbouniol/albert-ml-oscar-predictions.git`
- **Scope** : Films nomines aux Oscars (2000-2022)
- **Langue du code** : Python, conventions en anglais (noms de variables, fonctions, commits)
- **Langue des commentaires/docs** : Francais
- **Equipe** : Anna, Keira, Robin, Jonathan

## Donnees

### IMDb Non-Commercial Datasets (`Data/Raw/IMDb/`)

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

**Important** : Ces fichiers font ~10 Go au total. Toujours filtrer tot (ex: `titleType == 'movie'`, `startYear >= 2000`) et utiliser la lecture par chunks ou l'optimisation des dtypes avec pandas.

### Donnees Oscar (a venir)

Un dataset supplementaire avec les nominations et victoires aux Oscars est necessaire — les donnees IMDb seules ne contiennent pas cette info. Sources probables : Kaggle, scraping de la base Academy Awards.

### Convention des repertoires Data

- `Data/Raw/` — fichiers sources non modifies (non commites, trop volumineux)
- `Data/Processed/` — datasets nettoyes, merges, feature-engineeres (generes par les notebooks)

## Structure du projet

```
applied_ml_for_business/
├── Data/
│   ├── Raw/IMDb/          # Fichiers TSV sources (gitignored)
│   └── Processed/         # Datasets generes (gitignored)
├── Notebooks/             # Jupyter notebooks (numerotes : 01_, 02_, etc.)
├── src/                   # Modules Python reutilisables (principe DRY)
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

# Lancer les notebooks
jupyter lab

# Lancer l'app Streamlit
streamlit run app/main.py
```

## Contraintes du projet

- **Principe DRY** : Extraire la logique repetee dans des modules `src/`. Exigence explicite du professeur.
- **Feature engineering obligatoire** : Ne pas supposer que les donnees brutes sont pretes pour le ML.
