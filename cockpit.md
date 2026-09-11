# Mode opératoire — Cockpit de gestion des demandes d'achat

## Overview Page SAP Fiori elements

**KPI · Suivi de conversion · Navigation vers les applications**
**SAP S/4HANA (Private Cloud and On-Premise) 2021 — FPS02**

| Attribut | Valeur |
|---|---|
| Référence du document | MO-OVP-PRCOCKPIT-2021FPS02-v1.0 |
| Documents liés | [MO Service RAP](MO%20Service%20RAP.md) · [MO Annotations CDS](MO%20Annotations%20CDS.md) |
| Objet créé | Overview Page de pilotage des demandes d'achat et de leur conversion |
| Architecture | 100 % ABAP — CDS analytique, OData V2, Gateway embedded |
| Protocole | OData V2 |
| Version produit | SAP S/4HANA 2021 FPS02 — Private Cloud / On-Premise |
| Outils | Eclipse + ABAP Development Tools · VS Code + SAP Fiori tools · SAP GUI |
| Auteur | … |
| Vérifié par | … |
| Approuvé par | … |
| Date de rédaction | … |
| Statut | Version de travail |

> **Convention de ce document**
> Chaque emplacement de copie d'écran est signalé par un bloc `📸 COPIE D'ÉCRAN N°XX`.
> Déposez vos images dans `images/OVP-COCKPIT/` puis remplacez la ligne indiquée par le lien Markdown correspondant.

---

## Sommaire

- [1. Objet et architecture](#1-objet-et-architecture)
- [2. Principes de l'Overview Page](#2-principes-de-loverview-page)
- [3. Prérequis](#3-prérequis)
- [Partie A — Modèle de données analytique](#partie-a--modèle-de-données-analytique)
- [Partie B — Génération de l'Overview Page](#partie-b--génération-de-loverview-page)
- [Partie C — Navigation vers les applications](#partie-c--navigation-vers-les-applications)
- [Partie D — Déploiement et publication](#partie-d--déploiement-et-publication)
- [Partie E — Recette, transport et exploitation](#partie-e--recette-transport-et-exploitation)
- [Annexe A — Diagnostic des incidents fréquents](#annexe-a--diagnostic-des-incidents-fréquents)
- [Annexe B — Index des copies d'écran](#annexe-b--index-des-copies-décran)
- [Annexe C — Historique des versions](#annexe-c--historique-des-versions)

---

## 1. Objet et architecture

### 1.1 Objet

Ce mode opératoire décrit la création d'un cockpit de pilotage des demandes d'achat et de leur conversion en commandes, sous forme d'une **Overview Page SAP Fiori elements**. Le cockpit présente des indicateurs, permet de suivre le reste à convertir et donne accès en un clic aux applications de traitement.

### 1.2 Choix d'architecture

Le besoin initial mentionnait le modèle de programmation CAP. Ce modèle produit un service Node.js ou Java qui exige son propre runtime et sa propre persistance : la pile ABAP de S/4HANA ne l'exécute pas, et le Gateway embedded n'expose que des services ABAP. L'architecture retenue est donc entièrement ABAP, ce qui répond au besoin fonctionnel sans dépendance à une plateforme externe.

| Couche | Technologie retenue | Justification |
|---|---|---|
| Données | Vues CDS analytiques | Agrégation native, pas de réplication |
| Service | OData V2 publié depuis CDS | Protocole supporté par l'Overview Page |
| Interface | Overview Page Fiori elements | Cartes de KPI et navigation natives |
| Exécution | Gateway embedded | Aucune infrastructure supplémentaire |
| Déploiement | Repository SAPUI5 ABAP | Même chaîne que les applications existantes |

> **⚠️ Effet de bord favorable**
> L'Overview Page fonctionne en OData V2. Le cockpit est donc accessible même si la route d'accès des utilisateurs refuse encore les URL OData V4. Seule la navigation vers l'application de suivi, qui est en V4, restera bloquée tant que ce point d'infrastructure n'est pas corrigé.

### 1.3 Contenu fonctionnel du cockpit

| Élément | Type de carte | Contenu |
|---|---|---|
| Postes en attente | Carte analytique | Volume non converti, par groupe d'acheteurs |
| Taux de conversion | Carte analytique | Rapport converti / total, avec seuils |
| Montant en attente | Carte analytique | Valorisation du reste à convertir |
| Demandes à traiter | Carte liste | Postes les plus anciens non convertis |
| Répartition par division | Carte graphique | Volume par site |
| Accès rapides | Carte de liens | Application de suivi, ME59N, traitement des DA |

### 1.4 Hors périmètre

- Modèle de programmation CAP et plateforme SAP BTP.
- Réplication de données vers un entrepôt externe.
- Création de l'application de suivi des demandes d'achat, traitée dans son propre mode opératoire.
- Personnalisation avancée des graphiques au-delà des annotations.
- Traduction des libellés.

---

## 2. Principes de l'Overview Page

### 2.1 Nature de l'objet

Une Overview Page est un modèle d'application SAP Fiori elements composé de **cartes**. Chaque carte affiche un extrait du modèle de données et peut mener vers une autre application. La composition, le contenu et le comportement des cartes se déclarent dans le descripteur de l'application et dans les annotations CDS : il n'y a pratiquement aucun code à écrire.

### 2.2 Types de cartes

| Type | Usage |
|---|---|
| Carte analytique | Indicateur chiffré et graphique associé — le cœur d'un cockpit |
| Carte liste | Liste d'objets, triée et filtrée |
| Carte tableau | Plusieurs colonnes par ligne |
| Carte de liens | Points d'entrée vers d'autres applications ou transactions |
| Carte pile | Ensemble d'objets à traiter, parcourus un par un |

### 2.3 Contrainte de protocole

L'Overview Page consomme des services **OData V2**. Le service du cockpit est donc publié en V2 depuis la vue CDS, ce qui est la voie standard et la plus simple sur cette version. Les applications cibles de la navigation peuvent, elles, reposer sur n'importe quel protocole : la navigation intentionnelle ne transporte qu'un objet sémantique et une action.

### 2.4 Chaîne de construction

| Ordre | Objet | Outil | Partie |
|---|---|---|---|
| 1 | Vue CDS cube | Eclipse ADT | A |
| 2 | Vue CDS de requête analytique | Eclipse ADT | A |
| 3 | Annotations de KPI | Eclipse ADT | A |
| 4 | Publication du service OData V2 | Eclipse ADT, SAP GUI | A |
| 5 | Génération de l'Overview Page | VS Code | B |
| 6 | Configuration des cartes | VS Code | B |
| 7 | Navigation vers les applications | VS Code, SAP GUI | C |
| 8 | Déploiement et publication | VS Code, SAP GUI | D |

---

## 3. Prérequis

| Élément | Exigence |
|---|---|
| Plateforme | SAP S/4HANA 2021 FPS02 |
| Analytique embarquée | Composants d'analytique embarquée actifs |
| Outils | Eclipse avec ABAP Development Tools ; VS Code avec SAP Fiori tools |
| Autorisations | `S_DEVELOP`, `S_TRANSPRT`, administration Gateway et Launchpad |
| Package | Package de développement Z et ordres de transport |
| Données | Jeu de demandes d'achat représentatif, converties et non converties |
| Applications cibles | Application de suivi déployée ; transaction ME59N accessible |

> **ℹ️ Vérification préalable**
> Avant d'écrire la moindre ligne, tester une URL OData V2 par la route d'accès des utilisateurs et vérifier qu'elle répond. Ce contrôle de deux minutes évite de découvrir un blocage d'infrastructure après plusieurs jours de développement.

> **📸 COPIE D'ÉCRAN N°01** — Navigateur : test d'une URL OData V2 par la route d'accès des utilisateurs
> *Remplacer cette ligne par :* `![Copie 01](images/OVP-COCKPIT/capture-01.png)`

---

## Partie A — Modèle de données analytique

### A1 — Créer la vue cube

#### Rôle

Le cube porte les dimensions d'analyse et les mesures. Il ne contient aucune agrégation : celle-ci est réalisée à l'exécution par la couche analytique. Les mesures dérivées, comme l'indicateur de conversion, se calculent ici.

#### Mode opératoire

1. Créer un objet de type « Data Definition » dans le package.
2. Saisir la définition ci-dessous, en vérifiant au préalable dans Eclipse les noms de champs exacts de la vue standard source.
3. Activer et contrôler par un aperçu de données.
4. Vérifier la cohérence des volumes avec une extraction de contrôle.

#### Code — vue `ZI_PurReqToPoCube`

```abap
@AbapCatalog.sqlViewName: 'ZIPRPOCUBE'
@AbapCatalog.compiler.compareFilter: true
@AccessControl.authorizationCheck: #NOT_REQUIRED
@EndUserText.label: 'Cube - demandes d''achat et conversion en commandes'
@Analytics.dataCategory: #CUBE

