# Lab 5 — Déployer le modèle de scoring sur KServe · outils & commandes

Ce lab fait passer l'image Docker du Lab 4 (`localhost:5001/scoring-api:1.0.0`) à un service production-ready : installer KServe dans le cluster kind du cours (sans le recréer), exposer l'InferenceService `scoring-model` via la passerelle Kourier (port `30080`), configurer l'autoscaling (scale-to-zero → 5 pods), benchmarker avec Locust (100/500/1000 utilisateurs) et déployer une v2.0.0 en canary 90/10. Livrable : ISVC déployé + benchmark P50/P95/P99 + canary 90/10.

## Outils de la stack

| Outil/service | Rôle / utilité |
|---|---|
| KServe (`v0.21.0`, ns `kserve`) | CRD `InferenceService`, contrôleur et runtimes de modèles (sklearn, Triton…) : sert un modèle à partir d'un seul YAML. |
| Knative Serving (`knative-serving`) | Révisions immuables (image + configuration), routage du trafic entre versions, autoscaler KPA et scale-to-zero. |
| Kourier (`kourier-system`) | Passerelle HTTP Envoy de Knative, exposée en NodePort `30080` ; le routage se fait par l'en-tête `Host`. |
| cert-manager (`cert-manager`) | Émet les certificats TLS des webhooks de KServe. |
| metrics-server (`kube-system`) | Alimente `kubectl top` (CPU / mémoire des pods) ; installé avec `--kubelet-insecure-tls` (requis sur kind). |
| KEDA (`v2.21.0`, ns `keda`, option) | Autoscaling événementiel (files, Prometheus, cron) ; s'installe avec `KSERVE_WITH_KEDA=1 make kserve`. |
| kind | Cluster Kubernetes local du cours ; le port `30080` y est mappé depuis le Lab 0, précisément pour la passerelle KServe. |
| Registry local `localhost:5001` | Registry du cours branché au cluster : les nœuds tirent eux-mêmes les images (`localhost:5001/...`), plus de `kind load`. |
| Locust `2.46.6` | Générateur de charge : mesure le débit (RPS) et les latences P50/P95/P99 ; fait partie du socle `uv sync`. |
| Triton (`kserve-tritonserver`) | Serveur standard d'inférence, protocole V2, modèle ONNX téléchargé depuis S3 (comparaison de l'étape 8). |
| BentoML | Image HTTP du Lab 4 (pipeline scikit-learn, port `3000`) servie telle quelle par KServe. |
| SeaweedFS (S3) | Stockage objet du cours (`s3://models/triton` pour Triton), filer visible sur `http://localhost:8888/buckets/`. |
| MLflow | Registry du modèle : alias `champion` et paramètre `decision_threshold` = seuil métier servi partout. |
| uv | Exécution des scripts Python du repo dans un environnement synchronisé. |

## Cibles Makefile et scripts

