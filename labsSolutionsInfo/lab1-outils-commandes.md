# Lab 1 — Versionner un projet ML end-to-end · outils & commandes

Ce lab pose le projet fil rouge du cours : un modèle de *credit scoring* que l'on fera grandir de module en module. Il s'agit de versionner le projet de bout en bout — données avec DVC, 20+ runs tracés et comparés dans MLflow, promotion d'un modèle dans le Model Registry, features servies par Feast. Tout se fait à la racine du fork, sans nouveau dépôt.

## Outils de la stack

| Outil/service | Rôle / utilité |
| --- | --- |
| Git | Versionne le code, `dataset.yaml`, `params.yaml`, `dvc.yaml`, `dvc.lock` et les pointeurs DVC. C'est ce qui se partage entre postes. |
| DVC | Versionne les données et le modèle (contenu dans un remote S3, pointeurs dans Git). Décrit le pipeline `download → prepare → train → evaluate` et sait quand relancer une étape (`dvc repro`). |
| SeaweedFS (S3) | Stockage objet du cours (remplace MinIO). Héberge le bucket `dvc-store/scoring` pour les données versionnées et `mlflow-artifacts` pour les artefacts MLflow. |
| MLflow (tracking) | Serveur unique du cours : enregistre les runs (paramètres, métriques, tags, modèles) dans l'expérience `scoring`. Backend store SQLite sur volume persistant, artefacts en S3 via proxy. |
| MLflow Model Registry | Versionne les modèles promus (`scoring-model`) et gère les alias `@champion` / `@challenger` (remplacent les anciens « stages »). |
| Feast | Feature store local : une seule définition des features, deux accès (hors ligne pour l'entraînement, en ligne pour le serving) afin d'éviter le *training/serving skew*. |
| scikit-learn | Fournit le prétraitement (`ColumnTransformer`) et les estimateurs (régression logistique, forêts, gradient boosting, dummy). |
| Python 3.12 + uv | Environnement du cours ; le venv (`uv sync`) contient les dépendances et est réactivé à chaque séance. |
| kind / kubectl | Cluster Kubernetes local hébergeant SeaweedFS et MLflow (`kubectl get nodes` doit montrer 3 nœuds Ready). |
| Make | Raccourcis d'infrastructure et de vérification (`make up`, `make pf`, `make storage`, `make check-m1`…). |
| aws-lab | Client S3 du cours pour inspecter le contenu des buckets (`aws-lab s3 ls`). |
| GitHub CLI (`gh`) | Ouvre la pull request du lab (facultatif, alternative web). |

## Cibles Makefile et scripts

| Cible / script | Rôle |
| --- | --- |
| `make up` | Démarre la stack du cluster (utilisé sur un poste neuf). |
| `make pf` | Lance les port-forwards : S3 `:8333` et MLflow `:5000`. À vérifier en début de séance. |
| `make storage` | Crée les buckets S3 du cours (idempotent) ; nécessaire avant un `dvc push`. |
| `make lint` | Vérifie le style du code (PEP 8). Utilisé dans les critères de propreté du code. |
| `make check-m0` | Vérifie le prérequis (Lab 0 : cluster, SeaweedFS, MLflow) ; doit être entièrement vert. |
| `make check-m1` | Vérifie automatiquement tout le Lab 1 (contrat, DVC, MLflow, registry, Feast) sans rien modifier. |
| `make check-mX` | Vérification générique de l'état des modules/labs. |
| `src/download.py` | Télécharge le dataset UCI (repli OpenML), décode les catégories et écrit le CSV décrit par `raw_path`. |
| `src/dataset_config.py` | Lit et valide `dataset.yaml`, expose une API typée (features, cible, split, préprocesseur, coût métier, seuil, groupes sensibles). |
| `src/prepare.py` | Crée l'identifiant de demande, fait le split stratifié et écrit `train.csv`, `test.csv` et la source hors ligne Feast `features.parquet`. |
| `src/train.py` | Entraîne un modèle (Pipeline complet) et trace le run MLflow (params, métriques, tags, modèle + signature). |
| `src/grid_search.py` | Lance 22 runs (un sous-processus par run) pour comparer les configurations dans MLflow. |
| `src/promote.py` | Enregistre un run dans le Model Registry et pose les alias `@challenger` / `@champion` selon le seuil de qualité. |
| `src/evaluate.py` | Relit le modèle local et écrit `reports/metrics.json`, versionné et comparable via DVC. |
| `features/definitions.py` et `features/feature_store.yaml` | Définitions Feast (entité, source, vue, service) construites depuis `dataset.yaml`. |

## Commandes clés, expliquées

### Début de séance et branche de travail

- `cd ~/mlops-cours-2026` — se placer à la racine du fork : toutes les commandes s'y exécutent.
- `kubectl get nodes` — vérifie que le cluster kind est actif (3 nœuds Ready).
- `make pf` — ouvre les port-forwards S3 (`:8333`) et MLflow (`:5000`) vers le cluster.
- `source .venv/bin/activate` — active le venv du cours.
- `set -a && source .env && set +a` — charge les variables d'environnement (dont `MLFLOW_TRACKING_URI`) en les exportant.
- `git switch main && git pull` — part d'un `main` à jour.
- `git switch -c lab/m1-versioning` — crée la branche de travail du lab (convention du Lab 0).

### Étape 1 — Contrat de données et téléchargement

- `python -m src.download` — télécharge et écrit le CSV brut ; lancé en module pour trouver le paquet `src` sans bricoler le `PYTHONPATH`.
- `head -3 data/raw/dataset.csv` — affiche l'en-tête et les premières lignes pour contrôler les colonnes.
- `wc -l data/raw/dataset.csv` — compte les lignes (1 000 demandes + 1 en-tête attendus).
- `python - <<'EOF' … EOF` — exploration rapide : dimensions, valeurs manquantes, déséquilibre de la cible et taux de *bad* par groupe sensible.
- `python -c "from src.dataset_config import load_config; print(load_config())"` — valide le contrat de données.
- `uv run pytest tests/test_dataset_config.py` — exécute les tests du contrat (utile si dataset personnalisé).

### Étape 2 — Versionner les données avec DVC

- `dvc init` — initialise DVC dans le fork (crée `.dvc/`, `.dvc/config` et `.dvcignore`).
- `git status --short` — montre les nouveaux fichiers DVC à commiter.
- `dvc stage add -n download -d src/download.py -d src/dataset_config.py -p dataset.yaml:raw_path,target,features -o data/raw/dataset.csv python -m src.download` — déclare la stage `download` : `-d` = dépendances de code, `-p` = clés de paramètres surveillées, `-o` = sortie versionnée et ignorée par Git.
- `cat dvc.yaml` — relit la définition de la stage écrite par DVC.
- `dvc repro` — exécute la stage : DVC calcule les hashes et génère `dvc.lock`.
- `cat dvc.lock` — vérifie le hash des dépendances, les valeurs des paramètres et l'empreinte MD5 de la sortie.
- `dvc remote add -d storage s3://dvc-store/scoring` — enregistre et définit le remote par défaut du projet.
- `dvc remote modify storage endpointurl http://localhost:8333` — pointe le remote vers le SeaweedFS du cluster.
- `dvc remote modify --local storage access_key_id mlops` (et `secret_access_key mlops-lab-secret`) — stocke les identifiants hors Git, uniquement dans `.dvc/config.local`.
- `cat .dvc/config` / `cat .dvc/config.local` — vérifie que la config publique ne contient pas les secrets.
- `dvc push` — pousse les données vers S3 (rangées sous leur empreinte MD5).
- `aws-lab s3 ls --recursive s3://dvc-store/scoring/` — confirme la présence des objets dans le bucket.
- `git add .dvc/config .dvc/.gitignore .dvcignore dvc.yaml dvc.lock data/raw/.gitignore` — commite les pointeurs et la config DVC, pas les données.

### Étape 3 — Vérifier le serveur MLflow

- `echo $MLFLOW_TRACKING_URI` — affiche l'URL du serveur MLflow utilisée par le client.
- `curl http://localhost:5000/health` — vérifie que le serveur répond (`OK`).
- `env -u AWS_ACCESS_KEY_ID -u AWS_SECRET_ACCESS_KEY python -c "import mlflow; print([e.name for e in mlflow.search_experiments()])"` — prouve que le client MLflow n'a besoin que de l'URL, pas des identifiants S3 (le serveur fait proxy).

### Étape 4 — Préparation et entraînement instrumenté

- `python -m src.prepare` — crée l'identifiant de demande, le split stratifié et la source hors ligne Feast.
- `ls data/processed/` — vérifie la présence de `features.parquet`, `test.csv` et `train.csv`.
- `python -m src.train` — smoke test d'entraînement avec les paramètres par défaut de `params.yaml`.
- `python -m src.train --model random_forest --n_estimators 50 --max_depth 5` — smoke test avec surcharge d'hyper-paramètres (options de la CLI prioritaires sur `params.yaml`).
- `git add params.yaml src/prepare.py src/train.py` — commite les scripts et paramètres (jamais les données `data/processed/` ni `models/`).

### Étape 5 — Grid search de 20+ runs

- `python -m src.grid_search` — lance les 22 runs (dummy, régression logistique, forêts, gradient boosting) dans l'expérience `scoring` ; chaque run est isolé et tagué `launched_by=grid_search`.
- `git add src/grid_search.py && git commit -m "feat: grid search (22 runs)"` — commite le script de grid search.

### Étape 6 — Comparer et sélectionner dans MLflow

- UI `http://localhost:5000`, expérience `scoring` — explorer les runs (24 attendus), ajouter les colonnes `metrics.roc_auc`, `metrics.business_cost`, `tags.launched_by` et trier par `roc_auc`.
- `Compare` sur 3 runs (un de chaque type), vues `Parameters`, `Metrics`, `Parallel Coordinates Plot` — comprendre quels paramètres influencent le ROC AUC puis le coût métier.
- `python - <<'EOF' … client.search_runs([exp.experiment_id], order_by=["metrics.roc_auc DESC"], max_results=5) … EOF` — identifie les 5 meilleurs runs en CLI, triés par métrique primaire.

### Étape 7 — Promouvoir dans le Model Registry

- `python -m src.promote` — enregistre le meilleur run comme version de `scoring-model`, pose `@challenger`, et `@champion` seulement si la métrique atteint `promotion_threshold` ; idempotent (pas de doublon).
- `python -m src.promote --filter "params.model = 'dummy'"` — illustre le seuil : le modèle naïf devient `@challenger` mais n'obtient pas `@champion` (roc_auc < seuil).
- `python - <<'EOF' … mlflow.sklearn.load_model("models:/scoring-model@champion") … EOF` — charge le champion par son alias et lit le seuil de décision rangé dans ses métadonnées : c'est ce que servira l'API.
- `git add src/promote.py && git commit -m "feat: promotion par alias avec seuil de qualité"` — commite le script de promotion.

### Étape 8 — Pipeline DVC complet

- `dvc dag` — affiche le graphe `download → prepare → train → evaluate` déduit de `dvc.yaml`.
- `dvc repro` — relance uniquement les stages dont une dépendance (code ou paramètre) a changé.
- `dvc metrics show` — affiche les métriques de `reports/metrics.json`.
- `python -m src.promote --filter "tags.launched_by = 'dvc'"` — promeut le modèle produit par le pipeline (traçabilité complète : commit, données, paramètres, métriques).
- `sed -i 's/^  class_weight: balanced/  class_weight: none/' params.yaml` — modifie un paramètre pour tester la réactivité du pipeline (sur macOS : `sed -i ''`).
- `dvc status` — signale la stage impactée par la modification (`train: changed deps: params.yaml`).
- `dvc params diff` — montre l'écart de paramètre entre HEAD et l'espace de travail.
- `dvc metrics diff` — montre l'écart de métriques entre les versions (n'affiche que celles qui changent).
- `git checkout params.yaml` puis `dvc repro` — restaure la config commitée ; DVC reconnaît la combinaison déjà calculée et restaure les sorties depuis son *run-cache* (« skipping run »).
- `dvc push` — pousse les nouvelles sorties (données préparées, modèle) vers le remote.
- `git add dvc.yaml dvc.lock src/evaluate.py data/processed/.gitignore models/.gitignore reports/metrics.json` — commite le pipeline complet.