define view ZI_PurReqToPoCube
  as select from I_PurchaseRequisitionItem as PurReqItem
{
  key PurReqItem.PurchaseRequisition,
  key PurReqItem.PurchaseRequisitionItem,

      PurReqItem.PurchasingGroup,
      PurReqItem.Plant,
      PurReqItem.Material,
      PurReqItem.PurchaseRequisitionType,
      PurReqItem.PurchaseRequisitionReleaseDate,
      PurReqItem.PurchaseOrder,
      PurReqItem.PurchaseOrderItem,

      -- indicateur de conversion
      case when PurReqItem.PurchaseOrder <> ''
           then 1 else 0 end                    as ConvertedItemCount,

      -- compteur de postes
      1                                          as PurReqItemCount,

      -- délai de conversion en jours
      case when PurReqItem.PurchaseOrder <> ''
           then dats_days_between(
                  PurReqItem.PurchaseRequisitionReleaseDate,
                  PurReqItem.PurchaseRequisitionReleaseDate )
           else 0 end                            as ConversionLeadTime,

      @Semantics.amount.currencyCode: 'Currency'
      PurReqItem.PurchaseRequisitionPrice        as ItemAmount,
      PurReqItem.Currency
}
```

> **📸 COPIE D'ÉCRAN N°02** — Eclipse ADT : vue cube, code source
> *Remplacer cette ligne par :* `![Copie 02](images/OVP-COCKPIT/capture-02.png)`

> **📸 COPIE D'ÉCRAN N°03** — Eclipse ADT : aperçu de données du cube
> *Remplacer cette ligne par :* `![Copie 03](images/OVP-COCKPIT/capture-03.png)`

> **⚠️ À vérifier**
> Les noms de champs de la vue standard des postes de demande d'achat dépendent de la version. Les relever dans Eclipse avant activation. Le calcul du délai de conversion est donné comme structure : il doit être adapté à la date réellement disponible pour la commande.

---

### A2 — Créer la vue de requête analytique

#### Rôle

La requête définit ce qui est exposé au cockpit : dimensions en lignes, dimensions libres utilisables en filtre, mesures et formules. C'est elle qui est publiée en service OData.

#### Code — vue `ZC_PurReqToPoQuery`

```abap
@AbapCatalog.sqlViewName: 'ZCPRPOQUERY'
@AccessControl.authorizationCheck: #NOT_REQUIRED
@EndUserText.label: 'Requête cockpit - DA vers commandes'
@Analytics.query: true
@OData.publish: true

