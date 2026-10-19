# Lab 8 — Audit de gouvernance du modèle de scoring · outils & commandes

Ce lab fait passer le modèle de scoring de crédit par une revue complète de gouvernance : sécurité de l'image et des modèles, mesure puis atténuation des disparités selon les attributs sensibles, explicabilité SHAP, Model Card et conformité AI Act / RGPD. La fiche « Commandes » du cours liste les commandes sans dire à quoi elles servent : cette fiche comble ce manque en expliquant l'utilité et le rôle de chaque outil, cible et commande.

## Outils de la stack

| Outil/service | Rôle / utilité |
|---|---|
| Trivy | Scanner de vulnérabilités (CVE) des images Docker ; compare l'image `scoring-api:1.0.0` du Lab 4 et l'image durcie, et sert de garde-fou en CI. Tourne en conteneur, version épinglée. |
| ModelScan | Détecte statiquement les opérateurs dangereux d'un fichier de modèle (pickle/joblib) ; son code retour est exploitable par la CI. |
| skops | Sérialisation sûre des modèles MLflow 3 ; `get_untrusted_types` liste les types que le fichier demande d'instancier, pour auditer le modèle du Registry. |
| Fairlearn | Mesure les disparités (métriques par groupe, parité démographique, règle des 4/5, equalized odds) et fournit les atténuations (`ThresholdOptimizer`, `ExponentiatedGradient`). |
| SHAP | Explique le modèle : importance globale (|SHAP| moyen) et explications locales (waterfall) pour motiver un refus. |
| Model Cards | Documentation standard du modèle en 11 sections, alimentée par les rapports produits. |
| External Secrets Operator (ESO) | Recopie des secrets d'un « coffre » vers Kubernetes : source de vérité unique, rotation et resynchronisation automatiques. |
| Docker / Docker Hardened Images (DHI) | Construction des images ; DHI fournit une base distroless (Python 3.12, `nonroot`, sans shell) pour durcir l'image de serving. |
| Kubernetes / kubectl / RBAC | Applique et vérifie les identités minimales (`ServiceAccount`, `Role`, `RoleBinding`) et les droits effectifs. |
| KServe (InferenceService) | Sert le modèle en production ; on lui assigne le ServiceAccount `scoring-sa`. |
| Argo CD | GitOps : applique l'état de `main` ; tout changement de prod passe par Git, pas par `kubectl patch`. |
| MLflow Registry | Source du champion (`scoring-model@champion`), de sa version et de son seuil de décision. |
| SeaweedFS (S3) | Stockage objet des artefacts ; le secret `s3-credentials` du namespace `storage` sert de coffre simulé. |
| uv / pytest / git / gh | Environnement Python et scripts (`uv run`), tests K8s, versionnement et pull request. |

## Cibles Makefile et scripts

