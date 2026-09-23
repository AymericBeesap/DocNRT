# Application spécifique « Détermination Source Appro » — V1 (révision 2)

**Contexte :** SAP S/4HANA Cloud Private Edition 2021 FPS02 (ABAP Platform 2021 / SAP_BASIS 7.56, SAP_UI 7.56, SAPUI5 1.96.x)
**Objet :** créer de zéro le modèle CDS, le service OData et l'application SAP Fiori elements (List Report / Object Page, OData V2) affichant les postes de demande d'achat de type `ZCTL` avec leur source d'approvisionnement, plus une boîte de dialogue de consultation des sources candidates.
**Périmètre V1 :** affichage uniquement. Aucune écriture, aucune action.

> **Révision 2** — remplace la première version du document. Le modèle CDS a été revu pour supprimer les causes d'échec d'activation : plus d'association croisée entre vues d'interface, plus de couche de projection ni de `provider contract`, plus d'annotations dépendantes d'objets standard incertains. Les annotations UI sont désormais livrées en deux niveaux, activables l'un après l'autre.

---

## Sommaire

1. [Ce qui a changé et pourquoi](#1-ce-qui-a-changé-et-pourquoi)
2. [Architecture simplifiée](#2-architecture-simplifiée)
3. [Prérequis](#3-prérequis)
4. [Étape 0 – Vérifier les objets standard avant d'écrire une ligne](#étape-0--vérifier-les-objets-standard-avant-décrire-une-ligne)
5. [Étape 1 – Package et ordre de transport](#étape-1--package-et-ordre-de-transport)
6. [Étape 2 – Vue d'interface des sources](#étape-2--vue-dinterface-des-sources)
7. [Étape 3 – Vue d'interface des postes de DA](#étape-3--vue-dinterface-des-postes-de-da)
8. [Étape 4 – Vue de consommation des sources](#étape-4--vue-de-consommation-des-sources)
9. [Étape 5 – Vue de consommation des postes de DA](#étape-5--vue-de-consommation-des-postes-de-da)
10. [Étape 6 – Annotations UI niveau 1](#étape-6--annotations-ui-niveau-1)
11. [Étape 7 – Annotations UI niveau 2](#étape-7--annotations-ui-niveau-2)
12. [Étape 8 – Contrôle d'accès (DCL)](#étape-8--contrôle-daccès-dcl)
13. [Étape 9 – Service definition et service binding](#étape-9--service-definition-et-service-binding)
14. [Étape 10 – Générer l'application Fiori elements](#étape-10--générer-lapplication-fiori-elements)
15. [Étape 11 – Bouton personnalisé et boîte de dialogue](#étape-11--bouton-personnalisé-et-boîte-de-dialogue)
16. [Étape 12 – Test local, déploiement, Launchpad, transport](#étape-12--test-local-déploiement-launchpad-transport)
17. [Annexe A – Erreurs d'activation ADT : message par message](#annexe-a--erreurs-dactivation-adt--message-par-message)
18. [Annexe B – Plan de repli : modèle basé sur EBAN](#annexe-b--plan-de-repli--modèle-basé-sur-eban)
19. [Annexe C – Préparer la V2 (couche de projection RAP)](#annexe-c--préparer-la-v2-couche-de-projection-rap)
20. [Annexe D – Catalogue des annotations démontrées](#annexe-d--catalogue-des-annotations-démontrées)
21. [Captures à réaliser](#captures-à-réaliser)
22. [Références](#références)

---

## 1. Ce qui a changé et pourquoi

| Problème rencontré | Cause | Correction apportée dans cette révision |
|---|---|---|
| Les vues ne s'activent pas, chacune réclame l'autre | `ZI_PurReqnItemSourcing` déclarait une association vers `ZI_SourceOfSupplyCandidate` : dépendance croisée si l'ordre de création n'est pas respecté | **L'association est descendue au niveau consommation uniquement.** Les deux vues d'interface sont totalement indépendantes |
| Erreur sur `root` / entité racine | `define root view entity` n'a de sens que pour la racine d'un business object RAP | **Plus aucun `root` en V1.** Il n'apparaît qu'en V2 (annexe C) |
| Erreur sur la projection | `provider contract transactional_query` impose de redirriger toutes les associations exposées et attend un contexte de business object | **Plus de vue de projection.** Les vues de consommation sont de simples `define view entity … as select from …` |
| Erreur sur `redirected to composition child` | Syntaxe réservée aux **compositions** d'un BO ; une association normale se redirige avec `redirected to <vue>` | Redirection supprimée : l'association du niveau consommation pointe directement vers la vue de consommation cible |
| Erreurs sur des annotations | `@ObjectModel.text.association` vers une vue non textuelle, `qualifier` inexistant dans `@UI.dataPoint`, aides à la saisie vers des vues `…StdVH` non garanties | Annotations remaniées et **scindées en niveau 1 (sûr) et niveau 2 (optionnel)** |
| Erreur sur la quantité | Une zone quantité doit référencer son unité | `@Semantics.quantity.unitOfMeasure` + exposition de `BaseUnit` : rendu obligatoire et expliqué |
| Avertissements d'autorisation | `#CHECK` sans DCL | On démarre en `#NOT_REQUIRED`, le DCL est ajouté à l'étape 8 |

**Principe directeur de cette révision :** chaque vue s'active seule, dans l'ordre, et se teste immédiatement avec le *Data Preview* avant de passer à la suivante.

---

## 2. Architecture simplifiée

```text
Étape 2   ZI_SourceOfSupplyCandidate     ← EORD                     (aucune dépendance Z)
Étape 3   ZI_PurReqnItemSourcing         ← I_PurchaseRequisitionItemAPI01   (aucune dépendance Z)
             │                                   │
Étape 4   ZC_SourceOfSupplyCandidate  ────┘      │
             │                                   │
Étape 5   ZC_PurReqnItemSourcing  ───────────────┘
             │   association [0..*] _SourceCandidate → ZC_SourceOfSupplyCandidate
             │
Étape 6/7 ZC_PurReqnItemSourcing.ddlx + ZC_SourceOfSupplyCandidate.ddlx  (annotations UI)
Étape 9   ZUI_PR_SOURCING (service definition) → ZUI_PR_SOURCING_O2 (binding OData V2 UI)
Étape 10  Application Fiori elements « zmmprsourcing »
```

Les flèches ne vont que dans un sens. Il n'existe **aucune référence circulaire**, et l'association unique du modèle est déclarée au dernier niveau.

| Objet | Type | Créé à l'étape |
|---|---|---|
| `ZI_SourceOfSupplyCandidate` | DDLS (vue d'interface) | 2 |
| `ZI_PurReqnItemSourcing` | DDLS (vue d'interface) | 3 |
| `ZC_SourceOfSupplyCandidate` | DDLS (vue de consommation) | 4 |
| `ZC_PurReqnItemSourcing` | DDLS (vue de consommation) | 5 |
| `ZC_SourceOfSupplyCandidate` | DDLX (metadata extension) | 6 |
| `ZC_PurReqnItemSourcing` | DDLX (metadata extension) | 6 et 7 |
| `ZI_PURREQNITEM_SOURCING` | DCLS | 8 |
| `ZUI_PR_SOURCING` / `ZUI_PR_SOURCING_O2` | SRVD / SRVB | 9 |
| `ZMM_PR_SRC` | WAPA (application BSP) | 12 |

---

## 3. Prérequis

| Élément | Valeur / contrôle | Où |
|---|---|---|
| Type de document DA `ZCTL` | Existe, avec des DA créées | `SPRO` → MM → Achats → Demande d'achat → Définir types de document (`V_T161`) ; `ME53N` |
| Liste des sources alimentée | Entrées dans `EORD` pour l'article/division testés | `ME03`, `SE16N` sur `EORD` |
| ADT | Eclipse + ABAP Development Tools à jour, projet ABAP connecté | — |
| Fiori tools | VS Code (ou BAS) + Node.js LTS | — |
| Nœuds ICF | `/sap/opu/odata`, `/sap/bc/ui5_ui5`, `/sap/bc/lrep`, `/sap/bc/ui2`, `/sap/bc/adt` actifs | `SICF` |
| Autorisations développeur | `S_DEVELOP` (`DDLS`, `DDLX`, `DCLS`, `SRVD`, `SRVB`, `WAPA`, `DEVC`), `S_TRANSPRT`, `S_ADT_RES` | `PFCG` |
| Autorisations Gateway | Publication du service, `/IWFND/MAINT_SERVICES` | `PFCG` |
| Utilisateur de test | `M_BANF_WRK`, `M_BANF_EKG`, `M_BANF_EKO`, `M_BANF_BSA` en activité `03` | `PFCG` / `SU01` |

---

## Étape 0 – Vérifier les objets standard avant d'écrire une ligne

C'est l'étape qui évite 80 % des erreurs d'activation. Elle prend cinq minutes.

### 0.1 La vue standard des postes de DA existe-t-elle ?

1. Dans ADT : `Ctrl+Shift+A` → saisir `I_PurchaseRequisitionItemAPI01` → *Open*.
2. Si la vue n'existe pas sur votre release, essayer dans l'ordre : `I_PurchaseRequisitionItem`, `I_PurchaseReqnItem`, puis se reporter à l'[annexe B](#annexe-b--plan-de-repli--modèle-basé-sur-eban) (modèle direct sur `EBAN`).

### 0.2 Les zones que j'utilise existent-elles, et sous quel nom ?

Dans la vue ouverte :

| Action | Résultat |
|---|---|
| `Ctrl+O` (Quick Outline) puis saisir un morceau de nom (`Supplier`, `Contract`, `Source`…) | Liste filtrée des éléments |
| Curseur sur un élément + `F2` (Element Info) | Type, élément de données, annotations |
| Clic droit sur la vue → *Open With → Data Preview* (ou `F8`) | Contenu réel : on voit immédiatement si `ZCTL` existe et si les zones source sont alimentées |

Relever et noter dans un tableau les noms exacts :

| Rôle | Nom attendu | Nom sur votre système |
|---|---|---|
| N° de DA | `PurchaseRequisition` | |
| Poste | `PurchaseRequisitionItem` | |
| Type de document | `PurchaseRequisitionType` | |
| Texte du poste | `PurchaseRequisitionItemText` | |
| Article | `Material` | |
| Groupe de marchandises | `MaterialGroup` | |
| Division | `Plant` | |
| Groupe d'acheteurs | `PurchasingGroup` | |
| Organisation d'achats | `PurchasingOrganization` | |
| Quantité | `RequestedQuantity` | |
| Unité | `BaseUnit` | |
| Date de livraison | `DeliveryDate` | |
| Fournisseur fixe | `FixedSupplier` | |
| Division de livraison | `SupplyingPlant` | |
| Fiche info achat | `PurchasingInfoRecord` | |
| Contrat | `PurchaseContract` | |
| Poste de contrat | `PurchaseContractItem` | |
| Indicateur source affectée | `SourceOfSupplyIsAssigned` | |

> Si une zone manque, **retirez-la simplement** des vues ci-dessous : aucune n'est structurante sauf les clés, `PurchaseRequisitionType`, `Material` et `Plant`.

### 0.3 La table EORD et ses zones

`SE11` → `EORD` → onglet *Zones*. Vérifier : `MATNR`, `WERKS`, `ZEORD`, `LIFNR`, `EKORG`, `EBELN`, `EBELP`, `RESWK`, `FLIFN`, `NOTKZ`, `VDATU`, `BDATU`.

### 0.4 Les vues d'aide à la saisie

Dans ADT, vérifier l'existence de `I_Plant`, `I_Product`, `I_Supplier`. On n'utilisera **que** des vues vérifiées ; les vues de type `…StdVH` ne sont pas employées dans cette révision.

📸 *Capture 4-01 : Element Info (F2) sur une zone de la vue standard*
📸 *Capture 4-02 : Data Preview de la vue standard filtrée sur ZCTL*

---

## Étape 1 – Package et ordre de transport

ADT : clic droit sur le projet → **New → ABAP Package**

| Champ | Valeur |
|---|---|
| Name | `ZMM_SOURCING` |
| Description | `Determination source approvisionnement - DA ZCTL` |
| Package Type | `Development` |
| Software Component | `HOME` |
| Application Component | `MM-PUR-REQ` |
| Transport Layer | couche Z du paysage |

→ *Next* → **Create new request** : `App determination source appro V1`. Noter le numéro `S4DK9xxxxx`.

> **Astuce activation** : ADT permet de créer plusieurs objets puis de tout activer d'un coup (`Ctrl+Shift+F3`, *Activate inactive ABAP development objects*). Le framework résout alors lui-même l'ordre des dépendances. Dans ce document, on active néanmoins objet par objet : c'est plus lent mais chaque erreur est localisée immédiatement.

---

## Étape 2 – Vue d'interface des sources

**Toujours créer cette vue en premier** : elle ne dépend d'aucun objet Z.

ADT : clic droit sur `ZMM_SOURCING` → **New → Other ABAP Repository Object → Core Data Services → Data Definition**

| Champ | Valeur |
|---|---|
| Name | `ZI_SourceOfSupplyCandidate` |
| Description | `Sources appro candidates - liste des sources` |
| Transport Request | `S4DK9xxxxx` |
| Template | **Define View Entity** |

```abap
@EndUserText.label: 'Sources appro candidates - liste des sources'
@AccessControl.authorizationCheck: #NOT_REQUIRED
@Metadata.ignorePropagatedAnnotations: true
define view entity ZI_SourceOfSupplyCandidate
  as select from eord as SourceList

  association [0..1] to I_Supplier as _Supplier
    on $projection.Supplier = _Supplier.Supplier

{
  key SourceList.matnr as Material,
  key SourceList.werks as Plant,
  key SourceList.zeord as SourceListRecord,

      SourceList.lifnr as Supplier,
      SourceList.ekorg as PurchasingOrganization,
      SourceList.ebeln as PurchaseAgreement,
      SourceList.ebelp as PurchaseAgreementItem,
      SourceList.reswk as SupplyingPlant,
      SourceList.flifn as IsFixedSupplier,
      SourceList.notkz as IsBlocked,
      SourceList.vdatu as ValidityStartDate,
      SourceList.bdatu as ValidityEndDate,

      /* Associations */
      _Supplier
}
where
  SourceList.notkz = ''
```

**Activer** (`Ctrl+F3`).

**Test immédiat** : clic droit → *Open With → Data Preview*. Vous devez voir vos enregistrements de liste des sources. Si la vue est vide, ce n'est pas un problème de code : il n'y a pas d'entrées `EORD` non bloquées (`ME01` pour en créer).

Points de vigilance :

| Sujet | Détail |
|---|---|
| Alias `SourceList` | Obligatoire dès qu'on préfixe les zones ; évite toute ambiguïté |
| Pas de filtre de validité | Le filtre `bdatu >= $session.system_date` a été retiré : il masque les enregistrements dont la date de fin est vide. Si vous le voulez, ajoutez `and ( SourceList.bdatu = '00000000' or SourceList.bdatu >= $session.system_date )` |
| Pas d'annotation `@Semantics.booleanIndicator` | Elle exige un type `abap.char(1)` avec valeurs `X`/vide ; on la déplace au niveau consommation, en option |
| Aucune zone calculée | Les criticités sont calculées au niveau consommation |

📸 *Capture 4-03 : Data Preview de `ZI_SourceOfSupplyCandidate`*

---

## Étape 3 – Vue d'interface des postes de DA

Même procédure. **Cette vue ne référence aucune vue Z** : c'est le point clé de la correction.

| Champ | Valeur |
|---|---|
| Name | `ZI_PurReqnItemSourcing` |
| Description | `Postes de DA ZCTL - donnees sourcing` |

```abap
@EndUserText.label: 'Postes de DA ZCTL - donnees sourcing'
@AccessControl.authorizationCheck: #NOT_REQUIRED
@Metadata.ignorePropagatedAnnotations: true
define view entity ZI_PurReqnItemSourcing
  as select from I_PurchaseRequisitionItemAPI01 as PurReqnItem

  association [0..1] to I_Supplier as _Supplier
    on $projection.FixedSupplier = _Supplier.Supplier

{
      --- Clés
  key PurReqnItem.PurchaseRequisition,
  key PurReqnItem.PurchaseRequisitionItem,

      --- Identification du besoin
      PurReqnItem.PurchaseRequisitionType,
      PurReqnItem.PurchaseRequisitionItemText,
      PurReqnItem.Material,
      PurReqnItem.MaterialGroup,
      PurReqnItem.Plant,
      PurReqnItem.PurchasingGroup,
      PurReqnItem.PurchasingOrganization,

      --- Quantité : la zone quantité DOIT référencer son unité
      @Semantics.quantity.unitOfMeasure: 'BaseUnit'
      PurReqnItem.RequestedQuantity,
      @Semantics.unitOfMeasure: true
      PurReqnItem.BaseUnit,

      PurReqnItem.DeliveryDate,

      --- Source d'approvisionnement telle qu'elle est actuellement renseignée
      PurReqnItem.FixedSupplier,
      PurReqnItem.SupplyingPlant,
      PurReqnItem.PurchasingInfoRecord,
      PurReqnItem.PurchaseContract,
      PurReqnItem.PurchaseContractItem,
      PurReqnItem.SourceOfSupplyIsAssigned,

      /* Associations */
      _Supplier
}
where
  PurReqnItem.PurchaseRequisitionType = 'ZCTL'
```

**Activer**, puis **Data Preview**.

| Contrôle | Si ça échoue |
|---|---|
| Activation | Retirer une à une les zones signalées comme inconnues (étape 0.2) |
| Data Preview vide | Le type `ZCTL` n'existe pas ou aucune DA de ce type : vérifier avec `SE16N` sur `EBAN` (`BSART = 'ZCTL'`). Pour valider le modèle, remplacer temporairement `'ZCTL'` par `'NB'` |
| Erreur quantité | L'annotation `@Semantics.quantity.unitOfMeasure` et l'exposition de `BaseUnit` sont obligatoires : ne pas les retirer |

> **Le filtre `ZCTL` est volontairement en dur.** Si vous voulez le rendre paramétrable plus tard : paramètre d'entrée `with parameters p_type : esart` et `where PurchaseRequisitionType = $parameters.p_type`, ou table de personnalisation Z lue par une jointure.

📸 *Capture 4-04 : Data Preview de `ZI_PurReqnItemSourcing`*

---

## Étape 4 – Vue de consommation des sources

| Champ | Valeur |
|---|---|
| Name | `ZC_SourceOfSupplyCandidate` |
| Description | `Sources appro candidates - consommation` |

```abap
@EndUserText.label: 'Sources appro candidates - consommation'
@AccessControl.authorizationCheck: #NOT_REQUIRED
@Metadata.allowExtensions: true
define view entity ZC_SourceOfSupplyCandidate
  as select from ZI_SourceOfSupplyCandidate as Source

{
  key Source.Material,
  key Source.Plant,
  key Source.SourceListRecord,

      Source.Supplier,
      Source._Supplier.SupplierName as SupplierName,   // expression de chemin : pas d'association à exposer
      Source.PurchasingOrganization,
      Source.PurchaseAgreement,
      Source.PurchaseAgreementItem,
      Source.SupplyingPlant,
      Source.IsFixedSupplier,
      Source.ValidityStartDate,
      Source.ValidityEndDate,

      // 3 = source fixe (verte), 2 = source autorisée (jaune)
      case when Source.IsFixedSupplier = 'X' then 3 else 2 end as SourceCriticality
}
```

**Activer**, puis **Data Preview**.

| Sujet | Détail |
|---|---|
| `_Supplier.SupplierName` | Expression de chemin : la valeur est ramenée dans une zone simple, l'association n'a pas besoin d'être exposée. Si `SupplierName` n'existe pas sur votre `I_Supplier`, supprimer la ligne |
| `@Metadata.allowExtensions: true` | **Obligatoire** pour pouvoir créer la metadata extension de l'étape 6 |
| `case … then 3 else 2 end` | Si l'activation se plaint du type, écrire `cast( case when … then 3 else 2 end as abap.int4 ) as SourceCriticality` |

---

## Étape 5 – Vue de consommation des postes de DA

C'est **ici, et uniquement ici**, qu'est déclarée l'association vers les sources.

| Champ | Valeur |
|---|---|
| Name | `ZC_PurReqnItemSourcing` |
| Description | `Determination source appro - consommation` |

```abap
@EndUserText.label: 'Determination source appro - consommation'
@AccessControl.authorizationCheck: #NOT_REQUIRED
@Metadata.allowExtensions: true
@Search.searchable: true
define view entity ZC_PurReqnItemSourcing
  as select from ZI_PurReqnItemSourcing as Item

  association [0..*] to ZC_SourceOfSupplyCandidate as _SourceCandidate
    on  $projection.Material = _SourceCandidate.Material
    and $projection.Plant    = _SourceCandidate.Plant

{
  key Item.PurchaseRequisition,
  key Item.PurchaseRequisitionItem,

      Item.PurchaseRequisitionType,

      @Search.defaultSearchElement: true
      Item.PurchaseRequisitionItemText,

      @Search.defaultSearchElement: true
      Item.Material,
      Item.MaterialGroup,
      Item.Plant,
      Item.PurchasingGroup,
      Item.PurchasingOrganization,

      @Semantics.quantity.unitOfMeasure: 'BaseUnit'
      Item.RequestedQuantity,
      @Semantics.unitOfMeasure: true
      Item.BaseUnit,

      Item.DeliveryDate,

      Item.FixedSupplier,
      @Semantics.text: true
      Item._Supplier.SupplierName as FixedSupplierName,
      Item.SupplyingPlant,
      Item.PurchasingInfoRecord,
      Item.PurchaseContract,
      Item.PurchaseContractItem,
      Item.SourceOfSupplyIsAssigned,

      --- Zones calculées (supprimables si l'activation pose problème)
      // 3 = source affectée (vert), 1 = aucune source (rouge)
      case when Item.SourceOfSupplyIsAssigned = 'X' then 3 else 1 end as SourcingCriticality,

      // Nombre de jours avant la date de livraison souhaitée
      dats_days_between( $session.system_date, Item.DeliveryDate )     as DaysToDelivery,

      // 1 = urgent (< 3 j), 2 = à surveiller (< 10 j), 3 = confortable
      case when dats_days_between( $session.system_date, Item.DeliveryDate ) < 3  then 1
           when dats_days_between( $session.system_date, Item.DeliveryDate ) < 10 then 2
           else 3
      end                                                              as DeliveryCriticality,

      /* Associations */
      _SourceCandidate
}
```

**Activer**, puis **Data Preview**.

### Contrôles et pièges de cette vue

| Sujet | Détail |
|---|---|
| Association déclarée ici | Le `ON` utilise `$projection.Material` et `$projection.Plant` : les deux zones **doivent** figurer dans la liste de sélection, c'est le cas |
| `_SourceCandidate` exposé | Une association exposée dans une vue de consommation simple ne nécessite **aucune redirection** : c'est la différence avec une vue de projection |
| Aucun `root`, aucun `provider contract` | Inutiles en lecture seule et sources d'erreur ; ils arrivent en V2 (annexe C) |
| `FixedSupplierName` | Expression de chemin sur l'association `_Supplier` de la vue d'interface. Avec `@Semantics.text: true`, elle servira de texte au niveau 2 des annotations |
| `dats_days_between` | Disponible dans les vues entités depuis 7.55. Si l'activation la refuse, supprimer les trois zones calculées de date : l'application fonctionne sans |
| Ordre d'activation | Si l'erreur mentionne `ZC_SourceOfSupplyCandidate` inconnue, c'est que l'étape 4 n'a pas été activée |

📸 *Capture 4-05 : Data Preview de `ZC_PurReqnItemSourcing` avec les zones calculées*

---

## Étape 6 – Annotations UI niveau 1

Le niveau 1 ne contient que des annotations qui fonctionnent dans tous les cas. **On l'active et on le teste avant d'ajouter le niveau 2.**

### 6.1 Metadata extension des sources

Clic droit sur `ZC_SourceOfSupplyCandidate` → **New → Metadata Extension** → nom `ZC_SourceOfSupplyCandidate`, template *annotate view*.

```abap
@Metadata.layer: #CORE
annotate view ZC_SourceOfSupplyCandidate with
{
  @UI.lineItem: [{ position: 10, label: 'Fournisseur', importance: #HIGH }]
  Supplier;

  @UI.lineItem: [{ position: 20, label: 'Nom', importance: #HIGH }]
  SupplierName;

  @UI.lineItem: [{ position: 30, label: 'Org. achats', importance: #MEDIUM }]
  PurchasingOrganization;

  @UI.lineItem: [{ position: 40, label: 'Contrat', importance: #MEDIUM }]
  PurchaseAgreement;

  @UI.lineItem: [{ position: 50, label: 'Fixe', importance: #MEDIUM }]
  IsFixedSupplier;

  @UI.lineItem: [{ position: 60, label: 'Valide jusqu''au', importance: #LOW }]
  ValidityEndDate;

  @UI.hidden: true
  SourceCriticality;
}
```

Activer.

### 6.2 Metadata extension des postes de DA

```abap
@Metadata.layer: #CORE

@UI.headerInfo: {
  typeName:       'Poste de DA',
  typeNamePlural: 'Postes de DA',
  title:          { type: #STANDARD, value: 'PurchaseRequisition' },
  description:    { type: #STANDARD, value: 'PurchaseRequisitionItemText' }
}
annotate view ZC_PurReqnItemSourcing with
{
  @UI.facet: [
    { id: 'General', purpose: #STANDARD, type: #IDENTIFICATION_REFERENCE,
      label: 'Informations generales', position: 10 },
    { id: 'Candidates', purpose: #STANDARD, type: #LINEITEM_REFERENCE,
      label: 'Sources candidates', position: 20, targetElement: '_SourceCandidate' }
  ]

  @UI.lineItem:       [{ position: 10, label: 'Demande d''achat', importance: #HIGH }]
  @UI.identification: [{ position: 10, label: 'Demande d''achat' }]
  @UI.selectionField: [{ position: 40 }]
  PurchaseRequisition;

  @UI.lineItem:       [{ position: 20, label: 'Poste', importance: #HIGH }]
  @UI.identification: [{ position: 20, label: 'Poste' }]
  PurchaseRequisitionItem;

  @UI.hidden: true
  PurchaseRequisitionType;

  @UI.lineItem:       [{ position: 30, label: 'Designation', importance: #HIGH }]
  @UI.identification: [{ position: 30, label: 'Designation' }]
  PurchaseRequisitionItemText;

  @UI.lineItem:       [{ position: 40, label: 'Article', importance: #HIGH }]
  @UI.identification: [{ position: 40, label: 'Article' }]
  @UI.selectionField: [{ position: 30 }]
  Material;

  @UI.lineItem:       [{ position: 50, label: 'Groupe march.', importance: #LOW }]
  MaterialGroup;

  @UI.lineItem:       [{ position: 60, label: 'Division', importance: #HIGH }]
  @UI.identification: [{ position: 50, label: 'Division' }]
  @UI.selectionField: [{ position: 10 }]
  Plant;

  @UI.lineItem:       [{ position: 70, label: 'Groupe acheteurs', importance: #MEDIUM }]
  @UI.selectionField: [{ position: 20 }]
  PurchasingGroup;

  @UI.lineItem:       [{ position: 80, label: 'Quantite', importance: #HIGH }]
  @UI.identification: [{ position: 60, label: 'Quantite' }]
  RequestedQuantity;

  @UI.lineItem:       [{ position: 90, label: 'Livraison souhaitee', importance: #HIGH }]
  @UI.identification: [{ position: 70, label: 'Livraison souhaitee' }]
  DeliveryDate;

  @UI.lineItem:       [{ position: 100, label: 'Fournisseur fixe', importance: #HIGH }]
  @UI.identification: [{ position: 80, label: 'Fournisseur fixe' }]
  FixedSupplier;

  @UI.lineItem:       [{ position: 110, label: 'Contrat', importance: #MEDIUM }]
  @UI.identification: [{ position: 90, label: 'Contrat' }]
  PurchaseContract;

  @UI.identification: [{ position: 100, label: 'Poste de contrat' }]
  PurchaseContractItem;

  @UI.identification: [{ position: 110, label: 'Fiche info achat' }]
  PurchasingInfoRecord;

  @UI.identification: [{ position: 120, label: 'Division de livraison' }]
  SupplyingPlant;

  @UI.lineItem:       [{ position: 120, label: 'Source affectee', importance: #HIGH }]
  @UI.selectionField: [{ position: 50 }]
  SourceOfSupplyIsAssigned;

  @UI.hidden: true
  SourcingCriticality;
  @UI.hidden: true
  DeliveryCriticality;
  @UI.hidden: true
  DaysToDelivery;
  @UI.hidden: true
  BaseUnit;
  @UI.hidden: true
  FixedSupplierName;
}
```

Activer.

> **Chaque élément annoté doit exister dans la vue**, sinon l'activation échoue avec *Element … not found*. Si vous avez retiré des zones aux étapes 3 ou 5, retirez les blocs correspondants ici.
>
> `#IDENTIFICATION_REFERENCE` évite de manipuler des qualifiants de `fieldGroup` : c'est la façon la plus simple de remplir l'Object Page.

📸 *Capture 4-06 : les deux metadata extensions activées*

---

## Étape 7 – Annotations UI niveau 2

À n'appliquer **qu'après** avoir vu l'application fonctionner avec le niveau 1 (étapes 9 et 10). Chaque bloc est indépendant : ajoutez-les un par un, en réactivant à chaque fois.

### 7.1 Statuts colorés (criticality)

```abap
  @UI.lineItem: [{ position: 100, label: 'Fournisseur fixe', importance: #HIGH,
                   criticality: 'SourcingCriticality' }]
  FixedSupplier;

  @UI.lineItem: [{ position: 120, label: 'Source affectee', importance: #HIGH,
                   criticality: 'SourcingCriticality' }]
  SourceOfSupplyIsAssigned;
```

Dans `ZC_SourceOfSupplyCandidate.ddlx` :

```abap
  @UI.lineItem: [{ position: 10, label: 'Fournisseur', importance: #HIGH,
                   criticality: 'SourceCriticality' }]
  Supplier;
```

### 7.2 Indicateur de progression et KPI d'en-tête

Remplacer le bloc `DaysToDelivery` masqué par :

```abap
  @UI.lineItem:  [{ position: 130, label: 'Jours avant livraison',
                    type: #AS_DATAPOINT, importance: #MEDIUM }]
  @UI.dataPoint: { title: 'Jours avant livraison',
                   criticality: 'DeliveryCriticality',
                   visualization: #PROGRESS,
                   targetValue: 30 }
  DaysToDelivery;
```

et ajouter la facette d'en-tête dans le tableau `@UI.facet` :

```abap
    { id: 'Delai', purpose: #HEADER, type: #DATAPOINT_REFERENCE,
      targetQualifier: 'DaysToDelivery', position: 10, label: 'Delai' },
```

> `@UI.dataPoint` ne possède **pas** de sous-annotation `qualifier` : le qualifiant généré est le nom de l'élément, ici `DaysToDelivery`, et c'est ce nom qui est repris dans `targetQualifier`. C'était l'une des erreurs de la version précédente.

### 7.3 Code et texte du fournisseur

```abap
  @UI.lineItem:        [{ position: 100, label: 'Fournisseur fixe', importance: #HIGH }]
  @UI.textArrangement: #TEXT_LAST
  @ObjectModel.text.element: ['FixedSupplierName']
  FixedSupplier;

  // et retirer le @UI.hidden posé sur FixedSupplierName
  @UI.hidden: true
  FixedSupplierName;
```

> `@ObjectModel.text.element` pointe vers **une zone de la même vue** annotée `@Semantics.text: true` (déclarée à l'étape 5). C'est bien plus robuste que `@ObjectModel.text.association`, qui exige une vue de texte en bonne et due forme.

### 7.4 Aides à la saisie

```abap
  @Consumption.valueHelpDefinition: [{ entity: { name: 'I_Plant', element: 'Plant' } }]
  Plant;

  @Consumption.valueHelpDefinition: [{ entity: { name: 'I_Product', element: 'Product' } }]
  Material;
```

Ces deux vues doivent être exposées dans la service definition (étape 9).

### 7.5 Tri par défaut et onglets de variantes

En tête de la metadata extension, à côté de `@UI.headerInfo` :

```abap
@UI.presentationVariant: [{
  sortOrder:      [{ by: 'DeliveryDate', direction: #ASC }],
  visualizations: [{ type: #AS_LINEITEM }]
}]
@UI.selectionVariant: [
  { qualifier: 'SansSource', text: 'Sans source',
    parameters: [{ name: 'SourceOfSupplyIsAssigned', value: '' }] },
  { qualifier: 'AvecSource', text: 'Avec source',
    parameters: [{ name: 'SourceOfSupplyIsAssigned', value: 'X' }] }
]
```

Les onglets ne s'affichent qu'avec le réglage `quickVariantSelectionX` du manifest (étape 10.3).

### 7.6 Navigation contextuelle (smart links)

```abap
  @Consumption.semanticObject: 'Supplier'
  FixedSupplier;

  @Consumption.semanticObject: 'PurchaseContract'
  PurchaseContract;
```

---

## Étape 8 – Contrôle d'accès (DCL)

Une fois le modèle stable, on remplace `#NOT_REQUIRED` par un contrôle réel.

1. Clic droit sur `ZI_PurReqnItemSourcing` → **New → Access Control** → nom `ZI_PURREQNITEM_SOURCING` → template *Define Role with PFCG Aspect*.

```abap
@EndUserText.label: 'Controle acces postes DA ZCTL'
@MappingRole: true
define role ZI_PURREQNITEM_SOURCING {
  grant select on ZI_PurReqnItemSourcing
    where ( Plant )           = aspect pfcg_auth( M_BANF_WRK, WERKS, ACTVT = '03' )
      and ( PurchasingGroup ) = aspect pfcg_auth( M_BANF_EKG, EKGRP, ACTVT = '03' );
}
```

2. Dans `ZI_PurReqnItemSourcing`, passer l'annotation à `@AccessControl.authorizationCheck: #CHECK` et réactiver.
3. Faire de même pour `ZC_PurReqnItemSourcing` avec un DCL hérité :

```abap
@EndUserText.label: 'Controle acces consommation'
@MappingRole: true
define role ZC_PURREQNITEM_SOURCING {
  grant select on ZC_PurReqnItemSourcing
    where inheriting conditions from entity ZI_PurReqnItemSourcing;
}
```

4. Test : `Data Preview` avec un utilisateur restreint, ou `SU53` après un accès refusé.

> Si `inheriting conditions from entity` pose problème, laisser `#NOT_REQUIRED` sur la vue de consommation : le contrôle de la vue d'interface s'applique de toute façon à la lecture. ADT émettra un avertissement, pas une erreur.

---

## Étape 9 – Service definition et service binding

### 9.1 Service definition

Clic droit sur `ZC_PurReqnItemSourcing` → **New Service Definition** :

| Champ | Valeur |
|---|---|
| Name | `ZUI_PR_SOURCING` |
| Description | `Determination source appro - DA ZCTL` |
| Template | `defineService` |

```abap
@EndUserText.label: 'Determination source appro - DA ZCTL'
define service ZUI_PR_SOURCING {
  expose ZC_PurReqnItemSourcing     as PurReqnItemSourcing;
  expose ZC_SourceOfSupplyCandidate as SourceOfSupplyCandidate;
  expose I_Plant                    as Plant;      // aide à la saisie (niveau 2)
  expose I_Product                  as Product;    // aide à la saisie (niveau 2)
}
```

Activer. Si `I_Plant` ou `I_Product` n'existent pas, les retirer et renoncer aux aides à la saisie du §7.4.

### 9.2 Service binding

Clic droit sur la service definition → **New Service Binding** :

| Champ | Valeur |
|---|---|
| Name | `ZUI_PR_SOURCING_O2` |
| Description | `Determination source appro - OData V2 UI` |
| Binding Type | **OData V2 – UI** |
| Service Definition | `ZUI_PR_SOURCING` |

1. **Activer** (`Ctrl+F3`).
2. Cliquer sur **Publish** (*Local Service Endpoint*).
3. Sélectionner l'entity set `PurReqnItemSourcing` → **Preview** : l'application Fiori elements générée par ADT doit afficher le tableau avec vos colonnes.

```text
Service Binding  ZUI_PR_SOURCING_O2        Binding Type: OData V2 - UI
┌────────────────────────────────────────────────────────────────────┐
│ Service URL  /sap/opu/odata/sap/ZUI_PR_SOURCING_O2   [ Publish ]   │
│ ▾ Entity Set and Association                                       │
│    • PurReqnItemSourcing        [ Preview ]                        │
│    • SourceOfSupplyCandidate    [ Preview ]                        │
└────────────────────────────────────────────────────────────────────┘
```

| Contrôle | Où | Attendu |
|---|---|---|
| Service enregistré | `/IWFND/MAINT_SERVICES` | Alias `LOCAL`, nœud ICF vert |
| Métadonnées | `…/ZUI_PR_SOURCING_O2/$metadata` | Annotations `UI.LineItem`, `UI.HeaderInfo`, `UI.Facets` |
| Données | `…/ZUI_PR_SOURCING_O2/PurReqnItemSourcing?$top=5&$format=json` | Postes `ZCTL` |
| Association | `…/PurReqnItemSourcing(PurchaseRequisition='10000123',PurchaseRequisitionItem='00010')/to_SourceCandidate` | Sources de l'article/division |

> Le nom de la navigation property (`to_SourceCandidate`, `_SourceCandidate`…) dépend de la génération : relevez-le dans `$metadata`, il servira au fragment de l'étape 11.

📸 *Capture 4-07 : service binding publié*
📸 *Capture 4-08 : aperçu Fiori elements depuis ADT*

---

## Étape 10 – Générer l'application Fiori elements

### 10.1 Connexion

**VS Code** : `Ctrl+Shift+P` → `Fiori: Add SAP System` (nom `S4H_DEV_100`, URL, mandant, identifiants).
**BAS** : destination BTP + Cloud Connector.

### 10.2 Générateur

`Fiori: Open Application Generator` :

| Écran | Champ | Valeur |
|---|---|---|
| Template | Type | **List Report Page** (OData V2) |
| Data Source | Source / Service | `S4H_DEV_100` / `ZUI_PR_SOURCING_O2` |
| Entity Selection | Main entity | `PurReqnItemSourcing` |
| | Navigation entity | `_SourceCandidate` |
| | Add table columns automatically | Oui |
| Project Attributes | Module name | `zmmprsourcing` |
| | Application title | `Determination source appro` |
| | Minimum SAPUI5 version | `1.96.x` |
| | Add deployment configuration | Oui |
| Deployment | SAPUI5 ABAP Repository | `ZMM_PR_SRC` |
| | Package / Transport | `ZMM_SOURCING` / `S4DK9xxxxx` |

### 10.3 Onglets de variantes (si §7.5 appliqué)

`webapp/manifest.json` → `sap.ui.generic.app` → page `ListReport|PurReqnItemSourcing` → `component.settings` :

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

---

## Étape 11 – Bouton personnalisé et boîte de dialogue

### 11.1 Déclaration dans le manifest

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

`webapp/i18n/i18n.properties` :

```properties
ASSIGN_SOURCE=Affecter une source
SRC_DIALOG_TITLE=Sources candidates
SRC_SELECT_ONE=Sélectionnez une ligne de DA
```

### 11.2 Fragment `webapp/ext/fragment/SourceOfSupplyDialog.fragment.xml`

```xml
<core:FragmentDefinition xmlns="sap.m" xmlns:core="sap.ui.core">
    <Dialog id="sourceDialog" title="{i18n>SRC_DIALOG_TITLE}"
            contentWidth="42rem" contentHeight="24rem" resizable="true" draggable="true">
        <content>
            <MessageStrip id="srcContextStrip" text="" type="Information"
                          showIcon="true" class="sapUiSmallMargin"/>
            <Table id="sourceTable"
                   items="{ path: '/SourceOfSupplyCandidate' }"
                   noDataText="Aucune source candidate pour cet article et cette division"
                   growing="true" growingThreshold="20">
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
                            <ObjectIdentifier title="{Supplier}" text="{SupplierName}"/>
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
            <Button id="assignBtn" text="Affecter" type="Emphasized"
                    enabled="false" tooltip="Disponible en version 2"/>
        </beginButton>
        <endButton>
            <Button id="closeBtn" text="Fermer" press=".onCloseSourceDialog"/>
        </endButton>
    </Dialog>
</core:FragmentDefinition>
```

### 11.3 Contrôleur `webapp/ext/controller/ListReportExt.controller.js`

```javascript
sap.ui.define([
    "sap/ui/core/Fragment",
    "sap/ui/model/Filter",
    "sap/ui/model/FilterOperator",
    "sap/m/MessageToast"
], function (Fragment, Filter, FilterOperator, MessageToast) {
    "use strict";

    return {

        onOpenSourceDialog: function () {
            var oView = this.getView();
            var aContexts = this.extensionAPI.getSelectedContexts();

            if (!aContexts || aContexts.length !== 1) {
                MessageToast.show(oView.getModel("i18n").getResourceBundle()
                                       .getText("SRC_SELECT_ONE"));
                return;
            }

            var oItem = aContexts[0].getObject();

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
                Fragment.byId(oView.getId(), "srcContextStrip").setText(
                    "DA " + oItem.PurchaseRequisition + " / poste " + oItem.PurchaseRequisitionItem +
                    " — article " + (oItem.Material || "sans article") +
                    " — division " + oItem.Plant);

                var oBinding = Fragment.byId(oView.getId(), "sourceTable").getBinding("items");
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

---

## Étape 12 – Test local, déploiement, Launchpad, transport

### 12.1 Test local

```bash
cd zmmprsourcing
npm install
npm start
```

| Contrôle | Attendu |
|---|---|
| Barre de filtres | Division, Groupe acheteurs, Article, DA, Source affectée |
| Bouton *Affecter une source* | Visible, désactivé sans sélection |
| Après *Exécuter* | Colonnes annotées ; statuts colorés et progression si le niveau 2 est appliqué |
| Clic sur une ligne | Object Page avec la section *Informations generales* et le tableau *Sources candidates* |
| Sélection + bouton | Pop-up filtrée sur article et division |

### 12.2 Déploiement

```bash
npm run deploy
```

Contrôles : `SE80` → application BSP `ZMM_PR_SRC` ; `/UI5/APP_INDEX_CALCULATE`.

### 12.3 Launchpad

| Étape | Transaction | Valeurs |
|---|---|---|
| Objet sémantique | `/UI2/SEMOBJ` | `ZSourceOfSupply` — *Determination source appro* |
| Catalogue technique + target mapping | `/UI2/FLPAM` | Type `SAPUI5 Fiori App`, Semantic Object `ZSourceOfSupply`, Action `determine`, URL `/sap/bc/ui5_ui5/sap/zmm_pr_src`, ID `zmmprsourcing` |
| Tuile | `/UI2/FLPAM` | Titre *Détermination source appro*, sous-titre *DA ZCTL*, icône `sap-icon://supplier` |
| Catalogue métier | `/UI2/FLPD_CONF` ou `/UI2/FLPCM_CONF` | `Z_BC_MM_SOURCING` référençant tuile et target mapping |
| Rôle | `PFCG` | `Z_BR_SOURCING` : catalogue, `S_SERVICE` sur `ZUI_PR_SOURCING_O2`, `M_BANF_*` en `03` |

Test : `…/sap/bc/ui2/flp?sap-client=100#ZSourceOfSupply-determine`

### 12.4 Transport

| Ordre | Objets |
|---|---|
| Workbench | `DEVC ZMM_SOURCING`, `DDLS` (4 vues), `DDLX` (2), `DCLS` (1 ou 2), `SRVD ZUI_PR_SOURCING`, `SRVB ZUI_PR_SOURCING_O2`, `WAPA ZMM_PR_SRC` |
| Customizing | `/UI2/SEMOBJ`, contenu FLP client-spécifique |

Après import : vérifier le service dans `/IWFND/MAINT_SERVICES` (le republier depuis ADT s'il est absent), `/UI5/APP_INDEX_CALCULATE`, `/IWFND/CACHE_CLEANUP`, `/IWBEP/CACHE_CLEANUP`, `/UI2/INVALIDATE_GLOBAL_CACHES`.

---

## Annexe A – Erreurs d'activation ADT : message par message

| Message ADT (ou équivalent) | Cause | Correction |
|---|---|---|
| *ZC_SourceOfSupplyCandidate does not exist* / *… is unknown* | Vue cible non encore activée | Respecter l'ordre étapes 2 → 5, ou activer en masse (`Ctrl+Shift+F3`) |
| *Cyclic dependency* / activation en boucle | Deux vues qui se référencent | Ne déclarer l'association que dans la vue de consommation (étape 5) |
| *The association must be redirected* | `provider contract` présent sur la vue | Supprimer `provider contract transactional_query` en V1 |
| *redirected to composition child … is not allowed* | Syntaxe réservée aux compositions d'un BO | En V1 : pas de redirection. En V2 : `redirected to <vue de projection>` |
| *Root entity expected* / *… is not a root entity* | `define root view entity` sur une vue sans BO | Retirer `root` en V1 |
| *Behavior definition … not found* | `provider contract transactional_query` sans BDEF | Retirer le contrat, ou créer la BDEF (V2, annexe C) |
| *Element … not found* dans la DDLX | Zone annotée absente de la vue | Aligner la metadata extension sur la liste réelle des zones |
| *Metadata extension not allowed for this entity* | `@Metadata.allowExtensions: true` manquant | Ajouter l'annotation sur la vue de consommation et réactiver |
| *Quantity field … requires a unit of measure* | Zone quantité sans référence d'unité | `@Semantics.quantity.unitOfMeasure: 'BaseUnit'` + exposer `BaseUnit` avec `@Semantics.unitOfMeasure: true` |
| *Annotation qualifier not allowed* / *unknown annotation* | `qualifier` dans `@UI.dataPoint`, ou annotation inventée | Supprimer ; le qualifiant d'un dataPoint est le nom de l'élément |
| *Text association … invalid* | `@ObjectModel.text.association` vers une vue non textuelle | Utiliser `@ObjectModel.text.element` avec une zone `@Semantics.text: true` de la même vue |
| *Entity I_ProductStdVH does not exist* | Vue d'aide à la saisie inexistante sur la release | Pointer vers une vue vérifiée (`I_Product`, `I_Plant`) ou supprimer l'aide |
| *Field SourceOfSupplyIsAssigned unknown* | Nom de zone différent sur la release | Étape 0.2 |
| *Function dats_days_between unknown* | Fonction non disponible | Supprimer les trois zones calculées de délai |
| *Type mismatch in CASE* | Littéraux de types différents | `cast( case … end as abap.int4 )` |
| *No access control exists* (avertissement) | `#CHECK` sans DCL | Créer le DCL (étape 8) ou rester en `#NOT_REQUIRED` le temps des tests |
| Service binding : *entity has no key* | Clés non déclarées dans la vue de consommation | Reprendre les `key` de l'étape 5 |

---

## Annexe B – Plan de repli : modèle basé sur EBAN

Si `I_PurchaseRequisitionItemAPI01` n'existe pas ou pose problème, la vue d'interface se construit directement sur la table. Le reste du document est inchangé.

```abap
@EndUserText.label: 'Postes de DA ZCTL - donnees sourcing (EBAN)'
@AccessControl.authorizationCheck: #NOT_REQUIRED
define view entity ZI_PurReqnItemSourcing
  as select from eban as PurReqnItem

  association [0..1] to I_Supplier as _Supplier
    on $projection.FixedSupplier = _Supplier.Supplier

{
  key PurReqnItem.banfn as PurchaseRequisition,
  key PurReqnItem.bnfpo as PurchaseRequisitionItem,

      PurReqnItem.bsart as PurchaseRequisitionType,
      PurReqnItem.txz01 as PurchaseRequisitionItemText,
      PurReqnItem.matnr as Material,
      PurReqnItem.matkl as MaterialGroup,
      PurReqnItem.werks as Plant,
      PurReqnItem.ekgrp as PurchasingGroup,
      PurReqnItem.ekorg as PurchasingOrganization,

      @Semantics.quantity.unitOfMeasure: 'BaseUnit'
      PurReqnItem.menge as RequestedQuantity,
      @Semantics.unitOfMeasure: true
      PurReqnItem.meins as BaseUnit,

      PurReqnItem.lfdat as DeliveryDate,

      PurReqnItem.flief as FixedSupplier,
      PurReqnItem.reswk as SupplyingPlant,
      PurReqnItem.infnr as PurchasingInfoRecord,
      PurReqnItem.konnr as PurchaseContract,
      PurReqnItem.ktpnr as PurchaseContractItem,

      // indicateur reconstitué : aucune zone standard équivalente sur EBAN
      case when PurReqnItem.flief <> ''
             or PurReqnItem.konnr <> ''
             or PurReqnItem.infnr <> ''
             or PurReqnItem.reswk <> ''
           then 'X' else '' end as SourceOfSupplyIsAssigned,

      _Supplier
}
where
      PurReqnItem.bsart = 'ZCTL'
  and PurReqnItem.loekz = ''
```

Différences à connaître : pas de contrôle d'autorisation hérité du standard (le DCL de l'étape 8 devient indispensable), et `SourceOfSupplyIsAssigned` est calculé.

---

## Annexe C – Préparer la V2 (couche de projection RAP)

La V2 (document 06) ajoute une action RAP, ce qui impose la couche que l'on a volontairement écartée ici. Voici exactement ce qui change :

1. `ZI_PurReqnItemSourcing` devient **racine** :

```abap
define root view entity ZI_PurReqnItemSourcing
```

2. `ZC_PurReqnItemSourcing` devient une **vue de projection racine** avec contrat :

```abap
@Metadata.allowExtensions: true
define root view entity ZC_PurReqnItemSourcing
  provider contract transactional_query
  as projection on ZI_PurReqnItemSourcing
{
  key PurchaseRequisition,
  key PurchaseRequisitionItem,
      …
      /* redirection obligatoire : association simple, PAS de "composition child" */
      _SourceCandidate : redirected to ZC_SourceOfSupplyCandidate
}
```

3. L'association `_SourceCandidate` doit alors être **déclarée dans la vue d'interface** `ZI_PurReqnItemSourcing` (vers `ZI_SourceOfSupplyCandidate`), puisqu'une projection ne peut que projeter ce qui existe en dessous. C'est ce qui réintroduit la dépendance entre les deux vues d'interface : **créer et activer `ZI_SourceOfSupplyCandidate` en premier**, puis `ZI_PurReqnItemSourcing`.
4. `ZC_SourceOfSupplyCandidate` devient une projection : `define view entity ZC_SourceOfSupplyCandidate as projection on ZI_SourceOfSupplyCandidate`. Les zones calculées (`SourceCriticality`) doivent alors remonter dans la vue d'interface, une projection ne calculant rien.
5. Behavior definition, behavior pool et behavior projection : voir document 06.

> Tant que la V2 n'est pas lancée, **ne mettez pas cette couche en place** : elle n'apporte rien en lecture seule et c'est elle qui a généré les erreurs d'activation.

---

## Annexe D – Catalogue des annotations démontrées

| Annotation | Niveau | Effet (Fiori elements V2 / UI5 1.96) |
|---|---|---|
| `@UI.headerInfo` | 1 | Titre et type d'objet de l'Object Page |
| `@UI.lineItem` (`position`, `label`, `importance`) | 1 | Colonnes du tableau et priorité responsive |
| `@UI.identification` + facette `#IDENTIFICATION_REFERENCE` | 1 | Section de l'Object Page |
| `@UI.facet` `#LINEITEM_REFERENCE` + `targetElement` | 1 | Tableau des sources candidates dans l'Object Page |
| `@UI.selectionField` | 1 | Champs de la barre de filtres |
| `@UI.hidden` | 1 | Masquage des zones techniques |
| `@Search.searchable` / `@Search.defaultSearchElement` | 1 | Recherche libre |
| `@Semantics.quantity.unitOfMeasure` / `@Semantics.unitOfMeasure` | 1 | Quantité formatée avec son unité |
| `@UI.lineItem.criticality` | 2 | Statut coloré rouge/orange/vert |
| `@UI.lineItem` `#AS_DATAPOINT` + `@UI.dataPoint` `#PROGRESS` | 2 | Barre de progression dans le tableau |
| `@UI.facet` `#DATAPOINT_REFERENCE` `purpose: #HEADER` | 2 | KPI dans l'en-tête de l'Object Page |
| `@ObjectModel.text.element` + `@UI.textArrangement` | 2 | Affichage « code – libellé » |
| `@Consumption.valueHelpDefinition` | 2 | Aide à la saisie F4 sur les filtres |
| `@UI.presentationVariant` | 2 | Tri par défaut |
| `@UI.selectionVariant` + `quickVariantSelectionX` | 2 | Onglets Toutes / Sans source / Avec source |
| `@Consumption.semanticObject` | 2 | Smart link de navigation contextuelle |

---

## Captures à réaliser

| N° | Écran | Fichier suggéré |
|---|---|---|
| 4-01 | Element Info sur la vue standard | `img/04-01-element-info.png` |
| 4-02 | Data Preview du standard filtré ZCTL | `img/04-02-preview-std.png` |
| 4-03 | Data Preview `ZI_SourceOfSupplyCandidate` | `img/04-03-preview-sources.png` |
| 4-04 | Data Preview `ZI_PurReqnItemSourcing` | `img/04-04-preview-da.png` |
| 4-05 | Data Preview `ZC_PurReqnItemSourcing` | `img/04-05-preview-conso.png` |
| 4-06 | Metadata extensions activées | `img/04-06-ddlx.png` |
| 4-07 | Service binding publié | `img/04-07-srvb.png` |
| 4-08 | Aperçu Fiori depuis ADT | `img/04-08-preview-adt.png` |
| 4-09 | Application en local | `img/04-09-local.png` |
| 4-10 | Pop-up des sources | `img/04-10-dialog.png` |
| 4-11 | Application dans le FLP | `img/04-11-flp.png` |

---

## Références

- SAP Help – *CDS View Entities* : `define view entity`, associations, expressions de chemin
- SAP Help – *CDS Metadata Extensions* : `@Metadata.allowExtensions`, `@Metadata.layer`
- SAP Help – *CDS Annotations* : `@UI`, `@Consumption`, `@Search`, `@Semantics`, `@ObjectModel`
- SAP Help – *CDS Access Control (DCL)* : `aspect pfcg_auth`, `inheriting conditions from entity`
- SAP Learning – *Defining an OData UI Service* (publication du local service endpoint)
- SAPUI5 Demo Kit – *Adding Custom Actions Using Extension Points* (OData V2), *Using the extensionAPI*
- Documents internes : `05_Adaptation_Project_F1048_Navigation_VSCode.md`, `06_App_RAP_V2_Action_Affectation_Source.md`