define view ZC_PurReqToPoQuery
  as select from ZI_PurReqToPoCube
{
  @AnalyticsDetails.query.axis: #ROWS
  @EndUserText.label: 'Groupe d''acheteurs'
  PurchasingGroup,

  @AnalyticsDetails.query.axis: #ROWS
  @EndUserText.label: 'Division'
  Plant,

  @AnalyticsDetails.query.axis: #FREE
  PurchaseRequisitionType,

  @AnalyticsDetails.query.axis: #FREE
  PurchaseRequisitionReleaseDate,

  @DefaultAggregation: #SUM
  @EndUserText.label: 'Postes de DA'
  PurReqItemCount,

  @DefaultAggregation: #SUM
  @EndUserText.label: 'Postes convertis'
  ConvertedItemCount,

  @DefaultAggregation: #SUM
  @Semantics.amount.currencyCode: 'Currency'
  @EndUserText.label: 'Montant'
  ItemAmount,

  Currency,

  @AnalyticsDetails.query.formula: '$projection.PurReqItemCount - $projection.ConvertedItemCount'
  @EndUserText.label: 'Postes en attente'
  1 as OpenItemCount,

  @AnalyticsDetails.query.formula: 'case when $projection.PurReqItemCount > 0 then $projection.ConvertedItemCount / $projection.PurReqItemCount * 100 else 0 end'
  @EndUserText.label: 'Taux de conversion'
  1 as ConversionRate
}
```

> **📸 COPIE D'ÉCRAN N°04** — Eclipse ADT : vue de requête analytique, code source
> *Remplacer cette ligne par :* `![Copie 04](images/OVP-COCKPIT/capture-04.png)`

| Annotation | Effet |
|---|---|
| `@Analytics.query: true` | Déclare la vue comme requête analytique |
| `@AnalyticsDetails.query.axis: #ROWS` | Dimension affichée en ligne |
| `@AnalyticsDetails.query.axis: #FREE` | Dimension disponible en filtre, non affichée |
| `@DefaultAggregation: #SUM` | Mode d'agrégation de la mesure |
| `@AnalyticsDetails.query.formula` | Mesure calculée à partir d'autres mesures |
| `@OData.publish: true` | Génère et enregistre le service OData V2 |

> **ℹ️ Formules**
> Les mesures calculées se déclarent avec une valeur factice dans la liste de champs, la formule étant portée par l'annotation. Vérifier le résultat dans l'aperçu de données avant de passer à la suite : une formule erronée ne produit pas d'erreur d'activation.

---

### A3 — Annoter les indicateurs

#### Rôle

Les annotations de point de données définissent chaque indicateur : son titre, sa valeur cible et ses seuils de criticité. Celles de graphique et de variante de présentation décrivent la restitution visuelle. L'Overview Page s'y réfère ensuite par leur **qualificateur**.

#### Code — extension de métadonnées

```abap
@Metadata.layer: #CORE

@UI.chart: [ { qualifier:      'ConversionParGroupe',
               chartType:      #COLUMN,
               dimensions:     [ 'PurchasingGroup' ],
               measures:       [ 'OpenItemCount' ],
               dimensionAttributes: [ { dimension: 'PurchasingGroup',
                                        role: #CATEGORY } ],
               measureAttributes:   [ { measure: 'OpenItemCount',
                                        role: #AXIS_1,
                                        asDataPoint: true } ] } ]

@UI.selectionVariant: [ { qualifier: 'ToutesDA',
                          text:      'Toutes les demandes d''achat' } ]

@UI.presentationVariant: [ { qualifier: 'ParGroupe',
                             sortOrder: [ { by: 'OpenItemCount',
                                            direction: #DESC } ],
                             visualizations: [ { type: #AS_CHART,
                                                 qualifier: 'ConversionParGroupe' } ] } ]

annotate view ZC_PurReqToPoQuery with
{
  @UI.dataPoint: { qualifier:     'PostesEnAttente',
                   title:         'Postes de DA en attente',
                   criticalityCalculation: {
                     improvementDirection: #MINIMIZE,
                     toleranceRangeHighValue:  50,
                     deviationRangeHighValue: 100 } }
  OpenItemCount;

  @UI.dataPoint: { qualifier:  'TauxConversion',
                   title:      'Taux de conversion',
                   targetValue: 95,
                   criticalityCalculation: {
                     improvementDirection: #MAXIMIZE,
                     toleranceRangeLowValue:  80,
                     deviationRangeLowValue:  60 } }
  ConversionRate;
}
```

