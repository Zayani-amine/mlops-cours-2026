# Lab 3 — Gate de validation dans le pipeline · outils & commandes

Ce lab ajoute au pipeline KFP `scoring-pipeline` du Lab 2 trois gates bloquants qui interdisent d'entraîner, d'évaluer ou d'enregistrer un modèle à partir de données ou de modèles non conformes au contrat. On y construit une suite Great Expectations, on simule et détecte un drift, puis on valide le modèle (Evidently + tests comportementaux) avant son enregistrement. Le principe reste le *fail-fast* : un gate en échec arrête le run et n'exécute aucun step dépendant.

## Outils de la stack

| Outil/service | Rôle / utilité |
|---|---|
| Great Expectations (GX) 1.23 | Contrat de données : suite d'expectations générée depuis `dataset.yaml`, versionnée dans `gx/`, exécutée par un checkpoint. Produit Data Docs + rapport HTML. |
| Evidently 0.7 | Rapports HTML de drift (`DataDriftPreset`) et de classification (`ClassificationPreset`), avec tests explicites (`gte`, `eq` + `Reference`). |
| scipy | Tests statistiques de drift : Kolmogorov-Smirnov (`ks_2samp`), chi2 (`chi2_contingency`), distance de Wasserstein. |
| pytest | Tests comportementaux du modèle (performance minimale, invariance, directionnels) ; rapport JUnit lisible par le gate. |
| KFP 2.17 | Orchestrateur : exécute les composants dans l'image du projet, publie les artifacts (`HTML`, `Markdown`, `Metrics`) dans l'UI, gère les gates via `.after()`. |
| MLflow | Suivi des runs et Model Registry : `register-model` publie une nouvelle version de `scoring-model` seulement après validation. |
| DVC | Gestion des données et du modèle du Lab 1 (`dvc pull`, `dvc repro`). |
| Docker / registry local | Construit et pousse l'image du pipeline (`localhost:5001/scoring-pipeline:<tag>`), qui embarque `src/`, `gx/`, `tests/`. |
| Kubernetes / kubectl | Cluster `mlops-lab`, `kubectl get nodes`, port-forwards, accès au SeaweedFS interne. |
| SeaweedFS (S3) | Stockage des artifacts KFP (bucket `mlpipeline`), remplace MinIO depuis KFP 2.15 ; API S3 compatible. |
| MLflow RemoteStore / AWS CLI | Récupération des rapports HTML stockés dans SeaweedFS via un client S3 (`amazon/aws-cli`). |
| pandas / numpy / pyarrow | Manipulation des DataFrames, lecture/écriture CSV, `data/processed/features.parquet`. |
| Pandera (alternative) | Contrôle de schéma rapide en test unitaire (Lab 6), lui aussi généré depuis `dataset.yaml`. |
| Airflow | DAG `dags/scoring_dag.py` : mêmes composants, même image que KFP (`make check-m2` vérifie la cohérence). |

## Cibles Makefile et scripts

| Cible / script | Rôle |
|---|---|
| `make pf` | Port-forwards S3 (:8333) et MLflow (:5000) en début de séance. |
| `make kfp-ui` | Expose l'UI et l'API KFP sur `http://localhost:8080`. |
| `make pipeline-image` | Reconstruit l'image du pipeline avec le nouveau tag. |
| `make check-m2` | Vérifie l'état du Lab 2 (pipeline existant) avant de démarrer. |
| `make check-m3` | Vérification automatique du Lab 3 (GX, drift, modèle, code, KFP, MLflow). |
| `src/validate_data.py` | Gate 1 : exécute la suite GX, écrit `reports/validation.json` + `reports/validation.html` + Data Docs ; code retour 1 si échec. `--build-suite` (re)génère la suite. |
| `src/simulate.py` | Fabrique des données de test : `corrupt` (bugs silencieux) et `drift` (nouvelle population selon `params.yaml`). |
| `src/detect_drift.py` | Gate 2 : compare référence/ courant par colonne (KS/χ² + PSI + Wasserstein), écrit `reports/drift.{json,md,html}` ; code retour 1 si drift. |
| `src/validate_model.py` | Gate 3 : Evidently (ClassificationPreset au seuil) + tests comportementaux pytest ; écrit `reports/model_validation.*` ; code retour 1 si un test échoue. |
| `pipelines/components.py` | Définit les composants KFP (`validate_data`, `detect_drift`, `validate_model`…) sur `PIPELINE_IMAGE`. |
| `pipelines/stage.py` | Lance une étape (`run_stage`) : copie entrées/sorties, propage l'échec (labo local recopie les rapports même en échec). |
| `pipelines/scoring_pipeline.py` | Pipeline principal avec les trois gates ; compile en `scoring_pipeline.yaml`. |
| `pipelines/test_bad_data.py` | Pipeline de test qui doit ÉCHOUER au gate 1 sur données corrompues. |
| `pipelines/drift_demo.py` | Pipeline de démonstration : drift simulé détecté par `detect_drift`. |
| `labs/m3-validation/check.sh` | Script appelé par `make check-m3` : contrôles locaux en dossiers temporaires. |