| Cible / script | Rôle |
|---|---|
| `make pf` | Lance les port-forwards (dont l'accès aux services) : à vérifier en début de séance. |
| `make kserve` | Installe Knative, Kourier, cert-manager, KServe et metrics-server via `infra/kserve/install.sh`, versions lues dans `infra/versions.env`. Idempotent : relançable en cas d'erreur réseau. |
| `make check-m5` | Vérification automatique non destructive (infra, service, seuil, autoscaling, canary, livrables) ; code retour = nombre de contrôles en échec. |
| `infra/kserve/install.sh` | Enchaîne l'installation ; configure Knative (ingress-class kourier, domaine `example.com`, NodePort 30080), exclut `localhost:5001` de la résolution des tags et patche `inferenceservice-config` (mode Knative). |
| `serving/monitor.sh` | Tableau de bord « live » de l'autoscaling (ISVC, KPA, pods, CPU/mémoire) rafraîchi toutes les 2 s. |
| `serving/payloads.py` | Construit les corps de requête V1 (`instances`) et V2 (tenseurs) depuis `dataset.yaml` et de vraies lignes de `test.csv`. |
| `serving/locustfile.py` | Scénario Locust : rejoue des requêtes de scoring `/predict` (ou V2 Triton selon `API_FORMAT`). |
| `serving/locust_report.py` | Agrège les CSV Locust en un tableau Markdown (RPS, P50/P95/P99, erreurs). |
| `serving/prepare_triton.py` | Dépose le graphe ONNX et `config.pbtxt` sur S3 au format attendu par Triton. |
| `serving/v2_client.py` | Interroge Triton en protocole V2 et décide au seuil métier publié ; `--compare` rejoue les mêmes demandes vers l'API du Lab 4. |
| `packaging/sample_request.py` | Génère un corps de requête de test à partir de vraies demandes (script du Lab 4). |
| `labs/m5-serving/check.sh` | Script exécuté par `make check-m5`. |

## Commandes clés, expliquées

### Étape 1 — Installer KServe dans le cluster du cours

- `cd ~/mlops-cours-2026` — se placer à la racine du fork.
- `git switch main && git pull` puis `git switch -c lab/m5-serving` — créer la branche de travail du lab depuis `main` à jour.
- `set -a && source .env && set +a` — charger les variables d'environnement (S3, registry…) dans le shell.
- `make pf` — démarrer les port-forwards nécessaires.
- `kubectl config current-context` / `kubectl get nodes` — confirmer le contexte `kind-mlops-lab` et les 3 nœuds Ready.
- `docker port mlops-lab-control-plane` — vérifier le mapping `30080/tcp -> 0.0.0.0:30080`.
- `make kserve` — installer Knative, Kourier, cert-manager, KServe et metrics-server en une commande.
- `kubectl get pods -n knative-serving` (idem `kourier-system`, `cert-manager`, `kserve`) — vérifier que chaque brique tourne.
- `kubectl get crd | grep serving.kserve.io` et `kubectl get clusterservingruntimes` — vérifier les CRD et les runtimes KServe.
- `kubectl get svc kourier -n kourier-system` — confirmer le service NodePort `80:30080/TCP`.

### Étape 2 — Rendre l'image disponible pour le cluster

- `curl -s http://localhost:5001/v2/scoring-api/tags/list` — confirmer que le tag `1.0.0` est bien publié dans le registry (plus besoin de `kind load`).
- `kubectl create namespace scoring` — créer le namespace qui hébergera le service.
- `mkdir -p serving reports/m5` — créer les dossiers des manifests et des rapports.

### Étape 3 — Créer l'InferenceService minimal

- `kubectl apply -f serving/scoring-isvc-v1.yaml` — déployer l'ISVC (conteneur personnalisé `kserve-container`, image `1.0.0`, mode Knative).
- `kubectl get isvc -n scoring -w` — suivre jusqu'à `READY=True` (Ctrl-C ensuite).
- `kubectl get ksvc,revisions,kpa -n scoring` — inspecter le Knative Service, la révision et l'autoscaler créés pour vous.
- `kubectl get pods -n scoring` — vérifier le pod `2/2` (`kserve-container` + `queue-proxy`).
- `kubectl describe isvc scoring-model -n scoring | tail -15` — lire l'état détaillé et les conditions.

### Étape 4 — Tester via la passerelle (port 30080) + curl

- `ISVC_HOST=$(kubectl get isvc scoring-model -n scoring -o jsonpath='{.status.url}' | cut -d/ -f3)` — extraire le nom d'hôte (domaine `example.com` sans DNS réel).
- `curl -s -H "Host: ${ISVC_HOST}" http://localhost:30080/health` — appeler le service via Kourier ; sans en-tête `Host`, Kourier répond 404.
- `uv run python packaging/sample_request.py -n 2 > /tmp/request.json` — fabriquer un corps de requête à partir de vraies lignes de `test.csv`.
- `curl -s -X POST http://localhost:30080/predict -H "Host: ${ISVC_HOST}" -H "Content-Type: application/json" -d @/tmp/request.json` — tester une prédiction et vérifier le seuil `0.3182`.
- `uv run python -c "... MlflowClient ... get_model_version_by_alias('scoring-model','champion') ... decision_threshold ..."` — vérifier le seuil du champion dans MLflow.
- `curl -s -H "Host: ${ISVC_HOST}" http://localhost:30080/metadata | python3 -m json.tool | grep -E '"threshold"|"model_version"'` — comparer le seuil et la version servis à ceux du champion.
- `kubectl run -n scoring client-test --rm -i --restart=Never --image=python:3.12-slim -- python -c "... urlopen('http://scoring-model-predictor.scoring/health') ..."` — appeler le service par son adresse interne (sans le port 30080).

### Étape 5 — Configurer l'autoscaling

- `kubectl apply -f serving/scoring-isvc-v1-autoscale.yaml` — passer à `minReplicas: 0`, `maxReplicas: 5`, `scaleMetric: rps`, `scaleTarget: 200`.
- `kubectl get isvc scoring-model -n scoring` — voir la nouvelle révision prendre 100 % du trafic.
- `kubectl get kpa -n scoring` — voir les autoscalers et l'arrêt des pods de l'ancienne révision.
- `kubectl get revision scoring-model-predictor-00002 -n scoring -o jsonpath='{.metadata.annotations}' | python3 -m json.tool | grep autoscaling` — vérifier les annotations KPA (metric, target, min-scale, max-scale, window).
- `chmod +x serving/monitor.sh && ./serving/monitor.sh` — lancer le tableau de bord dans un terminal dédié.
- `kubectl get pods -n scoring -w` — observer le scale-to-zero (pod qui passe à `Terminating` puis disparaît).
- `curl -s -o /dev/null -w "HTTP %{http_code} en %{time_total} s\n" -H "Host: ${ISVC_HOST}" http://localhost:30080/health` — lancer deux fois : mesurer le démarrage à froid puis l'appel chaud.
- `kubectl rollout restart -n kube-system deploy/metrics-server` — corriger un `Metrics API not available` de `kubectl top`.

### Étape 6 — Test de charge avec Locust

- `export ISVC_HOST` — exporter le nom d'hôte que Locust placera dans l'en-tête `Host`.
- `uv run locust -f serving/locustfile.py --host http://localhost:30080 --users 100 --spawn-rate 20 --run-time 2m --headless --csv=reports/m5/run_100 --html=reports/m5/run_100.html` — run 1 (répéter avec 500 puis 1000 utilisateurs) ; produit les rapports.
- `uv run python serving/locust_report.py` — générer le tableau Markdown des trois runs (requêtes, RPS, P50, P95, P99, erreurs).
- `BATCH_SIZE=50 uv run locust -f serving/locustfile.py --host http://localhost:30080 --users 20 --spawn-rate 20 --run-time 1m --headless --csv=reports/m5/batch_50` — bonus « batch » : 50 demandes par requête.
- `uv run python serving/locust_report.py "reports/m5/batch_*"` — résumer le run batch (comparer les prédictions/s).

### Étape 7 — Déployer v2.0.0 en canary 90/10

- `MODEL_VERSION=$(uv run python -c "... metadata.json['model_version'] ...")` — récupérer la version du modèle à embarquer.
- `docker build -f packaging/Dockerfile -t localhost:5001/scoring-api:2.0.0 --build-arg VERSION=2.0.0 --build-arg MODEL_VERSION="$MODEL_VERSION" .` — construire l'image v2 (avec la route `/version`).
- `docker push localhost:5001/scoring-api:2.0.0` — publier l'image dans le registry du cours.
- `docker run -d --rm -p 8000:8000 --name scoring-v2 localhost:5001/scoring-api:2.0.0` puis `sleep 5 && curl -s http://localhost:8000/version` puis `docker stop scoring-v2` — tester l'image en local avant déploiement.
- `kubectl apply -f serving/scoring-isvc-canary.yaml` — déployer le canary (`canaryTrafficPercent: 10`, `enable-tag-routing: true`).
- `kubectl get isvc scoring-model -n scoring -w` — vérifier `PREV 90 / LATEST 10`.
- `for i in $(seq 1 200); do curl -s -H "Host: ${ISVC_HOST}" http://localhost:30080/metadata | grep -oE '"api_version":"[^"]+"'; done | sort | uniq -c` — compter la répartition réelle par version.
- `curl -s -H "Host: latest-${ISVC_HOST}" http://localhost:30080/version` et `curl -s -H "Host: prev-${ISVC_HOST}" http://localhost:30080/metadata | grep -oE '"api_version":"[^"]+"'` — interroger chaque version par son adresse dédiée.
- `kubectl patch isvc scoring-model -n scoring --type merge -p '{"spec":{"predictor":{"canaryTrafficPercent":100}}}'` — promouvoir la v2 à 100 % ; avec `0` à la place, rollback vers la v1.

### Étape 8 — Triton et BentoML (optionnel)

- `uv run python serving/prepare_triton.py` — déposer le graphe ONNX et `config.pbtxt` sur S3 au format Triton.
- `kubectl create secret generic s3-credentials -n scoring --from-literal=AWS_ACCESS_KEY_ID="$AWS_ACCESS_KEY_ID" --from-literal=AWS_SECRET_ACCESS_KEY="$AWS_SECRET_ACCESS_KEY" --dry-run=client -o yaml | kubectl apply -f -` — créer le Secret S3 depuis `.env` (jamais commité).
- `kubectl annotate secret s3-credentials -n scoring --overwrite serving.kserve.io/s3-endpoint=s3.storage:8333 serving.kserve.io/s3-usehttps=0 serving.kserve.io/s3-region="$AWS_DEFAULT_REGION"` — indiquer le point d'accès SeaweedFS.
- `kubectl apply -f serving/kserve-s3-sa.yaml` — créer le ServiceAccount `kserve-s3` qui rattache le Secret aux ISVC.
- `kubectl apply -f serving/scoring-triton-isvc.yaml` puis `kubectl get isvc scoring-triton -n scoring -w` — déployer Triton (1er démarrage long).
- `TRITON_HOST=$(kubectl get isvc scoring-triton -n scoring -o jsonpath='{.status.url}' | cut -d/ -f3)` puis `curl -s -H "Host: ${TRITON_HOST}" http://localhost:30080/v2/models/scoring-model | python3 -m json.tool | head -20` — lire les métadonnées du modèle en protocole V2.
- `uv run python serving/v2_client.py --host "$TRITON_HOST" -n 5 --compare "$ISVC_HOST"` — décider au seuil publié et comparer aux prédictions de l'API du Lab 4.
- `docker tag scoring-api:bentoml localhost:5001/scoring-bento:1.0.0` puis `docker push ...` — publier l'image BentoML du Lab 4.
- `kubectl apply -f serving/scoring-bento-isvc.yaml` puis `kubectl wait --for=condition=Ready isvc/scoring-bento -n scoring --timeout=300s` — déployer et attendre la disponibilité.
- `BENTO_HOST=$(kubectl get isvc scoring-bento -n scoring -o jsonpath='{.status.url}' | cut -d/ -f3)` puis `curl -s -X POST http://localhost:30080/predict -H "Host: ${BENTO_HOST}" -H "Content-Type: application/json" -d @/tmp/request.json` — tester BentoML.
- `ISVC_HOST=$BENTO_HOST API_FORMAT=v1 uv run locust -f serving/locustfile.py --host http://localhost:30080 --users 20 --spawn-rate 20 --run-time 1m --headless --only-summary --csv=reports/m5/compare_bento` — même banc d'essai pour les trois serveurs (répéter pour `compare_fastapi` en v1 et `compare_triton` en v2).
- `uv run python serving/locust_report.py "reports/m5/compare_*"` — comparer les trois serveurs.
- `kubectl delete -f serving/scoring-triton-isvc.yaml -f serving/scoring-bento-isvc.yaml` — supprimer les variantes après la mesure.

### Étape 9 — Rédiger le rapport et livrer

- `cp labs/m5-serving/starter/rapport-m5.md reports/rapport-m5.md` — partir du modèle de rapport fourni.
- `git add serving packaging/app/serve.py packaging/Dockerfile reports/rapport-m5.md reports/m5/*_stats.csv` — indexer les livrables (les HTML Locust restent sur le poste).
- `git commit -m "Lab 5: serving KServe (autoscaling, canary, benchmark)"` — commiter.
- `git push -u origin lab/m5-serving` puis `gh pr create` — pousser et ouvrir la pull request.
- `make check-m5` — lancer la vérification automatique finale.

## Pièges / points d'attention

- `make kserve` : erreur x509 / connection refused sur un webhook — le webhook n'est pas encore prêt ; le script réessaie seul, sinon relancer `make kserve` (idempotent).
- ISVC `READY=False` avec pod en `ImagePullBackOff` — image absente du registry ou nom erroné ; vérifier `curl localhost:5001/v2/scoring-api/tags/list`. Pas de `kind load` : le registry est branché au cluster.
- `failed to resolve image to digest` / `RevisionMissing` — Knative tente de résoudre le tag `localhost:5001/...` ; relancer `make kserve` (exclusion `registries-skipping-tag-resolving`), puis supprimer et réappliquer l'ISVC.
- Un tag publié est immuable : ne jamais repousser une image différente sous un tag existant ; chaque nouvelle version reçoit son propre tag.
- `update rejected: deploymentMode cannot be changed` — le mode est figé à la création de l'ISVC : supprimer (`kubectl delete isvc`) puis réappliquer.
- Canary `100/0` au lieu de `90/10` — ISVC en mode Standard (où `canaryTrafficPercent` est ignoré) ou absence de révision précédente déployée à 100 %.
- `/version` renvoie `{"detail":"Not Found"}` — 9 requêtes sur 10 tombent sur la révision 1.0.0 (image sans `/version`) ; tester via l'adresse `latest-...` ou lire `/metadata`.
- Deux `model_version` différents entre MLflow et le service — image construite avant un changement de champion : réexporter le modèle et reconstruire l'image (Lab 4).
- `curl localhost:30080` : `Connection refused` — service `kourier` pas en NodePort 30080, ou cluster créé sans le mapping ; vérifier `kubectl get svc kourier -n kourier-system` et `docker port mlops-lab-control-plane`.
- `curl` : 404 avec réponse vide — en-tête `Host` absent ou erroné ; le recalculer via `jsonpath='{.status.url}'`.
- `curl /predict` : 503 — pod pas encore prêt ; `kubectl wait --for=condition=Ready pod -l serving.kserve.io/inferenceservice=scoring-model -n scoring`.
- Premier appel lent (quelques secondes) — démarrage à froid normal après scale-to-zero ; garder `minReplicas: 1` pour l'éviter.
- `kubectl top pods` : `Metrics API not available` — metrics-server pas prêt ; attendre 60 s ou `kubectl rollout restart -n kube-system deploy/metrics-server`.
- Sous charge, `DESIREDSCALE` reste à 1 — `scaleTarget` au-dessus de ce qu'un pod sait servir ; baisser la cible ou passer à `scaleMetric: concurrency`.
- Locust 100 % d'échecs / 404 — `ISVC_HOST` non exporté dans le terminal de Locust (`export ISVC_HOST=...`).
- Locust : connection reset au run 1000 — service saturé (attendu) ou Locust lui-même (`CPU usage above 90%`) ; augmenter `maxReplicas`, relâcher les limites CPU, ou ajouter `--processes 4`.
- Triton : `ProgressDeadlineExceeded` — téléchargement de l'image Triton (≈ 6 Go) trop long ; l'annotation `serving.knative.dev/progress-deadline` est déjà à 30 min.
- Triton : `storage-initializer` en erreur (`NoCredentialsError`, `EndpointConnectionError`) — Secret absent/mal annoté ou ServiceAccount non référencé ; revoir l'étape 8.2 et `serviceAccountName: kserve-s3`.
- Triton : modèle `UNAVAILABLE`, failed to load — mauvaise arborescence : `storageUri` doit désigner le dossier contenant `scoring-model/`.
- Triton : 400 `unexpected datatype` — type d'entrée différent du graphe ONNX ; construire la requête avec `serving/payloads.py` (`v2_body`).
- `prepare_triton.py` : `XAmzContentSHA256Mismatch` — export `AWS_REQUEST_CHECKSUM_CALCULATION=when_required` puis relancer.
- Kourier : `SETTINGS_MAX_CONCURRENT_STREAMS` (connexion gRPC périmée) — `kubectl rollout restart deploy/3scale-kourier-gateway -n kourier-system`.
- Passerelle indisponible — port-forward de secours : `kubectl port-forward -n scoring deploy/scoring-model-predictor-00001-deployment 8001:8000` (adapter le n° de révision ; le port 8080 est pris par l'UI KFP).
- Nœud kind plein (`no space left`) — images accumulées : `docker exec mlops-lab-worker crictl rmi --prune` (idem `mlops-lab-worker2` et `mlops-lab-control-plane`).
- Ressources : prévoir ≥ 8 Go de RAM pour Docker ; Knative + Kourier + cert-manager + KServe ≈ 1,5 Go. Libérer Airflow (Lab 2) et mettre KFP en pause plutôt que recréer le cluster.
- La cible de l'autoscaler (`scaleTarget`) dépend du coût d'une prédiction : un modèle plus lourd demandera une cible plus basse ; un gros modèle allonge aussi le démarrage à froid du scale-to-zero.
- Le protocole V2 de Triton ne transporte pas de valeur manquante numérique en JSON : le client V2 écarte ces lignes (elles restent servies par l'API du Lab 4 qui accepte `null`).

*Fiche générée à partir de Lab5-Enonce.pdf et Lab5-Commandes.pdf — cours MLOps.*