> **📸 COPIE D'ÉCRAN N°05** — Eclipse ADT : extension de métadonnées des indicateurs
> *Remplacer cette ligne par :* `![Copie 05](images/OVP-COCKPIT/capture-05.png)`

| Élément | Rôle |
|---|---|
| `dataPoint` | Définition de l'indicateur, cible et seuils |
| `criticalityCalculation` | Calcul automatique de la couleur selon les seuils |
| `improvementDirection` | Sens d'amélioration : minimiser ou maximiser |
| `chart` | Type de graphique, dimensions et mesures |
| `presentationVariant` | Tri et visualisation associée |
| `selectionVariant` | Filtre appliqué à la carte |
| `qualifier` | Identifiant repris dans le descripteur de l'application |

> **ℹ️ Les qualificateurs sont structurants**
> Chaque carte du cockpit référencera ces annotations par leur qualificateur. Les nommer de façon explicite et stable dès le départ évite une reprise complète du descripteur.

---

### A4 — Publier le service OData V2

1. Vérifier la présence de l'annotation de publication sur la vue de requête.
2. Activer la vue : le service est généré automatiquement.
3. Lancer la transaction `/IWFND/MAINT_SERVICE` et rechercher le service généré.
4. L'ajouter s'il n'est pas déjà enregistré, en renseignant l'alias système, le package et l'ordre de transport.
5. Vérifier que le statut ICF est actif.
6. Relever le nom exact du service : il sera repris dans le descripteur de l'application.

```
Nom du service généré : <nom_de_vue_SQL>_CDS
```

> **📸 COPIE D'ÉCRAN N°06** — Transaction `/IWFND/MAINT_SERVICE` : service généré, enregistré et actif
> *Remplacer cette ligne par :* `![Copie 06](images/OVP-COCKPIT/capture-06.png)`

---

### A5 — Tester le service

1. Lancer la transaction `/IWFND/GW_CLIENT`.
2. Appeler le document de métadonnées et vérifier la présence des mesures et des annotations.
3. Exécuter une lecture agrégée et contrôler la cohérence des chiffres.
4. Rejouer le même appel depuis un navigateur, par la route d'accès des utilisateurs.

> **📸 COPIE D'ÉCRAN N°07** — Transaction `/IWFND/GW_CLIENT` : métadonnées du service analytique
> *Remplacer cette ligne par :* `![Copie 07](images/OVP-COCKPIT/capture-07.png)`

> **📸 COPIE D'ÉCRAN N°08** — Navigateur : lecture agrégée du service par la route des utilisateurs
> *Remplacer cette ligne par :* `![Copie 08](images/OVP-COCKPIT/capture-08.png)`

#### Critères de validation de la partie A

- Le cube et la requête sont activés et cohérents.
- Le service répond en HTTP 200 par la route des utilisateurs.
- Les annotations d'indicateur apparaissent dans les métadonnées.

---

## Partie B — Génération de l'Overview Page

### B1 — Générer le projet

1. Ouvrir VS Code et lancer le générateur d'application SAP Fiori.
2. Choisir le modèle **Overview Page**.
3. Sélectionner comme source de données le système ABAP puis le service publié en partie A.
4. Sélectionner l'entité de filtre global : l'entité de la requête analytique.
5. Renseigner le nom du module, le titre et l'espace de noms.
6. Renseigner la configuration Launchpad : objet sémantique, action et titre de la tuile.
7. Terminer la génération et lancer l'application en local.

> **📸 COPIE D'ÉCRAN N°09** — VS Code : générateur, choix du modèle Overview Page
> *Remplacer cette ligne par :* `![Copie 09](images/OVP-COCKPIT/capture-09.png)`

> **📸 COPIE D'ÉCRAN N°10** — VS Code : sélection du service analytique comme source de données
> *Remplacer cette ligne par :* `![Copie 10](images/OVP-COCKPIT/capture-10.png)`

> **📸 COPIE D'ÉCRAN N°11** — VS Code : configuration Launchpad du cockpit
> *Remplacer cette ligne par :* `![Copie 11](images/OVP-COCKPIT/capture-11.png)`

> **📸 COPIE D'ÉCRAN N°12** — VS Code : arborescence du projet généré
> *Remplacer cette ligne par :* `![Copie 12](images/OVP-COCKPIT/capture-12.png)`

> **⚠️ Version SAPUI5**
> Sélectionner la version correspondant à celle du système cible, et non la plus récente proposée par défaut. Une application générée pour une version postérieure se charge en local mais échoue une fois déployée.

---

### B2 — Descripteur de l'application

Le descripteur déclare la source de données, le point d'entrée dans le Launchpad et l'ensemble des cartes. C'est le fichier central du cockpit.

#### Source de données et point d'entrée

```json
"sap.app": {
  "id": "zovp.purreqcockpit",
  "type": "application",
  "dataSources": {
    "mainService": {
      "uri": "/sap/opu/odata/sap/ZCPRPOQUERY_CDS/",
      "type": "OData",
      "settings": { "odataVersion": "2.0" }
    }
  },
  "crossNavigation": {
    "inbounds": {
      "cockpit-display": {
        "semanticObject": "PurReqCockpit",
        "action": "display",
        "title": "Cockpit demandes d'achat",
        "subTitle": "Suivi et conversion",
        "icon": "sap-icon://business-objects-experience",
        "signature": {
          "parameters": {},
          "additionalParameters": "allowed"
        }
      }
    }
  }
}
```

