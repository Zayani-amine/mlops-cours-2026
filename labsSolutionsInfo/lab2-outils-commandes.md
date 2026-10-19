# Lab 2 — Pipeline ML sur Kubeflow + Airflow · outils & commandes

Ce lab transforme le pipeline DVC du Lab 1 en un DAG Kubeflow Pipelines (KFP) production-ready (`scoring-pipeline` : ingest → validate → prepare → train → evaluate → register), puis en écrit une variante Airflow (`scoring_dag`) avec la même image et le même code. Il s'appuie sur le cluster kind du Lab 0, réutilise les scripts de `src/` via une image « projet » et trace chaque run dans MLflow. Objectif final : comparer les deux orchestrateurs (DX, scaling, debugging) et évaluer cette intégration de bout en bout.

## Outils de la stack

| Outil/service | Rôle / utilité |
| --- | --- |
| Kubeflow Pipelines (KFP 2.17) | Orchestrateur de DAG ML sur Kubernetes : API server, UI, SDK Python, upload/exécution du pipeline `scoring-pipeline`. |
| Argo Workflows (4.0) | Moteur d'exécution réel des DAG KFP (fournit la CRD `workflows.argoproj.io`). |
| Kubernetes / kind (`mlops-lab`) | Support d'exécution : 1 step KFP = 1 pod, 1 task Airflow = 1 pod. Cluster local en conteneurs Docker. |
| Docker + registry local `localhost:5001` | Build et stockage de l'image « projet » `scoring-pipeline:1.0.0`, tirée par le kubelet de chaque nœud. |
| SeaweedFS interne KFP (ns `kubeflow`) | Stockage S3 interne de KFP (bucket `mlpipeline`) : transporte et range les artefacts typés entre composants ; jamais manipulé à la main. |
| SeaweedFS du cours (ns `storage`) | S3 du cours (buckets `mlflow-artifacts`, `dvc-store`, `airflow-data`) utilisé par MLflow, DVC et le dossier de travail Airflow. |
| MLflow 3 (ns `mlflow`) | Tracking des runs et Model Registry : expérience `scoring`, versions de `scoring-model`, alias `@challenger` / `@champion`. |
| MySQL | Base des métadonnées de KFP. |
| ML Metadata (`metadata-grpc`) | Métadonnées et lineage des artefacts KFP. |
| Airflow 3.3 (Docker Compose) | Orchestrateur alternatif ; DAG équivalent lancé depuis `http://localhost:8081`. |
| KubernetesPodOperator (KPO) | Opérateur Airflow qui exécute chaque task dans un pod du cluster (même image que KFP). |
| DVC (Lab 1) | Référence métier : mêmes stages et mêmes commandes que les composants KFP. |
| `kfp` / `kfp-kubernetes` / `kfp.local` | SDK KFP, extensions K8s (`kubernetes.set_timeout`), exécution locale (`SubprocessRunner`). |
| `mlflow-skinny`, `scikit-learn`, `skops`, `pandas`, `pyarrow`, `boto3` | Dépendances figées dans l'image du pipeline (`pipelines/requirements.txt`). |
| `gh` (GitHub CLI) | Création facultative de la pull request de fin de lab. |

## Cibles Makefile et scripts

| Cible / script | Rôle |
| --- | --- |
| `make cluster` | Prépare le cluster kind et le registry local `localhost:5001` (Lab 0). |
| `make pf` | Lance les port-forwards S3 (`:8333`) et MLflow (`:5000`) en arrière-plan. |
| `make mlflow` | Déploie le serveur MLflow du cours (ns `mlflow`). |
| `make kfp` | Applique les manifests KFP (script `infra/kfp/install.sh`, version lue dans `infra/versions.env`) puis attend la disponibilité des déploiements. |
| `make kfp-ui` | Port-forward de l'UI/API KFP sur `:8080` (journal `/tmp/mlops-port-forward/kfp-ui.log`). |
| `make pipeline-image` | Construit et pousse l'image `scoring-pipeline` dans le registry local. |
| `make airflow` | Installe Airflow en Docker Compose (`infra/airflow/install.sh`) et recopie le secret `s3-credentials`. |
| `make check-m1` | Vérifie que le Lab 1 est vert (prérequis). |
| `make check-m2` | Vérifie le Lab 2 (script `labs/m2-pipelines/check.sh`) sans rien modifier ; code retour = nombre d'échecs. |
| `infra/versions.env` | Source unique des versions (`KFP_VERSION`, `AIRFLOW_VERSION`). |
| `labs/m2-pipelines/starter/rapport-m2.md` | Modèle du rapport comparatif à recopier dans `reports/`. |

