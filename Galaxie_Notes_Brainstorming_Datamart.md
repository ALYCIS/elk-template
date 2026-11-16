# Galaxie --- Notes pour le brainstorming Datamart Elastic

*16 novembre 2026 --- Support de discussion technique*

## 1. Ce que j'ai compris du besoin jusqu'ici

Le programme Galaxie vise à moderniser la CMDB et à rendre les
informations d'infrastructure plus simples à rechercher, à rapprocher et
à exploiter. Les référentiels restent distribués par domaine : réseau,
datacenter, applications, ITSM et autres sources. La CMDB Galaxie,
construite autour d'Oracle Exadata, consolide ces informations et
demeure le référentiel de référence.

Je situe le Datamart Elastic en aval de cette CMDB. Son rôle n'est pas
de remplacer Oracle ni de devenir un second outil de saisie. Il sert à
présenter une vue de consultation efficace : retrouver un CI, afficher
ses caractéristiques, comprendre ses relations, identifier les données
incomplètes, suivre des indicateurs de qualité et exposer des résultats
au portail ou à des consommateurs via API et exports.

Une instance Elasticsearch existe déjà, mais sa version, sa licence, son
dimensionnement et son éventuel partage avec l'observabilité restent à
confirmer. La chaîne Kafka utilisée dans le programme CMDB avance
également ; je ne considère pas pour autant que l'alimentation Exadata →
Datamart est déjà définie. C'est précisément l'un des contrats à
clarifier.

Le premier résultat visible attendu semble être une démonstration métier
à partir de données CSV. Je vois cette étape comme une occasion de
valider ensemble les données, les écrans et les requêtes, sans figer
prématurément le mode d'alimentation définitif.

Le parcours métier que je proposerais comme fil conducteur est simple :
**rechercher une application, retrouver les serveurs associés, consulter
les dépendances et faire ressortir une anomalie de qualité**. Il permet
de juger la valeur du Datamart autrement qu'en regardant seulement si
des documents sont présents dans Elasticsearch.

## 2. Cinq questions pour ouvrir le cadrage

Je garderais volontairement cinq questions au début de la séance. Elles
portent sur la cible du Datamart, pas sur la manière de charger un CSV.

### 1. Quel est le premier usage métier que nous voulons réellement rendre disponible ?

**Réponse que j'anticipe :** la recherche de CI, la fiche détaillée, les
dépendances et quelques tableaux de bord de qualité.

**Ma proposition :** choisir un parcours utilisateur prioritaire et les
trois à cinq indicateurs qui permettront de dire que le résultat est
utile. Les index et les requêtes seront ensuite définis à partir de ce
parcours.

### 2. Quelle donnée fait autorité et à quel moment devient-elle publiable vers le Datamart ?

**Réponse que j'anticipe :** Oracle Exadata est le référentiel consolidé
; la fédération et les contrôles de qualité sont encore en cours de
construction.

**Ma proposition :** identifier le point exact de publication après
validation métier, ainsi que le contrat de données : identifiant stable,
type d'objet, attributs, version, date de modification, statut et
suppression. Je distinguerais bien les flux bruts d'entrée de la donnée
consolidée destinée au Datamart.

### 3. Comment la chaîne actuelle prévoit-elle d'alimenter le Datamart après la CMDB ?

**Réponse que j'anticipe :** Kafka est déjà présent pour la fédération,
mais le raccordement spécifique du Datamart et ses garanties restent à
préciser.

**Ma proposition :** comparer la publication d'événements validés, une
extraction incrémentale depuis Oracle et une première alimentation par
lots. Je retiendrais le mécanisme qui permet de reprendre un flux après
incident et de réconcilier le résultat avec Oracle, sans imposer Kafka
uniquement parce qu'il existe déjà.

### 4. Quels volumes, quelle fraîcheur et quels engagements d'exploitation visons-nous ?

**Réponse que j'anticipe :** le parc est de l'ordre de plusieurs
centaines de milliers d'objets au maximum, mais les chiffres réels de
CI, de relations, de changements et les objectifs de délai ne sont pas
encore stabilisés.

**Ma proposition :** distinguer le stock de CI, le nombre de relations,
les changements quotidiens, les pics et le nombre d'utilisateurs
simultanés. Cela donnera une base sérieuse pour les shards, les
réplicas, la capacité, la reprise et les tests de charge. Je préfère une
mesure sur échantillon à un dimensionnement annoncé trop tôt.

### 5. Qui pourra consulter quelles données, et depuis quelle interface ?

**Réponse que j'anticipe :** un portail Galaxie unifié est envisagé,
avec des droits par profil et des usages complémentaires de Kibana ; les
règles fines restent à arbitrer.

