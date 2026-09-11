# Mode opératoire — Annotations RAP et impact sur le front-end

## Application de suivi des demandes d'achat

**Référence des annotations · Domaine de statut et liste déroulante**
**SAP S/4HANA (Private Cloud and On-Premise) 2021 — FPS02**

| Attribut | Valeur |
|---|---|
| Référence du document | MO-RAP-ANNOT-2021FPS02-v1.0 |
| Documents liés | [MO Service RAP](MO%20Service%20RAP.md) · [MO Annotations CDS](MO%20Annotations%20CDS.md) |
| Objet concerné | `ZI_PurReqTracking` / `ZC_PurReqTracking` |
| Protocole | OData V4, SAP Fiori elements |
| Outil | Eclipse + ABAP Development Tools |
| Auteur | … |
| Vérifié par | … |
| Date de rédaction | … |
| Statut | Version de travail |

> **Convention de ce document**
> Chaque emplacement de copie d'écran est signalé par un bloc `📸 COPIE D'ÉCRAN N°XX`.
> Déposez vos images dans `images/RAP-ANNOT/` puis remplacez la ligne indiquée par le lien Markdown correspondant.

---

## Sommaire

- [1. Objet et principe](#1-objet-et-principe)
- [2. Où placer quelle annotation](#2-où-placer-quelle-annotation)
- [3. Référence des annotations par effet visuel](#3-référence-des-annotations-par-effet-visuel)
  - [3.1 Libellés](#31-libellés)
  - [3.2 Visibilité](#32-visibilité)
  - [3.3 Colonnes de la liste](#33-colonnes-de-la-liste)
  - [3.4 En-tête de la page objet](#34-en-tête-de-la-page-objet)
  - [3.5 Structure de la page objet](#35-structure-de-la-page-objet)
  - [3.6 Filtres et variantes](#36-filtres-et-variantes)
  - [3.7 Criticité et couleurs](#37-criticité-et-couleurs)
  - [3.8 Textes associés](#38-textes-associés)
  - [3.9 Aides à la saisie](#39-aides-à-la-saisie)
  - [3.10 Sémantique des données](#310-sémantique-des-données)
  - [3.11 Actions](#311-actions)
  - [3.12 Recherche](#312-recherche)
- [4. Ce qui ne relève pas des annotations](#4-ce-qui-ne-relève-pas-des-annotations)
- [5. Mise en œuvre : domaine de statut et liste déroulante](#5-mise-en-œuvre--domaine-de-statut-et-liste-déroulante)
  - [5.1 Créer le domaine](#51-créer-le-domaine)
  - [5.2 Créer l'élément de données](#52-créer-lélément-de-données)
  - [5.3 Adapter la table de persistance](#53-adapter-la-table-de-persistance)
  - [5.4 Créer la vue d'aide à la saisie](#54-créer-la-vue-daide-à-la-saisie)
  - [5.5 Adapter la vue d'interface](#55-adapter-la-vue-dinterface)
  - [5.6 Adapter la vue de projection](#56-adapter-la-vue-de-projection)
  - [5.7 Adapter l'extension de métadonnées](#57-adapter-lextension-de-métadonnées)
  - [5.8 Adapter la behavior definition](#58-adapter-la-behavior-definition)
  - [5.9 Adapter l'implémentation](#59-adapter-limplémentation)
  - [5.10 Activer et tester](#510-activer-et-tester)
- [6. Fiche de recette](#6-fiche-de-recette)
- [Annexe A — Diagnostic](#annexe-a--diagnostic)
- [Annexe B — Index des copies d'écran](#annexe-b--index-des-copies-décran)
- [Annexe C — Historique des versions](#annexe-c--historique-des-versions)

---

## 1. Objet et principe

Ce document a deux parties.

La première est une **référence** : quelles annotations sont disponibles sur un objet métier RAP, où les écrire, et ce qu'elles changent concrètement à l'écran. Elle sert de catalogue à consulter quand on cherche à obtenir un comportement précis.

La seconde est une **mise en œuvre complète** : remplacer le statut de suivi, aujourd'hui saisi librement, par un domaine à quatre valeurs proposées dans une liste déroulante. Elle traverse toutes les couches, du dictionnaire à l'écran.

> **ℹ️ Principe de fond**
> Dans une application Fiori elements, l'interface n'est pas codée : elle est décrite. Le modèle de données porte la structure, la behavior definition porte le comportement, les annotations portent la présentation. Trois responsabilités distinctes — confondre les deux dernières est la source d'erreur la plus fréquente.

---

## 2. Où placer quelle annotation

Trois emplacements possibles, avec une règle de priorité claire.

| Emplacement | Portée | Quand l'utiliser |
|---|---|---|
| Élément de données du dictionnaire | Toutes les vues, toutes les applications | Libellé d'un champ réutilisé partout |
| Vue CDS d'interface ou de projection | Cette vue et ses consommateurs | Sémantique, associations, textes |
| Extension de métadonnées | Cette vue uniquement | Toute la présentation |

**La règle à retenir** : tout ce qui relève de l'affichage va dans l'extension de métadonnées. Le modèle reste lisible, et les évolutions d'écran n'obligent jamais à toucher aux vues.

Deux exceptions justifiées :

- `@Consumption.valueHelpDefinition` peut être refusée dans une extension de métadonnées selon le niveau de support. Dans ce cas, elle se place dans la vue de projection.
- `@ObjectModel.*` et `@Semantics.*` décrivent la nature de la donnée, pas sa présentation. Leur place est dans la vue.

> **⚠️ Syntaxe**
> Utiliser `ANNOTATE VIEW` et non `ANNOTATE ENTITY` : la seconde forme accepte un jeu d'annotations plus restreint et fait échouer certaines annotations d'interface sans message explicite.

---

## 3. Référence des annotations par effet visuel

### 3.1 Libellés

| Annotation | Emplacement | Effet |
|---|---|---|
| `@EndUserText.label` | Élément CDS | Libellé général du champ, tous écrans |
| `label` dans `@UI.lineItem` | Extension | Libellé de la colonne uniquement |
| `label` dans `@UI.identification` | Extension | Libellé dans la page objet uniquement |
| Libellé de l'élément de données | Dictionnaire | Libellé par défaut, toutes vues confondues |

```abap
@EndUserText.label: 'Statut de suivi'
@UI: { lineItem:       [ { position: 30, label: 'Statut' } ],
       identification: [ { position: 30, label: 'Statut de suivi' } ] }