## Commandes clés, expliquées

### Début de séance

- `cd ~/mlops-cours-2026` — se placer à la racine du fork, prérequis de toutes les commandes du lab.
- `kubectl get nodes` — confirmer que les 3 nœuds kind sont `Ready`.
- `make pf` — ouvrir les port-forwards S3 et MLflow (déjà actifs dans les terminaux suivants).
- `source .venv/bin/activate` puis `set -a && source .env && set +a` — activer le venv du cours et charger les variables d'environnement (dont les identifiants S3).

### Branche de travail

- `git switch main && git pull` — partir de `main` à jour.
- `git switch -c lab/m2-pipelines` — créer la branche du lab, terminée par une pull request relue.

### Étape 1 — Installer KFP

- `source infra/versions.env` — charger `KFP_VERSION=2.17.2`.
- `kubectl apply -k "$KFP_MANIFESTS/cluster-scoped-resources?ref=$KFP_VERSION"` — installer les ressources cluster-scoped (CRD, RBAC).
- `kubectl wait --for condition=established ... crd/workflows.argoproj.io` — attendre que les CRD (dont Argo) soient prêtes avant la suite.
- `kubectl apply -k "$KFP_MANIFESTS/env/platform-agnostic?ref=$KFP_VERSION"` — déployer la stack KFP dans le namespace `kubeflow`.
- `kubectl -n kubeflow get pods -w` — suivre le démarrage ; les 14 pods doivent finir `Running` (CrashLoopBackOff initial sur `ml-pipeline`/`metadata-grpc` normal).
- `make kfp-ui` — exposer l'UI puis ouvrir `http://localhost:8080`.
- `python -c "import kfp; print(kfp.__version__)"` — vérifier le SDK (attendu `2.17.0`).
- `curl -s http://localhost:8080/apis/v2beta1/healthz` — tester l'accès à l'API (`"multi_user":false`).

### Étape 2 — Composants KFP, validation, image

- `python -m src.validate_data; echo "Code retour : $?"` — exécuter le gate de validation (code retour `0` si conforme).
- Génération d'un dataset corrompu (script inline) puis `python -m src.validate_data /tmp/dataset_bad.csv` — vérifier le fail-fast (colonne manquante → code retour `1`).
- `mkdir -p pipelines` / `touch pipelines/__init__.py` — créer le package des modules d'orchestration.
- Test de `run_stage` via script inline puis `cat /tmp/validation.json` — exécuter une stage dans un dossier temporaire sans modifier le repo.
- `docker build -f pipelines/Dockerfile --build-arg VERSION=1.0.0 -t localhost:5001/scoring-pipeline:1.0.0 .` — construire l'image « projet » (contexte = racine).
- `docker push localhost:5001/scoring-pipeline:1.0.0` — pousser l'image (sinon les pods restent en `ImagePullBackOff`).
- `curl -s http://localhost:5001/v2/scoring-pipeline/tags/list` — vérifier que le tag `1.0.0` est publié.
- `docker run --rm localhost:5001/scoring-pipeline:1.0.0 python -c "..."` — contrôler que l'image embarque bien `dataset.yaml` et `src/`.

### Étape 3 — Écrire et compiler le pipeline