| Cible / script | Rôle |
|---|---|
| `make check-m8` | Vérification automatique des livrables du lab ; le code retour est le nombre de contrôles en échec (0 = tout vert). |
| `make external-secrets` | Installe External Secrets Operator dans le cluster (étape 8). |
| `make pf` | Lance les port-forwards nécessaires à la séance. |
| `governance/__init__.py` | Paquet Python des scripts d'audit, lancés en module (`python -m governance.…`) depuis la racine, comme `src/`. |
| `governance.fairness_analysis` | Mesure les disparités pour chaque attribut sensible de `dataset.yaml` au seuil du champion ; écrit `reports/fairness_baseline.json`. |
| `governance.fairness_mitigation` | Compare trois atténuations (sans l'attribut, ThresholdOptimizer, ExponentiatedGradient) ; écrit `reports/fairness_mitigated.json`. |
| `governance.explain` | Importance globale SHAP ; produit `reports/shap_summary_bar.png`, `shap_beeswarm.png` et `shap_importance.json`. |
| `governance.explain_local` | Trois explications locales (waterfall) + motifs rédigés ; produit `reports/shap_waterfall_*.png` et `explanations_local.md`. |
| `packaging/scan-demo/make_pickles.py` | Génère deux pickles de démonstration (un sain, un piégé) pour ModelScan. |
| `packaging/sample_request.py` | Produit une requête d'exemple pour tester l'API `/predict`. |
| `packaging/Dockerfile.distroless` | Construit l'image de serving durcie (stage builder `-dev` + runtime distroless). |
| `tests/test_k8s.py` | Tests des overlays Kustomize (dont le `serviceAccountName`). |

## Commandes clés, expliquées

### Branche de travail

- `cd ~/mlops-cours-2026` : racine du fork ; toutes les commandes s'y exécutent.
- `git switch main && git pull` : met `main` à jour avant de partir d'une base saine.
- `git switch -c lab/m8-governance` : crée la branche de travail du lab (convention Lab 0).

### Étape 1.1 — Préparation

- `set -a && source .env && set +a` : charge et exporte les variables d'environnement pour la session.
- `make pf` : démarre les port-forwards (dont MLflow).
- `mkdir -p reports model-card governance k8s` : crée l'arborescence des livrables.
- `uv run python -c "import fairlearn, shap; print(fairlearn.__version__, shap.__version__)"` : vérifie la présence des bibliothèques d'audit (0.14.0 / 0.52.0).
- `trivy() { docker run ... aquasec/trivy:0.75.0 "$@" }` : encapsule Trivy dans un conteneur, version épinglée, avec le volume `trivy-cache` qui conserve la base de vulnérabilités entre deux scans.

### Étape 1.2 — Scan Trivy de l'image du Lab 4

- `docker pull localhost:5001/scoring-api:1.0.0` : récupère l'image dans le démon local (Trivy le lit via le socket Docker).
- `trivy image localhost:5001/scoring-api:1.0.0 > reports/trivy_full.txt` : scan complet, toutes sévérités.
- `trivy image --severity HIGH,CRITICAL localhost:5001/scoring-api:1.0.0 > reports/trivy_critical.txt` : scan focalisé sur les vulnérabilités graves.
- `grep "^Total" reports/trivy_full.txt reports/trivy_critical.txt` : extrait les compteurs par cible scannée (système Debian, puis paquets Python).
- `grep -c "CVE-" reports/trivy_critical.txt` : compte les CVE listées.
- `trivy image --severity HIGH,CRITICAL --ignore-unfixed ...` : ne montre que les CVE corrigeables aujourd'hui.
- `docker buildx imagetools inspect aquasec/trivy:0.75.0` : donne le `sha256:…` pour épingler l'image par empreinte en CI.

### Étape 1.3 — Corriger via une base distroless

- `docker login dhi.io` : s'authentifie au registre DHI (compte Docker gratuit + jeton d'accès personnel).
- `MODEL_VERSION=$(uv run python -c "import json; print(json.load(open('packaging/model/metadata.json'))['model_version'])")` : lit la version du modèle pour l'injecter au build.
- `docker build -f packaging/Dockerfile.distroless -t scoring-api:distroless --build-arg VERSION=1.0.0 --build-arg MODEL_VERSION="$MODEL_VERSION" .` : construit l'image durcie depuis la racine du repo.
- `docker images scoring-api` : liste les images (taille, tag).
- `docker run --rm --entrypoint python scoring-api:distroless --version` : vérifie le Python embarqué (3.12.x).
- `docker inspect -f '{{.Config.User}}' scoring-api:distroless` : vérifie l'utilisateur non-root (65532:65532).
- `docker run --rm --entrypoint sh scoring-api:distroless -c "echo shell" || echo "pas de shell dans l'image"` : confirme l'absence de shell (échec attendu).
- `uv run python packaging/sample_request.py > /tmp/request.json` : prépare une requête de test.
- `docker run -d --rm -p 8001:8000 --name scoring-distroless scoring-api:distroless` : démarre l'API durcie.
- `curl -s http://localhost:8001/health` : vérifie la santé de l'API.
- `curl -s -X POST http://localhost:8001/predict -H "Content-Type: application/json" -d @/tmp/request.json` : compare la prédiction à celle du Lab 4.
- `docker inspect -f '{{.State.Health.Status}}' scoring-distroless` : lit l'état du `HEALTHCHECK`.
- `docker stop scoring-distroless` : arrête le conteneur.
- `trivy image --severity HIGH,CRITICAL scoring-api:distroless > reports/trivy_distroless.txt` : re-scan de l'image durcie.
- `echo "AVANT : $(grep -c CVE- reports/trivy_critical.txt)"` / `echo "APRÈS : ..."` : compare le nombre de CVE avant/après.
- `docker tag scoring-api:distroless localhost:5001/scoring-api:1.0.0-distroless` puis `docker push ...` : publie l'image durcie à côté de la 1.0.0.
- `docker images --digests localhost:5001/scoring-api` : récupère le digest qui identifie réellement l'image auditée.

### Étape 2 — ModelScan : pickle vs ONNX