TrackingStatus;
```

> Un champ typé avec un type intégré (`abap.char`, `abap.dats`) n'a **aucun** libellé au niveau du dictionnaire. Sans annotation, il s'affiche sans étiquette.

---

### 3.2 Visibilité

| Annotation | Effet |
|---|---|
| `@UI.hidden: true` | Masque le champ dans tous les écrans |
| `@UI.hidden: 'ChampBooleen'` | Masquage conditionnel selon un champ du modèle |
| `@Consumption.hidden: true` | Retire le champ du service exposé |
| `@UI.lineItem: [ ]` | Retire la colonne sans masquer le champ ailleurs |

```abap
@UI.hidden: true
TrackingUUID;
```

La distinction compte : `@UI.hidden` masque à l'écran mais le champ reste dans le service et reste interrogeable. `@Consumption.hidden` le retire du modèle exposé.

---

### 3.3 Colonnes de la liste

| Propriété | Valeurs | Effet |
|---|---|---|
| `position` | Entier | Ordre des colonnes |
| `label` | Texte | En-tête de colonne |
| `importance` | `#HIGH` `#MEDIUM` `#LOW` | Ordre de disparition sur écran étroit |
| `criticality` | Nom de champ | Couleur de la valeur |
| `type` | `#AS_CHART`, `#FOR_ACTION`… | Nature de la cellule |

```abap
@UI.lineItem: [ { position: 10, importance: #HIGH } ]
PurchaseRequisition;
```

> **📸 COPIE D'ÉCRAN N°01** — Liste avec colonnes ordonnées et hiérarchisées
> *Remplacer cette ligne par :* `![Copie 01](images/RAP-ANNOT/capture-01.png)`

---

### 3.4 En-tête de la page objet

```abap
@UI.headerInfo: {
  typeName:       'Suivi de DA',
  typeNamePlural: 'Suivis de DA',
  title:          { type: #STANDARD, value: 'PurchaseRequisition' },
  description:    { value: 'PurReqItemText' },
  imageUrl:       'IconUrl'
}
```

| Clé | Effet |
|---|---|
| `typeName` | Nom du type d'objet, au singulier |
| `typeNamePlural` | Titre de la liste |
| `title` | Titre de la page objet |
| `description` | Sous-titre |
| `imageUrl` | Vignette, si le modèle porte une URL |

---

### 3.5 Structure de la page objet

Trois niveaux emboîtés : les facettes découpent la page en sections, les groupes de champs regroupent les éléments, l'identification porte la section principale.

```abap
@UI.facet: [ { id: 'General', purpose: #STANDARD,
               type: #IDENTIFICATION_REFERENCE,
               label: 'Informations générales', position: 10 },
             { id: 'Suivi', purpose: #STANDARD,
               type: #FIELDGROUP_REFERENCE,
               targetQualifier: 'GroupeSuivi',
               label: 'Suivi', position: 20 } ]
```

| Type de facette | Contenu |
|---|---|
| `#IDENTIFICATION_REFERENCE` | Champs annotés `@UI.identification` |
| `#FIELDGROUP_REFERENCE` | Champs d'un groupe, par qualificateur |
| `#LINEITEM_REFERENCE` | Table d'entités liées |
| `#COLLECTION` | Regroupement de plusieurs facettes |

