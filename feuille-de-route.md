# Feuille de route : déployer un Datamart CMDB sur Elastic, de la dev à la prod

> La démarche complète pour mettre en production un Datamart CMDB
> (type Galaxie : CMDB sur Oracle Exadata, ServiceNow, ITAM) : les environnements, le dépôt,
> la chaîne de promotion, les phases avec leurs livrables, le détail de chaque brique
> (mappings, Kafka, Logstash, Kibana, sécurité) et la liste de contrôle du passage en
> production. En dernière partie : où chaque étape existe déjà dans le lab.
> Pour les notions : [../formation/galaxie/datamart-explique.md](../formation/galaxie/datamart-explique.md).

## 1. Les principes

1. **Git est la seule source de vérité** : mappings, pipelines, Logstash, tableaux de bord,
   droits. Ce qui n'est pas dans Git n'existe pas.
2. **Le même code passe d'un environnement à l'autre** : dev → recette → prod, dans cet
   ordre. Seules changent les variables (adresse, taille, rétention).
3. **Rien à la main en production** : la prod est en lecture pour les humains, écrite par
   la CI. Une modification manuelle est écrasée au déploiement suivant.
4. **Aucun secret dans Git** : les mots de passe et clés d'API sont dans un coffre (Vault,
   OpenBao, CyberArk), lus au moment du déploiement.
5. **Tout est rejouable** : redéployer, recharger, rejouer un flux ne crée ni doublon ni
   perte (identifiants stables, `_id` déterministes).

## 2. Les environnements

| | DEV | RECETTE (préprod) | PROD |
|---|---|---|---|
| But | construire, essayer | valider avant la prod : montées de version, charge, métier | servir les utilisateurs |
| Cluster | 1 nœud, petit | même topologie que la prod, taille réduite | 3 nœuds maîtres dédiés, nœuds de données, réplicas |
| Données | fictives ou extraits | copie anonymisée de la prod, ou flux réel en parallèle | réelles |
| Sources | bouchons, CSV, une table de test | sources de recette du client (Exadata de recette, instance ServiceNow de test) | sources de production |
| Accès | l'équipe projet | l'équipe projet, les testeurs métier | les utilisateurs, en lecture ; l'exploitation |
| Déploiement | automatique à chaque fusion sur `develop` | automatique après la dev, ou sur étiquette | sur `main`, après validation de la recette |
| Versions | la prochaine | **la même que la prod**, ou la prochaine pendant une montée de version | la version validée |

Les **espaces Kibana** séparent des équipes ou des usages sur un même cluster, pas des
environnements : dev et prod sont des **clusters distincts**.

## 3. Le dépôt

```
datamart-cmdb/
  elasticsearch/
    ingest_pipelines/       PUT _ingest/pipeline/<nom>
    component_templates/    briques : réglages communs, groupes de champs
    index_templates/        assemblent les briques, par motif d'index
    indices/                les index et leurs alias (ci-v1 → alias ci)
    ilm/                    politiques de cycle de vie (historique : chaud, tiède, suppression)
    security/               rôles (lecture métier, écriture du flux, administration)
  logstash/
    pipelines.yml           la liste des pipelines
    pipelines/              un fichier par pipeline (cmdb-ci.conf, cmdb-relations.conf…)
  kafka/                    topics : nom, partitions, rétention (ex. Strimzi KafkaTopic)
  kibana/                   tableaux de bord en NDJSON (exportés ou générés par script)
  tests/                    jeux d'essai, documents attendus, requêtes de contrôle
  envs/
    dev.yml  recette.yml  prod.yml   les seules différences entre environnements
  .gitlab-ci.yml            la chaîne de promotion
```

Dans `envs/` : l'adresse du cluster, le nombre de réplicas, la rétention, le nombre de
partitions Kafka, le dimensionnement de Logstash. Jamais de secret : seulement le **chemin**
du secret dans le coffre.

| Variable | dev | recette | prod |
|---|---|---|---|
| `es_url` | `https://es-dev…` | `https://es-rec…` | `https://es…` |
| `number_of_replicas` | 0 | 1 | 1 |
| rétention de l'historique (ILM) | 30 jours | 90 jours | 3 ans (à fixer avec le métier et l'audit) |
| partitions Kafka par topic | 1 | 3 | 6 |
| workers Logstash | 1 | 2 | 4 |
| secret du compte d'écriture | `kv/dev/datamart/ecriture` | `kv/rec/…` | `kv/prod/…` |

## 4. La chaîne de promotion