- `uv run python packaging/scan-demo/make_pickles.py` : génère les deux pickles de démonstration (sain / piégé).
- `uv run modelscan -p packaging/scan-demo/ > reports/modelscan_pickle.txt` : scanne le pickle piégé (détecte un CRITICAL `posix.system`).
- `uv run modelscan -p packaging/model/model.onnx > reports/modelscan_onnx.txt` : scanne l'ONNX (« No issues found », en fait « skipped »).
- `grep -E "Total Issues|CRITICAL|Description" reports/modelscan_pickle.txt` : lit le résultat exploitable.
- `for f in packaging/scan-demo/model.pkl packaging/scan-demo/malicious.pkl packaging/model/model.onnx; do uv run modelscan -p "$f" > /dev/null 2>&1; echo "$f -> code $?"; done` : montre les codes retour (0, 1, 3) — la CI doit échouer si code 1 ou 2.
- `uv run modelscan -p packaging/scan-demo/ -r json -o reports/modelscan_pickle.json` : rapport JSON exploitable par un outil.
- `uv run mlflow artifacts download -u models:/scoring-model@champion -d /tmp/champion` : télécharge le champion du Registry.
- `uv run python -c "from skops.io import get_untrusted_types; print(get_untrusted_types(file='/tmp/champion/model.skops'))"` : liste les types demandés par le fichier skops (attendu `['numpy.dtype']`).

### Étape 3 — Fairlearn : mesurer les disparités

- `grep -A6 "^sensitive:" dataset.yaml` : affiche la section des attributs sensibles (le script ne code aucun nom en dur).
- `uv run python -m governance.fairness_analysis | tee reports/fairness_baseline.txt` : lance l'analyse, affiche le rapport et l'archive.

### Étape 4 — Atténuation des disparités

- `uv run python -m governance.fairness_mitigation | tee reports/fairness_mitigated.txt` : compare les stratégies et chiffre le compromis coût métier / équité.

### Étape 5 — SHAP

- `uv run python -m governance.explain` : importance globale + graphiques (bar, beeswarm, JSON).
- `uv run python -m governance.explain_local` : trois explications locales (refus net, acceptation nette, cas limite) + motifs rédigés.
- `uv run python -m governance.explain_local --id 819` : explique une demande précise du jeu de test.
- `ls reports/shap_waterfall_*.png` : vérifie les fichiers produits.

### Étape 6 — Générer la Model Card

- `cp labs/m8-governance/starter/model-card.md model-card/model-card.md` : part du squelette (11 sections).
- `grep "^## " model-card/model-card.md` : liste les sections du document.
- heredoc `uv run python - <<'EOF' ... EOF` : génère les tableaux par groupe depuis `reports/fairness_baseline.json`, à coller dans la section 4 (chiffres réels, jamais recopiés).

### Étape 7 — RBAC Kubernetes minimal

- `kubectl apply -f k8s/rbac-scoring.yaml` : crée le `ServiceAccount` `scoring-sa`, le `Role` `scoring-reader` et le `RoleBinding`.
- `kubectl get serviceaccount scoring-sa -n scoring` : vérifie la création.
- `kubectl kustomize k8s/overlays/prod | grep serviceAccountName` : rend l'overlay de prod et vérifie le patch.
- `uv run pytest -q tests/test_k8s.py` : exécute les tests des overlays.
- `git add k8s/rbac-scoring.yaml k8s/overlays/prod/kustomization.yaml` puis `git commit -m "feat(k8s): ..."` : versionne le RBAC et le patch prod.
- `SA=system:serviceaccount:scoring:scoring-sa` : identité sous laquelle on teste les droits.
- `kubectl auth can-i create pods --as=$SA -n scoring` (attendu no), `list secrets` (no), `get secret/s3-credentials -n scoring` (yes), `get secret/s3-credentials -n storage` (no) : vérifie le moindre privilège.

### Étape 8 — ExternalSecret pour les identifiants S3