## Commandes clés, expliquées

### Démarrage de séance

- `cd ~/mlops-cours-2026` — se placer à la racine du repo.
- `kubectl get nodes` — vérifie que les 3 nœuds du cluster sont `Ready`.
- `make pf` — ouvre les port-forwards S3 (:8333) et MLflow (:5000).
- `source .venv/bin/activate` — active le venv du cours.
- `set -a && source .env && set +a` — charge les variables d'environnement (export automatique).
- `dvc pull` — récupère données/modèle du Lab 1 (`dvc repro` si le remote DVC est vide).
- `make kfp-ui` — expose l'UI/API KFP (comme au Lab 2).

### Branche de travail

- `git switch main && git pull` — repartir d'un `main` à jour.
- `git switch -c lab/m3-validation` — crée la branche dédiée du lab.

### Étape 1 — Suite Great Expectations

- `python -c "import great_expectations as gx; print(gx.__version__)"` — vérifie GX dans l'environnement (attendu `1.23.2`, import lent ~3 s, normal).
- `python -m src.validate_data --build-suite` — (re)génère la suite depuis `dataset.yaml` et les données d'entraînement : écrit `gx/expectations/`, `gx/validation_definitions/`, `gx/checkpoints/`.
- `python -m src.validate_data; echo "Code retour : $?"` — valide les données brutes et affiche le résumé JSON ; `0` = conforme.
- `xdg-open gx/uncommitted/data_docs/local_site/index.html` — ouvre les Data Docs (site GX).
- `xdg-open reports/validation.html` — ouvre le rapport HTML autonome du gate.
- `python -m src.simulate corrupt data/raw/dataset.csv /tmp/dataset_corrupted.csv` — fabrique des données avec trois bugs silencieux (unité, code inconnu, cible manquante).
- `python -m src.validate_data /tmp/dataset_bad.csv; echo "Code retour : $?"` — validation de données corrompues : doit ressortir `success=false` et code `1`.
- `git add gx/ src/validate_data.py src/simulate.py` — versionne la suite (contrat), le script et le simulateur (jamais `gx/uncommitted/`).
- `git commit -m "feat: contrat de données Great Expectations (gx/)"` — commit du contrat de données.

### Étape 2 — Composant KFP et image 1.1.0

- `sed -i 's/scoring-pipeline:1.0.0/scoring-pipeline:1.1.0/' pipelines/components.py dags/scoring_dag.py` — passe le tag d'image (nouveau code = nouveau tag). Sur macOS : `sed -i '' 's/…/…/'`.
- `grep -n "scoring-pipeline:" pipelines/components.py dags/scoring_dag.py` — vérifie que les deux fichiers portent bien le nouveau tag.
- `docker build -f pipelines/Dockerfile --build-arg VERSION=1.1.0 -t localhost:5001/scoring-pipeline:1.1.0 .` — reconstruit l'image (embarque `gx/`, `tests/`, nouvelles dépendances).
- `docker push localhost:5001/scoring-pipeline:1.1.0` — pousse l'image dans le registry local.
- `docker run --rm localhost:5001/scoring-pipeline:1.1.0 python -c "import great_expectations as gx, evidently; print(gx.__version__, evidently.__version__)"` — vérifie que GX/Evidently sont bien dans l'image.
- `docker run --rm localhost:5001/scoring-pipeline:1.1.0 ls gx` — vérifie que la suite versionnée (sans `uncommitted/`) est dans l'image.

### Étape 3 — Compiler et tester en local

- `python -m pipelines.scoring_pipeline` — compile le pipeline en `scoring_pipeline.yaml`.
- `wc -l scoring_pipeline.yaml` — compte les lignes du YAML compilé.
- `grep -c "image: localhost:5001/scoring-pipeline:1.1.0" scoring_pipeline.yaml` — vérifie que chaque exécuteur utilise la bonne image (6 attendus à ce stade).
- Bloc `kfp.local` (`local.init(runner=local.SubprocessRunner(use_venv=False))` puis `scoring_pipeline(mlflow_uri="http://localhost:5000")`) — exécute le pipeline en local sans cluster.

