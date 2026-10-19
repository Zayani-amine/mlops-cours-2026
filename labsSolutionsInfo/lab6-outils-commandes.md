# Lab 6 — CI/CD/CT de bout en bout sur K8s · outils & commandes

Ce lab ferme la boucle MLOps : une **dérive** détectée par Evidently déclenche un **ré-entraînement** (KFP), une **gate** promeut (ou refuse) le modèle dans MLflow, puis une mise en **production GitOps** (Argo CD) déploie un canary KServe — sans intervention humaine. Trois chaînes tournent en parallèle et ne communiquent que par **Git** : la CI (GitHub Actions), le CD (Argo CD dans le cluster) et le CT (webhook sur le poste).

## Outils de la stack

| Outil/service | Rôle / utilité |
|---|---|
| GitHub Actions | Exécute la CI sur runners hébergés : lint, tests, build de l'image, test de fumée, publication GHCR, puis commit du tag de staging. |
| GHCR (`ghcr.io`) | Registry d'images ; stocke `scoring-api:2.1.N`, tiré par les nœuds kind. |
| Git / GitHub | Source unique de vérité : toute décision de déploiement est un commit (GitOps). |
| Argo CD | CD GitOps : lit `k8s/overlays/*` et synchronise `scoring-staging`/`scoring-prod` depuis Git, avec `prune` et `selfHeal`. |
| Kustomize | Construit les manifests (une base + un overlay par environnement) qu'Argo CD applique. |
| KServe (`InferenceService`) | Sert le modèle ; fournit le canary natif en mode Knative. |
| Knative | Mode de déploiement KServe activant `canaryTrafficPercent` et le scale-to-zero. |
| Kubeflow Pipelines (KFP) | Exécute le pipeline de ré-entraînement `ct-pipeline`. |
| MLflow | Registry du modèle (alias `@champion`/`@challenger`) et rangement des lots et rapports de dérive (`scoring-drift`). |
| Evidently 0.7 | Détecte la dérive entre les données de référence (train) et le lot récent. |
| Webhook CT (`trigger_ct.py`) | Reçoit l'événement de dérive, décide, lance le pipeline KFP et suit le run. |
| Pipelines GitOps (`gitops.py`) | Transforme une décision de déploiement en commit (staging, canary, promote, rollback). |
| kind | Cluster Kubernetes local du lab (K8s in Docker). |
| kubectl | Pilote le cluster : Applications, InferenceServices, secrets, port-forwards. |
| Docker | Construit l'image (test de fumée en CI ; registry local `localhost:5001`). |
| Registry local (`localhost:5001`) | Registry des images du cluster (prod initiale du Lab 5). |
| `uv` | Gère l'environnement Python (`uv.lock`) et exécute les modules du projet. |
| ruff | Lint et vérification de format. |
| pytest | Tests (contrat, API, manifests, pipeline, webhook). |
| `gh` (CLI) | Création de PR et suivi des runs (facultatif). |
| Argo Rollouts | **Non utilisé** : il pilote un Deployment, pas un `InferenceService` (le canary KServe le remplace). |

## Cibles Makefile et scripts

| Cible / script | Rôle |
|---|---|
| `make pf` | Lance les port-forwards (MLflow, etc.) nécessaires au poste. |
| `make kfp-ui` | Port-forward de l'UI/API KFP (port 8080) ; expose `apis/v2beta1`. |
| `make argocd` | Installe Argo CD sur kind via `infra/argocd/install.sh` (v3.5.3). |
| `make pipeline-image` | Construit/pousse l'image du pipeline (`scoring-pipeline`, Lab 2). |
| `make check-m6` | Vérification automatique du lab (fichiers, CI, overlays, Argo CD, CT, Git) sans rien modifier. |
| `infra/argocd/install.sh` | Applique le manifest Argo CD non-HA (`--server-side`), règle la réconciliation à 60 s, met Dex/notifications/ApplicationSets à 0 réplique. |
| `pipelines/gitops.py` | Outil GitOps : sous-commandes `status`, `payload`, `deploy`, `canary`, `promote`, `rollback`, `release` (+ `local-ci`). |
| `pipelines/trigger_ct.py` | Webhook CT : sous-commandes `simulate`, `detect`, `send`, `serve`. |
| `pipelines/ct_pipeline.py` | Pipeline KFP de ré-entraînement ; se compile avec `python -m pipelines.ct_pipeline`. |
| `packaging/app/export_onnx.py` | Exporte le champion MLflow en ONNX + métadonnées dans `packaging/model/`. |
| `labs/m6-cicd/check.sh` | Script exécuté par `make check-m6`. |
| `labs/m6-cicd/starter/git-server.yaml` | Git daemon en cluster (alternative locale sans GitHub Actions). |
| `labs/m6-cicd/starter/rapport-m6.md` | Gabarit du rapport final (copié dans `reports/`). |

## Commandes clés, expliquées

### Étape 1 — Structurer le repo (tests, manifests)

