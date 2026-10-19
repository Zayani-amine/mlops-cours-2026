# Lab 7 — Monitoring en production du modèle de scoring · outils & commandes

Ce lab installe les signaux d'alerte autour de l'API de scoring servie par KServe : un modèle qui se dégrade ne plante pas, il répond `200 OK` avec des probabilités plausibles pendant que la population change. On instrumente l'API avec Prometheus, on construit un dashboard Grafana, on détecte le drift avec Evidently, on agrège les logs avec Loki, on configure des alertes SLO et on démontre le cycle drift → alerte → ré-entraînement puis la décision de promotion d'un canary à partir des métriques.

## Outils de la stack

| Outil/service | Rôle / utilité |
| --- | --- |
| `prometheus-client` | Bibliothèque Python instrumentant l'API : compteurs, histogrammes, `Info`, route `GET /metrics`. Version 0.24.1 imposée par le `uv.lock` (Feast exige `<0.25`). |
| Prometheus | Base de séries temporelles : scrape l'API toutes les 15 s, stocke les métriques (4 signaux d'or + métriques métier) et évalue les règles d'alerte. |
| Grafana | Visualisation : dashboard versionné `scoring-overview` (SLO, modèle, drift, infra, logs) + source de données Loki. |
| Alertmanager | Routage des alertes : règles SLO (burn rate), drift, taux de refus ; envoi vers Slack/webhook par sévérité. |
| kube-prometheus-stack | Chart Helm (91.9.0) qui déploie Prometheus, Alertmanager, Grafana, operator, kube-state-metrics et node-exporter. |
| Pushgateway | Reçoit en `PUT` les métriques poussées par le job de drift (un CronJob vit trop peu pour être scrapé directement). |
| Evidently (0.7) | Calcule le drift (méthode PSI) entre les données d'entraînement et les demandes réelles, et produit un rapport HTML. |
| Loki | Stockage des logs, mode monolithique (un pod), rétention 72 h (les prédictions sont des données personnelles). |
| Grafana Alloy | Collecte des logs de tous les pods via l'API Kubernetes et les envoie à Loki (successeur de Promtail, en fin de vie depuis mars 2026). |
| KServe / InferenceService | Sert le modèle de scoring ; expose `scoring-model` et gère la répartition du trafic entre révisions (canary). |
| Knative | Sous-couche de l'InferenceService : renomme le port 8000 en `user-port`, ajoute le label `serving.knative.dev/revision` (clé pour comparer stable/canary). |
| DVC | Versionne la référence (`data/processed/train.csv`) : le fichier est rangé sur le S3 sous son empreinte MD5, `dvc.api.get_url` donne l'adresse sans téléchargement. |
| S3 (SeaweedFS) | Stockage objet du cours : référence DVC et rapports de drift HTML (`s3://monitoring/drift-reports`). |
| Helm | Installe les charts d'observabilité avec versions épinglées (`helm upgrade --install`, idempotent). |
| kind / Docker Desktop | Cluster Kubernetes local ; kube-prometheus-stack ≈ 2 Go, Loki + Alloy ≈ 0,5 Go, au moins 10 Go alloués à Docker. |
| Argo CD / GitOps | Déploie par Git : fusion d'une PR → image GHCR → staging ; la production ne bouge que par commit de canary/promotion/rollback. |
| webhook.site / Slack | Récepteur de test (webhook.site) ou canal réel du webhook d'alerte Alertmanager. |
| KFP (Kubeflow Pipelines) | Exécute le pipeline de ré-entraînement `ct-pipeline` appelé par le webhook continu du Lab 6. |

## Cibles Makefile et scripts