### Étape 9 — Feature store local avec Feast

- `cd features; feast apply; cd ..` — importe `definitions.py` et enregistre entité, source, vue et service dans le registre ; crée la table SQLite en ligne.
- `feast materialize-incremental "$(date -u +%Y-%m-%dT%H:%M:%S)"` — copie les dernières valeurs de l'offline store vers l'online store (à lancer depuis `features/`).
- `python - <<'EOF' … store.get_historical_features(entity_df=…, features=store.get_feature_service("scoring_v1")) … EOF` — construit le jeu d'entraînement hors ligne et vérifie qu'il est identique à `train.csv`.
- `python - <<'EOF' … store.get_online_features(features=store.get_feature_service("scoring_v1"), entity_rows=[{id_col: first}]) … EOF` — sert les features en ligne par clé (quelques ms) pour une demande donnée.
- `grep -A8 'Lab 1 :' .gitignore` — vérifie que registre et online store (`features/data/`) ainsi que `mlruns/`/`mlflow.db` sont bien ignorés.
- `git add features/feature_store.yaml features/definitions.py` — commite les définitions Feast (pas les fichiers locaux générés).

### Étape 10 — Reproductibilité et pull request

- `cat >> NOTES.md <<'EOF' … EOF` — documente la procédure de reproduction (livrable du lab).
- `git push -u origin lab/m1-versioning` — pousse la branche du lab.
- `gh pr create` — ouvre la pull request (alternative : « Compare & pull request » sur GitHub) ; relecture par le binôme puis fusion.
- `git switch main && git pull` — revient sur `main` à jour après fusion.
- `git clone … repro-test` puis `dvc pull; dvc repro; dvc metrics show` — simule un autre développeur sur le même poste : `dvc pull` restaure les données depuis le remote, `dvc repro` ne relance rien (tout « skipping »), mêmes métriques.
- `dvc repro --force` puis `dvc metrics diff --all` et `git diff --stat` — preuve forte : tout est recalculé (nouveau run MLflow), les métriques restent identiques ; seul `dvc.lock` change (hash de `models/model`).
- `dvc pull` (autre poste) puis `dvc repro; dvc metrics diff --all; dvc push` — sur un autre cluster le remote est vide : `dvc pull` échoue (missing cache files), on recalcule tout puis on remplit son propre remote.
- `make check-m1` — vérification automatique finale ; attendu `★ 18/18 vérifications OK — Lab 1 validé`.

