# Lab 0 — Setup de l'environnement · outils & commandes

Ce lab installe une fois pour toutes la stack commune à tous les modules : un cluster Kubernetes local (kind), un registry d'images local, un stockage objet S3 (SeaweedFS), un serveur MLflow partagé, un environnement Python reproductible (`uv`) et la chaîne d'outils MLOps (DVC, AWS CLI, Git). L'objectif est d'éviter la friction pendant les 36 h de cours. Le livrable est une capture de `make check-m0` à 100 % et la première pull request (branche `lab/m0-setup`) fusionnée.

## Outils de la stack

| Outil / service | Rôle / utilité |
| --- | --- |
| Docker | Moteur de conteneurs. Fait tourner les nœuds kind, le registry local, les outils en conteneur (client S3) et les services du cours. Version 27 minimum. |
| Docker Compose | Plugin `docker compose` : orchestration multi-conteneurs locale. L'ancien `docker-compose` v1 est abandonné. |
| kubectl | Client officiel pour piloter n'importe quel cluster Kubernetes (à ±1 version mineure du cluster, ici v1.37). |
| helm | Gestionnaire de paquets Kubernetes (charts : Prometheus, Argo CD aux labs suivants). |
| k9s | TUI pour explorer le cluster sans taper cinquante `kubectl get`. Raccourcis : `:pods`, `:svc`, `:ns`, `l` (logs), `s` (shell), `d` (describe), `Ctrl-D` (delete), `q` (quit). |
| kind (Kubernetes IN Docker) | Fait tourner un cluster K8s complet dans des conteneurs Docker : démarrage en ~1 min, suppression en une commande, isolation totale. |
| registry local (`localhost:5001`) | Registry d'images Docker local branché au cluster ; sert dès le Lab 2 (images du pipeline, puis de l'API). Port 5001 car 5000 est pris sur macOS (AirPlay) et réservé au port-forward MLflow. |
| SeaweedFS | Stockage objet S3-compatible (remplace MinIO, archivé en 2026). Backend DVC + artefacts MLflow + stockage générique. Même API S3 qu'AWS, donc portable. |
| MLflow | Serveur d'expériences partagé, dans le cluster. `backend store` = métadonnées (SQLite/PVC) ; `artifact store` = `s3://mlflow-artifacts`. Mode proxy : seul le serveur détient les identifiants S3. |
| uv | Gestionnaire Python rapide (Rust) : remplace pip + virtualenv + pip-tools + pyenv. Lockfile reproductible, gestion des versions Python intégrée. |
| Python 3.12 | Interpréteur installé et piloté par `uv` ; version du venv du repo. |
| DVC | Versioning de données/modèles avec un remote S3. Adressage par contenu (MD5) : deux fichiers identiques ne sont stockés qu'une fois. |
| AWS CLI (`aws-lab`) | Client S3 en conteneur, rien à installer (fonction définie dans `~/.bashrc`). |
| boto3 | Client S3 Python, livré par le venv ; sert de secours si `aws-lab` échoue et est utilisé par `check.sh`. |
| Git | Versioning du code. Convention du cours : branche `lab/mX-<slug>` + pull request relue ; jamais de commit direct sur `main`. |
| GitHub | Héberge la copie (fork) du repo `mlops-cours-2026`, les branches et les pull requests ; authentification par clé SSH. |
| Images `kindest/node:v1.37.0` | Image des nœuds kind (control-plane + workers), ~1 Go. |
| Image `registry:3` | Registry Docker de référence, lancé comme conteneur `kind-registry`. |
| Image `ghcr.io/chrislusf/seaweedfs:4.48` | Image du service de stockage S3. |
| Image `ghcr.io/mlflow/mlflow:v3.16.1` | Image du serveur MLflow (boto3 installé au démarrage, ~1 Go). |
| Image `amazon/aws-cli:2.37.9` | Client AWS CLI utilisé en conteneur (Job `create-buckets`, fonction `aws-lab`). |
| Bibliothèques du socle | `mlflow`, `dvc[s3]`, `boto3`, `pandas`, `scikit-learn`, `feast` (Lab 1), `kfp` (Lab 2), `great-expectations`, `evidently` (Lab 3), `onnxruntime`, `fastapi`, `bentoml` (Lab 4), `locust`, `prometheus-client` (Lab 7), `fairlearn`, `shap`, `modelscan` (Lab 8). |
| Outils dev | Groupe `dev` installé par défaut : `ruff` (lint/format), `pytest`, `yamllint`. |

## Cibles Makefile et scripts