- `make external-secrets` : installe External Secrets Operator.
- `kubectl get pods -n external-secrets` : vérifie les trois déploiements (contrôleur, webhook, cert-controller).
- `kubectl apply -f k8s/mlflow-creds-source.yaml` : crée ServiceAccount, Role, RoleBinding, `SecretStore` et `ExternalSecret`.
- `kubectl wait --for=condition=Ready secretstore/course-secrets -n scoring --timeout=60s` : attend que le SecretStore soit prêt.
- `kubectl wait --for=condition=Ready externalsecret/s3-credentials -n scoring --timeout=60s` : attend la synchronisation du Secret.
- `kubectl get secretstore,externalsecret -n scoring` : état des objets.
- `kubectl get secret s3-credentials -n scoring -o jsonpath='{.metadata.ownerReferences[0].kind}{"\n"}{.metadata.annotations}{"\n"}'` : vérifie que le Secret appartient à l'ExternalSecret et porte les annotations KServe.
- `for ns in storage scoring; do kubectl get secret s3-credentials -n $ns -o jsonpath='{.data.AWS_ACCESS_KEY_ID}' | base64 -d; echo " ($ns)"; done` : compare les valeurs à la source de vérité.
- `kubectl delete secret s3-credentials -n scoring` puis `sleep 10 && kubectl get secret s3-credentials -n scoring` : montre que l'opérateur recrée le Secret.
- `kubectl get externalsecret s3-credentials -n scoring` : reste en `SecretSynced`.

### Étape 9 — Conformité AI Act, PR et vérification