**Ma proposition :** préciser les consommateurs --- métier, experts,
API, exports --- et les restrictions par domaine ou par champ avant de
choisir la stratégie d'indexation. Un espace Kibana ne remplace pas à
lui seul une règle de sécurité sur les documents.

## 3. Ma méthode de travail : un dépôt Git comme référence technique

Je souhaite proposer un dépôt Git dès les premières itérations. Ce n'est
pas une démarche destinée à ralentir le prototype : au contraire, elle
permet de reproduire le même résultat, de comprendre les changements et
d'éviter qu'une configuration validée en développement soit impossible à
reconstruire en recette.

Dans Elasticsearch, une grande partie de l'architecture vit sous forme
de configuration : mappings, templates, pipelines d'ingestion, alias,
politiques de rétention et rôles. Logstash ajoute ses propres pipelines
; Kibana possède des objets exportables. Si ces éléments sont créés
uniquement à la main dans les interfaces, la configuration réelle finit
par dépendre de l'historique des manipulations.

Je reprendrais la structure de ma feuille de route :

``` text
datamart-cmdb/
├── elasticsearch/
│   ├── ingest_pipelines/
│   ├── component_templates/
│   ├── index_templates/
│   ├── indices/
│   ├── ilm/
│   └── security/
├── logstash/
│   ├── pipelines.yml
│   └── pipelines/
├── kafka/
├── kibana/
├── tests/
├── envs/
│   ├── dev.yml
│   ├── recette.yml
│   └── prod.yml
└── .gitlab-ci.yml
```

**Ce que chaque partie apporte :** `elasticsearch/` décrit les
structures et les règles du moteur ; `logstash/` contient les
traitements d'entrée ; `kafka/` documente les topics et leurs paramètres
si Kafka est retenu ; `kibana/` conserve les vues et tableaux de bord
exportés ; `tests/` contient les jeux de données et les résultats
attendus ; `envs/` porte les différences autorisées entre environnements
; `.gitlab-ci.yml` décrit les vérifications et les déploiements.

Je présenterais une progression raisonnable : validation des fichiers et
des mappings à chaque merge request, déploiement automatique en DEV,
tests de chargement et de requêtes, promotion en recette puis
déploiement en production après validation. Les secrets resteraient dans
le coffre prévu par la plateforme, jamais dans le dépôt.

Le point que je défendrais est la **traçabilité**. Quand un champ change
de type, quand un index doit être reconstruit ou quand un tableau de
bord est modifié, on retrouve la raison du changement, sa validation et
la version déployée. Git décrit l'état attendu ; les contrôles de
déploiement vérifient aussi l'état réellement présent dans les clusters.

Pour la première démonstration, je ne chercherais pas à construire toute
la chaîne de production. Un dépôt minimal contenant un mapping, un
chargeur, un jeu de test, deux requêtes et un tableau de bord suffit
déjà à poser cette discipline.

## 4. La structure d'indexation que je propose d'examiner

Je ne pars pas du principe qu'un index doit correspondre à une source ou
à un topic Kafka. **Je découpe les index selon la nature des données,
les usages de lecture, les fréquences de mise à jour, les règles de
sécurité et les durées de conservation.** Les huit familles ci-dessous
constituent une cible de discussion, pas huit index à créer
immédiatement.

  -------------------------------------------------------------------------
  Famille d'index           Contenu envisagé        Usage et justification
  ------------------------- ----------------------- -----------------------
  `galaxie-ci-*`            État courant des CI :   Recherche, filtres,
                            applications, serveurs, fiche 360°. C'est le
                            équipements, bases,     cœur du Datamart. Un
                            etc.                    découpage par domaine
                                                    reste une option si les
                                                    schémas, les droits ou
                                                    les cycles de vie
                                                    diffèrent réellement.

  `galaxie-relations`       Liens `source_id`,      Recherche des
                            `target_id`,            dépendances sans
                            `relation_type`,        recopier tous les liens
                            éventuellement validité dans chaque CI. Les
                            et provenance           relations peuvent
                                                    évoluer indépendamment
                                                    des fiches.

  `galaxie-ref-*`           Sites, équipes,         Uniformiser les
                            catégories et autres    libellés et faciliter
                            référentiels partagés   certains
                                                    enrichissements.
                                                    L'intérêt d'un index de
                                                    lookup dépend des
                                                    requêtes et de la
                                                    version Elastic.

  `galaxie-itsm`            Incidents, changements  Croiser une fiche CI
                            ou enrichissements      avec son contexte ITSM,
                            ServiceNow retenus      sans confondre les
                                                    tickets avec
                                                    l'inventaire de
                                                    référence.

  `galaxie-historique-*`    Versions datées ou      Retrouver un état
                            événements de           passé, expliquer une
                            changement des CI       évolution, alimenter
                                                    l'audit. Le choix entre
                                                    versions complètes,
                                                    événements et
                                                    instantanés reste à
                                                    préciser.

  `galaxie-kpi-*`           Mesures de qualité et   Afficher des tendances
                            agrégats datés          sans recalculer tout le
                                                    parc à chaque ouverture
                                                    de tableau de bord.

  `galaxie-docs-*`          Lexique, règles de      Recherche documentaire,
                            gestion, documentation  éventuellement enrichie
                            utile                   plus tard ; hors cœur
                                                    du premier parcours CI.

  `galaxie-abonnements-*`   Critères d'abonnement   Préparer les data
                            ou règles de sélection  products et
                                                    notifications, si ce
                                                    besoin est confirmé. Le
                                                    mécanisme percolator
                                                    n'est qu'une option à
                                                    évaluer.
  -------------------------------------------------------------------------

