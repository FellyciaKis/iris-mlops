# iris-mlops

TP MLOps : entraînement d'un modèle Iris, API FastAPI, frontend Nginx, le tout orchestré avec Docker Compose.

## Lancer le projet
    make up
    # puis ouvrir http://localhost:8080

## Commandes utiles
    make ps      # état des services
    make logs    # journaux
    make train   # ré-entraîner le modèle
    make test    # lancer les tests
    make down    # arrêter