> **📸 COPIE D'ÉCRAN N°13** — VS Code : descripteur de l'application, source de données et point d'entrée
> *Remplacer cette ligne par :* `![Copie 13](images/OVP-COCKPIT/capture-13.png)`

---

### B3 — Configurer la section des cartes

La section dédiée à l'Overview Page définit le filtre global et la liste des cartes. Le filtre global s'applique simultanément à toutes les cartes qui le référencent.

```json
"sap.ovp": {
  "globalFilterModel": "mainModel",
  "globalFilterEntityType": "ZC_PurReqToPoQueryType",
  "containerLayout": "resizable",
  "enableLiveFilter": true,
  "considerAnalyticalParameters": true,
  "cards": {
    "card01_kpi_attente": {
      "model": "mainModel",
      "template": "sap.ovp.cards.charts.analytical",
      "settings": {
        "title": "Postes de DA en attente",
        "subTitle": "Par groupe d'acheteurs",
        "entitySet": "ZC_PurReqToPoQuery",
        "chartAnnotationPath": "com.sap.vocabularies.UI.v1.Chart#ConversionParGroupe",
        "dataPointAnnotationPath": "com.sap.vocabularies.UI.v1.DataPoint#PostesEnAttente",
        "selectionAnnotationPath": "com.sap.vocabularies.UI.v1.SelectionVariant#ToutesDA",
        "presentationAnnotationPath": "com.sap.vocabularies.UI.v1.PresentationVariant#ParGroupe"
      }
    }
  }
}
```

> **📸 COPIE D'ÉCRAN N°14** — VS Code : section de configuration de l'Overview Page
> *Remplacer cette ligne par :* `![Copie 14](images/OVP-COCKPIT/capture-14.png)`

| Paramètre | Rôle |
|---|---|
| `globalFilterEntityType` | Entité alimentant la barre de filtres du cockpit |
| `containerLayout` | Disposition des cartes ; redimensionnable recommandé |
| `enableLiveFilter` | Actualisation immédiate à la saisie d'un filtre |
| `template` | Type de carte |
| `entitySet` | Jeu d'entités interrogé par la carte |
| `chartAnnotationPath` | Référence vers l'annotation de graphique |
| `dataPointAnnotationPath` | Référence vers l'annotation d'indicateur |
| `selectionAnnotationPath` | Filtre propre à la carte |
| `presentationAnnotationPath` | Tri et visualisation |

> **ℹ️ Chemins d'annotation**
> Ils reprennent le qualificateur défini en partie A, précédé du nom complet du terme d'annotation. Une erreur de frappe ne produit aucun message : la carte reste vide. C'est la cause la plus fréquente de cartes blanches.

---

### B4 — Ajouter une carte liste

La carte liste affiche les objets à traiter. Elle s'appuie sur l'annotation de poste de liste du modèle et peut porter sa propre variante de sélection.

```json
"card02_liste_attente": {
  "model": "mainModel",
  "template": "sap.ovp.cards.list",
  "settings": {
    "title": "Demandes d'achat à traiter",
    "subTitle": "Non converties en commande",
    "listType": "extended",
    "listFlavor": "standard",
    "entitySet": "ZC_PurReqToPoQuery",
    "annotationPath": "com.sap.vocabularies.UI.v1.LineItem",
    "selectionAnnotationPath": "com.sap.vocabularies.UI.v1.SelectionVariant#ToutesDA",
    "identificationAnnotationPath": "com.sap.vocabularies.UI.v1.Identification"
  }
}
```

> **📸 COPIE D'ÉCRAN N°15** — Navigateur : carte liste des demandes d'achat à traiter
> *Remplacer cette ligne par :* `![Copie 15](images/OVP-COCKPIT/capture-15.png)`

---

### B5 — Exécuter en local

1. Lancer l'application en local depuis VS Code.
2. Vérifier l'affichage de chaque carte et la cohérence des chiffres.
3. Tester la barre de filtres globale et son effet sur les cartes.
4. Corriger les chemins d'annotation des cartes restées vides.
5. Contrôler le comportement sur une largeur d'écran réduite.

> **📸 COPIE D'ÉCRAN N°16** — Navigateur : cockpit complet exécuté en local
> *Remplacer cette ligne par :* `![Copie 16](images/OVP-COCKPIT/capture-16.png)`

> **📸 COPIE D'ÉCRAN N°17** — Navigateur : effet du filtre global sur les cartes
> *Remplacer cette ligne par :* `![Copie 17](images/OVP-COCKPIT/capture-17.png)`

---

## Partie C — Navigation vers les applications

### C1 — Principe de la navigation intentionnelle

Une application ne désigne jamais sa cible par une URL. Elle exprime une **intention** : un objet sémantique et une action. Le Launchpad résout cette intention en consultant les target mappings des catalogues affectés à l'utilisateur, et ouvre l'application correspondante.

| Conséquence | Implication pratique |
|---|---|
| La cible est résolue à l'exécution | Le cockpit ne dépend pas de l'URL de la cible |
| La résolution dépend du rôle | Un lien invisible signale un target mapping non affecté |
| Le protocole de la cible est indifférent | Une cible en V2 ou en V4 se déclare de la même façon |
| Une transaction est une cible comme une autre | ME59N s'atteint par une intention, pas par une URL |