> **📸 COPIE D'ÉCRAN N°02** — Page objet structurée en sections
> *Remplacer cette ligne par :* `![Copie 02](images/RAP-ANNOT/capture-02.png)`

---

### 3.6 Filtres et variantes

| Annotation | Niveau | Effet |
|---|---|---|
| `@UI.selectionField` | Élément | Champ dans la barre de filtres |
| `@UI.selectionVariant` | Entité | Filtre appliqué par défaut |
| `@UI.presentationVariant` | Entité | Tri et mode de restitution par défaut |

```abap
@UI.presentationVariant: [ { sortOrder: [ { by: 'TargetDate', direction: #ASC } ],
                             visualizations: [ { type: #AS_LINEITEM } ] } ]
```

---

### 3.7 Criticité et couleurs

La criticité colore une valeur selon un code numérique porté par un autre champ du modèle.

| Valeur | Rendu |
|---|---|
| 0 | Neutre |
| 1 | Négatif — rouge |
| 2 | Critique — orange |
| 3 | Positif — vert |

```abap
@UI.lineItem: [ { position: 30, criticality: 'StatusCriticality' } ]
TrackingStatus;
```

> Une extension de métadonnées ne peut pas créer d'élément. Le champ de criticité doit exister dans la vue — il se calcule dans la vue d'interface.

---

### 3.8 Textes associés

```abap
@ObjectModel.text.element: [ 'TrackingStatusName' ]
@UI.textArrangement: #TEXT_ONLY
TrackingStatus;
```

| Disposition | Rendu |
|---|---|
| `#TEXT_FIRST` | Texte puis code entre parenthèses |
| `#TEXT_LAST` | Code puis texte entre parenthèses |
| `#TEXT_ONLY` | Texte seul — le code reste stocké |
| `#TEXT_SEPARATE` | Deux colonnes distinctes |

Pour un statut, `#TEXT_ONLY` est presque toujours le bon choix : l'utilisateur n'a aucune raison de voir le code technique.

---

### 3.9 Aides à la saisie

C'est ici que se joue le choix entre liste déroulante et boîte de dialogue.

| Annotation | Emplacement | Effet |
|---|---|---|
| `@Consumption.valueHelpDefinition` | Vue de projection | Déclare l'aide à la saisie |
| `@ObjectModel.resultSet.sizeCategory` | Vue d'aide | **Détermine le rendu** |
| `@ObjectModel.dataCategory: #TEXT` | Vue d'aide | Déclare une vue de texte |
| `@ObjectModel.representativeKey` | Vue d'aide | Désigne la clé métier |

| Taille déclarée | Rendu obtenu |
|---|---|
| `#XS` ou `#S` | **Liste déroulante** |
| `#M`, `#L`, `#XL` | Boîte de dialogue de recherche |

> **C'est la clé de la demande.** Une liste déroulante ne s'obtient pas par une annotation dédiée, mais en déclarant que l'ensemble de valeurs est très petit. Le framework en déduit le rendu approprié.

---

### 3.10 Sémantique des données

Ces annotations décrivent la nature de la donnée. Elles vont dans la vue, pas dans l'extension.

| Annotation | Effet |
|---|---|
| `@Semantics.amount.currencyCode` | Formatage monétaire, alignement, décimales de la devise |
| `@Semantics.quantity.unitOfMeasure` | Formatage des quantités |
| `@Semantics.user.createdBy` | Champ reconnu comme auteur de création |
| `@Semantics.systemDateTime.lastChangedAt` | Champ reconnu comme horodatage |
| `@Semantics.text: true` | Champ reconnu comme texte descriptif |
| `@Semantics.language: true` | Clé de langue d'une vue de texte |

Omettre l'annotation de devise sur un montant produit un affichage sans devise et un mauvais nombre de décimales.

---

### 3.11 Actions

```abap
@UI.lineItem: [ { position: 90,
                  type: #FOR_ACTION,
                  dataAction: 'markAsCompleted',
                  label: 'Clôturer' } ]
```

| Emplacement | Rendu |
|---|---|
| `@UI.lineItem` avec `#FOR_ACTION` | Bouton dans la barre de la liste |
| `@UI.identification` avec `#FOR_ACTION` | Bouton dans l'en-tête de la page objet |
| `@UI.lineItem` avec `inline: true` | Bouton dans chaque ligne |

La **disponibilité** du bouton — actif ou grisé — ne vient pas des annotations mais de la behavior definition, par les caractéristiques d'instance.

---

### 3.12 Recherche

```abap
@Search.searchable: true          -- niveau entité

@Search.defaultSearchElement: true
@Search.fuzzinessThreshold: 0.8
Comments;
```

