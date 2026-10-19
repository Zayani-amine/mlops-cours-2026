# Lab 10 — Mini-projet MLOps · outils & commandes

Le mini-projet consiste à faire passer un modèle du notebook à la production, à deux, de façon reproductible, observable, auditable et éthique. Le repo construit lab après lab (Labs 0 à 8) forme le socle : il est déployé chez un fournisseur cloud, complété par une extension individuelle, puis défendu en soutenance avec un défi en direct. Cette fiche explique l'utilité et le rôle de chaque outil, cible et commande regroupés dans la fiche « Commandes » du cours.

## Outils de la stack

| Outil/service | Rôle / utilité |
|---|---|
| `kind` | Cluster Kubernetes local : fait tourner tout le socle, c'est là que passent les checks et le défi en direct. |
| `kubectl` | Pilotage du cluster : observer pods, événements, logs, décrire une ressource, lire l'état d'un service. |
| `git` / GitHub | Versionner le code, gérer branches et PR, porter les tags de jalon, héberger le repo et les règles de protection. |
| `dvc` | Versionner les données et le modèle, rejouer le pipeline (`repro`), pousser/tirer vers le remote DVC. |
| MLflow | Tracer les runs et le registre de modèles ; gère l'alias `champion` et les versions. |
| KFP (Airflow) | Orchester le pipeline `scoring-pipeline` (Airflow utilisé comme point de comparaison). |
| KServe | Servir le modèle via une `InferenceService` (`scoring-model`, namespace `scoring`), autoscaling, canary. |
| ONNX Runtime | Exporter/exécuter le modèle de l'API (`scoring-api`), optimisation et quantification. |
| GitHub Actions | CI : build et test de l'image, publication sur GHCR, exécution des gates. |
| Argo CD | GitOps : synchronise le cluster depuis `main`, détecte les écarts (sync/diff). |
| Prometheus / Grafana | Collecte de métriques, tableaux de bord, alertes (avec runbook). |
| Evidently | Détection de drift des features et des prédictions. |
| GX | Validation des données (colonnes, domaines, gates du Lab 3). |
| Fairlearn | Analyse d'équité par groupe, base du gate d'équité. |
| SHAP | Explicabilité des décisions. |
| Trivy / ModelScan | Scans de sécurité de l'image et de l'artefact modèle. |
| SeaweedFS (S3) | Stockage objet du cluster, sert de remote DVC (`s3://dvc-store/scoring` sur `localhost:8333`). |
| registry local | Stockage des images (`localhost:5001`) à côté de GHCR. |
| Fournisseur cloud (Azure, Google Cloud, AWS…) | Héberge l'API de scoring en ligne, en HTTPS, publiquement accessible. |
| `uv` / `uvx` | Gestion de l'environnement Python (`uv sync`, `uv run`) et exécution d'outils ponctuels sans install (`uvx`). |
| `jq` | Lecture et filtrage de réponses JSON (ex. `/metadata`). |
| `qrcode[pil]` | Génération des QR codes PNG du poster. |
| `gh` CLI | Vérification des relectures de PR `ext/*`. |

## Cibles Makefile et scripts

| Cible / script | Rôle |
|---|---|
| `make up` | Démarre le cluster et l'infrastructure du socle. |
| `make pf` | Port-forward : rend accessibles les services du cluster (MLflow, etc.). À vérifier en début de séance. |
| `make down` | Arrête/retire l'environnement ; à manier avec précaution (à ne jamais utiliser pour « réparer »). |
| `make check-m0` … `make check-m8` | Vérification automatique d'une brique du socle (cluster, DVC/MLflow, pipeline, gates, image, service, CI/CD, monitoring, gouvernance). |
| `make check-mX` | Vérification générique d'une brique du socle. |
| `make check-m10` | Pré-contrôle de rendu du mini-projet ; par défaut le jalon « final », paramétrable par `JALON=`. |
| `./labs/m10-capstone/check.sh` | Pré-contrôle complet du rendu ; options `--sans-cluster` (repo seul) et `--gh` (relectures de PR). |
| `packaging/sample_request.py` | Produit une requête d'exemple pour tester l'API. |
| `src/dataset_config.py` | Lit `dataset.yaml` : tout nom de colonne, cible, classe ou seuil vient de là, rien en dur. |
| `src/download.py` | Télécharge et prépare le jeu de données (script de reproductibilité). |
| `docs/journal.md` | Journal de bord, une entrée datée par séance. |
| `docs/adr/` | Architecture Decision Records (3 minimum au rendu). |
| `docs/runbook.md` | Procédures d'exploitation par incident. |

## Commandes clés, expliquées

### Mise en place et premier tour de piste

- `git clone git@github.com:PROPRIETAIRE/mlops-cours-2026.git` — clone le repo (le repo commun en option A, le sien en option B).
- `git remote add upstream https://github.com/mlops-itbs-2026/mlops-cours-2026.git` — ajoute le repo du cours comme remote de référence (corrections, `solution/*`).
- `cp .env.example .env` — crée la configuration locale propre à chaque membre.
- `make up && make pf` — démarre l'infrastructure puis ouvre les accès ; chacun a son propre cluster.
- `git switch -c lab/m0-journal` — crée une branche de travail pour le lab (convention `lab/mX-…`).
- `$EDITOR docs/journal.md` — rédige l'entrée du journal de bord.
- `git add docs/journal.md` / `git commit -m "lab/m0 : journal de bord"` — indexe et commite la modification.
- `git push -u origin lab/m0-journal` — pousse la branche et déclenche la proposition de PR (relue par le partenaire par convention).