- `git switch main && git pull` puis `git switch -c lab/m6-cicd` : partir de main à jour et créer la branche du lab.
- `uv sync` : installe l'environnement exact de `uv.lock`.
- `set -a && source .env && set +a` : exporte les variables d'environnement (dont le jeton du webhook).
- `make pf` : ouvre les port-forwards vers MLflow.
- `git ls-files packaging/model` : vérifie que le modèle du Lab 4 (`model.onnx`, `metadata.json`) est versionné.
- `mkdir -p .github/workflows k8s/base k8s/overlays/staging k8s/overlays/prod` : crée l'arborescence CI + manifests.
- `cp labs/m6-cicd/starter/gitops.py pipelines/gitops.py` : installe l'outil GitOps fourni.
- `uv run python -m pipelines.gitops --help` : liste la surface de l'outil (`status`, `payload`, `deploy`, `canary`, `promote`, `rollback`, `release`).
- `uv run ruff check . && uv run ruff format --check .` : lint et contrôle de format.
- `uv run pytest -q` : exécute la suite (24 tests attendus : contrat, API, manifests).

### Étape 2 — Pipeline CI (lint + tests + build + push)

- `git add .github k8s tests pipelines/gitops.py` puis `git commit -m "..."` et `git push -u origin lab/m6-cicd` : pousse la branche.
- `gh pr create` : ouvre la pull request (déclenche la CI en test, sans publication).
- `gh pr checks --watch` : suit les checks de la PR.
- `gh run watch` : suit le run de main (publication + déploiement de staging après fusion).
- `git log --oneline -3` : montre le commit GitOps poussé par le job `deploy-staging`.
- `kubectl create namespace "$ns" --dry-run=client -o yaml | kubectl apply -f -` : crée les namespaces de façon idempotente.
- `kubectl create secret docker-registry ghcr-pull ...` : secret de tirage GHCR (uniquement si le package reste privé).
- `curl ... https://ghcr.io/v2/${REPO}/tags/list` : vérifie l'accès anonyme au package public.

### Étape 3 — Installer Argo CD

- `make argocd` : installe Argo CD et règle la réconciliation.
- `kubectl port-forward -n argocd svc/argocd-server 8443:443 &` : expose l'interface web sur `https://localhost:8443`.
- `kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath=... | base64 -d` : récupère le mot de passe admin initial.
- `curl -sSL .../argocd-linux-amd64` + `sudo install ...` : installe la CLI `argocd` (même version que le serveur).
- `argocd login localhost:8443 --username admin --insecure` : connexion de la CLI.

### Étape 4 — Les Applications Argo CD

- `kubectl kustomize k8s/overlays/prod` : rendu de l'overlay prod (l'`InferenceService` unique, image + canary).
- `sed -i "s#VOTRE_USER#.../#" k8s/argocd-app.yaml` : remplace le compte GitHub dans les `repoURL`.
- `kubectl apply -f k8s/argocd-app.yaml` : crée les deux Applications (`scoring-staging`, `scoring-prod`).
- `kubectl get applications -n argocd -w` : suit l'état `Synced`/`Healthy`.
- `kubectl get isvc -A` : liste les `InferenceService` (prod + staging).
- `kubectl get isvc scoring-model -n scoring -o jsonpath='{.metadata.annotations...,tracking-id}'` : preuve qu'Argo CD suit l'ISVC de prod.
- `kubectl get revisions -n scoring` : liste les révisions KServe.
- `kubectl patch isvc ... canaryTrafficPercent:50` puis `kubectl get isvc ... -w` : démontre le `selfHeal` (retour à 100 % en < 70 s).

### Étape 5 — Déploiement par git push

- `git commit -am "..."` + `git push` + `gh pr create` : modifie l'app et ouvre la PR.
- `git switch main && git pull` : récupère la fusion et les commits des outils.
- `kubectl get applications -n argocd` : état des Applications.
- `uv run python -m pipelines.gitops status` : compare ce que Git dit de staging/prod et ce qui est réellement servi.

### Étape 6 — Dérive Evidently + webhook