```
 branche ──MR──► CI : vérifier ───────────► fusion sur develop ──► DEV
                 · JSON et YAML valides                            · déploiement auto
                 · logstash --config.test_and_exit                 · tests d'intégration
                 · _simulate des pipelines sur les jeux d'essai    │
                 · plan : ce qui changera (+ ~ =)                  ▼
                                                                 RECETTE
                                                                 · même déploiement
                                                                 · recette métier, charge
                                                                 │ validation
                                                                 ▼
                                               fusion sur main ─► PROD
                                                                 · plan relu, puis apply
                                                                 · contrôles après déploiement
```

Chaque déploiement suit **le même ordre**, car chaque brique a besoin de la précédente :

1. rôles et comptes ;
2. politiques ILM ;
3. pipelines d'ingestion ;
4. briques de modèle (*component templates*), puis modèles d'index ;
5. index et alias (créés une seule fois, jamais recréés) ;
6. topics Kafka ;
7. pipelines Logstash (redémarrage progressif) ;
8. tableaux de bord Kibana (import, écrasement).

## 5. Les phases

| # | Phase | Contenu | Livrable | Fini quand |
|---|---|---|---|---|
| 0 | **Cadrage** | questions métier, KPI, consommateurs ; inventaire des sources ; volumes ; droits à demander ; ce que le client fournit (CI, relations, historique) | note de cadrage, liste des questions, matrice des sources | le métier a validé les questions et les KPI |
| 1 | **Socle** | les trois clusters, la sécurité (TLS, comptes, rôles), les sauvegardes (SLM), la supervision du cluster, le dépôt et la CI | **DAT** (architecture technique) | dev et recette répondent ; la CI déploie un index vide |
| 2 | **Modèle** | les index (`ci`, `relations`, `historique`), leurs mappings en `strict`, alias versionnés, ILM de l'historique | **modèle de données** | les documents d'essai passent `_simulate` et s'indexent sans refus |
| 3 | **Ingestion** | CDC sur Exadata, topics Kafka, pipelines Logstash, extraction ServiceNow et ITAM, file de rejets (DLQ) | pipelines dans Git, jeux d'essai | la dev reçoit un changement de la source en moins d'une minute |
| 4 | **Relations et historique** | liens déclarés, déduits, découverts ; historisation (CDC ou photos) ; reprise initiale | règles de rapprochement documentées | « état d'un CI à une date » donne la bonne réponse sur des cas connus |
| 5 | **Restitution** | vues de données, tableaux de bord en code, requêtes ES\|QL, API pour le portail, exports | tableaux de bord dans Git | le métier retrouve ses KPI sur la recette |
| 6 | **Recette** | réconciliation avec la source (mêmes comptes), tests de charge (volume × 2), test de reprise après panne, revue de sécurité | procès-verbal de recette | les écarts sont expliqués, la charge tient |
| 7 | **Mise en production** | reprise initiale, ouverture des flux, contrôles, ouverture aux utilisateurs | **DEX** (exploitation), plan de retour arrière | liste de contrôle du §7 cochée |
| 8 | **Exploitation et évolution** | alertes, montées de version (par la recette), nouveaux champs (v1 → v2), nouvelles sources | DEX tenu à jour | (continu) |

## 6. Le détail par brique

### Mappings et index

- **Nommage** : `<sujet>-v<n>` pour l'index, `<sujet>` pour l'alias. Les applications et
  Kibana ne lisent **que l'alias**.
- **`dynamic: strict`** : un champ inconnu est refusé et part en rejet, au lieu de faire
  dériver le modèle en silence.
- **Types choisis** : `keyword` pour filtrer et compter (identifiants, statuts, codes),
  `text` (avec un sous-champ `keyword`) pour chercher dans un libellé, `date` en UTC,
  nombres au plus juste.
- **Réglages par environnement** : un seul fragment change (`number_of_replicas`), le reste
  est identique.
- **Faire évoluer un champ** : on ne modifie pas un index existant. On crée `ci-v2` avec le
  nouveau mapping, on recopie (`_reindex`), on contrôle les comptes, on bascule l'alias en
  une requête atomique (`_aliases`), on garde `ci-v1` quelques jours pour revenir en arrière.
- **Historique** : un index par période (mois, trimestre) sous une politique ILM :
  chaud, tiède, puis suppression selon la durée de conservation décidée.

### Kafka

- **Un topic par table source** (`cmdb.ci`, `cmdb.relations`, `servicenow.incidents`…).
- **La clé du message = l'identifiant du CI** : tous les changements d'un même CI vont dans
  la même partition, donc **dans l'ordre**.