- `python -m pipelines.scoring_pipeline` — compiler le pipeline en `scoring_pipeline.yaml` (format IR).
- `ls -l scoring_pipeline.yaml` — constater le fichier compilé (~460 lignes).
- `grep -c "image: localhost:5001/scoring-pipeline:1.0.0" scoring_pipeline.yaml` — compter les exécuteurs sur l'image du projet (attendu `5` puis `6`).
- Script inline `local.init(runner=local.SubprocessRunner(use_venv=False))` — tester le DAG en local (sous-processus du venv, vrais artefacts) avant déploiement.

### Étape 4 — Exécuter sur le cluster

- UI KFP : `Pipelines → Upload pipeline` (nom exact `scoring-pipeline`), puis `Create run` dans l'expérience `scoring`.
- `kubectl -n kubeflow get pods -w | grep scoring` — suivre les pods driver/impl de chaque step.
- Script inline `kfp.Client(...).create_run_from_pipeline_package(...)` — soumettre un run sans l'UI (même API que la CI du Lab 6).
- UI KFP onglet Input/Output — explorer les artefacts (`raw_data`, `checks`, `model`, `metrics`, matrice de confusion).

### Étape 5 — Intégration MLflow

- `kubectl -n mlflow get pods,svc` — vérifier que MLflow tourne et expose le port `5000`.
- `kubectl -n kubeflow run mlflow-check ... python -c "..."` — tester la joignabilité de MLflow depuis un pod (via le DNS `mlflow.mlflow:5000`).
- `python -m pipelines.scoring_pipeline` — recompiler après avoir branché `register_model`.
- Script inline `MlflowClient()` — retrouver qui a produit `@challenger`/`@champion` et vérifier le tag `launched_by=kfp`.

### Étape 6 — Retries, ressources, caching

- `python -m pipelines.scoring_pipeline` — recompiler avec `set_retry`, limites CPU/mémoire et timeouts.
- `grep -nE "maxRetryCount|resourceMemoryLimit|activeDeadlineSeconds" scoring_pipeline.yaml` — vérifier que les policies sont dans le YAML (`maxRetryCount: 3`, six `resourceMemoryLimit`, deux `activeDeadlineSeconds`).
- Relancer deux runs identiques dans l'UI — observer les steps data en `Cached` et le train ré-exécuté (caching désactivé).

### Étape 7 — Airflow et DAG équivalent

- `source infra/versions.env` (dans `infra/airflow`) — charger `AIRFLOW_VERSION=3.3.2`, puis télécharger le `docker-compose.yaml` officiel.
- `mkdir -p ./logs ./plugins ./config ../../dags` — créer les dossiers attendus par Airflow.
- `echo "AIRFLOW_UID=$(id -u)" > .env` et `echo "AIRFLOW_IMAGE_NAME=apache/airflow:${AIRFLOW_VERSION}-python3.12" >> .env` — configurer Compose.
- `kind get kubeconfig --internal --name mlops-lab > kubeconfig` puis `chmod 600 kubeconfig` — donner aux conteneurs Airflow un accès interne au cluster.
- `kubectl create namespace airflow --dry-run=client -o yaml | kubectl apply -f -` — créer le namespace des pods de tasks (idempotent).
- `kubectl -n airflow create secret generic s3-credentials ... --dry-run=client -o yaml | kubectl apply -f -` — injecter les identifiants S3 du cours dans le namespace `airflow`.
- `docker compose up airflow-init` puis `docker compose up -d` et `docker compose ps` — initialiser puis démarrer Airflow (UI sur `:8081`).
- `kubectl -n airflow get pods -w` — suivre les pods des tasks au déclenchement du DAG.

### Étape 8 — Tester et comparer

- `docker stats --no-stream` — mesurer la mémoire des nœuds kind vs conteneurs Airflow.
- `wc -l pipelines/components.py pipelines/scoring_pipeline.py dags/scoring_dag.py` — comparer la taille du code des deux orchestrateurs.
- `python -m pipelines.stage src.download --workspace /tmp/ws` puis `python -m pipelines.stage src.validate_data --workspace /tmp/ws` — rejouer une task Airflow en local (dossier au lieu de S3).
- `ls -R /tmp/ws` — vérifier le contenu produit (`data/raw/dataset.csv`, `reports/validation.json`).