| Cible / script | Rôle |
| --- | --- |
| `make monitoring` | Installe/actualise toute la stack d'observabilité via Helm (idempotent : à relancer après toute modification des values). |
| `make pf` | Port-forwards du socle (MLflow, etc.) ; condition de départ de la séance avec `kubectl get nodes`. |
| `make kfp-ui` | Ouvre l'interface Kubeflow Pipelines (port 8080), nécessaire au ré-entraînement de l'étape 9. |
| `make check-m7` | Vérifie le lab sans rien modifier : fichiers, tests, stack prête, scraping, métriques, dashboard, alertes, drift, livrables. Code retour = nombre d'échecs. |
| `infra/monitoring/install.sh` | Ajoute les dépôts de charts et épingle les versions ; install de kube-prometheus-stack, Pushgateway, Loki, Alloy. |
| `infra/monitoring/port-forward.sh` | Expose les interfaces (Grafana 3000, Prometheus 9090, Alertmanager 9093, Loki 3100, Pushgateway 9091) ; `stop` les arrête. |
| `monitoring/traffic.py` | Générateur de trafic : rejoue des lignes réelles de `test.csv` à débit constant ; sait déformer les données (`--mode drift`) pour simuler un drift. |
| `monitoring/evidently-job/calc_drift.py` | Job de drift : référence DVC vs demandes réelles lues dans Loki, calcul PSI, push Pushgateway, rapport S3, webhook de ré-entraînement. |
| `monitoring/canary_analysis.py` | Compare stable et canary révision par révision (six garde-fous) et décide ROLLBACK / WAIT / palier suivant / PROMOTE ; `--apply --push` applique par Git. |
| `packaging/app/serve.py` | API de scoring instrumentée : `/metrics`, middleware `count_requests`, `observe()`, journal JSON des prédictions. |
| `packaging/sample_request.py` | Construit une charge de test (lit `test.csv` et `dataset.yaml`) ; option `-n` pour le nombre de demandes. |
| `monitoring/dashboards/kustomization.yaml` | Transforme les JSON versionnés en ConfigMap labellisée `grafana_dashboard: "1"` (détectée par le sidecar Grafana). |
| `monitoring/scoring-podmonitor.yaml` | PodMonitor ciblant les pods de l'InferenceService sur le port 8000, chemin `/metrics`. |
| `monitoring/evidently-cronjob.yaml` | CronJob `scoring-drift`, toutes les 15 min, qui lance l'image du job de drift. |
| `monitoring/scoring-alerts.yaml` | `PrometheusRule` : recording rules (`scoring.rules`) + six alertes (`scoring.alerts`). |
| `monitoring/alertmanager-config.yaml` | `AlertmanagerConfig` : routage des alertes `ml-platform` vers Slack par sévérité (webhook lu dans un Secret). |

## Commandes clés, expliquées

### Préparation de la branche

- `git switch main && git pull` puis `git switch -c lab/m7-monitoring` : part d'un `main` à jour et crée la branche de travail du lab.

### Étape 1 — Instrumenter l'API

- `set -a && source .env && set +a` : charge les variables d'environnement (identifiants S3, tokens…) dans le shell courant.
- `uv sync` : installe/synchronise l'environnement Python, notamment `prometheus-client`.
- `uv run python -c "from importlib.metadata import version; print(version('prometheus-client'))"` : vérifie la version installée (attendu `0.24.1`).
- `uv run python -m uvicorn app.serve:app --app-dir packaging --port 8000 > /tmp/api.log 2>&1 &` : lance l'API en arrière-plan, logs dans `/tmp/api.log`.
- `uv run python packaging/sample_request.py -n 20 > /tmp/request.json` : génère une charge de 20 demandes réelles.
- `for i in $(seq 1 20); do curl -s -X POST http://localhost:8000/predict -H "Content-Type: application/json" -d @/tmp/request.json > /dev/null; done` : envoie 20 × 20 requêtes à `/predict` pour alimenter les métriques.
- `curl -s http://localhost:8000/metrics | grep -E "^(ml_|http_requests)" | grep -v _bucket` : lit les métriques métier et HTTP exposées (hors `_bucket`).
- `grep -m1 '"event":"prediction"' /tmp/api.log` : vérifie qu'une ligne JSON de prédiction est bien journalisée.
- `kill %1` : arrête l'API lancée en arrière-plan.
- `cp -r labs/m7-monitoring/starter/monitoring .` et `cp labs/m7-monitoring/starter/tests/test_monitoring.py tests/` : copie les fichiers fournis (tests, trafic, canary, dashboard, runbooks).
- `uv run pytest tests/test_monitoring.py -v` : lance les tests du lab (attendu `13 passed`) ; ils tournent aussi dans la CI, bloquant une régression d'observabilité.

### Étape 2 — Déployer par GitOps