- **Rétention** : plusieurs jours (de quoi rejouer après une panne de l'aval), voire
  compactage pour garder le dernier état de chaque clé.
- **Groupes de consommateurs** : un par usage (Datamart, sécurité…). Chacun avance à son
  rythme et peut rejouer.
- **Droits** : le CDC écrit, Logstash lit, rien d'autre (ACL par compte).

### Logstash

- **Un pipeline par flux**, déclarés dans `pipelines.yml`. Les traitements communs (dates,
  normalisation des noms) dans un pipeline partagé (*pipeline-to-pipeline*).
- **`document_id` stable** (identifiant du CI, ou position Kafka) : rejouer réécrit au lieu
  de dupliquer.
- **File persistante** (`queue.type: persisted`) : un redémarrage ne perd rien de ce qui
  était en cours.
- **DLQ** (*dead letter queue*) : les documents refusés par Elasticsearch (400, champ
  inconnu, mauvais type) sont mis de côté pour analyse, sans bloquer le flux.
- **Configuration par variables** : `${ES_URL}`, `${KAFKA_BOOTSTRAP}`, mots de passe lus dans
  le coffre ; le fichier de pipeline est le même partout.
- **Tests** : `logstash --config.test_and_exit` dans la CI, puis des messages d'essai
  injectés en dev et les documents attendus comparés.
- **Ingest pipelines d'Elasticsearch** pour ce qui doit s'appliquer quelle que soit l'entrée
  (horodatage, champs calculés) : déclarés en `default_pipeline` dans le modèle d'index.

### Kibana

- **Un espace par usage** (CMDB, qualité des données, exploitation), avec ses droits.
- **Tableaux de bord en NDJSON dans Git**, construits en dev (à la souris puis exportés, ou
  générés par script), importés par la CI (`POST /api/saved_objects/_import?overwrite=true`).
- **Des vues de données à identifiant fixe** : les graphiques s'y réfèrent par cet
  identifiant, le même dans chaque environnement.

### Sécurité

| Rôle | Droits | Pour |
|---|---|---|
| écriture du flux | `create_doc`, `index` sur `ci*`, `relations*`, `historique*` | Logstash, les chargeurs |
| déploiement | modèles, ILM, pipelines, création d'index | la CI |
| lecture métier | `read` sur les alias, un espace Kibana | les utilisateurs |
| administration | tout | deux ou trois personnes, avec traçabilité |

Plus : TLS partout, journaux d'audit d'Elasticsearch en prod, comptes nominatifs pour les
humains (annuaire, SSO), comptes techniques limités pour les flux.

### Exploitation

- **Alertes** : retard de consommation Kafka, rejets en DLQ, cluster jaune ou rouge, disque
  au-dessus de 80 %, aucune écriture depuis N minutes (« le Datamart n'est plus à jour »).
- **Sauvegardes** : SLM quotidienne vers un dépôt externe, **restauration testée** en recette.
- **Réconciliation** : chaque jour, compter les CI dans la source et dans le Datamart ;
  écart au-delà du seuil = alerte.
- **Montées de version** : toujours la recette d'abord, avec la même procédure qu'en prod.

## 7. Liste de contrôle du passage en production

- [ ] Recette signée par le métier (KPI justes, écarts expliqués).
- [ ] Charge tenue en recette avec le double du volume attendu.
- [ ] Sauvegarde SLM active **et restauration testée**.
- [ ] Alertes en place et reçues (test d'une alerte de bout en bout).
- [ ] Comptes et rôles de prod créés, secrets dans le coffre, aucun secret dans Git.
- [ ] Reprise initiale lancée et contrôlée (mêmes comptes que la source).
- [ ] Flux ouverts, retard Kafka proche de zéro, DLQ vide ou expliquée.
- [ ] Tableaux de bord importés, accès des utilisateurs vérifiés.
- [ ] DEX à jour : démarrer, arrêter, rejouer un flux, restaurer, qui appeler.
- [ ] Plan de retour arrière écrit : revenir à l'alias précédent, couper un flux, restaurer.

## 8. Où chaque étape existe déjà dans le lab

| Étape | Dans le lab |
|---|---|
| Mappings en JSON, alias versionnés, `strict` | `datamart/*/elasticsearch/` appliqués par esctl |
| Promotion dev → prod | `develop` → `main` dans ce dépôt ; AWX Check (plan) puis Run |
| Secrets dans un coffre | OpenBao, copiés par ESO, lus par AWX |
| CDC, Kafka, Logstash | POC Galaxie : Debezium → Strimzi Kafka → Logstash (gitopsplatform `apps/galaxie/`) |
| Chargeur par lots | Datamart lab : `datamart/lab/charger.py`, toutes les heures par AWX |
| Historique par photos successives | empreinte dans `charger.py` |
| Tableaux de bord en code | `datamart/demo/tableau_de_bord.py` ; Job `kibana-dashboards` (gitopsplatform) |
| Rôles limités | comptes `datamart-lab`, `galaxie-logstash`, `galaxie-esctl` |
| Deux clusters séparés | ELK du lab et ES du POC Galaxie |
| Leçon d'exploitation | disque plein (15 nov. 2026) : alerte disque à ajouter avant le seuil |