### Étape 4 — Tester le gate sur données corrompues

- `python -m pipelines.test_bad_data` — compile le pipeline de test (`test_bad_data.yaml`).
- Bloc `kfp.local` avec `raise_on_error=False` — lance `bad_data_pipeline()` : l'échec du gate est affiché sans interrompre le script.
- `ls local_outputs/scoring-bad-data-test-*/validate-data/` — vérifie que `html` et `report` sont bien produits en local même en échec.

### Étape 5 — Détecter le drift

- `cat >> params.yaml <<'EOF' … EOF` — ajoute la section `drift` (seuils `ks_alpha`, `psi_threshold`, `drift_share` et scénario `simulation`).
- `python -m src.detect_drift; echo "Code retour : $?"` — compare entraînement et test (défauts) : tableau par colonne, statut du gate.
- `python -m src.simulate drift data/processed/test.csv /tmp/test_drifted.csv` — applique le scénario de drift au jeu de test.
- `python -m src.validate_data /tmp/test_drifted.csv | grep -E '"(success|n_failed)"'` — montre que les données déplacées respectent encore le contrat (le drift n'est pas une violation de contrat).
- `python -m src.detect_drift data/processed/train.csv /tmp/test_drifted.csv; echo "Code retour : $?"` — détecte le drift simulé (code retour `1`).
- `python -m pipelines.drift_demo` — compile le pipeline de démonstration de drift (`drift_demo.yaml`).
- Bloc `kfp.local` sur `drift_pipeline()` — exécute la démo : drift signalé sans bloquer.

### Étape 6 — Valider le modèle

- `python -m pytest -q tests/test_model_behavior.py` — lance les tests comportementaux (8 attendus).
- `python -m src.validate_model; echo "Code retour : $?"` — Evidently + pytest : rapport `reports/model_validation.*`, code `0` si le modèle est validé.
- `python -m pipelines.scoring_pipeline` — recompile le pipeline final avec les trois gates.
- `wc -l scoring_pipeline.yaml` — compte les lignes du pipeline final.
- `sed -i 's/scoring-pipeline:1.1.0/scoring-pipeline:1.2.0/' pipelines/components.py dags/scoring_dag.py` — passe au tag 1.2.0.
- `docker build -f pipelines/Dockerfile --build-arg VERSION=1.2.0 -t localhost:5001/scoring-pipeline:1.2.0 .` puis `docker push` — reconstruit (rapide, dépendances en cache) et pousse l'image 1.2.0.
- `python -m pipelines.drift_demo` puis `grep -c "image: …:1.2.0" scoring_pipeline.yaml` — recompile la démo et vérifie le tag.

### Étape 7 — Exporter et récupérer les rapports

- `kubectl -n kubeflow port-forward svc/seaweedfs 9090:8333 >/tmp/kfp-s3.log 2>&1 &` — port-forward vers le S3 interne de KFP.
- `KFP_S3_KEY=$(kubectl -n kubeflow get secret mlpipeline-minio-artifact -o jsonpath='{.data.accesskey}' | base64 -d)` — lit la clé S3 de KFP depuis le secret.
- `KFP_S3_SECRET=$(kubectl -n kubeflow get secret mlpipeline-minio-artifact -o jsonpath='{.data.secretkey}' | base64 -d)` — lit la clé secrète S3.
- Fonction `kfp-s3()` — enveloppe le client `amazon/aws-cli` (réseau host, montage `reports/kfp:/data`) pointé sur `--endpoint-url http://localhost:9090`.
- `kfp-s3 s3 ls s3://mlpipeline/v2/artifacts/scoring-pipeline/` — liste les runs stockés.
- `kfp-s3 s3 ls --recursive "s3://…/$RUN_ID/" | grep /html` — liste les rapports HTML d'un run.
- `kfp-s3 s3 cp "s3://mlpipeline/$KEY" /data/drift.html` — télécharge un rapport HTML précis.
- `kfp-s3 s3 cp --recursive "s3://…/$RUN_ID/" "/data/$RUN_ID/"` — récupère tout le run d'un coup.
- `pkill -f "port-forward svc/seaweedfs"` — arrête le port-forward.

### Commit et push final

- `git add params.yaml gx/ src/ tests/ pipelines/ dags/scoring_dag.py` — stage les sources et le contrat.
- `git status --short` — vérifie qu'aucun rapport, YAML compilé ou `gx/uncommitted/` n'est inclus.
- `git commit -m "feat: gates de validation (GX, drift, modèle) dans scoring-pipeline"` — commit des trois gates.
- `git push -u origin lab/m3-validation` — pousse la branche.
- `gh pr create` — ouvre la PR (ou « Compare & pull request » sur GitHub) ; le partenaire relit, notamment la suite GX.
- `make check-m3` — vérification automatique complète du lab.

## Pièges / points d'attention

- L'import de `great_expectations` prend ~3 s : ce n'est pas un bug.
- Le code GX 0.x ne fonctionne pas en 1.x : utiliser `context.data_sources`, `context.suites`, `ValidationDefinition`, `Checkpoint.run()`.
- Construire une suite exige un contexte actif : appeler `gx.get_context()` d'abord.
- `Suite absente` : lancer `--build-suite`, `git add gx/`, et copier `gx/` dans l'image avec un nouveau tag.
- `FileNotFoundError: data/processed/train.csv` : données du Lab 1 absentes (`dvc pull` ou `dvc repro`).
- La suite peut valider des données corrompues si moins de 1 % de lignes sont touchées (`mostly=0.99`) ou si elle a été regénérée depuis ces données : corrompre plus de lignes, ne jamais régénérer la suite depuis les données à valider.
- Un `expect_column_values_to_be_in_set` en échec = modalité inconnue : corriger la source ou faire évoluer le contrat (revue du `git diff gx/`).
- `validate-data` en `OOMKilled` : la limite de 512 Mi du Lab 2 est trop basse pour GX → `set_memory_limit("1Gi")`.
- `ModuleNotFoundError: great_expectations` dans un pod : l'image utilisée est encore 1.0.0 → mettre à jour le tag, repousser l'image, recompiler et uploader.
- `pytest: No module named 'src'` : lancer depuis la racine du repo (`python -m pytest` ou `uv run pytest`).
- Tests comportementaux tous *skipped* : pas de modèle local (`dvc pull` / `dvc repro`).
- Evidently `AttributeError: 'list' object has no attribute 'items'` : bogue de `ClassificationPreset(include_tests=False)` en 0.7.23 → désactiver les tests métrique par métrique (listes vides).
- Tests par défaut d'Evidently (« mieux qu'un modèle naïf ») en FAIL (TNR/FPR) : inadaptés au seuil coût-optimal → tests explicites seulement.
- Erreurs Deepchecks (`np.Inf`, `max_error`) : Deepchecks 0.19 incompatible NumPy 2 / scikit-learn ≥ 1.8 → remplacé par Evidently + pytest.
- `prep_task` démarre avant `validate` : `.after()` oublié.
- Un `sys.exit(1)` n'arrête pas le step si l'exception est attrapée : laisser remonter l'erreur pour que `run_stage` fasse échouer le step.
- Un artifact HTML ne s'affiche pas : n'utiliser qu'un seul fichier commençant par `<!DOCTYPE html>`, en UTF-8 (le rapport Evidently ≈ 4 Mo met quelques secondes).
- Pas de rapport HTML sur un step `validate-data` en échec : KFP ne publie que les sorties des steps réussis → lire les logs ou rejouer en local (`kfp.local`).
- KS p-value toujours très basse sur gros dataset : normal, d'où l'exigence d'un PSI ≥ 0,25.
- Valeurs PSI extrêmes : cases vides/trop fines → déciles de référence, ε = 1e-4, une case par modalité pour les discrètes.
- Evidently signale un drift non confirmé par le tableau : son verdict ne tient qu'à la p-value → comparer la colonne PSI (`DataDriftPreset(method="psi")` pour un verdict par magnitude).
- `detect-drift` échoue dans `scoring-pipeline` : le test ne ressemble pas à l'entraînement (petit dataset, split non stratifié) → lire le tableau, revoir le split, ajuster les seuils en le justifiant.
- `aws : Could not connect to the endpoint URL` : port-forward SeaweedFS absent.
- `aws : InvalidAccessKeyId` : identifiants du S3 du cours (mlops) au lieu de ceux de KFP → lire `mlpipeline-minio-artifact`.
- `aws s3 ls` sans résultat : mauvais préfixe (`v2/artifacts/<pipeline>/<run_id>/`), vérifier l'id dans l'URL de l'UI.
- Le split aléatoire et stratifié du Lab 1 ne crée aucun drift : Evidently peut signaler à tort des modalités rares à cause de son χ² sur effectifs → toujours accompagner un test d'une mesure de magnitude.
- Un gate ne se contente pas d'un avertissement : si l'exception est attrapée sans `raise`, le step reste vert et le run continue.

*Fiche générée à partir de Lab3-Enonce.pdf et Lab3-Commandes.pdf — cours MLOps.*