---

### C2 — Créer la carte d'accès rapides

La carte de liens est le point d'entrée vers les applications de traitement. Chaque entrée déclare l'objet sémantique et l'action de sa cible.

```json
"card03_acces_rapides": {
  "model": "mainModel",
  "template": "sap.ovp.cards.linklist",
  "settings": {
    "title": "Accès rapides",
    "listFlavor": "standard",
    "staticContent": [
      {
        "title": "Suivi des demandes d'achat",
        "subTitle": "Application de suivi",
        "imageUri": "sap-icon://cart",
        "semanticObject": "PurReqTracking",
        "action": "manage"
      },
      {
        "title": "Conversion automatique en commandes",
        "subTitle": "Transaction ME59N",
        "imageUri": "sap-icon://sales-order",
        "semanticObject": "PurchaseRequisition",
        "action": "convertAuto"
      },
      {
        "title": "Traitement des demandes d'achat",
        "subTitle": "Application standard",
        "imageUri": "sap-icon://documents",
        "semanticObject": "PurchaseRequisition",
        "action": "process"
      }
    ]
  }
}
```

> **📸 COPIE D'ÉCRAN N°18** — Navigateur : carte d'accès rapides et ses trois entrées
> *Remplacer cette ligne par :* `![Copie 18](images/OVP-COCKPIT/capture-18.png)`

> **ℹ️ Objets sémantiques**
> Reprendre exactement ceux déclarés dans les target mappings des applications cibles. Pour l'application de suivi, la valeur figure dans son propre target mapping. Pour les applications standard, elle se relève dans le Launchpad Designer.

---

### C3 — Créer le target mapping vers ME59N

#### Principe

Une transaction classique s'expose dans le Launchpad par un target mapping de type transaction. Le Launchpad l'ouvre alors dans un conteneur, sans que l'utilisateur ait à connaître le code de transaction.

#### Mode opératoire

1. Lancer la transaction `/UI2/FLPD_CUST` et ouvrir le catalogue Z du projet.
2. Créer un target mapping et renseigner l'objet sémantique et l'action retenus pour la conversion automatique.
3. Choisir le type d'application « Transaction ».
4. Renseigner le code de transaction `ME59N`.
5. Renseigner l'alias système correspondant au système où s'exécute la transaction.
6. Enregistrer et vérifier que le target mapping apparaît dans le catalogue.
7. Créer éventuellement une tuile associée, si la transaction doit aussi être accessible directement.

> **📸 COPIE D'ÉCRAN N°19** — Launchpad Designer : création du target mapping de type transaction
> *Remplacer cette ligne par :* `![Copie 19](images/OVP-COCKPIT/capture-19.png)`

> **📸 COPIE D'ÉCRAN N°20** — Launchpad Designer : code de transaction et alias système renseignés
> *Remplacer cette ligne par :* `![Copie 20](images/OVP-COCKPIT/capture-20.png)`

> **⚠️ Autorisations**
> Le target mapping rend la transaction atteignable, il n'autorise personne à l'exécuter. L'utilisateur doit disposer de l'autorisation de transaction correspondante dans son rôle, faute de quoi il obtiendra un refus à l'ouverture.

---

### C4 — Navigation depuis une carte de données

Au-delà de la carte de liens, une carte analytique ou liste peut elle-même mener vers une application. La cible se déclare dans les annotations du modèle, au moyen d'une annotation d'identification de type navigation intentionnelle.

```abap
@Metadata.layer: #CORE

annotate view ZC_PurReqToPoQuery with
{
  @UI.identification: [ { position: 10,
                          type: #WITH_INTENT_BASED_NAVIGATION,
                          semanticObject: 'PurReqTracking',
                          semanticObjectAction: 'manage',
                          label: 'Ouvrir le suivi des DA' } ]
  PurchasingGroup;
}
```

> **📸 COPIE D'ÉCRAN N°21** — Navigateur : navigation depuis une carte vers l'application de suivi
> *Remplacer cette ligne par :* `![Copie 21](images/OVP-COCKPIT/capture-21.png)`

---

### C5 — Tester la résolution des intentions

1. Ouvrir le Launchpad avec l'utilisateur de test.
2. Vérifier que les applications cibles sont accessibles individuellement depuis leurs propres tuiles.
3. Ouvrir le cockpit et tester chaque entrée de la carte d'accès rapides.
4. Vérifier le retour au cockpit après ouverture d'une cible.
5. Recommencer avec un utilisateur ne disposant pas des catalogues cibles, et constater que les liens ne sont pas proposés.

> **📸 COPIE D'ÉCRAN N°22** — Launchpad : ouverture de l'application de suivi depuis le cockpit
> *Remplacer cette ligne par :* `![Copie 22](images/OVP-COCKPIT/capture-22.png)`

> **📸 COPIE D'ÉCRAN N°23** — Launchpad : ouverture de ME59N depuis le cockpit
> *Remplacer cette ligne par :* `![Copie 23](images/OVP-COCKPIT/capture-23.png)`

