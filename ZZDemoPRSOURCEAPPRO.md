# Application spécifique « Détermination Source Appro » sur service RAP — V1

**Contexte :** SAP S/4HANA Cloud Private Edition 2021 FPS02 (ABAP Platform 2021 / SAP_BASIS 7.56, SAP_UI 7.56, SAPUI5 1.96.x)
**Objet :** créer de zéro un service **RAP en lecture seule** exposant les demandes d'achat de type `ZCTL` avec leur source d'approvisionnement, puis une application **SAP Fiori elements (List Report / Object Page, OData V2)** avec un bouton ouvrant une boîte de dialogue listant les sources candidates.
**Outils :** ABAP Development Tools (Eclipse) + SAP Fiori tools (VS Code ou SAP Business Application Studio)
**Périmètre V1 :** affichage uniquement. Aucune écriture, aucune action RAP. La pop-up est un écran de consultation des sources candidates (le bouton « Affecter » est présent mais désactivé, prévu pour la V2).

---

## Sommaire

1. [Objectif, périmètre et maquettes](#1-objectif-périmètre-et-maquettes)
2. [Architecture cible](#2-architecture-cible)
3. [Prérequis](#3-prérequis)
4. [Étape 1 – Package et ordre de transport](#étape-1--package-et-ordre-de-transport)
5. [Étape 2 – Vue d'interface des postes de DA](#étape-2--vue-dinterface-des-postes-de-da)
6. [Étape 3 – Vue des sources candidates](#étape-3--vue-des-sources-candidates)
7. [Étape 4 – Contrôle d'accès (DCL)](#étape-4--contrôle-daccès-dcl)
8. [Étape 5 – Vues de consommation (projections RAP)](#étape-5--vues-de-consommation-projections-rap)
9. [Étape 6 – Annotations UI (metadata extensions)](#étape-6--annotations-ui-metadata-extensions)
10. [Étape 7 – Service definition](#étape-7--service-definition)
11. [Étape 8 – Service binding OData V2 et publication](#étape-8--service-binding-odata-v2-et-publication)
12. [Étape 9 – Générer l'application Fiori elements](#étape-9--générer-lapplication-fiori-elements)
13. [Étape 10 – Bouton personnalisé et boîte de dialogue](#étape-10--bouton-personnalisé-et-boîte-de-dialogue)
14. [Étape 11 – Test en local](#étape-11--test-en-local)
15. [Étape 12 – Déploiement et Launchpad](#étape-12--déploiement-et-launchpad)
16. [Étape 13 – Transport](#étape-13--transport)
17. [Annexe A – Catalogue des annotations démontrées](#annexe-a--catalogue-des-annotations-démontrées)
18. [Annexe B – Ce qui est prévu en V2](#annexe-b--ce-qui-est-prévu-en-v2)
19. [Dépannage](#dépannage)
20. [Captures à réaliser](#captures-à-réaliser)
21. [Références](#références)

---

## 1. Objectif, périmètre et maquettes

### Fonctions V1

| # | Fonction | Moyen |
|---|---|---|
| F1 | Lister les postes de DA de type `ZCTL` | Vue CDS filtrée, List Report |
| F2 | Afficher uniquement les champs utiles | Sélection explicite dans la vue de consommation |
| F3 | Afficher la source d'approvisionnement si renseignée | Fournisseur fixe / contrat / fiche info + indicateur `SourceOfSupplyIsAssigned` |
| F4 | Bouton ouvrant une pop-up des sources candidates | Custom action Fiori elements + fragment XML |
| F5 | Démontrer plusieurs familles d'annotations UI | Metadata extension (voir [Annexe A](#annexe-a--catalogue-des-annotations-démontrées)) |
| F6 | Aucune modification de données | Service en lecture seule, pas de behavior definition |

### Maquette — List Report

```text
┌────────────────────────────────────────────────────────────────────────────────────┐
│ ◀  Détermination source d'approvisionnement                               🔍  👤  │
├────────────────────────────────────────────────────────────────────────────────────┤
│ Standard ▾              [Toutes] [Sans source] [Avec source]   ◀ onglets (Selection│
│ Division [    ]  Groupe acheteurs [   ]  Article [      ]  Source affectée [  ▾]   │
│ Recherche [            ]                       [Adapter les filtres]  [Exécuter]   │
├────────────────────────────────────────────────────────────────────────────────────┤
│ Postes de DA (12)                        ┏━━━━━━━━━━━━━━━━━━━━━━┓   ⚙  ⤓          │
│                                          ┃ Affecter une source  ┃ ◀ custom action  │
│                                          ┗━━━━━━━━━━━━━━━━━━━━━━┛                  │
│ ─────────────────────────────────────────────────────────────────────────────────  │
│ DA / Poste │ Article        │ Division │ Qté      │ Livraison │ Source     │ Délai │
│ 10000123/10│ TG11 Tube acier│ 1010     │ 100 PC   │ 12.10.2026│ ⬤ Aucune   │ ▓▓░ 5 │
│ 10000123/20│ TG12 Raccord   │ 1010     │  50 PC   │ 05.10.2026│ ⬤ 17300001 │ ▓░░ 2 │
└────────────────────────────────────────────────────────────────────────────────────┘
   ⬤ rouge = aucune source (criticality 1)   ⬤ vert = source affectée (criticality 3)
```

### Maquette — Boîte de dialogue

```text
        ┌──────────────────────────────────────────────────────────────┐
        │  Sources candidates — DA 10000123 / poste 10                 │
        ├──────────────────────────────────────────────────────────────┤
        │  Article TG11 · Division 1010 · Quantité 100 PC              │
        │ ──────────────────────────────────────────────────────────── │
        │  Fournisseur │ Org. achats │ Contrat    │ Fixe │ Validité    │
        │  17300001    │ 1010        │ 4600000012 │  ✔   │ →31.12.2026 │
        │  17300004    │ 1010        │            │      │ →31.12.2026 │
        │ ──────────────────────────────────────────────────────────── │
        │              [ Affecter (V2 – désactivé) ]      [ Fermer ]   │
        └──────────────────────────────────────────────────────────────┘
```

---

## 2. Architecture cible

```text
┌──────────────────────────── Couche UI ─────────────────────────────┐
│ App Fiori elements V2  « zmmprsourcing »  (BSP ZMM_PR_SRC)         │
│   manifest.json : custom action + controller extension + fragment  │
└───────────────────────────────┬────────────────────────────────────┘
                                │ OData V2
┌───────────────────────────────┴────────────────────────────────────┐
│ Service binding   ZUI_PR_SOURCING_O2   (OData V2 – UI)             │
│ Service definition ZUI_PR_SOURCING                                 │
├────────────────────────────────────────────────────────────────────┤
│ Projections (RAP, lecture seule)                                   │
│   ZC_PurReqnItemSourcing  ──▶ metadata ext. ZC_PurReqnItemSourcing │
│   ZC_SourceOfSupplyCandidate                                       │
├────────────────────────────────────────────────────────────────────┤
│ Vues d'interface                                                   │
│   ZI_PurReqnItemSourcing  (filtre type ZCTL)                       │
│   ZI_SourceOfSupplyCandidate                                       │
│   DCL : ZI_PURREQNITEM_SOURCING                                    │
├────────────────────────────────────────────────────────────────────┤
│ Modèle standard                                                    │
│   I_PurchaseRequisitionItemAPI01, I_Supplier, I_Product, I_Plant   │
│   Table EORD (liste des sources)                                   │
└────────────────────────────────────────────────────────────────────┘
```

> **Pourquoi OData V2 et non V4 ?** Sur SAPUI5 1.96, SAP Fiori elements **V2** couvre un éventail d'annotations beaucoup plus large (micro-graphiques, indicateurs de progression, smart links, onglets de variantes, contact quick view). Le binding `OData V4 – UI` est disponible dans RAP sur 2021, mais plusieurs annotations de démonstration n'y sont pas encore rendues en 1.96. Le choix V2 est noté ici comme décision d'architecture ; la V2 de l'application pourra basculer en V4 lors d'une montée de version UI5.

---

## 3. Prérequis

### 3.1 Système et paramétrage métier

| Élément | Valeur / contrôle | Transaction |
|---|---|---|
| Type de document DA `ZCTL` | Existe et est utilisé | `SPRO` → MM → Achats → Demande d'achat → **Définir types de document** (vue `V_T161`, `SM30`) |
| Jeu de données de test | Quelques DA `ZCTL` avec et sans source | `ME51N` / `ME53N` |
| Liste des sources alimentée | Enregistrements dans `EORD` | `ME01` / `ME03` |
| Nœud ICF OData V2 | `/sap/opu/odata` actif | `SICF` |
| Nœud ICF UI5 | `/sap/bc/ui5_ui5`, `/sap/bc/lrep`, `/sap/bc/adt`, `/sap/bc/ui2` actifs | `SICF` |
| Gateway | Déploiement embedded, alias `LOCAL` | `/IWFND/MAINT_SERVICES` |
| ADT | Eclipse + ABAP Development Tools à jour | — |
| Fiori tools | VS Code (ou BAS), Node.js LTS | — |

### 3.2 Objets à créer

| Objet | Nom | Type |
|---|---|---|
| Package | `ZMM_SOURCING` | DEVC |
| Vue d'interface DA | `ZI_PurReqnItemSourcing` | DDLS |
| Vue d'interface sources | `ZI_SourceOfSupplyCandidate` | DDLS |
| Contrôle d'accès | `ZI_PURREQNITEM_SOURCING` | DCLS |
| Projection DA | `ZC_PurReqnItemSourcing` | DDLS |
| Projection sources | `ZC_SourceOfSupplyCandidate` | DDLS |
| Metadata extension | `ZC_PurReqnItemSourcing` | DDLX |
| Service definition | `ZUI_PR_SOURCING` | SRVD |
| Service binding | `ZUI_PR_SOURCING_O2` | SRVB |
| Application UI5 | `ZMM_PR_SRC` (BSP) | WAPA |
| Objet sémantique | `ZSourceOfSupply` | `/UI2/SEMOBJ` |
| Catalogue / tuile / rôle | `Z_TC_MM_SOURCING`, `Z_BC_MM_SOURCING`, `Z_BR_SOURCING` | FLP / PFCG |

### 3.3 Autorisations

| Profil | Objets / rôles |
|---|---|
| Développeur | `S_DEVELOP` (DEVCLASS `ZMM_SOURCING`, OBJTYPE `DDLS`, `DCLS`, `DDLX`, `SRVD`, `SRVB`, `WAPA`, `DEVC`), `S_TRANSPRT`, `S_ADT_RES` |
| Publication service V2 | `S_SERVICE`, droits Gateway (`/IWFND/MAINT_SERVICES`) : rôle type `SAP_IWFND_ADMIN` ou équivalent |
| Utilisateur de test | Rôle dérivé acheteur : `M_BANF_BSA`, `M_BANF_EKG`, `M_BANF_EKO`, `M_BANF_WRK` (lecture DA) + `S_SERVICE` sur le nouveau service |
| Administrateur FLP | `SAP_UI2_ADMIN_700` ou équivalent |

---

## Étape 1 – Package et ordre de transport

### Dans ADT

1. Créer le projet ABAP si nécessaire : *File → New → ABAP Project* → système, mandant, utilisateur.
2. Clic droit sur le nœud du projet → **New → ABAP Package** :

| Champ | Valeur |
|---|---|
| Name | `ZMM_SOURCING` |
| Description | `Determination source approvisionnement - DA ZCTL` |
| Superpackage | (vide ou package chapeau Z) |
| Package Type | `Development` |
| Software Component | `HOME` |
| Application Component | `MM-PUR-REQ` |
| Transport Layer | couche Z du paysage |

3. *Next* → **Create new request** : `Creation app determination source appro` → l'ordre `S4DK9xxxxx` est mémorisé pour les objets suivants.

> Équivalent SAP GUI : `SE21` (package), `SE09` (ordre).

📸 *Capture 4-01 : création du package dans ADT*

---

## Étape 2 – Vue d'interface des postes de DA

Clic droit sur `ZMM_SOURCING` → **New → Other ABAP Repository Object → Core Data Services → Data Definition**
Name : `ZI_PurReqnItemSourcing` — Description : `Postes de DA ZCTL - donnees sourcing` — Template : *Define View Entity*.

```abap
@EndUserText.label: 'Postes de DA ZCTL - donnees sourcing'
@AccessControl.authorizationCheck: #CHECK
@Metadata.ignorePropagatedAnnotations: true
@ObjectModel.usageType: { serviceQuality: #C, sizeCategory: #M, dataClass: #TRANSACTIONAL }
@Search.searchable: true
define root view entity ZI_PurReqnItemSourcing
  as select from I_PurchaseRequisitionItemAPI01 as PurReqnItem

  association [0..1] to I_Supplier                 as _Supplier
    on  $projection.FixedSupplier = _Supplier.Supplier
  association [0..1] to I_Product                  as _Product
    on  $projection.Material = _Product.Product
  association [0..1] to I_Plant                    as _Plant
    on  $projection.Plant = _Plant.Plant
  association [0..*] to ZI_SourceOfSupplyCandidate as _SourceCandidate
    on  $projection.Material = _SourceCandidate.Material
    and $projection.Plant    = _SourceCandidate.Plant

{
      --- Clés
  key PurReqnItem.PurchaseRequisition,
  key PurReqnItem.PurchaseRequisitionItem,

      --- Identification du besoin
      PurReqnItem.PurchaseRequisitionType,
      @Search.defaultSearchElement: true
      @Search.fuzzinessThreshold: 0.8
      PurReqnItem.PurchaseRequisitionItemText,
      @Search.defaultSearchElement: true
      @ObjectModel.text.association: '_Product'
      PurReqnItem.Material,
      PurReqnItem.MaterialGroup,
      PurReqnItem.Plant,
      PurReqnItem.PurchasingGroup,
      PurReqnItem.PurchasingOrganization,

      --- Quantités et dates
      @Semantics.quantity.unitOfMeasure: 'BaseUnit'
      PurReqnItem.RequestedQuantity,
      @Semantics.unitOfMeasure: true
      PurReqnItem.BaseUnit,
      PurReqnItem.DeliveryDate,

      --- Source d'approvisionnement (si renseignée)
      PurReqnItem.FixedSupplier,
      PurReqnItem.SupplyingPlant,
      PurReqnItem.PurchasingInfoRecord,
      PurReqnItem.PurchaseContract,
      PurReqnItem.PurchaseContractItem,
      PurReqnItem.SourceOfSupplyIsAssigned,

      --- Éléments calculés pour la visualisation
      // 3 = vert (source affectée), 1 = rouge (aucune source)
      case when PurReqnItem.SourceOfSupplyIsAssigned = 'X' then 3 else 1 end          as SourcingCriticality,

      // Nombre de jours avant la date de livraison souhaitée
      dats_days_between( $session.system_date, PurReqnItem.DeliveryDate )             as DaysToDelivery,

      // 1 = urgent (< 3 jours), 2 = à surveiller (< 10 jours), 3 = confortable
      case when dats_days_between( $session.system_date, PurReqnItem.DeliveryDate ) < 3  then 1
           when dats_days_between( $session.system_date, PurReqnItem.DeliveryDate ) < 10 then 2
           else 3
      end                                                                             as DeliveryCriticality,

      --- Associations exposées
      _Supplier,
      _Product,
      _Plant,
      _SourceCandidate
}
where
  PurReqnItem.PurchaseRequisitionType = 'ZCTL'
```

**Activer** (`Ctrl+F3`).

Points d'attention :

| Sujet | À vérifier / adapter |
|---|---|
| Noms de zones | Ouvrir `I_PurchaseRequisitionItemAPI01` dans ADT (`Ctrl+Shift+A`) et comparer les noms exposés par votre niveau de SP |
| Filtre `ZCTL` | Codé en dur pour la V1. Alternative : paramètre d'entrée `P_PurReqnType` ou table de paramétrage `ZMM_SOURCING_TYP` |
| `#CHECK` | Nécessite le DCL de l'étape 4. Pour un premier test rapide, `#NOT_REQUIRED` fonctionne mais **désactive le contrôle d'autorisation** — ne pas laisser en production |
| `case … then 3 else 1 end` | Si l'activation se plaint du type, encadrer par `cast( … as abap.int4 )` |
| Vue de suppression | Ajouter `and PurReqnItem.PurchaseRequisitionItemIsDeleted = ''` si votre release expose cette zone |

📸 *Capture 4-02 : vue `ZI_PurReqnItemSourcing` activée*

---

## Étape 3 – Vue des sources candidates

Les sources proviennent de la **liste des sources standard** (`EORD`), sans aucun ajustement de table.

```abap
@EndUserText.label: 'Sources approvisionnement candidates (liste des sources)'
@AccessControl.authorizationCheck: #NOT_REQUIRED
@Metadata.ignorePropagatedAnnotations: true
@ObjectModel.usageType: { serviceQuality: #C, sizeCategory: #S, dataClass: #MASTER }
define view entity ZI_SourceOfSupplyCandidate
  as select from eord

  association [0..1] to I_Supplier as _Supplier
    on $projection.Supplier = _Supplier.Supplier

{
  key eord.matnr as Material,
  key eord.werks as Plant,
  key eord.zeord as SourceListRecord,

      @ObjectModel.text.association: '_Supplier'
      eord.lifnr as Supplier,
      eord.ekorg as PurchasingOrganization,
      eord.ebeln as PurchaseAgreement,
      eord.ebelp as PurchaseAgreementItem,
      eord.reswk as SupplyingPlant,

      @Semantics.booleanIndicator: true
      eord.flifn as IsFixedSupplier,
      @Semantics.booleanIndicator: true
      eord.notkz as IsBlocked,

      eord.vdatu as ValidityStartDate,
      eord.bdatu as ValidityEndDate,

      // 3 = source fixe, 2 = source autorisée
      case when eord.flifn = 'X' then 3 else 2 end as SourceCriticality,

      _Supplier
}
where
      eord.notkz = ''                       // sources non bloquées
  and eord.bdatu >= $session.system_date    // encore valides
```

> **Vérifications** : ouvrir `EORD` dans `SE11` et confirmer les noms de zones de votre release (`MATNR`, `WERKS`, `ZEORD`, `LIFNR`, `EKORG`, `EBELN`, `EBELP`, `RESWK`, `FLIFN`, `NOTKZ`, `VDATU`, `BDATU`). Si votre système expose une vue CDS standard released pour la liste des sources, la préférer à l'accès direct à la table (recherche dans ADT ou dans l'app *View Browser*).
>
> **Extension possible dès la V1.1** : ajouter en `union all` les fiches info achat (`EINA`/`EINE`) et les contrats-cadres (`I_PurchaseContractItemAPI01`) avec une zone `SourceType` (`LS` liste des sources / `IR` fiche info / `CT` contrat).

📸 *Capture 4-03 : vue `ZI_SourceOfSupplyCandidate` activée*

---

## Étape 4 – Contrôle d'accès (DCL)

Clic droit sur `ZI_PurReqnItemSourcing` → **New → Access Control** → nom `ZI_PURREQNITEM_SOURCING` → template *Define Role with Inherited Conditions*.

```abap
@EndUserText.label: 'Controle acces postes DA ZCTL'
@MappingRole: true
define role ZI_PURREQNITEM_SOURCING {
  grant select on ZI_PurReqnItemSourcing
    where inheriting conditions from entity I_PurchaseRequisitionItemAPI01;
}
```

Activer. Si l'entité source n'expose pas de DCL héritable sur votre release, utiliser le contrôle explicite :

```abap
define role ZI_PURREQNITEM_SOURCING {
  grant select on ZI_PurReqnItemSourcing
    where ( Plant )            = aspect pfcg_auth( M_BANF_WRK, WERKS, ACTVT = '03' )
      and ( PurchasingGroup )  = aspect pfcg_auth( M_BANF_EKG, EKGRP, ACTVT = '03' );
}
```

> Contrôle : `SU53` après un test, ou ADT → *Run As → ABAP Application (Console)* avec un `SELECT` sur la vue sous un utilisateur restreint.

---

## Étape 5 – Vues de consommation (projections RAP)

### 5.1 Projection racine

Clic droit sur `ZI_PurReqnItemSourcing` → **New Data Definition** → `ZC_PurReqnItemSourcing`, template *Define Projection View*.

```abap
@EndUserText.label: 'Determination source appro - consommation'
@AccessControl.authorizationCheck: #CHECK
@Metadata.allowExtensions: true
@Search.searchable: true
@ObjectModel.semanticKey: ['PurchaseRequisition', 'PurchaseRequisitionItem']
define root view entity ZC_PurReqnItemSourcing
  provider contract transactional_query
  as projection on ZI_PurReqnItemSourcing
{
  key PurchaseRequisition,
  key PurchaseRequisitionItem,
      PurchaseRequisitionType,
      PurchaseRequisitionItemText,
      @Consumption.valueHelpDefinition: [{ entity: { name: 'I_ProductStdVH', element: 'Product' } }]
      Material,
      MaterialGroup,
      @Consumption.valueHelpDefinition: [{ entity: { name: 'I_PlantStdVH', element: 'Plant' } }]
      Plant,
      PurchasingGroup,
      PurchasingOrganization,
      RequestedQuantity,
      BaseUnit,
      DeliveryDate,
      @Consumption.semanticObject: 'Supplier'
      FixedSupplier,
      SupplyingPlant,
      PurchasingInfoRecord,
      @Consumption.semanticObject: 'PurchaseContract'
      PurchaseContract,
      PurchaseContractItem,
      SourceOfSupplyIsAssigned,
      SourcingCriticality,
      DaysToDelivery,
      DeliveryCriticality,
      /* Associations */
      _Supplier,
      _Product,
      _Plant,
      _SourceCandidate : redirected to composition child ZC_SourceOfSupplyCandidate
}
```

> **Si l'activation échoue** sur `provider contract transactional_query` (demande d'une behavior definition) ou sur `redirected to composition child` : la V1 étant en lecture seule, remplacer par une vue de consommation simple :
> ```abap
> define view entity ZC_PurReqnItemSourcing as select from ZI_PurReqnItemSourcing { … _SourceCandidate }
> ```
> et exposer les deux vues indépendamment dans le service. Le comportement de l'application Fiori est identique en lecture seule ; les annotations restent valables.

### 5.2 Projection des sources

```abap
@EndUserText.label: 'Sources candidates - consommation'
@AccessControl.authorizationCheck: #NOT_REQUIRED
@Metadata.allowExtensions: true
define view entity ZC_SourceOfSupplyCandidate
  as projection on ZI_SourceOfSupplyCandidate
{
  key Material,
  key Plant,
  key SourceListRecord,
      Supplier,
      PurchasingOrganization,
      PurchaseAgreement,
      PurchaseAgreementItem,
      SupplyingPlant,
      IsFixedSupplier,
      ValidityStartDate,
      ValidityEndDate,
      SourceCriticality,
      _Supplier
}
```

📸 *Capture 4-04 : projections activées dans l'arborescence du package*

---

## Étape 6 – Annotations UI (metadata extensions)

Clic droit sur `ZC_PurReqnItemSourcing` → **New → Metadata Extension** → nom `ZC_PurReqnItemSourcing`, template *annotate view*.

```abap
@Metadata.layer: #CORE

@UI: {
  headerInfo: {
    typeName:       'Poste de DA',
    typeNamePlural: 'Postes de DA',
    title:          { type: #STANDARD, value: 'PurchaseRequisition' },
    description:    { type: #STANDARD, value: 'PurchaseRequisitionItemText' }
  },
  presentationVariant: [{
    sortOrder:      [{ by: 'DeliveryDate', direction: #ASC }],
    visualizations: [{ type: #AS_LINEITEM }]
  }],
  selectionVariant: [
    { qualifier: 'SansSource', text: 'Sans source',
      parameters: [{ name: 'SourceOfSupplyIsAssigned', value: '' }] },
    { qualifier: 'AvecSource', text: 'Avec source',
      parameters: [{ name: 'SourceOfSupplyIsAssigned', value: 'X' }] }
  ]
}
annotate view ZC_PurReqnItemSourcing with
{
  @UI.facet: [
    { id: 'HeaderDelai',   purpose: #HEADER, type: #DATAPOINT_REFERENCE,
      targetQualifier: 'DaysToDelivery', position: 10, label: 'Delai' },
    { id: 'General',       purpose: #STANDARD, type: #FIELDGROUP_REFERENCE,
      targetQualifier: 'General', position: 10, label: 'Informations generales' },
    { id: 'Source',        purpose: #STANDARD, type: #FIELDGROUP_REFERENCE,
      targetQualifier: 'Source',  position: 20, label: 'Source d''approvisionnement' },
    { id: 'Candidates',    purpose: #STANDARD, type: #LINEITEM_REFERENCE,
      position: 30, label: 'Sources candidates', targetElement: '_SourceCandidate' }
  ]

  @UI.lineItem:       [{ position: 10, label: 'Demande d''achat', importance: #HIGH }]
  @UI.identification: [{ position: 10 }]
  @UI.fieldGroup:     [{ qualifier: 'General', position: 10 }]
  PurchaseRequisition;

  @UI.lineItem:       [{ position: 20, label: 'Poste', importance: #HIGH }]
  @UI.fieldGroup:     [{ qualifier: 'General', position: 20 }]
  PurchaseRequisitionItem;

  @UI.hidden: true
  PurchaseRequisitionType;

  @UI.lineItem:       [{ position: 30, label: 'Designation', importance: #HIGH }]
  @UI.fieldGroup:     [{ qualifier: 'General', position: 30 }]
  PurchaseRequisitionItemText;

  @UI.lineItem:       [{ position: 40, label: 'Article', importance: #HIGH }]
  @UI.selectionField: [{ position: 30 }]
  @UI.fieldGroup:     [{ qualifier: 'General', position: 40 }]
  @UI.textArrangement: #TEXT_LAST
  Material;

  @UI.lineItem:       [{ position: 50, label: 'Groupe march.', importance: #LOW }]
  MaterialGroup;

  @UI.lineItem:       [{ position: 60, label: 'Division', importance: #HIGH }]
  @UI.selectionField: [{ position: 10 }]
  @UI.fieldGroup:     [{ qualifier: 'General', position: 50 }]
  Plant;

  @UI.lineItem:       [{ position: 70, label: 'Groupe acheteurs', importance: #MEDIUM }]
  @UI.selectionField: [{ position: 20 }]
  PurchasingGroup;

  @UI.lineItem:       [{ position: 80, label: 'Quantite', importance: #HIGH }]
  @UI.fieldGroup:     [{ qualifier: 'General', position: 60 }]
  RequestedQuantity;

  @UI.lineItem:       [{ position: 90, label: 'Livraison souhaitee', importance: #HIGH }]
  @UI.fieldGroup:     [{ qualifier: 'General', position: 70 }]
  DeliveryDate;

  @UI.lineItem:       [{ position: 100, label: 'Fournisseur fixe', importance: #HIGH,
                         criticality: 'SourcingCriticality' }]
  @UI.fieldGroup:     [{ qualifier: 'Source', position: 10 }]
  FixedSupplier;

  @UI.lineItem:       [{ position: 110, label: 'Contrat', importance: #MEDIUM }]
  @UI.fieldGroup:     [{ qualifier: 'Source', position: 20 }]
  PurchaseContract;

  @UI.fieldGroup:     [{ qualifier: 'Source', position: 30 }]
  PurchaseContractItem;

  @UI.fieldGroup:     [{ qualifier: 'Source', position: 40 }]
  PurchasingInfoRecord;

  @UI.fieldGroup:     [{ qualifier: 'Source', position: 50 }]
  SupplyingPlant;

  @UI.lineItem:       [{ position: 120, label: 'Source affectee', importance: #HIGH,
                         criticality: 'SourcingCriticality' }]
  @UI.selectionField: [{ position: 40 }]
  SourceOfSupplyIsAssigned;

  @UI.lineItem:       [{ position: 130, label: 'Jours avant livraison',
                         type: #AS_DATAPOINT, importance: #MEDIUM }]
  @UI.dataPoint:      { title: 'Jours avant livraison',
                        criticality: 'DeliveryCriticality',
                        visualization: #PROGRESS,
                        targetValue: 30,
                        qualifier: 'DaysToDelivery' }
  DaysToDelivery;

  @UI.hidden: true
  SourcingCriticality;
  @UI.hidden: true
  DeliveryCriticality;
}
```

Métadonnées de la vue des sources (fichier `ZC_SourceOfSupplyCandidate.ddlx`) :

```abap
@Metadata.layer: #CORE
annotate view ZC_SourceOfSupplyCandidate with
{
  @UI.lineItem: [{ position: 10, label: 'Fournisseur', importance: #HIGH,
                   criticality: 'SourceCriticality' }]
  Supplier;

  @UI.lineItem: [{ position: 20, label: 'Org. achats', importance: #HIGH }]
  PurchasingOrganization;

  @UI.lineItem: [{ position: 30, label: 'Contrat', importance: #MEDIUM }]
  PurchaseAgreement;

  @UI.lineItem: [{ position: 40, label: 'Fixe', importance: #MEDIUM }]
  IsFixedSupplier;

  @UI.lineItem: [{ position: 50, label: 'Valide jusqu''au', importance: #LOW }]
  ValidityEndDate;

  @UI.hidden: true
  SourceCriticality;
}
```

Activer les deux metadata extensions.

> **Onglets de variantes** (`selectionVariant` ci-dessus) : ils nécessitent un réglage complémentaire dans le `manifest.json` de l'application (`quickVariantSelectionX`), ajouté à l'[étape 9.3](#93-onglets-de-variantes-optionnel).

📸 *Capture 4-05 : metadata extension dans ADT*

---

## Étape 7 – Service definition

Clic droit sur `ZC_PurReqnItemSourcing` → **New Service Definition** :

| Champ | Valeur |
|---|---|
| Name | `ZUI_PR_SOURCING` |
| Description | `Determination source appro - DA ZCTL` |
| Package / OT | `ZMM_SOURCING` / `S4DK9xxxxx` |
| Template | `defineService` |

```abap
@EndUserText.label: 'Determination source appro - DA ZCTL'
define service ZUI_PR_SOURCING {
  expose ZC_PurReqnItemSourcing    as PurReqnItemSourcing;
  expose ZC_SourceOfSupplyCandidate as SourceOfSupplyCandidate;
  expose I_Supplier                as Supplier;
  expose I_Product                 as Product;
  expose I_PlantStdVH              as PlantVH;
  expose I_ProductStdVH            as ProductVH;
}
```

Activer (`Ctrl+F3`).

> Les entités `…StdVH` alimentent les aides à la saisie (F4) déclarées par `@Consumption.valueHelpDefinition`. Si un nom n'existe pas sur votre release, ADT le signale à l'activation : le retirer ou le remplacer.

---

## Étape 8 – Service binding OData V2 et publication

1. Clic droit sur la service definition → **New Service Binding** :

| Champ | Valeur |
|---|---|
| Name | `ZUI_PR_SOURCING_O2` |
| Description | `Determination source appro - OData V2 UI` |
| Binding Type | **OData V2 – UI** |
| Service Definition | `ZUI_PR_SOURCING` |

2. **Activer** le service binding (`Ctrl+F3`).
3. Cliquer sur **Publish** (bouton *Local Service Endpoint* dans l'éditeur du service binding). Les listes de tâches s'exécutent et le service est enregistré dans le hub local.
4. L'éditeur affiche l'URL de service et l'arborescence des entity sets :

```text
Service Binding  ZUI_PR_SOURCING_O2       Binding Type: OData V2 - UI
┌───────────────────────────────────────────────────────────────────────┐
│ Service Information        Local Service Endpoint   [ Publish ]       │
│ Service URL  /sap/opu/odata/sap/ZUI_PR_SOURCING_O2                    │
│ ▾ Entity Set and Association                                          │
│    • PurReqnItemSourcing          [ Preview ]                         │
│    • SourceOfSupplyCandidate      [ Preview ]                         │
└───────────────────────────────────────────────────────────────────────┘
```

5. **Contrôles :**

| Contrôle | Où | Attendu |
|---|---|---|
| Service enregistré | `/IWFND/MAINT_SERVICES` → filtre `ZUI_PR_SOURCING_O2` | Service présent, alias système `LOCAL`, nœud ICF vert |
| Métadonnées | `/sap/opu/odata/sap/ZUI_PR_SOURCING_O2/$metadata` | EntityType `PurReqnItemSourcingType` avec les annotations `UI.*` |
| Données | `/sap/opu/odata/sap/ZUI_PR_SOURCING_O2/PurReqnItemSourcing?$top=5&$format=json` | Postes de DA `ZCTL` |
| Aperçu Fiori elements | Bouton **Preview** sur l'entity set | List Report généré avec les colonnes annotées |

📸 *Capture 4-06 : éditeur du service binding après publication*
📸 *Capture 4-07 : aperçu Fiori elements depuis ADT*

---

## Étape 9 – Générer l'application Fiori elements

### 9.1 Connexion au système

**VS Code** (extension pack SAP Fiori tools) : palette `Ctrl+Shift+P` → `Fiori: Add SAP System` → saisir nom (`S4H_DEV_100`), URL `https://<host>:<port>`, mandant `100`, identifiants.
**BAS** : destination BTP + Cloud Connector (voir document 01, §2.4 et 2.5).

### 9.2 Générateur d'application

Palette → `Fiori: Open Application Generator` :

| Écran | Champ | Valeur |
|---|---|---|
| Template | Type | **List Report Page** (OData V2) |
| Data Source | Source | *Connect to a System* → `S4H_DEV_100` |
| | Service | `ZUI_PR_SOURCING_O2` (`/sap/opu/odata/sap/ZUI_PR_SOURCING_O2`) |
| Entity Selection | Main entity | `PurReqnItemSourcing` |
| | Navigation entity | `_SourceCandidate` (pour la table de l'Object Page) |
| | Automatically add table columns | Oui |
| Project Attributes | Module name | `zmmprsourcing` |
| | Application title | `Determination source appro` |
| | Application namespace | (vide) |
| | Description | `DA ZCTL et sources candidates` |
| | Minimum SAPUI5 version | `1.96.x` |
| | Add deployment configuration | **Oui** |
| | Add FLP configuration | Non (fait côté ABAP, étape 12) |
| Deployment | Target | ABAP — `S4H_DEV_100` |
| | SAPUI5 ABAP Repository | `ZMM_PR_SRC` |
| | Description | `Determination source appro` |
| | Package | `ZMM_SOURCING` |
| | Transport Request | `S4DK9xxxxx` |

Structure générée :

```text
zmmprsourcing/
├── ui5.yaml / ui5-deploy.yaml / ui5-local.yaml
├── package.json
└── webapp/
    ├── manifest.json
    ├── Component.js
    ├── i18n/i18n.properties
    ├── annotations/annotation.xml      (annotations locales éventuelles)
    └── ext/                            (créé à l'étape 10)
```

### 9.3 Onglets de variantes (optionnel)

Dans `webapp/manifest.json`, section `sap.ui.generic.app` → page `ListReport|PurReqnItemSourcing` → `component.settings` :

```json
"settings": {
  "quickVariantSelectionX": {
    "showCounts": true,
    "variants": {
      "0": { "key": "all",  "annotationPath": "" },
      "1": { "key": "sans", "annotationPath": "com.sap.vocabularies.UI.v1.SelectionVariant#SansSource" },
      "2": { "key": "avec", "annotationPath": "com.sap.vocabularies.UI.v1.SelectionVariant#AvecSource" }
    }
  }
}
```

📸 *Capture 4-08 : générateur — sélection du service*
📸 *Capture 4-09 : générateur — entités*

---

## Étape 10 – Bouton personnalisé et boîte de dialogue

### 10.1 Déclarer l'action dans le manifest

Dans `webapp/manifest.json`, sous `"sap.ui5"`, ajouter (ou compléter) la section `extends` :

```json
"extends": {
  "extensions": {
    "sap.ui.controllerExtensions": {
      "sap.suite.ui.generic.template.ListReport.view.ListReport": {
        "controllerName": "zmmprsourcing.ext.controller.ListReportExt",
        "sap.ui.generic.app": {
          "PurReqnItemSourcing": {
            "EntitySet": "PurReqnItemSourcing",
            "Actions": {
              "openSourceDialog": {
                "id": "openSourceDialogBtn",
                "text": "{i18n>ASSIGN_SOURCE}",
                "press": "onOpenSourceDialog",
                "requiresSelection": true
              }
            }
          }
        }
      }
    }
  }
}
```

Dans `webapp/i18n/i18n.properties` :

```properties
ASSIGN_SOURCE=Affecter une source
SRC_DIALOG_TITLE=Sources candidates
SRC_SELECT_ONE=Sélectionnez une ligne de DA
```

> `requiresSelection: true` désactive le bouton tant qu'aucune ligne n'est sélectionnée : comportement visible dès l'ouverture de l'application, sans données.

### 10.2 Fragment de la boîte de dialogue

`webapp/ext/fragment/SourceOfSupplyDialog.fragment.xml` :

```xml
<core:FragmentDefinition xmlns="sap.m" xmlns:core="sap.ui.core">
    <Dialog id="sourceDialog"
            title="{i18n>SRC_DIALOG_TITLE}"
            contentWidth="42rem"
            contentHeight="24rem"
            resizable="true"
            draggable="true">
        <content>
            <MessageStrip id="srcContextStrip"
                          text=""
                          type="Information"
                          showIcon="true"
                          class="sapUiSmallMargin"/>
            <Table id="sourceTable"
                   items="{ path: '/SourceOfSupplyCandidate' }"
                   noDataText="Aucune source candidate pour cet article et cette division"
                   growing="true"
                   growingThreshold="20">
                <columns>
                    <Column><Text text="Fournisseur"/></Column>
                    <Column><Text text="Org. achats"/></Column>
                    <Column><Text text="Contrat"/></Column>
                    <Column hAlign="Center"><Text text="Fixe"/></Column>
                    <Column><Text text="Valide jusqu'au"/></Column>
                </columns>
                <items>
                    <ColumnListItem>
                        <cells>
                            <ObjectIdentifier title="{Supplier}" text="{_Supplier/SupplierName}"/>
                            <Text text="{PurchasingOrganization}"/>
                            <Text text="{PurchaseAgreement}"/>
                            <CheckBox selected="{path: 'IsFixedSupplier', formatter: '.formatBoolean'}"
                                      editable="false"/>
                            <Text text="{path: 'ValidityEndDate', type: 'sap.ui.model.type.Date',
                                         formatOptions: { style: 'medium' } }"/>
                        </cells>
                    </ColumnListItem>
                </items>
            </Table>
        </content>
        <beginButton>
            <Button id="assignBtn"
                    text="Affecter"
                    type="Emphasized"
                    enabled="false"
                    tooltip="Disponible en version 2"/>
        </beginButton>
        <endButton>
            <Button id="closeBtn" text="Fermer" press=".onCloseSourceDialog"/>
        </endButton>
    </Dialog>
</core:FragmentDefinition>
```

### 10.3 Extension de contrôleur

`webapp/ext/controller/ListReportExt.controller.js` :

```javascript
sap.ui.define([
    "sap/ui/core/Fragment",
    "sap/ui/model/Filter",
    "sap/ui/model/FilterOperator",
    "sap/m/MessageToast"
], function (Fragment, Filter, FilterOperator, MessageToast) {
    "use strict";

    return {

        /**
         * Ouvre la boîte de dialogue des sources candidates
         * pour la ligne de DA sélectionnée (lecture seule en V1).
         */
        onOpenSourceDialog: function () {
            var oView = this.getView();
            var aContexts = this.extensionAPI.getSelectedContexts();

            if (!aContexts || aContexts.length !== 1) {
                MessageToast.show(oView.getModel("i18n").getResourceBundle()
                                       .getText("SRC_SELECT_ONE"));
                return;
            }

            var oItem = aContexts[0].getObject();
            var that  = this;

            if (!this._pDialog) {
                this._pDialog = Fragment.load({
                    id:         oView.getId(),
                    name:       "zmmprsourcing.ext.fragment.SourceOfSupplyDialog",
                    controller: this
                }).then(function (oDialog) {
                    oView.addDependent(oDialog);
                    return oDialog;
                });
            }

            this._pDialog.then(function (oDialog) {
                // Contexte affiché dans le bandeau
                Fragment.byId(oView.getId(), "srcContextStrip").setText(
                    "DA " + oItem.PurchaseRequisition + " / poste " + oItem.PurchaseRequisitionItem +
                    " — article " + (oItem.Material || "sans article") +
                    " — division " + oItem.Plant);

                // Filtrage des sources sur article + division
                var oTable = Fragment.byId(oView.getId(), "sourceTable");
                var oBinding = oTable.getBinding("items");
                if (oBinding) {
                    oBinding.filter([
                        new Filter("Material", FilterOperator.EQ, oItem.Material),
                        new Filter("Plant",    FilterOperator.EQ, oItem.Plant)
                    ]);
                }
                oDialog.open();
            });
        },

        onCloseSourceDialog: function () {
            this._pDialog.then(function (oDialog) { oDialog.close(); });
        },

        formatBoolean: function (sValue) {
            return sValue === "X" || sValue === true;
        }
    };
});
```

> **Règle Fiori elements** : n'utiliser que l'`extensionAPI` (`getSelectedContexts`, `refreshTable`, `securedExecution`…) et ne jamais manipuler directement les contrôles générés par le framework.

📸 *Capture 4-10 : arborescence `ext/` dans VS Code*
📸 *Capture 4-11 : pop-up ouverte avec les sources*

---

## Étape 11 – Test en local

```bash
cd zmmprsourcing
npm install
npm start           # ouvre /test/flpSandbox.html#zmmprsourcing-tile
```

Liste de contrôle :

| Contrôle | Attendu |
|---|---|
| Barre de filtres | Division, Groupe acheteurs, Article, Source affectée, recherche libre |
| Onglets | Toutes / Sans source / Avec source (si §9.3 appliqué) |
| Bouton *Affecter une source* | Visible et **désactivé** sans sélection |
| Après *Exécuter* | Colonnes annotées, statut coloré, indicateur de progression « Jours avant livraison » |
| Sélection d'une ligne + bouton | Pop-up avec les sources filtrées sur article/division |
| Object Page (clic sur une ligne) | En-tête + sections *Informations générales*, *Source d'approvisionnement*, tableau *Sources candidates* |
| Console `F12` | Aucune erreur `Fragment.load` / binding |

Astuces :
- `npm run start-noflp` pour lancer l'application hors sandbox FLP ;
- `?sap-ui-debug=true` pour déboguer l'extension ;
- `npm run start-mock` si un fichier de données mock a été généré (utile pour valider les annotations sans backend).

📸 *Capture 4-12 : List Report en local*

---

## Étape 12 – Déploiement et Launchpad

### 12.1 Déploiement du BSP

```bash
npm run deploy      # utilise ui5-deploy.yaml
```

`ui5-deploy.yaml` (extrait) :

```yaml
builder:
  customTasks:
    - name: deploy-to-abap
      afterTask: generateCachebusterInfo
      configuration:
        target:
          url: https://<host>:<port>
          client: "100"
        app:
          name: ZMM_PR_SRC
          description: Determination source appro
          package: ZMM_SOURCING
          transport: S4DK9xxxxx
        exclude:
          - /test/
```

Contrôles backend : `SE80` → *Application BSP* `ZMM_PR_SRC` ; `/UI5/APP_INDEX_CALCULATE`.

### 12.2 Objet sémantique

Transaction **`/UI2/SEMOBJ`** (objets sémantiques client-spécifiques) → *Nouvelles entrées* :

| Champ | Valeur |
|---|---|
| Objet sémantique | `ZSourceOfSupply` |
| Nom de l'objet sémantique | `Determination source appro` |
| Description | `Objet semantique application Z sourcing DA` |

Enregistrer dans un ordre **Customizing**.

### 12.3 Catalogue, tuile, target mapping

`/UI2/FLPAM` (Launchpad App Manager) ou `/UI2/FLPD_CUST` :

| Élément | Champ | Valeur |
|---|---|---|
| Catalogue technique | Nom | `Z_TC_MM_SOURCING` |
| App descriptor / target mapping | Application Type | `SAPUI5 Fiori App` |
| | Semantic Object | `ZSourceOfSupply` |
| | Action | `determine` |
| | URL | `/sap/bc/ui5_ui5/sap/zmm_pr_src` |
| | ID (composant SAPUI5) | `zmmprsourcing` |
| | Fiori ID | `ZF0001` |
| | Device Types | Desktop, Tablet |
| Tuile (*App Launcher – Static*) | Titre | `Détermination source appro` |
| | Sous-titre | `DA ZCTL` |
| | Icône | `sap-icon://supplier` |
| | Semantic Object / Action | `ZSourceOfSupply` / `determine` |
| Catalogue métier | Nom | `Z_BC_MM_SOURCING` (référence la tuile et le target mapping) |

### 12.4 Rôle PFCG

`PFCG` → `Z_BR_SOURCING` :

1. Menu → *SAP Fiori Launchpad* → **Launchpad Catalog** `Z_BC_MM_SOURCING` (+ espace/page ou groupe).
2. Autorisations : générer, puis vérifier
   - `S_SERVICE` pour `ZUI_PR_SOURCING_O2`,
   - `M_BANF_WRK` / `M_BANF_EKG` / `M_BANF_EKO` / `M_BANF_BSA` en activité `03`,
   - `S_RFC` si nécessaire selon votre paysage.
3. `SU01` → affecter au testeur.

### 12.5 Test dans le Launchpad

```text
https://<host>:<port>/sap/bc/ui2/flp?sap-client=100&sap-language=FR#ZSourceOfSupply-determine
```

📸 *Capture 4-13 : `/UI2/SEMOBJ`*
📸 *Capture 4-14 : target mapping*
📸 *Capture 4-15 : application dans le FLP*

---

## Étape 13 – Transport

| Ordre | Objets |
|---|---|
| Workbench | `R3TR DEVC ZMM_SOURCING`, `R3TR DDLS ZI_PurReqnItemSourcing`, `ZI_SourceOfSupplyCandidate`, `ZC_PurReqnItemSourcing`, `ZC_SourceOfSupplyCandidate`, `R3TR DCLS ZI_PURREQNITEM_SOURCING`, `R3TR DDLX ZC_PURREQNITEMSOURCING` (+ celle des sources), `R3TR SRVD ZUI_PR_SOURCING`, `R3TR SRVB ZUI_PR_SOURCING_O2`, `R3TR WAPA ZMM_PR_SRC` |
| Customizing | Objet sémantique `/UI2/SEMOBJ`, contenu FLP client-spécifique |
| Workbench FLP | Catalogues/tuiles cross-client selon votre configuration |

Actions après import dans le système cible :

1. `/IWFND/MAINT_SERVICES` : vérifier la présence de `ZUI_PR_SOURCING_O2` (alias `LOCAL`). Si absent, ajouter le service ou republier le service binding depuis ADT sur le système cible.
2. `/UI5/APP_INDEX_CALCULATE` pour `ZMM_PR_SRC`.
3. `/UI2/INVALIDATE_GLOBAL_CACHES`, `/IWFND/CACHE_CLEANUP`.
4. Test fonctionnel avec un utilisateur porteur du rôle `Z_BR_SOURCING`.

> Le détail du transport DEV → QUALITÉ, commun avec l'adaptation project, est traité dans le document **05**, §Transport.

---

## Annexe A – Catalogue des annotations démontrées

| Annotation | Où | Effet visuel (Fiori elements V2 / UI5 1.96) |
|---|---|---|
| `@UI.headerInfo` | Metadata ext. (en-tête) | Titre et type d'objet de l'Object Page |
| `@UI.lineItem` (`position`, `label`, `importance`) | Par zone | Colonnes du tableau, priorité d'affichage responsive |
| `@UI.lineItem.criticality` | `FixedSupplier`, `SourceOfSupplyIsAssigned` | Statut coloré (rouge/orange/vert) dans la cellule |
| `@UI.lineItem` `type: #AS_DATAPOINT` + `@UI.dataPoint.visualization: #PROGRESS` | `DaysToDelivery` | Barre de progression dans le tableau |
| `@UI.dataPoint` + facette `#DATAPOINT_REFERENCE` `purpose: #HEADER` | `DaysToDelivery` | KPI dans l'en-tête de l'Object Page |
| `@UI.selectionField` | Division, Groupe acheteurs, Article, Source affectée | Champs de la barre de filtres |
| `@UI.selectionVariant` + `quickVariantSelectionX` | Metadata ext. + manifest | Onglets « Toutes / Sans source / Avec source » |
| `@UI.presentationVariant` | En-tête | Tri par défaut sur la date de livraison |
| `@UI.fieldGroup` + `@UI.facet` `#FIELDGROUP_REFERENCE` | Object Page | Sections *Informations générales* et *Source* |
| `@UI.facet` `#LINEITEM_REFERENCE` + `targetElement` | `_SourceCandidate` | Tableau des sources candidates dans l'Object Page |
| `@UI.identification` | `PurchaseRequisition` | Zone d'identification de l'en-tête |
| `@UI.hidden` | Zones techniques de criticité | Masquage dans l'UI tout en restant exploitable |
| `@UI.textArrangement` + `@ObjectModel.text.association` | `Material` | Affichage « code – désignation » |
| `@Consumption.valueHelpDefinition` | `Material`, `Plant` | Aide à la saisie F4 dans les filtres |
| `@Consumption.semanticObject` | `FixedSupplier`, `PurchaseContract` | Smart link avec menu de navigation contextuelle |
| `@Search.searchable` / `@Search.defaultSearchElement` / `@Search.fuzzinessThreshold` | Vue + zones | Recherche libre floue dans la barre de filtres |
| `@Semantics.quantity.unitOfMeasure` / `@Semantics.unitOfMeasure` | `RequestedQuantity` / `BaseUnit` | Quantité formatée avec son unité |
| `@Semantics.booleanIndicator` | `IsFixedSupplier`, `IsBlocked` | Rendu booléen (case à cocher) |
| `@ObjectModel.semanticKey` | Projection | Clé sémantique affichée en gras, navigation par clé lisible |

> Autres pistes de démonstration si vous voulez enrichir : `@UI.chart` + `@UI.lineItem type: #AS_CHART` (micro-graphiques), `@Semantics.contact` (quick view fournisseur), `@UI.criticality` sur `@UI.dataPoint` `#RATING`, `@UI.connectedFields`, `@UI.textArrangement: #TEXT_ONLY`.

---

## Annexe B – Ce qui est prévu en V2

| Sujet | Impact technique |
|---|---|
| Affectation réelle de la source | Passage à un **RAP BO unmanaged** ou **managed with unmanaged save** : behavior definition + classe d'implémentation appelant l'API standard d'affectation de source (BAPI / EML) |
| Action déclarée | `action AssignSourceOfSupply parameter ZD_AssignSource result [1] $self;` exposée dans la behavior projection |
| Boîte de dialogue | Remplacée par une action annotée `@UI.lineItem: [{ type: #FOR_ACTION }]` avec écran de paramètres généré |
| Sources candidates | Ajout des fiches info achat et des contrats via `union all` et zone `SourceType` |
| Contrôle | BAdI de contrôle avant sauvegarde de la DA (voir document 02 pour le pattern) |
| Traçabilité | Table Z d'historique des affectations + champ `LastChangedBy` |

---

## Dépannage

| Symptôme | Cause probable | Solution |
|---|---|---|
| Activation CDS : zone inconnue | Noms de zones différents sur votre SP | Ouvrir `I_PurchaseRequisitionItemAPI01` / `EORD` et corriger |
| Activation : `provider contract transactional_query` refusé | Attente d'une behavior definition | Retirer l'addition (vue de consommation simple) — cf. note §5.1 |
| `Publish` du service binding en erreur | Autorisations Gateway, listes de tâches | Journal du bouton *Publish*, `STC01` (task list), rôle Gateway |
| Service absent de `/IWFND/MAINT_SERVICES` | Publication non effectuée | Republier depuis ADT ou ajouter le service manuellement (alias `LOCAL`) |
| Aucune donnée dans l'app | Pas de DA `ZCTL`, DCL trop restrictif, filtre date | `ME53N`, tester la vue dans ADT (*Open With → Data Preview*), `SU53` |
| Annotations non visibles | Metadata extension non activée / cache Gateway | Activer la DDLX, `/IWFND/CACHE_CLEANUP` puis `/IWBEP/CACHE_CLEANUP` |
| Bouton personnalisé absent | Entity set mal orthographié dans le manifest | Le nom doit correspondre à l'alias du service definition (`PurReqnItemSourcing`) |
| `this.extensionAPI is undefined` | Handler défini hors de l'extension de contrôleur déclarée | Vérifier `controllerName` et le chemin du fichier `ext/controller/…` |
| Pop-up vide | Filtres sur des zones absentes du set, ou aucune entrée `EORD` | Tester `…/SourceOfSupplyCandidate?$filter=Material eq '…' and Plant eq '…'` dans le navigateur |
| Smart link sans cible | Objet sémantique sans target mapping | Normal si l'app cible n'est pas configurée |

---

## Captures à réaliser

| N° | Écran | Fichier suggéré |
|---|---|---|
| 4-01 | Package ADT | `img/04-01-package.png` |
| 4-02 | Vue d'interface DA | `img/04-02-zi-da.png` |
| 4-03 | Vue sources | `img/04-03-zi-sources.png` |
| 4-04 | Projections | `img/04-04-projections.png` |
| 4-05 | Metadata extension | `img/04-05-ddlx.png` |
| 4-06 | Service binding publié | `img/04-06-srvb.png` |
| 4-07 | Aperçu Fiori depuis ADT | `img/04-07-preview-adt.png` |
| 4-08 | Générateur — service | `img/04-08-gen-service.png` |
| 4-09 | Générateur — entités | `img/04-09-gen-entities.png` |
| 4-10 | Dossier `ext/` | `img/04-10-ext.png` |
| 4-11 | Pop-up sources | `img/04-11-dialog.png` |
| 4-12 | Test local | `img/04-12-local.png` |
| 4-13 | `/UI2/SEMOBJ` | `img/04-13-semobj.png` |
| 4-14 | Target mapping | `img/04-14-targetmapping.png` |
| 4-15 | App dans le FLP | `img/04-15-flp.png` |

---

## Références

- SAP Help – *ABAP RESTful Application Programming Model* : vues de projection, contrats de fournisseur, service definition / binding
- SAP Learning – *Defining an OData UI Service* (publication du *local service endpoint*, choix V2/V4)
- SAPUI5 Demo Kit – *Adding Custom Actions Using Extension Points* (SAP Fiori elements pour OData V2)
- SAPUI5 Demo Kit – *Using the extensionAPI*
- SAP Help – *CDS Annotations* : `@UI`, `@Consumption`, `@Search`, `@Semantics`, `@ObjectModel`
- SAP Fiori Apps Reference Library – F1048 (application étendue dans le document 05)
- Documents internes : `01_Extension_UI_Adaptation_Project_F0842A.md`, `02_Extension_BAdI_F0842A.md`, `03_Key_User_Extensibility_F0842A.md`