Le seuil de tolérance accepte les fautes de frappe. Au-dessus de 0,9 la recherche devient stricte, en dessous de 0,7 elle devient bruyante.

---

## 4. Ce qui ne relève pas des annotations

Confusion fréquente : ces comportements viennent de la **behavior definition**, jamais des annotations.

| Besoin | Où le déclarer |
|---|---|
| Champ obligatoire | `field ( mandatory )` |
| Champ en lecture seule | `field ( readonly )` |
| Champ obligatoire selon contexte | `field ( features : instance )` |
| Action activée ou grisée | Caractéristiques d'instance |
| Contrôle bloquant | `validation` |
| Valeur par défaut | `determination` |
| Droit de création ou de modification | `authorization` |

> **⚠️ Conséquence pratique**
> Un champ déclaré `readonly` dans la behavior definition n'apparaîtra jamais en saisie, quelles que soient les annotations. Avant de chercher une annotation qui n'existe pas, vérifier la behavior definition.

---

## 5. Mise en œuvre : domaine de statut et liste déroulante

**Objectif** : remplacer le statut saisi librement par quatre valeurs proposées dans une liste déroulante à la création et à la modification.

| Code | Libellé | Criticité |
|---|---|---|
| `OP` | Ouvert | 2 — orange |
| `IP` | En cours de traitement | 2 — orange |
| `CL` | Clôturé | 3 — vert |
| `CA` | Annulé | 0 — neutre |

### Objets à créer ou modifier

| Objet | Action |
|---|---|
| `ZZ_PR_TRACK_STATUS` | Domaine à créer |
| `ZZ_PR_TRACK_STATUS` | Élément de données à créer |
| `ZMM_PR_TRACK` | Table à modifier |
| `ZI_PurReqTrackingStatusVH` | Vue d'aide à la saisie à créer |
| `ZI_PurReqTracking` | Vue d'interface à modifier |
| `ZC_PurReqTracking` | Vue de projection à modifier |
| `ZC_PurReqTracking` | Extension de métadonnées à modifier |
| `ZI_PurReqTracking` | Behavior definition à modifier |
| `ZBP_I_PurReqTracking` | Implémentation à modifier |

---

### 5.1 Créer le domaine

1. Lancer la transaction `SE11`, ou créer l'objet depuis Eclipse.
2. Créer le domaine `ZZ_PR_TRACK_STATUS`, type `CHAR`, longueur `2`.
3. Ouvrir l'onglet **Domaine de valeurs**.
4. Saisir les quatre valeurs fixes avec leurs libellés.
5. Activer et affecter au package et à l'ordre de transport.

| Valeur | Description |
|---|---|
| `OP` | Ouvert |
| `IP` | En cours de traitement |
| `CL` | Clôturé |
| `CA` | Annulé |

> **📸 COPIE D'ÉCRAN N°03** — Transaction `SE11` : domaine et ses quatre valeurs fixes
> *Remplacer cette ligne par :* `![Copie 03](images/RAP-ANNOT/capture-03.png)`

> **ℹ️ Pourquoi des valeurs fixes plutôt qu'une table de contrôle**
> Quatre valeurs stables, techniques, non paramétrables par le métier : les valeurs fixes du domaine sont l'outil adapté. Si le métier doit pouvoir ajouter des statuts sans développement, il faut au contraire une table de contrôle avec sa table de textes.

---

### 5.2 Créer l'élément de données

1. Créer l'élément de données `ZZ_PR_TRACK_STATUS` rattaché au domaine.
2. Renseigner les quatre libellés : court, moyen, long, en-tête.
3. Activer.

C'est ce qui donnera un libellé au champ dans tous les écrans, sans annotation supplémentaire.

> **📸 COPIE D'ÉCRAN N°04** — Transaction `SE11` : élément de données et ses libellés
> *Remplacer cette ligne par :* `![Copie 04](images/RAP-ANNOT/capture-04.png)`

---

### 5.3 Adapter la table de persistance

Remplacer le type intégré par l'élément de données, dans la table de base **et** dans la table de draft.

```abap
define table zmm_pr_track {
  key client            : abap.clnt not null;
  key tracking_uuid     : sysuuid_x16 not null;
  purchase_req          : banfn;
  purchase_req_item     : bnfpo;
  tracking_status       : zz_pr_track_status;
  comments              : abap.char(250);
  target_date           : abap.dats;
  created_by            : abp_creation_user;
  created_at            : abp_creation_tstmpl;
  last_changed_by       : abp_lastchange_user;
  last_changed_at       : abp_lastchange_tstmpl;
  local_last_changed_at : abp_locinst_lastchange_tstmpl;
}
```

> **⚠️ Conversion de table**
> Changer le type d'une colonne existante impose une conversion. Sur un environnement de développement, vider la table avant activation est plus simple. Les deux tables — base et draft — doivent rester strictement alignées, faute de quoi la behavior definition ne s'active plus.

