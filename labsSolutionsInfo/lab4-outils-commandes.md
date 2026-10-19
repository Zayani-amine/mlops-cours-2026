# Lab 4 — Packager le modèle de scoring en image Docker · outils & commandes

Ce lab transforme le modèle MLflow du Lab 1 (`scoring-model@champion`) en une image Docker portable, optimisée (< 500 Mo), scannée et prête pour la production. On y exporte le modèle en ONNX, on l'expose via une API FastAPI générique, on compare les latences scikit-learn vs ONNX, on produit une variante BentoML, on scanne images et modèles, puis on pousse l'image sur le registry local du cours.

## Outils de la stack

| Outil/service | Rôle / utilité |
|---|---|
| Docker Engine | Construit et exécute les images et conteneurs de serving. |
| Dockerfile multi-stage (BuildKit) | Sépare un stage « builder » (installation des dépendances) d'un stage « runtime » minimal, pour réduire la taille de l'image finale. |
| `.dockerignore` (`<Dockerfile>.dockerignore`) | Exclut fichiers et secrets du contexte de build ; la liste blanche du Dockerfile optimisé ne ré-inclut que le strict nécessaire. |
| ONNX / ONNX Runtime | Format de graphe statique (protobuf) et moteur d'inférence ; exécute le modèle sans scikit-learn ni MLflow. |
| skl2onnx | Convertit le pipeline scikit-learn (prétraitement inclus) en graphe ONNX. |
| MLflow Registry | Source du modèle (`models:/scoring-model@champion`) et de ses métadonnées (version, run_id, seuil). |
| SeaweedFS (S3) | Stockage d'artefacts du cluster, proxifié par le serveur MLflow : aucun identifiant S3 côté client. |
| FastAPI | Framework HTTP de l'API de scoring ; génère validation et doc OpenAPI (`/docs`). |
| uvicorn | Serveur ASGI qui sert l'application FastAPI. |
| Pydantic (`create_model`) | Construit dynamiquement le schéma d'entrée à partir de `dataset.yaml` (validation stricte). |
| BentoML | Alternative de serving : enregistre le modèle dans son model store et le sert/empaquette. |
| Locust | Test de charge de l'API (rejoue des lignes réelles de test). |
| Trivy | Scanne les images Docker à la recherche de vulnérabilités OS et libs (sévérités HIGH/CRITICAL). |
| ModelScan | Scanne les fichiers modèles à la recherche d'opérateurs dangereux (ex. `os.system` dans un pickle). |
| skops | Format de sérialisation de MLflow 3 ; reconstruit des objets Python uniquement dans une liste de types autorisés. |
| Hadolint | Linter de Dockerfile (versions épinglées, cache pip, etc.). |
| Registry local (`kind-registry`, `registry:3`) | Registry du cours sur `localhost:5001`, branché au cluster : les nœuds kind y tirent les images. |
| kubectl / cluster kind | Vérifie que le cluster voit et démarre l'image poussée (smoke test). |
| uv | Gestionnaire d'environnement : `uv sync`, `uv run` (socle incluant skl2onnx, onnxruntime, fastapi, bentoml, modelscan, locust). |

## Cibles Makefile et scripts