| Cible / script | Rôle |
| --- | --- |
| `make help` | Liste les cibles disponibles. |
| `make cluster` | Crée le cluster kind (1 control-plane + 2 workers) et le registry local sur `localhost:5001`. Idempotent : le relancer ne casse rien. |
| `make storage` | Déploie SeaweedFS (namespace `storage`) + le Job créant les buckets `mlflow-artifacts` et `dvc-store`. |
| `make mlflow` | Déploie le serveur MLflow (namespace `mlflow`) + PVC `mlflow-data`. |
| `make pf` | Lance les port-forwards en arrière-plan : S3→8333, Filer→8888, MLflow→5000. |
| `make pf-stop` | Arrête les port-forwards. |
| `make prefetch` | Pré-télécharge toutes les images du cours (nœuds kind, MLflow, SeaweedFS, client AWS, registry). |
| `make check-m0` | Vérifie l'environnement (19 contrôles) et valide le Lab 0. |
| `make down` | Détruit le cluster et toutes les données. |
| `make kfp` / `make kserve` | (Labs suivants) déploient Kubeflow Pipelines / KServe. |
| `infra/kind/create-cluster.sh` | Script idempotent appelé par `make cluster` : registry local + cluster + alias + réseau + ConfigMap. |
| `infra/versions.env` | Source unique des versions (kind, images, MLflow, KFP…) lue par les scripts. |
| `labs/m0-setup/check.sh` | Script de vérification exécuté par `make check-m0` (ne modifie rien). |
| `labs/m0-setup/smoke_mlflow.py` | Script du smoke test MLflow (logge paramètre, métriques, artefact). |

## Commandes clés, expliquées

### Vérifier Docker (étape 1)

- `docker --version` / `docker compose version` — versions installées (Docker 27+, Compose v2/v5).
- `docker run --rm hello-world` — smoke test du moteur et du réseau Docker ; `--rm` supprime le conteneur après exécution.
- `docker info | grep -E 'CPUs|Total Memory'` — ressources allouées au moteur (viser 4 CPU / 6 Go mini).
- `sudo usermod -aG docker $USER` puis `newgrp docker` — ajoute l'utilisateur au groupe docker pour éviter `sudo`.

### Installer les outils Kubernetes (étape 2)

- `curl -LO .../kubectl` + `chmod +x` + `sudo mv kubectl /usr/local/bin/` — installe le client kubectl (Linux x86_64) ; `brew install kubectl` sur macOS.
- `kubectl version --client` — vérifie la version du client (v1.36 minimum, à ±1 mineure du cluster).
- `curl -fsSL .../get-helm-4 | bash` / `brew install helm` — installe Helm 4 ; `helm version` contrôle.
- `wget .../k9s_linux_amd64.deb` + `sudo apt install ./k9s_linux_amd64.deb` / `curl -sS https://webinstall.dev/k9s | bash` / `brew install derailed/k9s/k9s` — installe k9s ; `k9s version`.
- `curl -Lo ./kind .../kind-linux-amd64` + `chmod +x` + `sudo mv ./kind /usr/local/bin/kind` / `brew install kind` — installe kind ; `kind version`.

### Fork, clone et configurer Git (étape 3)

- `ssh-keygen -t ed25519 -C "votre.email@etu.fr"` — génère une clé SSH si nécessaire (à copier dans GitHub).
- `ssh -T git@github.com` — teste l'authentification SSH GitHub.
- `git clone git@github.com:VOTRE_USER/mlops-cours-2026.git` — clone votre copie (fork) en SSH.
- `git remote add upstream https://github.com/mlops-itbs-2026/mlops-cours-2026.git` — déclare le repo du cours pour récupérer les correctifs ; `git remote -v` vérifie.
- `git fetch upstream && git merge upstream/main && git push` — récupère une mise à jour du cours.
- `git config --global user.name "Prenom Nom"` / `git config --global user.email "votre.email@etu.fr"` — identité Git ; `--global` car le smoke test DVC crée un second dépôt jetable ; `git config --list | grep user` vérifie.
- `ls` / `make help` — tour du propriétaire : découvre `infra/`, `labs/`, `Makefile`, `dataset.yaml`, `docs/`, `pyproject.toml`, `uv.lock`, `.env.example`.
- `cp .env.example .env` / `cat .env` — crée le fichier d'environnement local (jamais commité).
- `set -a && source .env && set +a` — charge et exporte les variables de `.env` dans chaque nouveau terminal.

### Créer le cluster kind + registry (étape 4)