> **📸 COPIE D'ÉCRAN N°05** — Eclipse ADT : table modifiée et activée
> *Remplacer cette ligne par :* `![Copie 05](images/RAP-ANNOT/capture-05.png)`

---

### 5.4 Créer la vue d'aide à la saisie

Cette vue lit les textes des valeurs fixes du domaine dans le dictionnaire. Elle porte les annotations qui déclenchent le rendu en liste déroulante.

```abap
@AbapCatalog.sqlViewName: 'ZIPRTRKSTATVH'
@AbapCatalog.compiler.compareFilter: true
@AccessControl.authorizationCheck: #NOT_REQUIRED
@EndUserText.label: 'Statuts de suivi - aide à la saisie'

@ObjectModel.dataCategory: #TEXT
@ObjectModel.representativeKey: 'TrackingStatus'
@ObjectModel.resultSet.sizeCategory: #XS
@ObjectModel.usageType: { serviceQuality: #A,
                          sizeCategory:   #XS,
                          dataClass:      #CUSTOMIZING }
@Search.searchable: true

define view ZI_PurReqTrackingStatusVH
  as select from dd07t
{
      @ObjectModel.text.element: [ 'TrackingStatusName' ]
      @Search.defaultSearchElement: true
      @EndUserText.label: 'Statut'
  key cast( domvalue_l as zz_pr_track_status preserving type ) as TrackingStatus,

      @Semantics.language: true
  key ddlanguage                                               as Language,

      @Semantics.text: true
      @EndUserText.label: 'Désignation du statut'
      ddtext                                                   as TrackingStatusName
}
where domname  = 'ZZ_PR_TRACK_STATUS'
  and as4local = 'A'
  and as4vers  = '0000'
```

| Annotation | Rôle |
|---|---|
| `@ObjectModel.dataCategory: #TEXT` | Déclare une vue de texte, avec clé de langue |
| `@ObjectModel.representativeKey` | Désigne la clé métier parmi les clés |
| `@ObjectModel.resultSet.sizeCategory: #XS` | **Déclenche le rendu en liste déroulante** |
| `@ObjectModel.text.element` | Désigne le texte associé au code |
| `@Semantics.language: true` | Identifie la clé de langue |

> **📸 COPIE D'ÉCRAN N°06** — Eclipse ADT : vue d'aide à la saisie, code source
> *Remplacer cette ligne par :* `![Copie 06](images/RAP-ANNOT/capture-06.png)`

> **📸 COPIE D'ÉCRAN N°07** — Eclipse ADT : aperçu de données, quatre statuts et leurs textes
> *Remplacer cette ligne par :* `![Copie 07](images/RAP-ANNOT/capture-07.png)`

> **ℹ️ Lecture de `DD07T`**
> Cette table du dictionnaire porte les textes des valeurs fixes. Elle est stable de longue date mais n'est pas une interface publiée par SAP : l'usage est courant et sans risque en on-premise, mais à documenter. Alternative plus formelle : créer une table de contrôle `Z` et sa table de textes, au prix d'un paramétrage supplémentaire.

---

### 5.5 Adapter la vue d'interface

Deux ajouts : l'association vers la vue de texte pour afficher le libellé, et le calcul de la criticité.

```abap
@AccessControl.authorizationCheck: #NOT_REQUIRED
@EndUserText.label: 'Suivi des demandes d''achat'
define root view entity ZI_PurReqTracking
  as select from zmm_pr_track

  association [0..1] to I_PurchaseRequisition as _PurReq
    on $projection.PurchaseRequisition = _PurReq.PurchaseRequisition

  association [0..1] to I_PurchaseRequisitionItem as _PurReqItem
    on  $projection.PurchaseRequisition     = _PurReqItem.PurchaseRequisition
    and $projection.PurchaseRequisitionItem = _PurReqItem.PurchaseRequisitionItem

  association [0..1] to ZI_PurReqTrackingStatusVH as _StatusText
    on  $projection.TrackingStatus = _StatusText.TrackingStatus
    and _StatusText.Language       = $session.system_language
{
  key tracking_uuid              as TrackingUUID,
      purchase_req               as PurchaseRequisition,
      purchase_req_item          as PurchaseRequisitionItem,

      @ObjectModel.text.association: '_StatusText'
      tracking_status            as TrackingStatus,

      _StatusText.TrackingStatusName as TrackingStatusName,

      case tracking_status
        when 'CL' then 3
        when 'OP' then 2
        when 'IP' then 2
        when 'CA' then 0
        else           0
      end                        as StatusCriticality,

      comments                   as Comments,
      target_date                as TargetDate,

      _PurReqItem.Material                    as Material,
      _PurReqItem.PurchaseRequisitionItemText as PurReqItemText,

      @Semantics.user.createdBy: true
      created_by                 as CreatedBy,
      @Semantics.systemDateTime.createdAt: true
      created_at                 as CreatedAt,
      @Semantics.user.lastChangedBy: true
      last_changed_by            as LastChangedBy,
      @Semantics.systemDateTime.lastChangedAt: true
      last_changed_at            as LastChangedAt,
      @Semantics.systemDateTime.localInstanceLastChangedAt: true
      local_last_changed_at      as LocalLastChangedAt,

      _PurReq,
      _PurReqItem,
      _StatusText
}
```