### Pourquoi séparer les CI et leurs relations ?

Une application peut dépendre de plusieurs serveurs ; un équipement peut
avoir des centaines ou des milliers de liens. Si toutes ces relations
sont intégrées dans la fiche du CI, chaque changement de relation peut
entraîner une réécriture de la fiche, et certains documents deviennent
disproportionnés.

Avec un index `galaxie-ci`, je retrouve les objets. Avec
`galaxie-relations`, je retrouve leurs liens. L'API peut assembler une
fiche et ses dépendances pour l'écran. Cette approche convient bien à
une navigation de proximité. Pour des parcours profonds, des plus courts
chemins ou des boucles, je demanderais d'abord des exemples concrets
avant d'envisager les capacités graphe d'Oracle ou une base dédiée.

### Pourquoi distinguer l'état courant, l'historique et les KPI ?

Ce ne sont pas les mêmes questions. L'état courant répond à « quel est
le statut de ce serveur ? ». L'historique répond à « quel était son
statut à telle date ? ». Les KPI répondent à « comment le taux de
complétude a-t-il évolué depuis trois mois ? ».

Les séparer évite de mélanger des mises à jour fréquentes, des données
conservées pour l'audit et des séries d'indicateurs. Cela facilite aussi
les règles de rétention et les tests. Pour le premier incrément, je
privilégierais **CI + relations**, puis un indicateur de qualité
démontrable ; l'historisation détaillée suivrait selon le besoin
confirmé.

### Ce que je souhaite valider avant de figer les mappings

Je partirais d'un dictionnaire de données, même partiel. Les
identifiants, statuts et codes seront généralement des `keyword` ; les
dates des `date` ; les volumes et scores des types numériques ; les
adresses IP du type `ip` si elles sont propres ; les libellés recherchés
en plein texte pourront utiliser `text` avec un sous-champ `keyword`.

J'envisage `dynamic: strict` pour les index contractualisés, à condition
d'avoir un circuit explicite de traitement des rejets. Un champ inconnu
ne doit pas disparaître silencieusement. Pour des domaines encore
instables, je peux prévoir une zone d'attributs contrôlée plutôt que de
multiplier les champs sans gouvernance.

Les alias permettront de dissocier le nom lu par les applications des
index physiques versionnés. Une évolution incompatible de mapping
conduira à reconstruire un nouvel index, vérifier les données puis
basculer l'alias. La bascule seule ne suffit pas : les modifications
reçues pendant la reconstruction doivent aussi être reprises.

### Kafka, Logstash et index : trois notions distinctes

Un **topic Kafka** transporte des événements ; un **pipeline Logstash**
lit, transforme et oriente ces événements ; un **index Elasticsearch**
organise les documents pour la consultation. Il n'existe pas de
correspondance obligatoire un pour un.

Un topic peut alimenter plusieurs index si un événement sert à mettre à
jour l'état courant et à conserver une trace historique. Plusieurs
topics peuvent aussi converger vers un même index de CI après
normalisation. À l'inverse, un flux homogène peut naturellement
alimenter un seul index.

Pour Galaxie, je souhaiterais connaître les messages réellement publiés
après consolidation Oracle : leur format, leur identifiant, leur
version, leur ordre, le traitement des suppressions et la possibilité de
les rejouer. Le choix des topics viendra de ce contrat, pas du nom des
index.

## 5. La proposition CSV de l'équipe : une démonstration utile, à rendre reproductible

L'équipe envisage d'injecter des données CSV afin de montrer rapidement
quelque chose au métier. C'est une bonne entrée en matière : un extrait
concret permet de confronter le modèle à la réalité, de vérifier les
recherches et d'obtenir des retours sur la présentation.