## Pièges / points d'attention

- `dvc push` : `NoSuchBucket` → le bucket `dvc-store` n'existe pas (job Lab 0 non terminé) ; lancer `make storage` (idempotent) puis `aws-lab s3 ls`.
- `dvc push` : `Could not connect to the endpoint URL` → port-forward S3 arrêté ; relancer `make pf`.
- `dvc push` : `Unable to locate credentials` → identifiants non configurés (clone frais, autre poste) ; `dvc remote modify --local storage access_key_id …` (et `secret_access_key`).
- `dvc push` : `SSL: WRONG_VERSION_NUMBER` → endpoint en `https://` alors que SeaweedFS écoute en HTTP ; corriger `endpointurl` en `http://localhost:8333`.
- `dvc repro` (download) : UCI indisponible puis erreur réseau → pas d'accès Internet ; récupérer `german.data` et lancer `python -m src.download --from-file german.data` puis `dvc commit download`.
- `ModuleNotFoundError: No module named 'src'` → script lancé par son chemin ou hors racine ; utiliser `python -m src.train` depuis `~/mlops-cours-2026`.
- `ModuleNotFoundError: mlflow / dvc / feast` (ou dans `dvc repro`) → venv non activé ; `source .venv/bin/activate` ou `uv run dvc repro`.
- `dvc repro` ne fait rien après modification → le fichier modifié n'est ni dans `deps` ni dans `params` ; vérifier `dvc.yaml`, `dvc status` dit ce que DVC surveille.
- `dvc add` : `cannot update 'dataset.csv': overlaps with an output of stage: 'download'` → un même fichier ne peut être à la fois `dvc add` et sortie de stage ; garder une seule méthode.
- MLflow : `Connection refused` sur `localhost:5000` → port-forward MLflow arrêté ; `make pf` puis `curl http://localhost:5000/health`.
- MLflow UI vide après un run → `MLFLOW_TRACKING_URI` non chargé (runs écrits dans `./mlruns` local) ; `set -a && source .env && set +a` puis supprimer `mlruns/`.
- `log_model` : « references untrusted types » → un type skops n'est pas dans `skops_trusted_types` ; ajouter le type cité (en vérifiant qu'il vient de scikit-learn ou numpy).
- `predict` : `Can not safely convert int64 to float64` → DataFrame lu par `pd.read_csv` passé tel quel ; utiliser `cfg.read_xy(chemin)` qui convertit les numériques.
- `promote` : « Aucun run avec la métrique … » → `metrics.primary` n'est pas une métrique loggée ou l'expérience est vide ; utiliser une métrique de `compute_metrics` et lancer `python -m src.grid_search`.
- `MlflowException: Registered model alias champion not found` → aucun modèle n'a encore passé le seuil ; lancer `python -m src.promote` et vérifier `metrics.promotion_threshold`.
- Ancien code `transition_model_version_stage` déprécié / `models:/…/Production` introuvable → les stages sont remplacés par les alias depuis MLflow 2.9 ; utiliser `set_registered_model_alias(...)` et `models:/scoring-model@champion`.
- `feast apply` : `ModuleNotFoundError: No module named 'src'` → `definitions.py` modifié sans la ligne `sys.path.insert(...)` ; garder `ROOT = …` et `sys.path.insert(0, str(ROOT))`.
- Feast : valeurs `None` en ligne → store en ligne non matérialisé ou fenêtre trop courte ; `feast materialize-incremental "$(date -u +%Y-%m-%dT%H:%M:%S)"` depuis `features/`.
- `dvc pull` : `KeyError: remote` / `no remote specified` → `.dvc/config` non commité ; `git add .dvc/config && git commit` dans le dépôt d'origine puis `git pull`.
- Autres points d'attention signalés : le même fichier ne peut être suivi à la fois par un `.dvc` et par une stage ; ne pas commiter `data/processed/` ni `models/` dans Git ; ne pas commiter le `dvc.lock` régénéré (seul le hash de `models/model` change, à cause de l'identifiant du run MLflow) ; un `pd.Timestamp.now()` dans `prepare.py` casserait la repro (horodatage fixe dans `params.yaml`) ; messages MLflow `Detected uv project` / `Failed to resolve installed pip version` sans gravité (le venv uv n'a pas pip).

*Fiche générée à partir de Lab1-Enonce.pdf et Lab1-Commandes.pdf — cours MLOps.*
