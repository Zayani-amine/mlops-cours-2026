# Rapport comparatif M2 : Kubeflow Pipelines vs Airflow

## Contexte
Nous avons migré le pipeline DVC du Lab 1 (dataset german-credit, 1000 lignes, cible good/bad, 30 % de classe bad) vers deux orchestrateurs : un pipeline Kubeflow Pipelines à 6 steps (ingest, validate, prepare, train, evaluate, register) et un DAG Airflow équivalent. Les deux exécutent le même code, grâce à l'image projet scoring-pipeline:1.0.0 avec un pod par step, et alimentent la même expérience MLflow, avec les tags launched_by=kfp et launched_by=airflow. Le gate de validation arrête le pipeline avant l'entraînement si les données ne respectent pas dataset.yaml.

## Résultats quantitatifs
| Dimension | KFP | Airflow |
|---|---|---|
| Installation | 10 min (make kfp) | 5 min (Docker Compose) |
| Run complet | 3 min 46 s | 1 min |
| Run relancé (cache) | 2 min 52 s (steps data en cache) | non applicable (pas de cache) |
| Lignes de code | 263 (components.py + pipeline) | 67 (scoring_dag.py) |
| Mémoire | environ 4,7 Gio pour les 3 nœuds kind, qui hébergent aussi MLflow et S3 | environ 1,9 Gio pour 7 conteneurs |

Le gain du cache KFP est modeste (environ 24 %), car le train, l'évaluation et l'enregistrement sont toujours ré-exécutés : le cache est désactivé sur le train pour produire un nouveau run MLflow à chaque fois.

## Analyse qualitative
**Developer experience.** KFP impose des composants typés et une compilation en YAML : c'est plus verbeux (263 lignes contre 67), mais les entrées et sorties sont explicites, et l'interface montre les artefacts, les métriques et la matrice de confusion sans effort. Airflow est plus rapide à écrire, puisqu'un DAG est un fichier Python rechargé automatiquement, mais il ne transporte aucun fichier : il a fallu passer par un dossier de travail S3 et par XCom pour les petites valeurs. Dans les deux cas, modifier le code de src/ impose de reconstruire l'image sous un nouveau tag.

**Scaling.** Dans KFP, chaque step est un pod et le parallélisme est natif. Le cache évite de recalculer les étapes de données pendant le tuning d'hyper-paramètres. Airflow parallélise aussi les tasks, mais demande davantage de composants (scheduler, worker, base de données, Redis) et n'offre pas de cache d'artefacts. Le run Airflow est plus court que celui de KFP, car il n'a ni pods driver ni transfert d'artefacts par SeaweedFS à chaque étape : KFP paie ce surcoût pour obtenir des artefacts typés et du lineage.

**Debugging.** Dans KFP, le lineage est visible en un clic, de l'artefact jusqu'au step qui l'a produit. Dans Airflow, on lit les logs de la task et on parcourt le préfixe S3 du run. Un de nos runs Airflow est resté longtemps en file d'attente parce que l'API server ne démarrait pas (port 8081 bloqué par Windows) : avec Airflow en Docker Compose hors du cluster, la disponibilité des conteneurs et des ports conditionne l'exécution, alors que sous KFP tout vit dans le cluster. Le gate de validation échoue proprement dans les deux outils grâce au code retour non nul de src.validate_data.

## Recommandation
Nous choisirions KFP pour des pipelines centrés sur le ML, avec des artefacts volumineux, du tuning fréquent et un besoin de lineage, à condition de disposer d'un cluster Kubernetes et d'une équipe prête à maintenir des images versionnées. Nous choisirions Airflow pour des workflows de données hétérogènes et planifiés (ETL, déclencheurs externes), dans une équipe qui l'exploite déjà, en veillant à externaliser les fichiers sur S3 et à ne passer que de petites valeurs par XCom.

## Limites de la comparaison
Ce lab utilise un petit dataset de 1000 lignes sur un cluster kind local, sans montée en charge réelle. Nous n'avons pas testé KFP sur un cluster managé, ni Airflow avec le KubernetesExecutor ou déployé sur Kubernetes. Les mesures de mémoire ne sont pas strictement comparables, car les nœuds kind hébergent aussi MLflow et S3. Les temps dépendent de la machine et du téléchargement initial des images, et ils ne disent rien du coût d'exploitation à long terme.
