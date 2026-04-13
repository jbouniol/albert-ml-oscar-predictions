# Oscar Predictions - Applied ML for Business

Projet de Machine Learning applique : prediction des nominees et gagnants aux Oscars.

## Equipe

- Anna
- Keira
- Robin
- Jonathan

## Objectif

Predire les gagnants des Oscars a partir de donnees combinees :
- Donnees IMDb (ratings, casting, equipe technique, metadata films)
- Donnees de nominations/victoires aux Oscars (a venir)
- Features supplementaires envisagees : box office, analyse sentimentale, biographies

## Donnees

### Sources
- **IMDb Non-Commercial Datasets** : [developer.imdb.com/non-commercial-datasets](https://developer.imdb.com/non-commercial-datasets/)
- **Oscar Nominations** : a venir

### Perimetre
Films nomines aux Oscars entre 2000 et 2022.

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
```

### Donnees IMDb

Les fichiers IMDb ne sont pas inclus dans le repo (trop volumineux). Telecharger les datasets depuis [developer.imdb.com/non-commercial-datasets](https://developer.imdb.com/non-commercial-datasets/) et les placer dans `Data/Raw/IMDb/`.

## Structure du projet

```
├── Data/
│   ├── Raw/           # Donnees brutes (non commitees)
│   └── Processed/     # Donnees traitees
├── Notebooks/         # Notebooks d'analyse et de modelisation
├── src/               # Modules Python reutilisables
└── app/               # Interface web
```

## Licence

Projet academique. Les donnees IMDb sont utilisees dans le cadre d'un usage non-commercial conformement aux [conditions IMDb](https://developer.imdb.com/non-commercial-datasets/).