- `git add …` / `git commit -m "feat(api): …"` / `git push -u origin lab/m7-monitoring` : pousse l'API instrumentée sur la branche.
- `gh pr create` puis `gh pr checks --watch` : ouvre la PR et suit la CI (lint, tests, build de l'image).
- `gh run watch` : suit le run de `main` (publication de l'image + déploiement staging).
- `uv run python -m pipelines.gitops status` : affiche l'état staging/prod (nouveau tag en staging, ancien en prod).
- `uv run python -m pipelines.gitops deploy --env prod --canary 10 --push` : premier canary en production, commité par l'outil GitOps (part de `main`).
- `kubectl get isvc scoring-model -n scoring -w` : suit l'InferenceService jusqu'à `READY=True` et `LATEST=10` (Ctrl-C).
- `ISVC_HOST=$(kubectl get isvc scoring-model -n scoring -o jsonpath='{.status.url}' | cut -d/ -f3)` puis `export ISVC_HOST` : extrait l'hôte de la passerelle KServe pour les requêtes avec `Host:`.
- `curl -s -H "Host: latest-${ISVC_HOST}" http://localhost:30080/metrics | grep ^ml_model_info` : vérifie que la nouvelle révision expose `ml_model_info` (l'ancienne répond 404 : non instrumentée).
- `uv run python -m pipelines.gitops promote --push` : promeut le canary à 100 % une fois le contrôle de fumée passé.

### Étape 3 — Installer la stack

- `make monitoring` : installe kube-prometheus-stack, Pushgateway, Loki et Alloy (relancer après toute modification des values).
- `kubectl get pods -n monitoring` : attend que tous les pods soient `Running`/prêts (3 à 5 min).
- `./infra/monitoring/port-forward.sh` : ouvre les interfaces locales (Grafana `admin`/`admin`, Prometheus, Alertmanager, Loki, Pushgateway).

### Étape 4 — PodMonitor

- `kubectl apply -f monitoring/scoring-podmonitor.yaml` : installe le PodMonitor ciblant les pods de l'ISVC (nécessaire car en Knative les Services n'exposent que le queue-proxy).
- `kubectl get podmonitor -n monitoring` : confirme sa présence.
- Requêtes Prometheus de contrôle : `up{namespace="scoring", isvc="scoring-model"}`, `ml_model_info{namespace="scoring"}`, `sum by (outcome) (ml_predictions_total{namespace="scoring"})`.

### Étape 5 — Trafic et dashboard

- `uv run python monitoring/traffic.py --url http://localhost:30080 --host-header "$ISVC_HOST" --duration 0` : trafic normal illimité (à laisser tourner jusqu'à l'étape 9).
- `kubectl apply -k monitoring/dashboards` : charge le dashboard versionné via la ConfigMap labellisée.
- `kubectl get configmap -n monitoring -l grafana_dashboard=1 | grep scoring` : vérifie que la ConfigMap `scoring-dashboards` existe.

### Étape 6 — CronJob Evidently

- `docker build -f monitoring/evidently-job/Dockerfile -t localhost:5001/scoring-drift:1.0.0 .` puis `docker push localhost:5001/scoring-drift:1.0.0` : construit et publie l'image du job de drift (contexte = racine du repo).
- `curl -s http://localhost:5001/v2/scoring-drift/tags/list` : vérifie le tag publié.
- `kubectl create secret generic s3-credentials -n monitoring --from-literal=… --dry-run=client -o yaml | kubectl apply -f -` : crée le Secret d'accès S3 dans `monitoring` (idempotent).
- `dvc push` : pousse la référence sur le S3 (sinon `NoSuchKey`).
- `REFERENCE_URI=$(uv run python -c "import dvc.api; print(dvc.api.get_url('data/processed/train.csv'))")` puis ConfigMap `scoring-drift-reference` : donne au job l'adresse de la référence versionnée.
- `kubectl apply -f monitoring/evidently-cronjob.yaml` : déploie le CronJob `scoring-drift` (`*/15 * * * *`).
- `kubectl create job --from=cronjob/scoring-drift manual-run-1 -n monitoring` puis `kubectl logs -n monitoring job/manual-run-1 -f` : déclenche un run immédiat et suit ses logs.
- `aws-lab s3 ls s3://monitoring/drift-reports/` et `aws-lab s3 cp s3://monitoring/drift-reports/<fichier>.html - > /tmp/drift.html` : liste et récupère le rapport HTML via la sortie standard.

### Étape 7 — Loki + Alloy

- Requêtes LogQL d'exploration : `{namespace="scoring", container="kserve-container"}`, le filtre `|= \`"event":"prediction"\` | json | prediction = "1"` (refus) et `sum by (revision) (count_over_time(… [1m]))` (comptage par révision).

### Étape 8 — Alertes et Slack

- `kubectl apply -f monitoring/scoring-alerts.yaml` : charge les recording rules et les six alertes.
- `kubectl create secret generic slack-webhook -n monitoring --from-literal=url="…" --dry-run=client -o yaml | kubectl apply -f -` : stocke l'URL du webhook (secret, jamais dans Git).
- `kubectl apply -f monitoring/alertmanager-config.yaml` : configure le routage par sévérité.
- `curl -s http://localhost:9093/api/v2/status | grep -o 'monitoring/scoring-slack/slack-ml[a-z-]*' | sort -u` : vérifie que les receivers préfixés sont enregistrés.
- `curl -s -X POST http://localhost:9093/api/v2/alerts -H "Content-Type: application/json" -d '[{…}]'` : injecte une alerte de test pour valider la chaîne sans attendre une panne.

### Étape 9 — Démo drift → alerte → ré-entraînement

- `uv run python monitoring/traffic.py --url http://localhost:30080 --host-header "$ISVC_HOST" --mode drift --drift age=0.6 --drift credit_amount=2.5 --duration 1200` : injecte un drift (âges × 0,6, montants × 2,5).
- `kubectl create job --from=cronjob/scoring-drift manual-run-2 -n monitoring` puis `kubectl logs -n monitoring job/manual-run-2 -f` : relance l'analyse de drift sans attendre le quart d'heure.
- `CT_AUTO_PROMOTE=0 uv run python -m pipelines.trigger_ct serve` : lance le webhook de ré-entraînement du Lab 6 sur `localhost:8095` (sans auto-promotion : il s'arrête au canary à 10 %).
- `REFERENCE_URI=… LOKI_URL=http://localhost:3100 PUSHGATEWAY_URL="" REPORTS_URI="" CT_WEBHOOK_URL=http://localhost:8095/drift uv run python monitoring/evidently-job/calc_drift.py` : rejoue le calcul depuis le poste en pointant le webhook ; attendu un HTTP 202 avec un `run_id`.
- `cp labs/m7-monitoring/starter/rapport-m7.md reports/rapport-m7.md` : crée le squelette du rapport (six sections).

### Étape 10 — Analyser le canary

- `kubectl get isvc scoring-model -n scoring` : affiche les colonnes `PREV`/`LATEST` (ex. 90/10).
- `uv run python monitoring/traffic.py --url http://localhost:30080 --host-header "$ISVC_HOST" --duration 180` : trafic normal pour juger le canary.
- `uv run python monitoring/canary_analysis.py` : analyse les six garde-fous et propose une décision (CONTINUE/ROLLBACK/PROMOTE).
- `uv run python monitoring/canary_analysis.py --apply --push` : applique la décision par Git (canary 50 %, puis promotion 100 %).
- `git log --oneline -3` : vérifie les commits `gitops(prod): …`.
- `uv run python -m pipelines.gitops deploy --env prod --canary 10 --image localhost:5001/scoring-api --tag 2.0.0 --push` : simule une release défectueuse (image sans métriques) → attendu ROLLBACK.
- `uv run python -m pipelines.gitops deploy --env prod --canary 100 --push` : redéploie la bonne version à 100 % après le rollback.
- `cat reports/m7/canary-analysis.md` : relit la trace écrite de la décision.
- `git switch -c lab/m7-observabilite` puis `git add reports monitoring && git commit -m "docs(m7): …"` et `git push -u origin lab/m7-observabilite` : seconde branche avec les fichiers des étapes 3 à 10.
- `make check-m7` : vérification automatique finale (code retour = nombre de contrôles en échec).

## Pièges / points d'attention

- **Ressources** : kube-prometheus-stack ≈ 2 Go + Loki/Alloy ≈ 0,5 Go ; prévoir ≥ 10 Go pour Docker. Libérer la mémoire des labs précédents (arrêter Airflow avec `(cd infra/airflow && docker compose down)`, mettre KFP en pause) plutôt que recréer le cluster.
- **Pods OOMKilled** : réduire la rétention Prometheus à 6 h, mettre KFP en pause, monter Docker à 12 Go.
- **node-exporter `CreateContainerError : not a shared or slave mount`** : sous Docker Desktop la racine du nœud kind n'est pas partagée ; garder `prometheus-node-exporter.hostRootFsMount.enabled: false` puis relancer `make monitoring`.
- **`make monitoring` en `context deadline exceeded`** : images longues à tirer au premier essai ; relancer (idempotent) et surveiller avec `kubectl get pods -n monitoring -w`.
- **PodMonitor non pris en compte** : `podMonitorSelectorNilUsesHelmValues` doit être `false`, sinon le label `release: kube-prom-stack` est exigé ; vérifier aussi le namespace (`scoring` et `scoring-staging`).
- **Cible `scoring-api` absente ou DOWN (404)** : service à zéro pod (staging) ou révision non instrumentée (image d'avant le Lab 7) — normal pendant un canary vers une ancienne image.
- **`/metrics` renvoie 307** : `app.mount("/metrics", make_asgi_app())` redirige vers `/metrics/` ; utiliser la route `GET /metrics` directe (ou `curl -L`).
- **`Duplicated timeseries in CollectorRegistry`** : métriques dans le registre global + `serve.py` importé deux fois (tests du Lab 6) ; utiliser un registre propre au module.
- **Compteurs incohérents avec plusieurs workers uvicorn** : chaque processus a ses propres compteurs ; un worker par pod (Knative ajoute des pods) ou mode multiprocess.
- **Grafana « No data »** : fenêtre `rate()` trop courte ou absence de trafic ; passer de `[1m]` à `[5m]` au démarrage et relancer `traffic.py`.
- **Dashboard absent de Grafana** : ConfigMap sans le label `grafana_dashboard: "1"` ou JSON invalide ; inspecter les logs du conteneur `grafana-sc-dashboard`.
- **Job de drift `NoSuchKey`** : `train.csv` non poussé sur le S3 ou empreinte DVC différente ; refaire `dvc push` puis recréer la ConfigMap `scoring-drift-reference`.
- **Job de drift « Moins de 500 demandes »** : trafic insuffisant dans la dernière heure ou Loki vide ; relancer le trafic et vérifier la requête LogQL.
- **Job de drift `CreateContainerConfigError`** : Secret `s3-credentials` ou ConfigMap absent du namespace `monitoring`.
- **Evidently `KeyError`** : API 0.4 (`as_dict()`, `drift_by_columns`) utilisée avec Evidently 0.7 ; utiliser `snapshot.dict()` et épingler `evidently==0.7.23`.
- **Pushgateway 404 / connection refused** : URL incorrecte ; le nom complet est `prometheus-pushgateway.monitoring:9091`.
- **Loki pods Pending** : volume persistant demandé sans classe de stockage adaptée ; garder `singleBinary.persistence.enabled: false`.
- **Loki `max entries limit per query exceeded`** : `limit` > 5 000 ; paginer comme `calc_drift.py`.
- **Alertmanager `config reload failed`** : erreur dans l'`AlertmanagerConfig` (Secret introuvable, matcher invalide) ; vérifier les logs de l'operator et le Secret `slack-webhook` (clé `url`).
- **Slack `invalid_payload`** : template de message incorrect ; simplifier et tester d'abord avec webhook.site.
- **Webhook CT : 401** : `CT_WEBHOOK_TOKEN` absent ou différent entre serveur et appel ; faire `set -a && source .env && set +a` dans les deux terminaux.
- **`canary_analysis.py` : WAIT en boucle** : pas assez de trafic sur le canary (10 %) ; relancer `traffic.py` plus longtemps ou monter le palier.
- **`canary_analysis.py --apply` : code 2** : l'outil GitOps refuse un arbre modifié dans `k8s/overlays/` ou une branche `lab/*` (les opérations partent de `main`).
- **Dataset personnalisé** : tout est lu dans `dataset.yaml` (variables, classe positive, attributs sensibles) ; un attribut « d'audit seulement » absent des features n'est pas envoyé à l'API et ne peut pas être suivi par groupe ; la classe positive n'est un « refus » que pour le crédit ; le seuil PSI 0,25 suppose une référence de quelques centaines de lignes.

*Fiche générée à partir de Lab7-Enonce.pdf et Lab7-Commandes.pdf — cours MLOps.*