> **📸 COPIE D'ÉCRAN N°08** — Eclipse ADT : vue d'interface modifiée
> *Remplacer cette ligne par :* `![Copie 08](images/RAP-ANNOT/capture-08.png)`

---

### 5.6 Adapter la vue de projection

L'annotation d'aide à la saisie se place ici : c'est la couche de consommation.

```abap
@AccessControl.authorizationCheck: #NOT_REQUIRED
@EndUserText.label: 'Suivi des demandes d''achat - projection'
@Metadata.allowExtensions: true
define root view entity ZC_PurReqTracking
  provider contract transactional_query
  as projection on ZI_PurReqTracking
{
  key TrackingUUID,
      PurchaseRequisition,
      PurchaseRequisitionItem,

      @Consumption.valueHelpDefinition: [
        { entity: { name:    'ZI_PurReqTrackingStatusVH',
                    element: 'TrackingStatus' } } ]
      TrackingStatus,

      TrackingStatusName,
      StatusCriticality,

      Comments,
      TargetDate,
      Material,
      PurReqItemText,
      LastChangedAt,
      LocalLastChangedAt,

      /* association exposée pour le texte */
      _StatusText : redirected to ZI_PurReqTrackingStatusVH
}
```

> **📸 COPIE D'ÉCRAN N°09** — Eclipse ADT : vue de projection modifiée
> *Remplacer cette ligne par :* `![Copie 09](images/RAP-ANNOT/capture-09.png)`

---

### 5.7 Adapter l'extension de métadonnées

```abap
@Metadata.layer: #CORE

@UI.headerInfo: { typeName:       'Suivi de DA',
                  typeNamePlural: 'Suivis de DA',
                  title:       { type: #STANDARD, value: 'PurchaseRequisition' },
                  description: { value: 'PurReqItemText' } }

@UI.facet: [ { id:       'General',
               purpose:  #STANDARD,
               type:     #IDENTIFICATION_REFERENCE,
               label:    'Informations générales',
               position: 10 } ]

annotate view ZC_PurReqTracking with
{
  @UI.hidden: true
  TrackingUUID;

  @UI.hidden: true
  StatusCriticality;

  @UI.hidden: true
  TrackingStatusName;

  @UI: { lineItem:       [ { position: 10, importance: #HIGH } ],
         identification: [ { position: 10 } ],
         selectionField: [ { position: 10 } ] }
  PurchaseRequisition;

  @UI: { lineItem:       [ { position: 20, importance: #HIGH } ],
         identification: [ { position: 20 } ] }
  PurchaseRequisitionItem;

  @UI: { lineItem:       [ { position: 30,
                             label: 'Statut',
                             criticality: 'StatusCriticality',
                             importance: #HIGH } ],
         identification: [ { position: 30, label: 'Statut de suivi' } ],
         selectionField: [ { position: 20 } ] }
  @UI.textArrangement: #TEXT_ONLY
  TrackingStatus;

  @UI: { lineItem:       [ { position: 40, label: 'Échéance' } ],
         identification: [ { position: 40, label: 'Date d''échéance' } ] }
  TargetDate;

  @UI: { identification: [ { position: 50, label: 'Commentaire' } ] }
  Comments;

  @UI: { lineItem: [ { position: 60, label: 'Article', importance: #LOW } ] }
  Material;
}
```

Trois annotations font ici tout le travail sur le statut : `criticality` pour la couleur, `textArrangement: #TEXT_ONLY` pour n'afficher que le libellé, `selectionField` pour proposer le filtre — qui bénéficiera lui aussi de la liste déroulante.

> **📸 COPIE D'ÉCRAN N°10** — Eclipse ADT : extension de métadonnées modifiée
> *Remplacer cette ligne par :* `![Copie 10](images/RAP-ANNOT/capture-10.png)`

---

### 5.8 Adapter la behavior definition

Deux modifications : rendre le statut obligatoire, et déclarer en lecture seule les champs dérivés.