- `echo "CT_WEBHOOK_TOKEN=$(openssl rand -hex 16)" >> .env` : génère le jeton du webhook (jamais commité).
- `make kfp-ui` et `curl -s localhost:8080/apis/v2beta1/healthz` : ouvrent et testent l'API KFP.
- `CT_AUTO_RELEASE=0 uv run python -m pipelines.trigger_ct serve` : lance le webhook sans mise en prod automatique.
- `curl -s localhost:8095/health` : état du webhook.
- `curl -s -X POST localhost:8095/drift -H ... -d '{"drift_share": 1}'` : envoie un événement (401 sans jeton → contrôle d'accès).
- `uv run python -m pipelines.trigger_ct simulate` : fabrique un lot récent étiqueté avec dérive (`data/drift/current.csv`).
- `uv run python -m pipelines.trigger_ct detect --no-send` : produit le rapport Evidently et l'événement, sans POST.

### Étape 7 — Pipeline CT (webhook → KFP → nouveau modèle)

- `uv run python -m pipelines.ct_pipeline && grep -c "^    exec-" ct_pipeline.yaml` : compile le pipeline (7 exécuteurs attendus).
- `uv run pytest -q` : suite complète (29 tests attendus).
- `uv run python -m pipelines.trigger_ct send --event reports/drift/drift_event.json` : envoie l'événement au webhook, qui soumet le run KFP.
- `uv run python -c "from mlflow import MlflowClient; ...get_model_version_by_alias('scoring-model','champion')..."` : inspecte l'alias champion et ses tags (`promotion`, `previous_champion`, `ct_trigger`).
- `git add pipelines tests` + `git commit` + `git push` + `gh pr create` : troisième PR du lab.

### Étape 8 — Canary KServe piloté par Git

- `uv run python packaging/app/export_onnx.py` : exporte le champion dans `packaging/model/`.
- `git add packaging/model && git commit -m "..." && git push` + `gh run watch` : publie le modèle comme le ferait un humain.
- `uv run python -m pipelines.gitops deploy --env prod --canary 10 --push` : un commit qui change l'image prod ET la part du canary.
- `git show --stat HEAD` : vérifie les fichiers modifiés (`kustomization.yaml`, `canary.yaml`).
- `kubectl get isvc scoring-model -n scoring` : répartition PREV/LATEST.
- `for i in $(seq 1 200); do curl -s -H "Host: ${ISVC_HOST}" .../version ...; done | sort | uniq -c` : mesure la part de trafic réelle.
- `uv run python -m pipelines.gitops promote --push` : promeut à 100 %.
- `uv run python -m pipelines.gitops canary 0 --push` : retour arrière instantané.
- `uv run python -m pipelines.gitops rollback --push` : canary 0 + restauration du `@champion` précédent.

### Étape 9 — Démo de bout en bout + rapport

- `uv run python -m pipelines.trigger_ct serve` : relance le webhook en mode automatique (mise en prod par défaut).
- `uv run python -m pipelines.trigger_ct simulate --factor 2` puis `detect` : dérive plus forte, cycle complet sans intervention.
- `git log --format='%ci %an | %s' -6` : horodatages des commits (temps de chaque étape).
- `curl -s localhost:8095/runs` : historique des runs suivis par le webhook.
- `cp labs/m6-cicd/starter/rapport-m6.md reports/rapport-m6.md` : installe le gabarit de rapport.
- `git switch main && git pull && git switch -c lab/m6-rapport` + `git add reports/...` + `gh pr create` : dernière PR.
- `make check-m6` : vérification automatique finale (code retour = nombre d'échecs).

## Pièges / points d'attention

- **`paths` de la CI** : un commit qui ne touche que `k8s/` ne relance rien — c'est volontaire (évite une boucle infinie).
- **GHCR en minuscules** : le nom de l'image doit être en minuscules (`${VAR,,}`), sinon le push échoue ; `packages: write` suffit via `GITHUB_TOKEN`.
- **Package GHCR privé à la création** : les nœuds kind ne peuvent pas le tirer (`ImagePullBackOff`) ; le rendre public ou créer un secret `ghcr-pull` (+ `imagePullSecrets`).
- **Tag immuable** : jamais `latest` ; un tag = une image, on ne réécrit pas une image publiée.
- **Modèle versionné dans Git** : `packaging/model/` voyage dans le repo ; au-delà de quelques dizaines de Mo, passer à Git LFS ou à un `storageUri`.
- **Dérive seulement sur le numérique continu** : `simulate` ne dérive que les variables à plus de 20 valeurs distinctes ; sinon fournir un vrai lot via `detect --current`.
- **Gate = étiquettes requises** : une gate n'a de sens que si le lot récent est étiqueté.
- **`kubectl patch` sans effet durable** : Argo CD (`selfHeal`) réapplique l'état de Git ; tout changement passe par un commit (`gitops.py`).
- **`gitops --push` depuis une branche `lab/*`** : refusé, l'outil ne pousse que depuis `main` à jour.
- **Webhook 401/503** : `CT_WEBHOOK_TOKEN` absent ou `.env` non chargé (`set -a && source .env && set +a` dans les deux terminaux).
- **Webhook 502** : port-forward KFP arrêté → relancer `make kfp-ui`.
- **`skipped` (cooldown)** : délai minimal entre deux ré-entraînements (`CT_COOLDOWN_SECONDS`), un seul run à la fois.
- **`uv sync` en CI** : `UV_LOCKED=1` fait échouer si `uv.lock` ne correspond plus à `pyproject.toml` (`uv lock` en local, commiter `uv.lock`).
- **`deploy-staging` en échec au push** : commit concurrent ou `main` protégée → relancer (rebase) ou autoriser le bot.
- **Argo CD `Unknown / repository not found`** : `repoURL` encore à `VOTRE_USER`, ou fork privé.
- **Gate REFUSÉE** : comportement normal — le challenger fait moins bien que le champion ; le champion reste en place.
- **Staging « servi : None »** : scale-to-zero (premier appel lent) ou révision en échec.
- **Sécurité du CT automatique** : une mise en prod directe est risquée ; garde-fous = gate (seuil + coût), passage par Git/CI puis par un canary à 10 %, et en réel une validation humaine (PR plutôt que push).

*Fiche générée à partir de Lab6-Enonce.pdf et Lab6-Commandes.pdf — cours MLOps.*