- `cp labs/m8-governance/starter/compliance-ai-act.md reports/compliance-ai-act.md` : part du squelette (5 sections + sources).
- `git add governance model-card k8s packaging/Dockerfile.distroless packaging/Dockerfile.distroless.dockerignore reports ':(exclude)packaging/scan-demo/*.pkl'` : committe l'audit sans les pickles de démonstration.
- `git commit -m "feat(lab8): audit de gouvernance (...)"` / `git push -u origin lab/m8-governance` / `gh pr create` : pousse la branche et ouvre la pull request.
- `kubectl annotate application scoring-prod -n argocd argocd.argoproj.io/refresh=normal --overwrite` : force Argo CD à resynchroniser sans attendre.
- `kubectl wait isvc/scoring-model -n scoring --timeout=180s --for=jsonpath='{.spec.predictor.serviceAccountName}'=scoring-sa` : attend le nouveau ServiceAccount sur l'InferenceService.
- `kubectl wait --for=condition=Ready isvc/scoring-model -n scoring --timeout=180s` : attend que l'InferenceService soit prêt.
- `POD=$(kubectl get pods -n scoring -l serving.kserve.io/inferenceservice=scoring-model --sort-by=.metadata.creationTimestamp -o name | tail -1)` : récupère le pod de la nouvelle révision.
- `kubectl get "$POD" -n scoring -o jsonpath='{.spec.serviceAccountName}{"\n"}'` : confirme que le pod tourne sous `scoring-sa`.
- `kubectl get "$POD" -n scoring -o yaml | grep -c kube-api-access` : résultat 0 = aucun jeton monté.
- `make check-m8` : vérification automatique finale (code retour = nombre d'échecs).

## Pièges / points d'attention

- **Trivy : `TOOMANYREQUESTS`** au téléchargement de la base — limite de débit du registre. Relancer plus tard ; la base est en cache dans `trivy-cache`, `--skip-db-update` réutilise le cache local.
- **Trivy : `unable to find the specified image`** — l'image n'est pas dans le démon Docker (Trivy le lit via le socket) : faire `docker pull localhost:5001/scoring-api:1.0.0` avant de scanner.
- **`docker build : unauthorized / pull access denied` sur `dhi.io`** — pas connecté au registre DHI : `docker login dhi.io` avec identifiant et jeton personnel.
- **Distroless : `ModuleNotFoundError` au démarrage** — venv non copié ou builder/runtime sur des Python différents : vérifier `COPY --from=builder /opt/venv /opt/venv`, `ENV PATH` et les deux `FROM` en `3.12-debian13`.
- **`docker exec … sh : executable file not found`** — normal, l'image distroless n'a pas de shell : déboguer avec `--entrypoint python ... -c "…"` ou une image `-dev`.
- **ModelScan : « No issues found » mais rien d'analysé** — format non supporté (ONNX, skops) : fichier « skipped », code retour 3 ; utiliser `--show-skipped`, et `skops.io.get_untrusted_types` pour skops.
- **ModelScan : `argument of type 'NoneType' is not iterable`** — pickle écrit par joblib (bug de ModelScan 0.8.8) : produire le pickle avec `pickle.dump`.
- **`MlflowException: Registered model alias champion not found`** — Lab 1 non terminé ou port-forward arrêté : `make pf`, puis vérifier l'alias dans l'UI MLflow.
- **`KeyError: 'decision_threshold'`** — champion entraîné avant l'ajout du paramètre : `dvc repro` puis `python -m src.promote`.
- **`ModuleNotFoundError: No module named 'governance'`** — script lancé par son chemin ou hors racine : lancer depuis la racine `uv run python -m governance.fairness_analysis`.
- **Fairlearn : `ValueError` sur `sensitive_features`** — attribut absent du test ou `bins` ne couvrant pas toutes les valeurs (NaN) : vérifier `sensitive` dans `dataset.yaml` ; les bins doivent englober min et max.
- **Un groupe affiche des taux à 0 % ou 100 %** — groupe minuscule : lire l'IC 95 % et l'alerte « moins de 30 demandes » ; non concluant, à documenter.
- **ThresholdOptimizer : `not binary` ou forme inattendue** — `predict_proba` ne renvoie pas `(n, 2)` : le modèle doit être un classifieur binaire (cible encodée 0/1).
- **SHAP : `Additivity check failed in TreeExplainer`** — explainer « interventionnel » sur des arbres scikit-learn : utiliser `shap.TreeExplainer(modèle)` sans données de fond.
- **SHAP waterfall : `IndexError` ou valeurs 3D** — classifieur à deux sorties `(n, colonnes, 2)` : prendre la classe positive (`sv[..., 1]`).
- **RoleBinding refusé ou sans effet** — mauvais namespace : le RoleBinding doit être dans le même namespace que le Role ; le sujet peut venir d'un autre namespace.
- **`kubectl patch isvc` sans effet durable** — Argo CD ré-applique le manifest de Git : modifier `k8s/overlays/prod/kustomization.yaml` et fusionner la PR.
- **ISVC toujours sur `default` après la fusion** — PR non fusionnée, Argo CD non synchronisé, ou patch mal indenté : `kubectl kustomize k8s/overlays/prod | grep serviceAccountName`, vérifier l'Application `Synced`, forcer l'annotation refresh.
- **ExternalSecret : `SecretSyncedError`** — SecretStore invalide ou droits de lecture manquants : `kubectl describe externalsecret` / `secretstore`, vérifier le Role `eso-s3-reader` du namespace `storage`.
- **SecretStore : `could not verify if client is valid`** — ESO n'obtient pas de jeton pour `eso-s3-reader` ou CA introuvable : vérifier le ServiceAccount dans `scoring` et la ConfigMap `kube-root-ca.crt`.
- **`no matches for kind "ExternalSecret" in version "external-secrets.io/v1beta1"`** — ancien manifest : `v1beta1` n'est plus servie, passer en `external-secrets.io/v1`.
- **`auth can-i : yes` sur `create pods`** — test lancé avec votre identité ou droits hérités : tester avec `--as=system:serviceaccount:scoring:scoring-sa`, puis `kubectl get clusterrolebinding -o wide | grep scoring-sa`.
- **Un tag n'est pas une preuve** — `scoring-api:distroless` peut être reconstruite demain avec un autre contenu : ce qui identifie l'image auditée est son digest ; le scanner lui-même peut être compromis (Trivy piégé en 2026), d'où l'épinglage par version postérieure à l'incident et, en CI, par digest.
- **Ne jamais charger `malicious.pkl`** — un pickle exécute du code arbitraire à son chargement ; ModelScan détecte des motifs statiques mais ne garantit pas l'absence de tout risque.
- **Pour Fairlearn, « sélectionner » = REFUSER** — la classe positive est le risque (`bad`) ; la règle des 4/5 se calcule sur les taux d'acceptation (l'issue favorable).
- **Auditer au bon seuil** — le modèle ne décide pas à 0,5 mais à `decision_threshold` (0,318) : auditer à ce seuil, sinon on audite un autre modèle.
- **Retirer l'attribut sensible ne suffit pas** (« fairness through unawareness ») — les proxies (AUC 0,71-0,78) permettent de deviner le groupe ; « ThresholdOptimizer » avec son objectif par défaut (accuracy) double le coût métier et applique un seuil différencié par groupe (juridiquement risqué).
- **Sécurité des jetons** — par défaut, les pods reçoivent un jeton d'API monté, inutile et donc dangereux : la défense en profondeur vise `automountServiceAccountToken: false` et `can-i : no/no/yes`.
- **SHAP décrit le modèle, pas une causalité** — l'explication due au demandeur (RGPD art. 22) doit être compréhensible, pas une formule.
- **Model Card Toolkit non maintenu** — générer les tableaux dans la CI, le texte d'analyse reste écrit par l'équipe.
- **Dataset personnel** — déclarer tous les attributs protégés dans `sensitive`, garder `target.positive_class` = classe défavorable, regrouper les modalités rares (< 30 individus = taux inexploitables).

*Fiche générée à partir de Lab8-Enonce.pdf et Lab8-Commandes.pdf — cours MLOps.*