```abap
managed implementation in class zbp_i_purreqtracking unique;
strict ( 1 );
with draft;

define behavior for ZI_PurReqTracking alias PurReqTracking
persistent table zmm_pr_track
draft table zmm_pr_track_d
lock master total etag LastChangedAt
authorization master ( global )
etag master LocalLastChangedAt
{
  field ( readonly, numbering : managed ) TrackingUUID;
  field ( mandatory )                     PurchaseRequisition,
                                          PurchaseRequisitionItem,
                                          TrackingStatus;
  field ( readonly )                      TrackingStatusName,
                                          StatusCriticality,
                                          Material, PurReqItemText,
                                          CreatedBy, CreatedAt,
                                          LastChangedBy, LastChangedAt,
                                          LocalLastChangedAt;

  create;
  update;
  delete;

  action ( features : instance ) markAsCompleted result [1] $self;

  determination setInitialStatus on modify { create; }

  validation validateRequisition on save { create; field PurchaseRequisition,
                                                         PurchaseRequisitionItem; }
  validation validateTargetDate  on save { create; update; field TargetDate; }
  validation validateStatus      on save { create; update; field TrackingStatus; }

  draft action Activate optimized;
  draft action Discard;
  draft action Edit;
  draft action Resume;
  draft determine action Prepare
  {
    validation validateRequisition;
    validation validateTargetDate;
    validation validateStatus;
  }

  mapping for zmm_pr_track
  {
    TrackingUUID            = tracking_uuid;
    PurchaseRequisition     = purchase_req;
    PurchaseRequisitionItem = purchase_req_item;
    TrackingStatus          = tracking_status;
    Comments                = comments;
    TargetDate              = target_date;
    CreatedBy               = created_by;
    CreatedAt               = created_at;
    LastChangedBy           = last_changed_by;
    LastChangedAt           = last_changed_at;
    LocalLastChangedAt      = local_last_changed_at;
  }
}
```

> **⚠️ Champs dérivés en lecture seule**
> `TrackingStatusName` et `StatusCriticality` sont calculés par la vue. S'ils ne sont pas déclarés en lecture seule, l'activation échoue : la behavior definition ne peut pas mapper vers la table des champs qui n'y existent pas.

> **📸 COPIE D'ÉCRAN N°11** — Eclipse ADT : behavior definition modifiée
> *Remplacer cette ligne par :* `![Copie 11](images/RAP-ANNOT/capture-11.png)`

---

### 5.9 Adapter l'implémentation

Ajout d'une validation qui garantit la cohérence même en cas d'appel direct du service, hors interface.

```abap
METHOD validateStatus.

  READ ENTITIES OF ZI_PurReqTracking IN LOCAL MODE
    ENTITY PurReqTracking
      FIELDS ( TrackingStatus )
      WITH CORRESPONDING #( keys )
    RESULT DATA(lt_tracking).

  SELECT domvalue_l
    FROM dd07l
    WHERE domname  = 'ZZ_PR_TRACK_STATUS'
      AND as4local = 'A'
      AND as4vers  = '0000'
    INTO TABLE @DATA(lt_allowed).

  LOOP AT lt_tracking INTO DATA(ls_tracking).

    IF NOT line_exists( lt_allowed[ table_line = ls_tracking-TrackingStatus ] ).

      APPEND VALUE #( %tky = ls_tracking-%tky ) TO failed-purreqtracking.

      APPEND VALUE #(
        %tky                     = ls_tracking-%tky
        %element-TrackingStatus  = if_abap_behv=>mk-on
        %msg = new_message(
                 id       = 'ZMM_PR_TRACK'
                 number   = '003'
                 severity = if_abap_behv_message=>severity-error
                 v1       = ls_tracking-TrackingStatus ) )
        TO reported-purreqtracking.

    ENDIF.

  ENDLOOP.

ENDMETHOD.
```

Adapter également la détermination, dont la valeur par défaut reste `OP` — désormais garantie valide par le domaine.

> **ℹ️ Pourquoi valider malgré le domaine**
> Les valeurs fixes du domaine sont contrôlées par l'interface, pas par la base. Un appel OData direct ou une insertion par EML peut écrire une valeur hors domaine. La validation ferme cette porte.

> **📸 COPIE D'ÉCRAN N°12** — Eclipse ADT : validation du statut
> *Remplacer cette ligne par :* `![Copie 12](images/RAP-ANNOT/capture-12.png)`

---

### 5.10 Activer et tester

Ordre d'activation impératif :

1. Domaine, puis élément de données.
2. Tables de base et de draft.
3. Vue d'aide à la saisie.
4. Vue d'interface, puis vue de projection.
5. Extension de métadonnées.
6. Behavior definition, puis classe d'implémentation.
7. Republier le service binding.
8. Vider les caches de métadonnées, puis le cache du navigateur.

Points à contrôler :

1. À la création, le champ statut propose une **liste déroulante** avec les quatre libellés.
2. Le code technique n'est pas visible — seul le libellé s'affiche.
3. Le statut par défaut est positionné à l'ouverture du formulaire.
4. Dans la liste, le statut apparaît coloré selon sa criticité.
5. Le filtre sur le statut propose lui aussi la liste déroulante.
6. Le champ est refusé s'il est laissé vide.