| Cible / script | Rôle |
|---|---|
| `make pf` | Lance les port-forwards du cluster : rend le serveur MLflow accessible sur `http://localhost:5000`. |
| `make cluster` | Démarre le cluster kind et le registry local `kind-registry` (`registry:3`) sur `localhost:5001`. |
| `make check-m4` | Vérification automatique du lab : modèle exporté, registry, image (< 500 Mo, non-root, HEALTHCHECK), conteneur, livrables. |
| `packaging/app/export_onnx.py` | Récupère `scoring-model@champion` et l'exporte en `model.onnx` + `metadata.json`. |
| `packaging/app/verify_onnx.py` | Compare les probabilités ONNX vs scikit-learn sur `data/processed/test.csv` (tolérance 1e-4). |
| `packaging/app/serve.py` | API FastAPI de scoring (seul script embarqué dans l'image). |
| `packaging/sample_request.py` | Fabrique un corps JSON d'exemple à partir de vraies lignes de test. |
| `packaging/app/benchmark.py` | Mesure P50/P95/P99 : scikit-learn vs ONNX (1 ligne / lot de 100) ; écrit `reports/benchmark.md`. |
| `packaging/locustfile.py` | Scénario de test de charge rejouant des lignes aléatoires de test. |
| `packaging/bento/save_model.py` | Copie le modèle MLflow exporté dans le model store BentoML. |
| `packaging/bento/service.py` | Variante BentoML de l'API (même route `/predict`, port 3000). |
| `packaging/bento/bentofile.yaml` | Description du Bento (service, dépendances, version Python, image Docker). |
| `packaging/scan-demo/make_pickles.py` | Fabrique deux pickles de démo (dont un piégé) pour illustrer ModelScan. |
| `packaging/Dockerfile.naive` | Dockerfile baseline volontairement naïf (grand et non sécurisé). |
| `packaging/Dockerfile` | Dockerfile multi-stage optimisé. |
| `packaging/requirements.txt` | Dépendances figées de l'image de serving uniquement. |
| `labs/m4-packaging/check.sh` | Script lancé par `make check-m4` (ne modifie rien, supprime son conteneur de test). |

## Commandes clés, expliquées

### Préparation et branche

- `cd ~/mlops-cours-2026` puis `set -a && source .env && set +a` : se placer à la racine du fork et exporter les variables d'environnement (dont `MLFLOW_TRACKING_URI`).
- `make pf` : établit les port-forwards, indispensable pour que MLflow réponde sur `localhost:5000`.
- `uv sync` : installe/synchronise l'environnement avec le socle du cours.
- `mkdir -p packaging/app packaging/model packaging/bento packaging/scan-demo reports` : crée l'arborescence de travail du lab.
- `git switch main && git pull` puis `git switch -c lab/m4-packaging` : crée la branche de travail dédiée depuis `main` à jour.

### 1.1 Vérifier le modèle du Lab 1

- `uv run python -c "from mlflow import MlflowClient; mv = MlflowClient().get_model_version_by_alias('scoring-model', 'champion'); print(mv.name, mv.version, mv.run_id)"` : confirme l'existence du champion et affiche sa version (≥ 7) et son `run_id`.

### 1.2 Exporter le modèle en ONNX

- `uv run python packaging/app/export_onnx.py` : charge `models:/scoring-model@champion`, le convertit avec skl2onnx (entrées dérivées de `dataset.yaml`) et écrit `packaging/model/model.onnx` + `metadata.json` (version, run_id, seuil, empreinte SHA-256).

### 1.3 Vérifier que ONNX = sklearn

- `uv run python packaging/app/verify_onnx.py` : compare les probabilités ONNX et scikit-learn sur tout le test set ; doit afficher `OK: predictions match.` sans `AssertionError`.

### 2.2 Test local de l'API

- `uv run python packaging/sample_request.py > /tmp/request.json` : génère une requête JSON valide depuis une vraie ligne de test ; `cat /tmp/request.json` l'affiche.
- `uv run python -m uvicorn app.serve:app --app-dir packaging --port 8000 &` : lance l'API localement ; `--app-dir packaging` ajoute `packaging/` au chemin d'import et `python -m uvicorn` garde la racine du repo importable.
- `sleep 3` puis `curl -s http://localhost:8000/health` : attend le démarrage et vérifie la sonde de vie (`{"status":"ok","model_loaded":true}`).
- `curl -s -X POST http://localhost:8000/predict -H "Content-Type: application/json" -d @/tmp/request.json` : envoie une demande et récupère probabilité + décision au seuil.
- `curl -s -X POST http://localhost:8000/predict ... -d '{"instances": [{"duration": "douze"}]}' | python3 -m json.tool | head -12` : provoque une erreur 422 détaillée (mauvais type, colonnes manquantes).
- `uv run python packaging/sample_request.py -n 3 | curl -s -X POST ... -d @-` : teste plusieurs demandes dans une seule requête (`instances` avec `-n 3`).
- `kill %1` : arrête le serveur local en arrière-plan.

### 3. Dockerfile naïf (baseline)

- `docker build -f packaging/Dockerfile.naive -t scoring-api:naive .` : build baseline depuis la racine du repo (contexte `.`), Dockerfile désigné par `-f`.
- `docker images scoring-api:naive --format "{{.Size}}"` : relève la taille (1,3 à 1,6 Go) pour le rapport.
- `docker run --rm scoring-api:naive id` : montre que le serveur tourne en `uid=0(root)`.
- `docker run --rm scoring-api:naive cat .env` : prouve que `COPY . /app` embarque les secrets du `.env`.

### 4. Dockerfile multi-stage optimisé

- `MODEL_VERSION=$(uv run python -c "import json; print(json.load(open('packaging/model/metadata.json'))['model_version'])")` : lit la version du modèle pour la passer en LABEL/`build-arg`.
- `docker build -f packaging/Dockerfile -t scoring-api:optimized --build-arg VERSION=1.0.0 --build-arg MODEL_VERSION="$MODEL_VERSION" .` : build l'image optimisée multi-stage avec métadonnées.
- `docker images scoring-api:optimized --format "{{.Size}}"` : vérifie la taille cible (< 500 Mo).
- `docker run -d --rm -p 8000:8000 --name scoring scoring-api:optimized` puis `sleep 5 && curl -s http://localhost:8000/health` : démarre le conteneur et teste la sonde.
- `curl -s -X POST http://localhost:8000/predict ... -d @/tmp/request.json` : vérifie que l'image renvoie la même prédiction qu'en local.
- `docker run --rm scoring-api:optimized id` : confirme l'utilisateur non-root (`uid=10001(ml)`).
- `docker inspect --format '{{.State.Health.Status}}' scoring` : lit l'état du HEALTHCHECK (`healthy`).
- `docker inspect --format '{{json .Config.Labels}}' scoring-api:optimized` : affiche les LABEL reliant l'image au Registry MLflow.
- `docker stop scoring` : arrête le conteneur.
- `docker run --rm -i hadolint/hadolint < packaging/Dockerfile` : lint le Dockerfile (versions épinglées, cache pip).

### 5. Benchmark et test de charge

- `uv run python packaging/app/benchmark.py` : compare scikit-learn et ONNX Runtime (P50/P95/P99) et écrit `reports/benchmark.md`.
- `docker run -d --rm -p 8000:8000 --name scoring scoring-api:optimized` : relance le conteneur pour le test de charge.
- `uv run locust -f packaging/locustfile.py --host http://localhost:8000 --users 100 --spawn-rate 10 --run-time 30s --headless` : lance 100 utilisateurs simulés en mode headless.
- `docker stop scoring` : arrête le conteneur.

### 6. Variante BentoML

- `uv run python packaging/bento/save_model.py` : enregistre le modèle (même version que l'image FastAPI + contrat d'entrée) dans le model store BentoML.
- `cd packaging/bento` puis `uv run bentoml build` : construit le Bento (`scoring_bento`).
- `uv run bentoml serve scoring_bento:latest --port 3000 &` puis `sleep 10` : sert le Bento sans Docker, sur le port 3000.
- `curl -s -X POST http://localhost:3000/predict ... -d @/tmp/request.json` puis `kill %1` : teste la même requête (réponse identique à 10⁻⁷ près) puis arrête le service.
- `uv run bentoml containerize scoring_bento:latest -t scoring-api:bentoml` : produit l'image Docker BentoML.
- `cd ../..` puis `docker images scoring-api --format "table {{.Repository}}:{{.Tag}}\t{{.Size}}"` : liste les trois images (`naive`, `optimized`, `bentoml`).

### 7. Scan sécurité (Trivy + ModelScan)

- `trivy() { docker run --rm -v /var/run/docker.sock:/var/run/docker.sock -v trivy-cache:/root/.cache/ aquasec/trivy:0.75.0 "$@"; }` : lance Trivy en conteneur (version épinglée, socket Docker monté, cache de base de vulnérabilités persistant).
- `trivy image --severity HIGH,CRITICAL scoring-api:optimized > reports/trivy_optimized.txt` : scanne l'image optimisée (idem pour `naive` et `bentoml`).
- `grep -c "CVE-" reports/trivy_*.txt` et `grep "^Total" reports/trivy_*.txt` : comptent les CVE et lisent les totaux par cible.
- `uv run modelscan -p packaging/model/model.onnx > reports/modelscan_onnx.txt` puis `cat` : scanne l'ONNX (`No issues found!`).
- `uv run modelscan -p packaging/model/model.onnx --show-skipped | tail -3` : montre que l'ONNX est ignoré (format non lu).
- `uv run python packaging/scan-demo/make_pickles.py` : fabrique `model.pkl` (modèle sérialisé) et `malicious.pkl` (charge cachée inoffensive).
- `uv run modelscan -p packaging/scan-demo/ > reports/modelscan_pickle.txt` puis `grep -A4 -- "--- CRITICAL ---" reports/modelscan_pickle.txt` : détecte l'opérateur `system` dans le pickle piégé.
- `uv run mlflow artifacts download -u models:/scoring-model@champion -d /tmp/champion` puis `ls /tmp/champion` : télécharge les artefacts du champion (`model.skops`).
- `uv run modelscan -p /tmp/champion` : montre que ModelScan ignore skops.
- `uv run python -c "from skops.io import get_untrusted_types; print(get_untrusted_types(file='/tmp/champion/model.skops'))"` : liste les types hors liste de confiance (`['numpy.dtype']`, autorisé au Lab 1).

### 8. Push sur le registry local + test pull

- `docker ps --filter name=kind-registry --format "{{.Names}}  {{.Image}}  {{.Ports}}"` et `curl -s http://localhost:5001/v2/_catalog` : confirment que le registry de cours tourne déjà (`registry:3` sur `localhost:5001`).
- `docker tag scoring-api:optimized localhost:5001/scoring-api:1.0.0` : tague l'image pour le registry local.
- `docker push localhost:5001/scoring-api:1.0.0` : pousse l'image.
- `curl -s http://localhost:5001/v2/_catalog` et `curl -s http://localhost:5001/v2/scoring-api/tags/list` : vérifient la présence du dépôt et du tag `1.0.0`.
- `docker rmi scoring-api:optimized localhost:5001/scoring-api:1.0.0` puis `docker pull localhost:5001/scoring-api:1.0.0` : simule un autre environnement (suppression locale + pull).
- `docker run -d --rm -p 8000:8000 --name scoring-from-reg localhost:5001/scoring-api:1.0.0` puis `sleep 5 && curl -s http://localhost:8000/health` et `docker stop scoring-from-reg` : prouve que l'image pullée fonctionne à l'identique.
- `kubectl run scoring-smoke --image=localhost:5001/scoring-api:1.0.0 --restart=Never` puis `kubectl wait --for=condition=Ready pod/scoring-smoke --timeout=90s` : vérifie que le cluster tire l'image depuis le registry local.
- `kubectl logs scoring-smoke | tail -2` puis `kubectl delete pod scoring-smoke` : lit les logs uvicorn puis nettoie le pod.

### 9. Rapport et commit

- `cp labs/m4-packaging/starter/rapport-m4.md reports/rapport-m4.md` : part du modèle de rapport à compléter (5 sections).
- `git add packaging reports ':(exclude)packaging/scan-demo/*.pkl'` : indexe le travail sans les pickles de démonstration.
- `git status` : vérifie qu'aucun `.pkl` ni `.env` n'est indexé.
- `git commit -m "Lab 4: packaging du modèle"`, `git push -u origin lab/m4-packaging`, `gh pr create` : commite, pousse et ouvre la PR (relecture par le partenaire du binôme).

## Pièges / points d'attention

- `skl2onnx: unsupported operator` / `MissingShapeCalculator` : le classifieur n'est pas supporté (CustomTransformer, XGBoost, LightGBM) ; rester sur un classifieur scikit-learn ou fournir un converter (`onnxmltools`).
- `NotImplementedError` sur `SimpleImputer` texte avec NaN : l'imputer de chaînes n'accepte pas NaN ; convertir via `onnx_ready()` (point (a) : `missing_values = ""`).
- `Divergence detected!` (ONNX ≠ sklearn) : entrées numériques en float32 — les arbres changent de branche ; utiliser `DoubleTensorType` + point (b) de `onnx_ready()` et passer des `float64`.
- `INVALID_ARGUMENT ... expected: (tensor(double))` : données envoyées en float32 à un graphe float64 ; convertir en `np.float64`.
- `Registered model alias champion not found` : Lab 1 non terminé ou port-forward MLflow arrêté ; relancer `make pf` et vérifier l'alias dans l'UI.
- `ModuleNotFoundError: No module named 'src'` : API lancée avec `uvicorn` au lieu de `python -m uvicorn`, ou hors de la racine ; lancer depuis la racine avec `--app-dir packaging`.
- `ImportError` sur `packaging.version` / pip / MLflow cassés : création accidentelle de `packaging/__init__.py` qui masque le paquet PyPI ; le supprimer.
- Ne créez jamais `packaging/__init__.py` : le dossier porterait le nom du paquet PyPI `packaging` et masquerait les imports.
- `docker build: no space left` : cache Docker trop gros ; `docker system prune -af --volumes` (attention : supprime aussi le volume `trivy-cache`).
- `model.onnx not found` au `COPY` : modèle non exporté ou build lancé depuis `packaging/` ; exporter puis builder depuis la racine.
- Image bloquée sur `pip install` : pas d'accès réseau au build ; vérifier le proxy Docker Desktop, pré-télécharger `python:3.12-slim`.
- `uvicorn: address already in use` : port 8000 occupé (server ou conteneur actif) ; `lsof -i:8000` + kill, ou changer de port.
- Trivy `unable to connect to Docker` : socket Docker non monté ; ajouter `-v /var/run/docker.sock:/var/run/docker.sock`.
- Trivy `TOOMANYREQUESTS` / base très lente : limite de débit ; garder `trivy-cache`, puis `--skip-db-update`.
- BentoML `model not found` : `save_model.py` jamais exécuté ; le relancer avant `bentoml build`.
- BentoML `AttributeError` sur `bentoml.Service`/`bentoml.io` : code écrit pour BentoML 1.2 ; utiliser `@bentoml.service` + `@bentoml.api`.
- `bentoml containerize : buildx introuvable` : plugin Docker `buildx` absent ; installer `docker-buildx-plugin`.
- `registry:2 : port is already allocated` : le registry du cours occupe déjà 5001 ; ne pas lancer de second registry, utiliser `kind-registry`.
- `docker push : connection refused` sur `localhost:5001` : conteneur `kind-registry` arrêté ; `docker start kind-registry` ou `make cluster`.
- Registry push `server gave HTTP response to HTTPS client` : registry désigné par un autre nom que `localhost` ; utiliser `localhost:5001` (sinon `insecure-registries`).
- `Container exit code 137` : OOM killer (RAM insuffisante) ; augmenter la RAM Docker Desktop.
- `FileNotFoundError` sur `model.onnx`/`dataset.yaml` au démarrage : fichier absent de l'image ; vérifier les `COPY` et la liste blanche du `.dockerignore`.
- ModelScan `argument of type 'NoneType' is not iterable` : il ne lit pas les pickles joblib ; sérialiser avec pickle (comme `make_pickles.py`).
- ModelScan `skipped` / code retour 3 : format non pris en charge (ONNX, skops) ; ce n'est pas une erreur.
- Ne chargez jamais `malicious.pkl` : `pickle.load`/`joblib.load` exécuteraient la charge au chargement ; ne pas commiter les `.pkl`.
- En prod, préférer ONNX (format statique, sans code) : charger un pickle revient à exécuter du code Python arbitraire ; jamais de pickle d'origine non maîtrisée.

*Fiche générée à partir de Lab4-Enonce.pdf et Lab4-Commandes.pdf — cours MLOps.*