- `make cluster` — crée le registry et le cluster (1 control-plane + 2 workers) ; premier lancement : télécharge l'image des nœuds (~1 Go).
- `kubectl cluster-info --context kind-mlops-lab` — adresse de l'API server.
- `kubectl get nodes -o wide` — contrôle les trois nœuds en état `Ready` (v1.37.0).
- `docker ps --filter name=kind-registry` — vérifie que le conteneur registry tourne.
- `curl -s http://localhost:5001/v2/_catalog` — le registry répond `{"repositories":[]}`.
- `kubectl config current-context` / `kubectl config use-context kind-mlops-lab` — vérifie / bascule le contexte courant.

### Déployer SeaweedFS puis MLflow (étape 5)

- `make storage` + `kubectl -n storage get pods,pvc,svc` — déploie SeaweedFS ; attend un pod `Running`, PVC `Bound`, service `s3`.
- `kubectl -n storage logs job/create-buckets` — affiche `make_bucket: mlflow-artifacts`, `dvc-store` (le Job s'efface 10 min après sa fin, `ttlSecondsAfterFinished`).
- `make mlflow` + `kubectl -n mlflow get pods,pvc` — déploie MLflow ; attend un pod `Running` et PVC `Bound` (2 à 5 min au premier lancement).
- `make pf` — lance les port-forwards en arrière-plan (journaux dans `/tmp/mlops-port-forward/`).
- `kubectl --context kind-mlops-lab -n storage port-forward svc/s3 8333:8333 8888:8888` — port-forward manuel vers S3 et le filer.
- `kubectl --context kind-mlops-lab -n mlflow port-forward svc/mlflow 5000:5000` — port-forward manuel vers MLflow.
- `curl http://localhost:5000/health` — MLflow répond `OK`.
- `cat >> ~/.bashrc <<'EOF' ... aws-lab() { docker run --rm --network host ... } EOF` + `source ~/.bashrc` — définit le client S3 en conteneur.
- `aws-lab s3 mb s3://mlops-test` — crée un bucket.
- `aws-lab s3 ls` — liste les buckets (`dvc-store`, `mlflow-artifacts`, `mlops-test`).

### Python, uv et pré-téléchargement (étape 6)

- `curl -LsSf https://astral.sh/uv/install.sh | sh` + `source ~/.bashrc` — installe uv ; `uv --version`.
- `uv python install 3.12` + `uv python list --only-installed` — installe et liste Python 3.12.
- `uv sync` — crée le venv et installe exactement les versions de `uv.lock`, désinstalle le reste. Ne pas faire `pip install`/`uv add` à la main.
- `source .venv/bin/activate` — active le venv ; ou préfixer par `uv run <cmd>` sans l'activer.
- `direnv` (`brew install direnv` / `sudo apt install direnv`, `eval "$(direnv hook bash)"`, `.envrc`, `direnv allow`) — activation automatique du venv et chargement de `.env`.
- `python --version` / `python -c "import sklearn, mlflow, dvc; print(...)"` / `dvc --version` — vérifications.
- `make prefetch` — pré-télécharge toutes les images du cours (10 à 20 min, sans surveillance).

### Smoke tests MLflow et DVC (étape 7)

- `set -a && source .env && set +a` — charge `MLFLOW_TRACKING_URI` ; `echo $MLFLOW_TRACKING_URI` contrôle.
- `python labs/m0-setup/smoke_mlflow.py` — logge un run (paramètre, métriques, artefact `hello.txt`) via le mode proxy du serveur.
- `rm -rf /tmp/dvc-smoke && mkdir -p /tmp/dvc-smoke && cd /tmp/dvc-smoke` — crée un dossier jetable hors du repo.
- `git init && dvc init && git commit -m "dvc init"` — initialise un dépôt Git + DVC de test.
- `dvc remote add -d storage s3://mlops-test/dvcstore` — configure SeaweedFS comme remote DVC par défaut.
- `dvc remote modify storage endpointurl http://localhost:8333` — fixe l'endpoint (HTTP) dans `.dvc/config` (fichier commité).
- `dvc remote modify --local storage access_key_id mlops` / `... secret_access_key mlops-lab-secret` — identifiants stockés dans `.dvc/config.local` (ignoré par Git).
- `mkdir -p data && echo "hello mlops" > data/sample.txt` + `dvc add data/sample.txt` — crée et tracke un fichier avec DVC.
- `git add data/.gitignore data/sample.txt.dvc .dvc/config && git commit -m "Track sample.txt with DVC"` — commite les métadonnées DVC.
- `dvc push` — pousse l'objet vers S3 (`1 file pushed`).
- `aws-lab s3 ls --recursive s3://mlops-test/dvcstore/` — voit l'objet rangé sous `files/md5/...`.

### Première pull request (étape 8)

- `git switch -c lab/m0-setup` — crée la branche du lab depuis `main` à jour.
- `git add NOTES.md labs/m0-setup/smoke_mlflow.py` / `git status` — prépare le commit ; `.env` ne doit jamais apparaître.
- `git commit -m "Lab 0: setup OK"` puis `git push -u origin lab/m0-setup` — pousse la branche et ouvre la PR.
- `gh pr create` — ouvre la PR en ligne de commande (facultatif, interactif).
- `git switch main && git pull` — revient sur `main` fusionnée ; `git log --oneline -3` confirme le merge.
- `rm NOTES.md labs/m0-setup/smoke_mlflow.py && git pull` — côté second membre du binôme (option A), récupère `main`.
- `make check-m0` — vérification automatique finale : attend `★ 19/19 vérifications OK — Lab 0 validé` (code retour = nombre d'échecs).

## Pièges / points d'attention

- **Docker Desktop arrêté** : `connection refused` sur l'API server, `docker` introuvable dans WSL. Ce n'est pas un problème de cluster : démarrer Docker Desktop et attendre que les nœuds repassent `Ready`.
- **SeaweedFS en `CrashLoopBackOff`** : le Service s'appelle `s3`, Kubernetes injecte `S3_PORT=tcp://<clusterIP>:8333` et SeaweedFS ne sait pas le convertir en entier. Corrigé dans `infra/storage/storage.yaml` en figeant `S3_PORT: "8333"` (une variable du conteneur a priorité sur celle du Service).
- **`make pf` → `make: *** [pf] Terminated`** : avec `.ONESHELL`, un `pkill -f "port-forward svc/s3 8333"` matche le shell lui-même et le tue. Corrigé dans le Makefile en suivant les PIDs dans des fichiers.
- **`aws-lab` → `exec format error`** : anomalie du store d'images de Docker Desktop (mauvaise architecture). Contournement : créer/lister les buckets via `boto3` depuis la venv (`uv run --no-sync python - <<'PY' ... PY`).
- **`kubectl -n storage logs job/create-buckets` → « not found »** : normal, le Job (`ttlSecondsAfterFinished: 600`) est supprimé 10 min après sa fin. Se fier à `kubectl -n storage get pods` (pod `Completed`) et à la liste des buckets.
- **Dépôt DVC jetable : `git commit` refusé (identité)** : `/tmp/dvc-smoke` est un nouveau dépôt sans `user.name`/`user.email`. Renseigner l'identité locale (ou via la config globale de l'étape 3.4).
- **Port-forwards fragiles** : ils s'arrêtent si le pod redémarre, si le poste se met en veille ou si la session se ferme (symptôme : `Connection refused` sur `localhost:5000`/`:8333`). Remède : relancer `make pf`, sans risque, au début de chaque séance.
- **Ports déjà utilisés** : 80/443/30080 (Apache/nginx/IIS) ou 5001 (autre registry) font échouer `make cluster` ; arrêter le service ou commenter le mapping dans `infra/kind/kind-config.yaml`.
- **Ressources insuffisantes** : moins de 4 CPU ou 6 Go de RAM et le cluster rame ; augmenter dans Docker Desktop (Settings > Resources).
- **Windows/WSL2** : travailler dans le système de fichiers Linux (`/home/...`), jamais dans `/mnt/c/...`, sous peine de lenteur et de fichiers introuvables ; Docker Desktop doit utiliser le backend WSL2.
- **`--network host`** : sur Docker Desktop (macOS/Windows), l'activer ou remplacer par `http://host.docker.internal:8333` dans `aws-lab`.
- **Ne jamais commiter `.env`** : il contient des identifiants et est listé dans `.gitignore` ; le vérifier avec `git status`.
- **Ne pas recréer le cluster** : SeaweedFS et MLflow y vivent jusqu'au mini-projet ; `make down` (ou `kind delete cluster`) détruit toutes les données (datasets, expériences, modèles).
- **`uv sync` fait foi** : ne pas installer de paquets à la main (`pip install`, `uv add`) dans le repo du cours ; ils disparaîtraient au prochain `uv sync` ou modifieraient `uv.lock`.
- **ImagePullBackOff sur `minio/minio`** : images retirées de Docker Hub en 2026 (d'où le passage à SeaweedFS) ; utiliser `make storage` et le client `aws-lab`.

*Fiche générée à partir de Lab0-Enonce.pdf et Lab0-Commandes.pdf — cours MLOps.*