> **📸 COPIE D'ÉCRAN N°13** — Application : liste déroulante des quatre statuts à la création
> *Remplacer cette ligne par :* `![Copie 13](images/RAP-ANNOT/capture-13.png)`

> **📸 COPIE D'ÉCRAN N°14** — Application : liste avec statuts colorés selon la criticité
> *Remplacer cette ligne par :* `![Copie 14](images/RAP-ANNOT/capture-14.png)`

> **📸 COPIE D'ÉCRAN N°15** — Application : filtre sur le statut avec liste déroulante
> *Remplacer cette ligne par :* `![Copie 15](images/RAP-ANNOT/capture-15.png)`

> **📸 COPIE D'ÉCRAN N°16** — Application : message d'erreur sur statut obligatoire
> *Remplacer cette ligne par :* `![Copie 16](images/RAP-ANNOT/capture-16.png)`

---

## 6. Fiche de recette

| N° | Point de contrôle | Résultat attendu | OK / KO |
|---|---|---|---|
| 1 | Domaine créé avec quatre valeurs fixes | Actif | ☐ |
| 2 | Élément de données et libellés | Actif | ☐ |
| 3 | Tables base et draft alignées | Activées | ☐ |
| 4 | Vue d'aide à la saisie | Quatre lignes | ☐ |
| 5 | Association de texte dans la vue d'interface | Texte remonté | ☐ |
| 6 | Criticité calculée | Conforme | ☐ |
| 7 | Aide à la saisie déclarée en projection | Active | ☐ |
| 8 | Extension de métadonnées activée | Active | ☐ |
| 9 | Behavior definition activée | Sans erreur | ☐ |
| 10 | Rendu en liste déroulante à la création | Quatre choix | ☐ |
| 11 | Libellé seul affiché, code masqué | Conforme | ☐ |
| 12 | Statut par défaut à la création | `OP` | ☐ |
| 13 | Couleurs selon la criticité | Conformes | ☐ |
| 14 | Liste déroulante dans le filtre | Présente | ☐ |
| 15 | Refus si le statut est vide | Message correct | ☐ |
| 16 | Refus d'une valeur hors domaine par appel direct | Message correct | ☐ |
| 17 | Libellés dans la langue de connexion | Conformes | ☐ |

> **📸 COPIE D'ÉCRAN N°17** — Synthèse de recette : application avec statuts opérationnels
> *Remplacer cette ligne par :* `![Copie 17](images/RAP-ANNOT/capture-17.png)`

---

## Annexe A — Diagnostic

| Symptôme | Cause probable | Action corrective |
|---|---|---|
| Boîte de dialogue au lieu d'une liste déroulante | `sizeCategory` absent ou trop grand | Déclarer `#XS` sur la vue d'aide |
| Aucune aide à la saisie | Annotation refusée dans l'extension | Déplacer l'annotation dans la vue de projection |
| Liste déroulante vide | Nom de domaine ou filtres erronés | Contrôler l'aperçu de données de la vue d'aide |
| Le code s'affiche au lieu du libellé | `textArrangement` absent | Ajouter `#TEXT_ONLY` |
| Le libellé reste vide | Association de texte non exposée | Exposer l'association dans la projection |
| La behavior definition ne s'active plus | Champs dérivés non déclarés en lecture seule | Compléter la clause `field ( readonly )` |
| Aucune couleur | Champ de criticité non calculé ou masqué à tort | Contrôler le calcul dans la vue d'interface |
| Le champ reste vide à l'ouverture | Détermination non déclenchée | Contrôler la clause `on modify` |
| Les libellés sont en anglais | Textes du domaine non traduits | Compléter la traduction des valeurs fixes |
| Rien ne change après activation | Cache de métadonnées | Vider les caches serveur et navigateur |

---

## Annexe B — Index des copies d'écran

| N° | Chapitre | Contenu attendu |
|---|---|---|
| 01 | 3.3 | Colonnes ordonnées et hiérarchisées |
| 02 | 3.5 | Page objet structurée |
| 03 à 04 | 5.1 et 5.2 | Domaine et élément de données |
| 05 | 5.3 | Table modifiée |
| 06 à 07 | 5.4 | Vue d'aide à la saisie |
| 08 à 09 | 5.5 et 5.6 | Vues d'interface et de projection |
| 10 | 5.7 | Extension de métadonnées |
| 11 à 12 | 5.8 et 5.9 | Comportement et validation |
| 13 à 16 | 5.10 | Tests dans l'application |
| 17 | 6 | Synthèse de recette |

---

## Annexe C — Historique des versions

| Version | Date | Auteur | Nature des modifications |
|---|---|---|---|
| 1.0 | … | … | Création du document |
| | | | |