### Étape 9 — Rapport et commit

- `cp labs/m2-pipelines/starter/rapport-m2.md reports/rapport-m2.md` — copier le modèle de rapport à compléter.
- `git add src/validate_data.py pipelines/ dags/ reports/rapport-m2.md` — indexer les fichiers du lab.
- `git status --short` — vérifier qu'aucun fichier ignoré (`kubeconfig`, `scoring_pipeline.yaml`, `local_outputs/`) n'apparaît.
- `git commit -m "feat: pipeline KFP scoring-pipeline + DAG Airflow scoring_dag"` — commiter le travail.
- `git push -u origin lab/m2-pipelines` puis `gh pr create` — pousser et ouvrir la PR (relue par le binôme).
- `make check-m2` — lancer la vérification automatique finale (port-forwards actifs).

## Pièges / points d'attention

- Le journal des problèmes fourni (`Lab 2 — Problèmes`) est un **modèle vide à remplir pendant le lab** (Symptôme / Cause probable / Solution) : aucun cas n'y est encore consigné.
- Ne pas confondre les **deux stockages S3** : le SeaweedFS du cours (`storage`, `s3.storage:8333`, secrets `s3-credentials`) et le SeaweedFS interne de KFP (`kubeflow`, `seaweedfs.kubeflow:8333`, bucket `mlpipeline`, secret `mlpipeline-minio-artifact`).
- Le **caching KFP** se base sur le nom de l'image, pas son contenu : reconstruire le même tag après un changement de code ressert l'ancien résultat (`imagePullPolicy: IfNotPresent`). Tout changement de code dans l'image = nouveau tag (`1.0.1`), reporté dans `PIPELINE_IMAGE`. Jamais `latest`.
- Une seule `limit` vaut aussi `request` : garder une `set_cpu_request("500m")` évite un pod `Pending (Insufficient cpu)` sur kind.
- Dans `components.py`, tout import doit être **dans** la fonction et aucune valeur par défaut ne doit dépendre d'une constante du module (sinon `NameError` dans le pod).
- `kfp.local` doit être lancé **depuis la racine du repo**, sinon `ModuleNotFoundError: No module named 'src'`.
- XCom ne supporte que les **petits objets** : y passer des clés/chemins S3, pas des DataFrames complets.
- `mlflow_uri` doit rester `http://mlflow.mlflow:5000` (autorisé par `--allowed-hosts`, Lab 0) ; les composants n'ont besoin d'aucun identifiant S3 (MLflow fait proxy).
- Le **gate de validation** doit rester fail-fast : si les données ne respectent pas `dataset.yaml`, le step échoue avant l'entraînement.
- Un step `OOMKilled` se corrige en augmentant son `set_memory_limit` (étape 6) ; les valeurs mesurées : train ~550 Mo, autres steps 200–500 Mo.
- Pod en `ImagePullBackOff` : image non poussée ou tag différent de `PIPELINE_IMAGE`.
- Ne jamais commiter ce que le lab régénère (`scoring_pipeline.yaml`, `kubeconfig`, `local_outputs/`, `reports/validation.json`).
- Ressources : compter ~6 Go de RAM pour KFP et ~4 Go pour Airflow ; arrêter Airflow (`docker compose down`) dès qu'il n'est plus utile.
- DAG Airflow `not found` : le fichier doit être dans `dags/` à la racine (attendre ~30 s le scan) ; en cas d'erreur d'import, utiliser `airflow dags list-import-errors`.
- Docker Compose trop ancien (`unknown tag !override`) : mettre à jour vers ≥ 2.24.4.

*Fiche générée à partir de Lab2-Enonce.pdf et Lab2-Commandes.pdf — cours MLOps.*
