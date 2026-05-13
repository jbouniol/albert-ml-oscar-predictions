# ORCHESTRATOR.md — Règles multi-agents (Oscar ML)

Règles d'orchestration pour ce projet. Les règles générales du projet sont dans `CLAUDE.md`.

---

## §1 — Orchestrator-Worker Pattern

Toute tâche multi-étapes suit ce schéma :

```
Orchestrateur (ce contexte principal)
  ├── Analyse la tâche → définit la stratégie → décompose en sous-tâches
  ├── Lance les sous-agents EN PARALLÈLE (jamais en série quand c'est possible)
  ├── Synthétise les résultats
  └── Décide si un tour supplémentaire est nécessaire

Sous-agents (workers)
  ├── Opèrent de façon indépendante dans leur périmètre
  ├── Exécutent 3+ tool calls en parallèle quand possible
  ├── Retournent des résultats filtrés et structurés (jamais de dump brut)
  └── Chacun a son propre contexte isolé
```

### Sélection du modèle par rôle

| Rôle | Modèle | Pourquoi |
|------|--------|----------|
| Orchestrateur | Opus | Planification stratégique, synthèse multi-sources |
| Worker spécialisé | Sonnet | Exécution focalisée sur un domaine précis |
| Extraction simple | Haiku | Pattern matching, formatage, lookups rapides |

---

## §2 — Effort Scaling

Calibrer le nombre d'agents à la complexité réelle. Ne pas sur-agenter les tâches simples.

| Complexité | Agents | Tool calls / agent | Quand l'utiliser |
|------------|--------|--------------------|-----------------|
| **Simple** | 1 | 3-10 | Lookup, fix d'un seul fichier, recherche rapide |
| **Modérée** | 2-4 | 10-15 chacun | Comparaison, changement multi-fichiers, investigation |
| **Complexe** | 5-10+ | Responsabilités divisées | Pipeline ML complet, feature engineering, modélisation |

**Règle** : Si un agent avec 3 tool calls peut résoudre le problème, ne pas en lancer cinq.

### Exemples pour ce projet

| Tâche | Complexité | Agents recommandés |
|-------|------------|-------------------|
| Corriger un bug dans un notebook | Simple | 1 |
| Explorer et nettoyer un nouveau dataset IMDb | Modérée | 2 (exploration + nettoyage) |
| Pipeline complet EDA → feature engineering → modèle | Complexe | 4-6 (un par étape majeure) |
| Comparer 3 algorithmes ML + rapport | Complexe | 3 workers parallèles + 1 synthèse |

---

## §3 — Enveloppe de tâche pour les sous-agents

Chaque sous-agent DOIT recevoir une enveloppe complète. Pas de délégations vagues.

| Champ | Requis | Description |
|-------|--------|-------------|
| **Objectif** | Oui | Ce qu'il faut trouver ou faire — précis et borné |
| **Format de sortie** | Oui | Comment structurer la réponse (tableau, liste, code) |
| **Outils / sources** | Oui | Quels outils utiliser, quelles sources prioriser |
| **Périmètre** | Oui | Ce qu'il ne faut PAS faire — évite les chevauchements |

**Mauvais** : "Analyse les données IMDb"  
**Bon** : "Dans `Data/Processed/oscar_imdb_merged.csv`, calcule la distribution des genres parmi les films nommés. Retourne un tableau genre → count → %. Ne modifie aucun fichier — un autre agent gère le feature engineering."

---

## §4 — Parallélisation

Grouper toutes les opérations indépendantes dans un seul message :

- Greps, lectures, éditions indépendants → parallèle
- Lancements d'agents indépendants → parallèle
- Opérations dépendantes → séquentiel

**Ne jamais sérialiser ce qui peut tourner en parallèle.**

---

## §5 — Protocole de communication

### Sous-agent → Orchestrateur
- Retourner des **résultats filtrés** avec compression intelligente
- Ne jamais dumper le output brut des outils — résumer et structurer
- Signaler l'incertitude : "Trouvé X mais impossible de confirmer Y"

### Orchestrateur → Utilisateur
- Aller droit au but : la réponse d'abord, pas le processus
- Partager les découvertes, pas la mécanique
- Calibrer la narration à la complexité : fix simple = "Fait." / investigation complexe = partager les étapes clés
