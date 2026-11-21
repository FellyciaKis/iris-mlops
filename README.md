# iris-mlops

Projet MLOps complet autour du jeu de données Iris : entraînement d'un modèle, API de prédiction, interface web, le tout conteneurisé avec Docker Compose et validé par une CI GitHub Actions.

**Frontend (Nginx) → API (FastAPI) → modèle (scikit-learn)**

## Architecture

| Service | Rôle | Port |
|---|---|---|
| `training` | Entraîne un RandomForest, vérifie le seuil de qualité, écrit le modèle dans `models/` puis s'arrête | – |
| `backend` | API FastAPI qui charge le modèle au démarrage et sert les prédictions | 8000 |
| `frontend` | Page web servie par Nginx, qui redirige `/api/` vers le backend | 8080 |

Ordre de démarrage : le training doit réussir (`service_completed_successfully`), puis l'API doit être en bonne santé (`service_healthy`), puis le frontend démarre.

## Structure du dépôt

```
iris-mlops/
├── .github/workflows/ci.yml   # CI : tests, build Docker, smoke test
├── training/                  # entraînement (train.py, tests, Dockerfile)
├── backend/                   # API FastAPI (app/, tests, Dockerfile)
├── frontend/                  # page web + nginx.conf + Dockerfile
├── models/                    # modèles entraînés (ignorés par Git)
├── docker-compose.yml
├── Makefile
└── README.md
```

## Prérequis

- WSL2 avec Ubuntu 24.04 (ou Linux)
- Docker et Docker Compose
- Git, et `make` pour les raccourcis

## Lancer le projet

```bash
export GIT_COMMIT=$(git rev-parse --short HEAD)
make up        # ou : docker compose up -d --build
make ps        # état des services
```

Attendu : `training` en `Exited (0)`, `backend` en `Up (healthy)`, `frontend` en `Up`. Ouvrez ensuite http://localhost:8080.

## Utiliser l'API

Documentation interactive : http://localhost:8000/docs

| Méthode | Route | Rôle |
|---|---|---|
| GET | `/health` | État de l'API et du modèle |
| GET | `/model/info` | Métadonnées du modèle en service |
| POST | `/predict` | Prédit l'espèce d'une fleur |
| POST | `/predict/batch` | Prédit l'espèce de 1 à 100 fleurs |

Exemple :

```bash
curl -s -X POST localhost:8000/predict \
  -H "Content-Type: application/json" \
  -d '{"sepal_length":6.7,"sepal_width":3.0,"petal_length":5.2,"petal_width":2.3}'
```

Les quatre mesures (en cm) doivent être comprises entre 0 et 10, sinon l'API répond 422.

## Ré-entraîner le modèle

```bash
N_ESTIMATORS=300 MAX_DEPTH=3 docker compose run --rm training
docker compose restart backend
curl -s localhost:8000/model/info
```

Le script applique un **quality gate** : si l'accuracy est inférieure à `MIN_ACCURACY` (0,90 par défaut), il s'arrête avec le code 1 et aucun modèle n'est publié. Le fichier `metadata.json` enregistre la version, le commit Git, les hyperparamètres et les métriques.

Configuration retenue : `n_estimators=300`, `max_depth=3`, accuracy 0,9667 sur 30 échantillons de test.

## Tests

```bash
make test      # tests du training et de l'API dans un conteneur Python 3.11
```

## Commandes utiles

| Commande | Effet |
|---|---|
| `make up` | Lance toute la stack |
| `make down` | Arrête la stack |
| `make logs` | Suit les journaux |
| `make train` | Ré-entraîne le modèle |
| `make clean` | Supprime conteneurs, images et modèles |

## Workflow Git et livraison

- `main` : version livrée, protégée (Pull Request et CI verte obligatoires).
- `develop` : branche d'intégration.
- `feature/*`, `ci/*`, `experiment/*` : une branche par sujet, fusionnée dans `develop` par Pull Request.
- Messages de commit : Conventional Commits (`feat`, `fix`, `test`, `docs`, `ci`, `chore`).
- Versions : tag `model-v1.0.0`, qui permet de reconstruire la même version avec `git checkout model-v1.0.0` puis `docker compose up -d --build`.

## Dépannage rapide

| Symptôme | Piste |
|---|---|
| `backend` reste en `Created` | Le training a échoué : `docker compose logs training` |
| Page web : « modèle absent » | `models/` est vide : `make train` puis `docker compose restart backend` |
| Port déjà utilisé | Arrêter l'autre programme ou changer le port dans `docker-compose.yml` |
