# Lab 9 — RAG MLOps évalué et observé · outils & commandes

Ce lab guide construit un système RAG sur la documentation du cours MLOps : on sert un LLM local (Ollama sur CPU ou vLLM sur GPU), on indexe les documents dans une base vectorielle Qdrant, on assemble une chaîne RAG LangChain avec citations de sources, puis on l'évalue automatiquement (Ragas), on l'observe (tracing Phoenix, métriques Prometheus) et on teste sa résistance au prompt injection. Contrairement aux labs du socle, tout tourne sur le poste, dans le dossier `rag/`, sans cluster kind.

## Outils de la stack

| Outil/service | Rôle / utilité |
| --- | --- |
| Ollama | Sert un LLM local, sur CPU ou GPU grand public (llama.cpp, modèles quantifiés 4 bits). Option A, recommandée pour le lab : un utilisateur, `qwen3.5:4b` (~20-40 s par requête). |
| vLLM | Sert un LLM sur GPU pour beaucoup d'utilisateurs en parallèle (PagedAttention, continuous batching, tensor parallelism) ; standard de la prod. Option B, GPU NVIDIA 16 Go minimum (~1-3 s par requête). |
| Ollama `/ vLLM` (API) | Exposent tous deux une API compatible OpenAI (`/v1/chat/completions`, `/v1/models`) : le code du lab fonctionne avec l'un ou l'autre. |
| Qdrant | Base vectorielle en Docker qui stocke les embeddings des documents du cours ; sert l'index et la recherche de proximité. |
| LangChain (`langchain-core`, LCEL) | Framework d'assemblage de la chaîne RAG : prompts, retrieval, passage à travers des étapes traçables (`RunnablePassthrough.assign`). |
| langchain-community / langchain-text-splitters | Fournit `PyPDFLoader`, `TextLoader`, `FastEmbedEmbeddings` et `RecursiveCharacterTextSplitter` (découpage en chunks). |
| langchain-qdrant / qdrant-client | Connecteur entre LangChain et Qdrant (upsert des chunks, retriever). |
| langchain-ollama / langchain-openai | Adaptateurs du LLM pour Ollama ou vLLM côté LangChain. |
| fastembed | Calcule les embeddings en ONNX sur CPU sans PyTorch (`bge-small-en-v1.5`, 384 dim., ~70 Mo). |
| pypdf | Lecture des fichiers PDF. |
| Ragas | Évaluation du RAG par LLM-juge : `faithfulness`, `answer_relevancy`, `context_precision`, `context_recall`. |
| Phoenix (Arize) | Serveur de traces OpenTelemetry spécialisé LLM : visualise chaque requête (retrieval, prompt, réponse, tokens, latence). |
| arize-phoenix-otel / openinference-instrumentation-langchain | Client OpenTelemetry et auto-instrumentation de LangChain pour envoyer les traces à Phoenix. |
| LangSmith | Alternative managée (SaaS) de tracing : mêmes chaînes tracées sans opérer de serveur, au prix de l'envoi des prompts/documents à un tiers. |
| FastAPI + uvicorn | Exposent la chaîne RAG en API (`/ask`, `/health`) ; `uvicorn` est le serveur ASGI. |
| prometheus_client | Expose des métriques métier scrapables par Prometheus (`rag_requests`, `rag_request_duration_seconds`, `rag_llm_tokens`), pour rejoindre le monitoring du Module 7. |
| Docker | Fait tourner Qdrant (et Ollama en variante conteneurisée). |

## Cibles Makefile et scripts

| Cible / script | Rôle |
| --- | --- |
| `make check-m0` … `make check-m8` | Vérifications du socle des labs précédents. Le Lab 9 ne fait pas partie de ce socle. |
| `make check-m9` | Vérifie le Lab 9 de bout en bout, sans rien modifier : LLM, Qdrant, API RAG, golden set, résultats Ragas, traces Phoenix, logs d'injection et rapport. |
| `labs/m9-llmops/check.sh` | Script appelé par `make check-m9` ; affiche ✔/✘ par item, le code retour est le nombre d'échecs (attendu : 17/17). |
| `rag/config.py` | Configuration commune : Qdrant, embeddings et LLM en un seul endroit, piloté par variables d'environnement. |
| `rag/ingest.py` | Indexe les documents (PDF/MD, ou doc du projet) dans Qdrant, en chunks. |
| `rag/rag.py` | Chaîne RAG LCEL (retrieval top-5 + prompt + LLM), avec sources tracées. |
| `rag/tracing.py` | Active le tracing Phoenix et auto-instrumente LangChain (importé quand `PHOENIX_TRACING=1`). |
| `rag/evaluate_rag.py` | Génère les réponses sur le golden set et calcule les 4 métriques Ragas ; peut servir de gate en CI. |
| `rag/phoenix_stats.py` | Calcule depuis les traces Phoenix les tokens, coût équivalent et latences P50/P95. |
| `rag/api.py` | Expose le RAG en FastAPI avec métriques Prometheus, en passant par le guardrail. |
| `rag/guardrails.py` | Guardrail d'entrée (patterns d'injection) et de sortie (fuite de secret/prompt), via `safe_query`. |
| `rag/injection_test.py` | Joue une liste d'attaques de prompt injection contre la chaîne (avant guardrail). |
| `rag/injection_test_guarded.py` | Rejoue les attaques plus une requête légitime, via `safe_query` (après guardrail). |
| `rag/golden_set.json` | Jeu de référence de Q/R (26 questions fournies, ≥ 20 exigées) pour évaluer le RAG. |
| `rag/RAPPORT.md` | Trame de rapport à compléter (9 sections). |
| `rag/reports/` | Sorties : `ragas_results.csv`, `ragas_summary.json`, `ragas_run.log`, `phoenix_stats.json`, `injection_before.log`, `injection_after.log`, `phoenix_trace.png`. |
| `rag/.gitignore` | Exclut `docs/*.pdf` : les supports du cours ne sont pas republiés dans le fork public. |