Je ne sais pas encore si l'import est prévu depuis l'interface Kibana ou
par un autre outil. Si l'option retenue est **Kibana → Upload a file**,
elle peut convenir à une exploration ponctuelle, sous réserve que la
fonctionnalité soit disponible et autorisée sur l'instance. Je ne la
prendrais pas comme mécanisme durable d'alimentation.

Mon idée est de conserver la rapidité de cette démonstration tout en
ajoutant une petite chaîne versionnée :

``` text
CSV représentatif et autorisé
          |
          v
Contrôle du format + normalisation
          |
          v
Chargeur versionné (script / Logstash / Bulk API)
          |
          v
Index et mappings déployés depuis Git
          |
          v
Requêtes de vérification + vue métier
```

Pour ce premier chargement, Kafka n'apporte pas nécessairement de
valeur. Le même modèle de données et les mêmes contrôles pourront
ensuite être conservés lorsque l'alimentation passera à un flux Oracle
ou Kafka. Le CSV devient alors **un jeu d'essai reproductible**, et non
une architecture parallèle à entretenir.

Je proposerais une démonstration avec un fichier CI et, si possible, un
fichier relations. Le métier pourrait rechercher une application, voir
ses serveurs et identifier des champs manquants. Les données seraient
fictives ou extraites avec autorisation et assainies selon les règles du
client.

### Ce que je regarderai particulièrement pendant l'import

**La signification des lignes.** Un CSV peut contenir un CI par ligne,
plusieurs lignes pour un même CI ou des relations mélangées aux objets.
Je veux comprendre la granularité avant de définir le `_id`.

**Les identifiants.** Je vérifierai leur stabilité et leur unicité, y
compris entre domaines. Un `_id` déterministe permet de relancer un
chargement sans multiplier les doublons. Cela ne règle pas à lui seul
les mises à jour arrivées dans le désordre.

**Les types et les valeurs absentes.** Dates françaises, nombres avec
virgule, booléens, cellules vides, chaînes « NULL », encodage UTF-8,
séparateur `;` ou `,` : ces détails modifient réellement la qualité des
recherches et des indicateurs. Je veux les traiter explicitement avant
l'indexation.

**Les relations orphelines.** Un lien vers un CI absent de l'extrait
peut être normal dans un jeu partiel ; il doit être identifiable. Je
distinguerai une donnée absente du CSV d'une incohérence dans la CMDB.

**Le comportement au second chargement.** Je referai volontairement le
même import, puis un import avec une modification et une suppression. Le
nombre de documents et leur état doivent rester explicables. Un import
CSV ne signifie pas automatiquement synchronisation complète : les CI
disparus du fichier ne doivent pas être supprimés sans règle validée.

**Les rejets et les écarts.** Je suivrai le nombre de lignes lues,
acceptées, mises à jour, rejetées et ignorées, avec des motifs
consultables. Une démonstration qui affiche 1 000 documents sans dire ce
qu'il est advenu des 50 autres ne prouve pas la qualité du pipeline.

**Les performances et la sécurité.** Je mesurerai la durée du
chargement, la taille de l'index et les premières latences de recherche.
Je vérifierai aussi les droits d'import, les données sensibles et
l'impact sur un cluster éventuellement partagé avec l'observabilité.

### Ce qui intéressera surtout le métier

Le métier ne jugera probablement pas la réussite au nombre de topics, de
pipelines ou de shards. Il voudra reconnaître ses objets, retrouver une
application par son nom, comprendre les liens, voir les données
manquantes, filtrer par domaine ou statut et obtenir des chiffres
cohérents.

Je présenterais donc la démonstration dans cet ordre : **recherche →
fiche CI → relations → anomalie qualité → indicateur simple**. Les
contrôles techniques resteront disponibles pour expliquer la fiabilité
du résultat, sans occuper tout le temps de présentation.

## 6. Ce que j'aimerais obtenir à l'issue du brainstorming

À la fin de la séance, mon objectif serait d'avoir un premier parcours
métier validé, un extrait de données représentatif, le nom des
responsables du contrat de publication depuis Oracle, les principales
contraintes d'accès et un accord sur la manière de versionner les
configurations.

Je ne chercherais pas à faire valider immédiatement huit index, un
nombre de nœuds ou une architecture Kafka définitive. Je proposerais
plutôt de figer **le premier modèle CI/relations, les règles de
chargement et les critères de réussite**, puis d'utiliser les mesures du
prototype pour arbitrer la suite.

Ma position tient en une phrase : **démarrer simplement pour rendre le
besoin visible, tout en gardant dès le départ une architecture que
l'équipe pourra expliquer, rejouer et faire évoluer jusqu'à la
production.**