> **ℹ️ Point de vigilance**
> Si la navigation vers l'application de suivi échoue alors que le reste fonctionne, vérifier d'abord que cette application s'ouvre depuis sa propre tuile. Un blocage sur son protocole d'exposition se manifeste au même endroit qu'une erreur de navigation, sans être de même nature.

---

## Partie D — Déploiement et publication

### D1 — Configurer le déploiement

Le déploiement s'appuie sur une configuration dédiée, qui désigne le système cible, le repository SAPUI5, le package et l'ordre de transport.

#### Fichier de configuration

```yaml
builder:
  resources:
    excludes:
      - "/test/**"
      - "/localService/**"
  customTasks:
    - name: deploy-to-abap
      afterTask: replaceVersion
      configuration:
        target:
          url: https://<hote>:<port>
          client: "050"
        app:
          name: ZOVP_PR_COCKPIT
          description: Cockpit demandes d'achat
          package: Z_MM_PUR_COCKPIT
          transport: <ordre_de_transport>
        exclude:
          - /test/
```

#### Commandes

```json
{
  "scripts": {
    "deploy": "npm run build && fiori deploy --config ui5-deploy.yaml",
    "deploy-test": "npm run build && fiori deploy --config ui5-deploy.yaml --testMode true"
  }
}
```

> **📸 COPIE D'ÉCRAN N°24** — VS Code : fichier de configuration du déploiement
> *Remplacer cette ligne par :* `![Copie 24](images/OVP-COCKPIT/capture-24.png)`

> **ℹ️ Essai à blanc**
> Exécuter d'abord le déploiement en mode test. Il valide la connexion, le package et l'ordre de transport sans rien écrire dans le système.

---

### D2 — Déployer et activer

1. Lancer le déploiement en mode test et contrôler le journal.
2. Lancer le déploiement réel.
3. Vérifier la création de l'application BSP dans le système.
4. Activer le nœud ICF de l'application via la transaction `SICF`.
5. Appeler directement l'application par son URL et contrôler son chargement.

> **📸 COPIE D'ÉCRAN N°25** — VS Code : journal de déploiement
> *Remplacer cette ligne par :* `![Copie 25](images/OVP-COCKPIT/capture-25.png)`

> **📸 COPIE D'ÉCRAN N°26** — Transaction `SE80` : application BSP du cockpit
> *Remplacer cette ligne par :* `![Copie 26](images/OVP-COCKPIT/capture-26.png)`

> **📸 COPIE D'ÉCRAN N°27** — Transaction `SICF` : nœud ICF du cockpit activé
> *Remplacer cette ligne par :* `![Copie 27](images/OVP-COCKPIT/capture-27.png)`

---

### D3 — Publier la tuile du cockpit

1. Lancer la transaction `/UI2/FLPD_CUST` et ouvrir le catalogue Z du projet.
2. Créer la tuile du cockpit avec titre, sous-titre et icône.
3. Créer le target mapping en reprenant l'objet sémantique et l'action déclarés dans le descripteur.
4. Renseigner l'identifiant du composant SAPUI5 et l'URL de l'application.
5. Vérifier que le catalogue contient bien la tuile du cockpit, le target mapping du cockpit et celui de la transaction.

> **📸 COPIE D'ÉCRAN N°28** — Launchpad Designer : tuile et target mapping du cockpit
> *Remplacer cette ligne par :* `![Copie 28](images/OVP-COCKPIT/capture-28.png)`

> **📸 COPIE D'ÉCRAN N°29** — Launchpad Designer : contenu complet du catalogue du projet
> *Remplacer cette ligne par :* `![Copie 29](images/OVP-COCKPIT/capture-29.png)`

---

### D4 — Affecter les rôles

1. Ouvrir le rôle PFCG du projet et ajouter le catalogue du cockpit.
2. Ajouter également les catalogues des applications cibles, sans lesquels la navigation ne se résout pas.
3. Ajouter le groupe ou l'espace contenant la tuile du cockpit.
4. Compléter les autorisations : exécution du service analytique, transaction ME59N, autorisations achat.
5. Générer le profil et affecter le rôle à l'utilisateur de test.
6. Invalider les caches du Launchpad.

| Autorisation | Objet |
|---|---|
| Exécution du service analytique | `S_SERVICE` sur le service OData V2 du cockpit |
| Exécution de la transaction | `S_TCODE` pour ME59N |
| Données d'achat | Objets du domaine achat selon le périmètre organisationnel |
| Navigation vers les cibles | Catalogues des applications cibles affectés au rôle |

> **📸 COPIE D'ÉCRAN N°30** — Transaction `PFCG` : catalogues et autorisations du rôle du cockpit
> *Remplacer cette ligne par :* `![Copie 30](images/OVP-COCKPIT/capture-30.png)`

> **📸 COPIE D'ÉCRAN N°31** — Launchpad : tuile du cockpit visible et cockpit ouvert sur données réelles
> *Remplacer cette ligne par :* `![Copie 31](images/OVP-COCKPIT/capture-31.png)`

---

## Partie E — Recette, transport et exploitation

### E1 — Fiche de recette