## Commandes clés, expliquées

### Préparation (branche de travail)
- `git switch main && git pull` puis `git switch -c lab/m9-llmops` : on part d'un `main` à jour et on crée la branche dédiée au lab.
- `kubectl get nodes` et `make pf` : vérifications de début de séance (le cluster n'est pas utilisé par le lab).

### Étape 1 — Installer et servir le LLM
- `curl -fsSL https://ollama.com/install.sh | OLLAMA_VERSION=0.35.1 sh` : installe Ollama en épinglant la version du cours (Linux/WSL2).
- `ollama --version` : confirme la version installée (attendu `0.35.1`).
- `ollama pull qwen3.5:4b` : télécharge le modèle (~3,4 Go).
- `ollama serve &` (ou `sudo systemctl start ollama`) : démarre le serveur sur `localhost:11434`.
- `ollama run qwen3.5:4b --think=false "…"` : teste le modèle sans phase de « réflexion » (réponse directe).
- `docker run -d --name ollama -p 11434:11434 -v ollama:/root/.ollama ollama/ollama:0.35.1` puis `docker exec ollama ollama pull qwen3.5:4b` : variante Ollama en conteneur.
- `uv venv ~/vllm-env --python 3.12` puis `source ~/vllm-env/bin/activate` et `uv pip install vllm==0.31.0` : environnement GPU séparé pour vLLM (PyTorch/CUDA).
- `vllm serve Qwen/Qwen3.5-4B --host 0.0.0.0 --port 8000 --max-model-len 16384 --gpu-memory-utilization 0.9 --language-model-only --reasoning-parser qwen3 --default-chat-template-kwargs '{"enable_thinking": false}'` : lance le serveur compatible OpenAI sans thinking.
- `curl http://localhost:8000/v1/models` : vérifie que vLLM répond.

### Étape 2 — Lancer Qdrant
- `docker run -d --name qdrant -p 6333:6333 -p 6334:6334 -v ~/qdrant-storage:/qdrant/storage qdrant/qdrant:v1.19.2` : lance Qdrant (REST `6333`, gRPC `6334`) avec stockage persistant.
- `curl http://localhost:6333/` : vérifie la version JSON renvoyée ; UI web sur `http://localhost:6333/dashboard`.

### Étape 3 — Ingérer les documents
- `mkdir -p rag/docs` puis `cp ~/Downloads/MLOps_*.pdf rag/docs/` et `ls -lh rag/docs/` : prépare et vérifie le corpus.
- `cd rag` puis `uv sync` : crée l'environnement isolé du RAG depuis `rag/pyproject.toml`.
- `uv run python -c "import ragas, phoenix; print(ragas.__version__, phoenix.__version__)"` : vérifie les dépendances (attendu `0.4.3 20.19.0`).
- `uv run python ingest.py` (ou `uv run python ingest.py --with-labs`) : indexe les documents dans Qdrant ; `--with-labs` indexe les énoncés `labs/*/README.md` sans les PDF.
- `QDRANT_COLLECTION=project_docs uv run python ingest.py --project` : indexe la doc de votre propre projet dans une collection séparée.
- `QDRANT_COLLECTION=project_docs uv run python rag.py` : interroge cette collection projet (après l'étape 4).

### Étape 4 — Construire la chaîne RAG
- `uv run python rag.py` : exécute la chaîne LCEL sur une question de test, affiche la réponse et 3 sources.

### Étape 5 — Créer un golden set
- `cp ../labs/m9-llmops/starter/golden_set.json .` : copie le golden set fourni.
- `uv run python -c "import json; print(len(json.load(open('golden_set.json'))), 'questions')"` : compte les questions.

### Étape 6 — Évaluer avec Ragas
- `mkdir -p reports` : prépare le dossier des sorties.
- `RAGAS_LIMIT=5 uv run python evaluate_rag.py 2>&1 | tee reports/ragas_run.log` : premier passage rapide sur 5 questions, journalisé.
- `RAGAS_MIN_FAITHFULNESS=0.7 uv run python evaluate_rag.py` : renvoie un code retour 1 si la `faithfulness` moyenne passe sous le seuil (gate CI).

### Étape 7 — Tracing Phoenix + API
- `uv run phoenix serve` : lance le serveur de traces (`http://localhost:6006`, OTLP sur `/v1/traces`).
- `PHOENIX_TRACING=1 uv run python rag.py` : relance la chaîne avec le tracing activé.
- `uv run python phoenix_stats.py` : calcule tokens, coût équivalent et P50/P95 depuis les traces et écrit `reports/phoenix_stats.json`.
- `PHOENIX_TRACING=1 uv run uvicorn api:app --port 8090 &` : démarre l'API RAG avec métriques.
- `curl -s -X POST localhost:8090/ask -H 'Content-Type: application/json' -d '{"question": "Que fait PagedAttention dans vLLM ?"}'` : interroge l'API.
- `curl -s localhost:8090/metrics | grep '^rag_'` : lit les métriques Prometheus exposées.
- `export LANGSMITH_TRACING=true` / `export LANGSMITH_API_KEY=lsv2_...` / `export LANGSMITH_PROJECT=rag-mlops` puis `uv run python rag.py` : variante de tracing managée LangSmith.

### Étape 8 — Prompt injection + guardrail
- `uv run python injection_test.py 2>&1 | tee reports/injection_before.log` : démontre l'attaque (4 prompts).
- `uv run python injection_test_guarded.py 2>&1 | tee reports/injection_after.log` : rejoue via le guardrail (3 bloquées, requête légitime passée).

### Étape 9 — Rapport final et versionnage
- `cp ../labs/m9-llmops/starter/RAPPORT.md .` : copie la trame du rapport.
- `cd ..`, `git add rag/`, `git status --short` (aucun PDF), `git commit -m "Lab 9 : RAG évalué (Ragas) et observé (Phoenix)"`, `git push -u origin lab/m9-llmops`, `gh pr create` : commit et PR.
- `docker start qdrant` puis `cd rag && uv sync`, `uv run python ingest.py`, `PHOENIX_TRACING=1 uv run python rag.py` : remise à niveau d'un second poste.
- `make check-m9` : vérification automatique (attendu `★ 17/17 vérifications OK`).

## Pièges / points d'attention

- Ollama `model not found` : le modèle n'est pas téléchargé ou `LLM_MODEL` diffère ; faire `ollama pull qwen3.5:4b` puis `ollama list` et vérifier `echo $LLM_MODEL`.
- Réponses Ollama très lentes : CPU faible ou modèle qui « réfléchit » ; utiliser `qwen3.5:0.8b` et vérifier que `reasoning=False` est bien passé.
- Contexte tronqué / réponses incohérentes : fenêtre par défaut de 4K tokens sur CPU ; relancer avec `OLLAMA_CONTEXT_LENGTH=8192 ollama serve`.
- Qdrant `connection refused` ou `collection not found` : conteneur non démarré (`docker start qdrant`) ou nom de collection différent entre ingestion et requête (garder le même `QDRANT_COLLECTION` partout et relancer `ingest.py`).
- Embeddings : échec de téléchargement (pas d'accès à Hugging Face, proxy) → définir `HF_ENDPOINT` vers un miroir ; embeddings lents → rester sur `bge-small-en-v1.5`, pas `bge-large`, sur CPU.
- RAG sources vides : collection vide ou `k=0` ; vérifier le nombre de points dans l'UI Qdrant et `search_kwargs={"k": 5}`.
- Réponse hors sujet ou obsolète : prompt trop permissif ou ancien document indexé ; un RAG n'est jamais plus à jour que son index.
- Imports LangChain : `ModuleNotFoundError: langchain.chains` = code écrit pour 0.x ; utiliser les imports 1.x ou `langchain_classic` ; `No module named 'langchain_community.chat_models.vertexai'` = `langchain-community` 0.4.2 installé à la main → `cd rag && uv sync` (épingle 0.4.1).
- Ragas lent (9 appels au juge par question) : utiliser `RAGAS_LIMIT=5` ou un juge plus rapide ; un échec `ValidationError` isolé (JSON invalide) donne un `NaN`, normal de temps en temps.
- Phoenix : port 6006 occupé → `PHOENIX_PORT=6007 uv run phoenix serve` et adapter `PHOENIX_COLLECTOR_ENDPOINT` ; aucune trace → `PHOENIX_TRACING=1` et Phoenix lancé avant le script.
- `make check-m9` 5a/5b en échec : l'API RAG n'est pas lancée (`cd rag && PHOENIX_TRACING=1 uv run uvicorn api:app --port 8090 &`).
- Avertissement `VIRTUAL_ENV … does not match` : le venv du socle est activé, `uv` utilise `rag/.venv` — sans conséquence (`deactivate` fait taire le message).
- Ne jamais indexer de secrets : tout ce qui est indexé peut sortir (LLM02). Le guardrail à base de regex se contourne et produit des faux positifs ; en prod, combiner guardrail d'entrée + de sortie + monitoring.

*Fiche générée à partir de Lab9-Enonce.pdf et Lab9-Commandes.pdf — cours MLOps.*