### Rendre un jalon

- `git switch main && git pull` — se place sur un `main` à jour avant le jalon.
- `make check-m10 JALON=j1` — lance le pré-contrôle correspondant au jalon visé (ici J1).
- `git tag -a j1 -m "J1 MVP : pipeline avec gate, modèle servi par KServe"` — pose un tag annoté sur le commit qui fait foi.
- `git push origin j1` — pousse le tag ; un tag poussé ne se déplace pas, il faut corriger via un nouveau commit sur `main`.

### Vérifier le déploiement cloud

- `URL=https://votre-api.exemple` — définit l'URL donnée par le fournisseur.
- `curl -fsS "$URL/health"` — vérifie que l'API répond (`{"status":"ok","model_loaded":true}`).
- `curl -fsS "$URL/metadata" | jq .model` — lit la version servie, à comparer à `scoring-model@champion`.
- `uv run python packaging/sample_request.py > /tmp/request.json` — génère une requête d'exemple.
- `curl -fsS -X POST "$URL/predict" -H "Content-Type: application/json" -d @/tmp/request.json` — envoie une prédiction à l'API.
- Ouvrir `$URL/docs` depuis un téléphone, hors Wi-Fi de l'école — vérifie l'essai d'une prédiction et la page de documentation.

### Diagnostiquer (méthode en cinq temps)

- `kubectl get pods -A | grep -v Running` — repère en un coup d'œil tout ce qui ne tourne pas, dans tous les namespaces.
- `kubectl -n scoring get events --sort-by=.lastTimestamp` — reconstitue le récit chronologique de ce qui s'est passé.
- `kubectl -n scoring describe pod <pod>` — inspecte conditions, sondes et montages du pod.
- `kubectl -n scoring logs <pod> --previous` — lit les logs du conteneur qui a planté avant son redémarrage.
- `kubectl -n scoring get inferenceservice scoring-model` — vérifie l'état READY, l'URL et la révision du service.
- `make check-m5` — confronte l'état réel à l'état attendu, item par item.

### Poster (QR codes)

- `uvx --from "qrcode[pil]" qr "https://votre-projet-deploye/docs" > docs/qr-projet.png` — génère le QR code du projet déployé.
- `uvx --from "qrcode[pil]" qr "https://github.com/votre-user/mlops-cours-2026" > docs/qr-repo.png` — génère le QR code du repo.

### Rattraper un lab (FAQ)

- `git fetch upstream` — récupère les nouveautés du repo du cours (corrections).
- `git switch -c lab/m4-rattrapage` — crée la branche de reprise.
- `git checkout upstream/solution/m4 -- packaging/` — ne récupère que les fichiers du lab concerné.
- `git commit -m "lab/m4 : reprise de la correction m4"` — trace la reprise dans l'historique.

### Vérification automatique

- `make check-m10` — pré-contrôle du jalon « final » (tout).
- `make check-m10 JALON=j1` — ce qui est attendu au jalon J1.
- `./labs/m10-capstone/check.sh --sans-cluster` — contrôle repo seul (docs, Git, tags).
- `./labs/m10-capstone/check.sh --gh` — contrôle étendu à la relecture des PR `ext/*` via `gh`.

## Pièges / points d'attention

- Ne jouer au jeu du chaos que sur des éléments versionnés ou recréables (git, Argo CD, make) ; jamais `make down`, ni suppression de PVC ou de bucket.
- `dvc pull` ne suffit pas : le remote DVC est propre à chaque poste ; il faut `dvc repro` (puis `dvc push`) pour reconstruire.
- Un tag poussé ne se déplace pas : `git tag -f` et `git push -f` sont refusés ; corriger par un commit sur `main`.
- Ne pas activer « Require a pull request before merging » : le bot GitOps du Lab 6 commite directement sur `main`.
- Décocher « Allow rebase merging » : un rebase effacerait le nom de branche `ext/<nom>` de l'historique.
- Rien en dur : tout (colonnes, cible, groupes sensibles, seuils) doit venir de `dataset.yaml` ou d'un fichier de configuration versionné.
- Aucun secret dans Git (`.env`, `.dvc/config.local`, jetons) ; un secret poussé doit être changé, pas seulement supprimé.
- Aucune donnée personnelle : données publiques uniquement (credit scoring ou dataset validé).
- Le service cloud peut descendre à zéro instance : appeler `/health` quelques minutes avant la soutenance ou fixer une instance minimale.
- Créer une alerte de budget dès l'ouverture du compte cloud et supprimer les ressources après évaluation.
- Répéter la démo au moins trois fois, dont une sur un poste vierge (`git clone` + `make up` + `uv sync` + `dvc repro` + `dvc push`).
- Ne pas fusionner l'extension la veille : sans relecture ni tests, elle casse souvent le socle et les checks.
- Le pré-contrôle vérifie que les livrables existent, il ne note pas la qualité ; un pré-contrôle rouge au tag final coûte des points.

*Fiche générée à partir de Lab10-Enonce.pdf et Lab10-Commandes.pdf — cours MLOps.*