| N° | Point de contrôle | Résultat attendu | OK / KO |
|---|---|---|---|
| 1 | Cube activé et cohérent | Conforme | ☐ |
| 2 | Requête analytique activée | Active | ☐ |
| 3 | Annotations d'indicateur activées | Actives | ☐ |
| 4 | Service OData V2 enregistré et actif | Statut vert | ☐ |
| 5 | Service accessible par la route des utilisateurs | HTTP 200 | ☐ |
| 6 | Chiffres conformes à une extraction de contrôle | Conformes | ☐ |
| 7 | Toutes les cartes affichent des données | Aucune carte vide | ☐ |
| 8 | Seuils de criticité et couleurs | Conformes | ☐ |
| 9 | Filtre global actif sur toutes les cartes | Conforme | ☐ |
| 10 | Carte d'accès rapides complète | Trois entrées visibles | ☐ |
| 11 | Navigation vers l'application de suivi | Fonctionnelle | ☐ |
| 12 | Navigation vers ME59N | Fonctionnelle | ☐ |
| 13 | Retour au cockpit après navigation | Conforme | ☐ |
| 14 | Tuile du cockpit visible | Visible | ☐ |
| 15 | Comportement sur écran réduit | Conforme | ☐ |
| 16 | Temps de chargement acceptable sur volume réel | Conforme | ☐ |
| 17 | Comportement avec un utilisateur d'un autre périmètre | Conforme | ☐ |

> **📸 COPIE D'ÉCRAN N°32** — Synthèse de recette : cockpit complet en fonctionnement
> *Remplacer cette ligne par :* `![Copie 32](images/OVP-COCKPIT/capture-32.png)`

### E2 — Transport

| Objet | Mode de propagation |
|---|---|
| Vues CDS et extensions de métadonnées | Ordre de workbench |
| Service OData généré | Ordre de workbench ; enregistrement à contrôler dans le système cible |
| Application déployée | Ordre de workbench ; activation du nœud ICF manuelle |
| Catalogue, tuiles et target mappings | Ordre de customizing |
| Rôle PFCG | Ordre de customizing |

> **⚠️ Après import**
> Activer les nœuds ICF, contrôler l'enregistrement du service, invalider les caches, puis rejouer la fiche de recette. Vérifier en particulier que les target mappings des applications cibles existent aussi dans le système d'arrivée, faute de quoi la navigation échouera.

### E3 — Exploitation

| Sujet | Point de vigilance |
|---|---|
| Performance | Contrôler le temps de réponse des cartes sur le volume de production |
| Volume de données | Limiter la portée des cartes par des variantes de sélection |
| Évolution des vues standard | Retester le cube après montée de version |
| Cohérence des chiffres | Rapprocher périodiquement d'une extraction de contrôle |
| Navigation | Revalider après toute évolution des catalogues cibles |

---

## Annexe A — Diagnostic des incidents fréquents

| Symptôme | Cause probable | Action corrective |
|---|---|---|
| Une carte reste vide | Chemin d'annotation erroné dans le descripteur | Comparer caractère par caractère avec le qualificateur défini en CDS |
| Toutes les cartes sont vides | Service inaccessible ou entité erronée | Tester le service seul dans le navigateur |
| Le cockpit ne se charge pas après déploiement | Version SAPUI5 incompatible | Contrôler la version minimale déclarée dans le descripteur |
| Les chiffres sont faux | Agrégation ou formule incorrecte | Contrôler l'aperçu de données de la requête |
| Aucune couleur sur les indicateurs | Seuils de criticité absents ou incohérents | Compléter le calcul de criticité |
| Un lien d'accès rapide est absent | Catalogue cible non affecté au rôle | Ajouter le catalogue de l'application cible |
| La navigation ouvre une page vide | Application cible inaccessible | Tester la cible depuis sa propre tuile |
| ME59N refuse l'ouverture | Autorisation de transaction manquante | Compléter le rôle |
| Le filtre global est sans effet | Entité de filtre non déclarée | Renseigner l'entité de filtre global |
| Le cockpit est lent | Volume trop important | Restreindre par variante de sélection |

---

## Annexe B — Index des copies d'écran

| N° | Chapitre | Contenu attendu |
|---|---|---|
| 01 | Chapitre 3 | Vérification préalable de la route OData V2 |
| 02 à 03 | Étape A1 | Vue cube et aperçu de données |
| 04 | Étape A2 | Vue de requête analytique |
| 05 | Étape A3 | Annotations des indicateurs |
| 06 | Étape A4 | Service enregistré |
| 07 à 08 | Étape A5 | Tests du service |
| 09 à 12 | Étape B1 | Génération du projet |
| 13 | Étape B2 | Descripteur de l'application |
| 14 | Étape B3 | Configuration des cartes |
| 15 | Étape B4 | Carte liste |
| 16 à 17 | Étape B5 | Exécution locale |
| 18 | Étape C2 | Carte d'accès rapides |
| 19 à 20 | Étape C3 | Target mapping vers ME59N |
| 21 | Étape C4 | Navigation depuis une carte |
| 22 à 23 | Étape C5 | Tests de navigation |
| 24 | Étape D1 | Configuration du déploiement |
| 25 à 27 | Étape D2 | Déploiement et activation |
| 28 à 29 | Étape D3 | Tuile et catalogue |
| 30 à 31 | Étape D4 | Rôle et cockpit en fonctionnement |
| 32 | Partie E | Synthèse de recette |

---

## Annexe C — Historique des versions

| Version | Date | Auteur | Nature des modifications |
|---|---|---|---|
| 1.0 | … | … | Création du document |
| | | | |
