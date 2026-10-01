# Value help en liste déroulante (dropdown) sur un statut à valeurs fixes de domaine — service RAP

**Contexte :** SAP S/4HANA Cloud Private Edition 2021 FPS02 — ABAP Platform 2021 (SAP_BASIS 7.56), SAP_UI 7.56, SAPUI5 1.96.x
**Service :** RAP, service binding **OData V2 – UI**, application SAP Fiori elements V2 (modèle des documents 04 REV2 et 06)
**Objet :** ajouter un champ **Statut de traitement** dont les valeurs proviennent d'un **domaine à valeurs fixes** (`OP`, `IP`, `CL`, `CA`) et l'afficher sous forme de **liste déroulante**, dans la barre de filtres et sur le champ, sans développement UI5.

> Ce document reprend tout depuis zéro et explicite, à chaque annotation, **à quoi elle sert et ce qui casse si elle manque**. La section [Annexe A](#annexe-a--pourquoi-une-liste-déroulante-ne-sort-pas--les-12-causes) liste les douze causes classiques d'échec.

---

## Sommaire

1. [Résultat attendu](#1-résultat-attendu)
2. [Cohérence des versions — à lire avant de commencer](#2-cohérence-des-versions--à-lire-avant-de-commencer)
3. [Chaîne complète : du domaine à la liste déroulante](#3-chaîne-complète--du-domaine-à-la-liste-déroulante)
4. [Étape 0 – Vérifications préalables](#étape-0--vérifications-préalables)
5. [Étape 1 – Domaine à valeurs fixes](#étape-1--domaine-à-valeurs-fixes)
6. [Étape 2 – Élément de données](#étape-2--élément-de-données)
7. [Étape 3 – Table de persistance du statut](#étape-3--table-de-persistance-du-statut)
8. [Étape 4 – Vue d'aide à la saisie](#étape-4--vue-daide-à-la-saisie)
9. [Étape 5 – Vue de consommation de l'aide à la saisie](#étape-5--vue-de-consommation-de-laide-à-la-saisie)
10. [Étape 6 – Intégration dans le modèle de l'application](#étape-6--intégration-dans-le-modèle-de-lapplication)
11. [Étape 7 – Annotations UI](#étape-7--annotations-ui)
12. [Étape 8 – Exposition dans le service et republication](#étape-8--exposition-dans-le-service-et-republication)
13. [Étape 9 – Contrôle du $metadata](#étape-9--contrôle-du-metadata)
14. [Étape 10 – Test dans l'application](#étape-10--test-dans-lapplication)
15. [Étape 11 – Rendre le champ modifiable](#étape-11--rendre-le-champ-modifiable)
16. [Étape 12 – Transport](#étape-12--transport)
17. [Annexe A – Pourquoi une liste déroulante ne sort pas : les 12 causes](#annexe-a--pourquoi-une-liste-déroulante-ne-sort-pas--les-12-causes)
18. [Annexe B – Variantes](#annexe-b--variantes)
19. [Annexe C – Où placer chaque annotation](#annexe-c--où-placer-chaque-annotation)
20. [Captures à réaliser](#captures-à-réaliser)
21. [Références](#références)

---

## 1. Résultat attendu

| Valeur | Texte | Criticité affichée |
|---|---|---|
| `OP` | Ouverte | neutre |
| `IP` | En cours | orange |
| `CL` | Clôturée | vert |
| `CA` | Annulée | rouge |

```text
┌──────────────────────────────────────────────────────────────────────────────────┐
│ Standard ▾                                                                       │
│ Division [    ]  Groupe acheteurs [   ]  ┏ Statut ─────────────────────┓         │
│                                          ┃ Ouverte                  ▾ ┃         │
│                                          ┣━━━━━━━━━━━━━━━━━━━━━━━━━━━━┫         │
│                                          ┃ ☑ Ouverte                  ┃         │
│                                          ┃ ☐ En cours                 ┃         │
│                                          ┃ ☐ Clôturée                 ┃         │
│                                          ┃ ☐ Annulée                  ┃         │
│                                          ┗━━━━━━━━━━━━━━━━━━━━━━━━━━━━┛         │
├──────────────────────────────────────────────────────────────────────────────────┤
│ DA / Poste │ Article │ Division │ Livraison │ Statut                             │
│ 10000123/10│ TG11    │ 1010     │ 12.10.2026│ ⬤ En cours                         │
│ 10000123/20│ TG12    │ 1010     │ 05.10.2026│ ⬤ Clôturée                         │
└──────────────────────────────────────────────────────────────────────────────────┘
```

Liste déroulante (`ComboBox` / `MultiComboBox`) et non boîte de dialogue de recherche : c'est l'annotation `@ObjectModel.resultSet.sizeCategory: #XS` sur la **vue d'aide à la saisie** qui produit ce rendu.

---

## 2. Cohérence des versions — à lire avant de commencer

| Élément | Version minimale | Notre système | Verdict |
|---|---|---|---|
| `define view entity` (vue entité) | ABAP 7.55 | 7.56 | ✅ |
| `@Consumption.valueHelpDefinition` | 7.50 | 7.56 | ✅ |
| `@ObjectModel.resultSet.sizeCategory: #XS` → `sap:value-list="fixed-values"` | **NW 7.52** (inopérant en 7.51) | 7.56 | ✅ |
| `@ObjectModel.text.element` dans la même vue | 7.50 | 7.56 | ✅ |
| Vue standard `I_DomainFixedValue` + `_DomainFixedValueText` | S/4HANA | à confirmer, étape 0 | ⚠️ repli `DD07L`/`DD07T` fourni |
| Rendu liste déroulante par SAP Fiori elements **V2** (SmartFilterBar, SmartField) | SAPUI5 1.4x | 1.96 | ✅ |
| Équivalent OData **V4** : `Common.ValueListWithFixedValues` | FE V4 | FE V4 incomplet en 1.96 | ❌ on reste en binding V2 |
| `useForValidation: true` dans `valueHelpDefinition` | releases récentes | incertain en 7.56 | ⚠️ optionnel, à retirer si l'activation le refuse |
| Champ modifiable (dropdown en saisie et pas seulement en filtre) | behavior definition RAP | 7.56 | ✅ voir étape 11 |

**Deux conséquences à retenir dès maintenant :**

1. Avec le binding **OData V2**, la liste déroulante est pilotée par `sap:value-list="fixed-values"`. Les recettes trouvées en ligne qui utilisent `Common.ValueListWithFixedValues` concernent OData V4 ou CAP : elles ne s'appliquent pas ici.
2. Dans une application **en lecture seule**, le champ n'est jamais en saisie : la liste déroulante n'apparaît alors **que dans la barre de filtres**. C'est normal et c'est souvent la cause du « ça ne marche pas ». L'étape 11 explique comment l'obtenir aussi sur le champ.

---

## 3. Chaîne complète : du domaine à la liste déroulante

```text
   SE11                     ADT / CDS                         Gateway                     SAPUI5 1.96
┌────────────┐   ┌────────────────────────────────┐   ┌────────────────────┐   ┌────────────────────────┐
│ Domaine    │   │ ZI_SrcStatusVH                 │   │ $metadata          │   │ SmartFilterBar         │
│ ZD_SRC_    │──▶│  @ObjectModel.resultSet        │──▶│ Property Status    │──▶│  → MultiComboBox       │
│ STATUS     │   │    .sizeCategory: #XS          │   │  sap:value-list=   │   │ SmartField (si saisie) │
│ OP IP CL CA│   │  @ObjectModel.text.element     │   │   "fixed-values"   │   │  → ComboBox            │
└────────────┘   │  @ObjectModel.representativeKey│   │  sap:text=         │   └────────────────────────┘
       │         └────────────────────────────────┘   │   "StatusText"     │
       │                      │                       │ EntitySet          │
       ▼                      ▼                       │  SrcStatusVH       │
┌────────────┐   ┌────────────────────────────────┐   │ Annotations        │
│ Élément de │   │ ZC_PurReqnItemSourcing         │   │  Common.ValueList  │
│ données    │──▶│  @Consumption.valueHelp        │──▶└────────────────────┘
│ ZE_SRC_    │   │    Definition (DDLX)           │              ▲
│ STATUS     │   │  @ObjectModel.text.element     │              │
└────────────┘   │  @UI.textArrangement           │   ┌────────────────────┐
       │         └────────────────────────────────┘   │ Service definition │
       ▼                                              │  expose les DEUX   │
┌────────────┐                                        │  entités           │
│ Table      │                                        └────────────────────┘
│ ZMM_SRC_   │
│ STATUS     │
└────────────┘
```

---

## Étape 0 – Vérifications préalables

| # | Contrôle | Comment | Si KO |
|---|---|---|---|
| 1 | Vue `I_DomainFixedValue` disponible | ADT → `Ctrl+Shift+A` → `I_DomainFixedValue` → ouvrir, puis `F8` (Data Preview) | Utiliser la variante B de l'étape 4 (`DD07L` / `DD07T`) |
| 2 | Association `_DomainFixedValueText` présente | `Ctrl+O` dans la vue ouverte | Variante B |
| 3 | Binding du service | Le service binding de l'application est bien `OData V2 – UI` | Voir §2, les annotations diffèrent en V4 |
| 4 | Version SAPUI5 | `Ctrl+Alt+Shift+P` dans le FLP → 1.96.x | Adapter les attentes de rendu |
| 5 | Application de base active | Modèle du document 04 REV2 activé et application affichant des données | Reprendre le document 04 REV2 |

📸 *Capture 8-01 : `I_DomainFixedValue` ouverte dans ADT avec son Data Preview*

---

## Étape 1 – Domaine à valeurs fixes

`SE11` → **Domaine** → `ZD_SRC_STATUS` → *Créer*.

**Onglet Définition**

| Champ | Valeur |
|---|---|
| Description courte | `Statut de traitement sourcing DA` |
| Type de données | `CHAR` |
| Nbre de caractères | `2` |
| Majuscules | coché |

**Onglet Domaine de valeurs → Valeurs individuelles**

| Valeur fixe | Description |
|---|---|
| `OP` | Ouverte |
| `IP` | En cours |
| `CL` | Clôturée |
| `CA` | Annulée |

Enregistrer dans le package `ZMM_SOURCING` et l'ordre Workbench, puis **Activer**.

> Le domaine porte à la fois le **contrôle de saisie** (SE16, SM30, saisie ABAP) et la **liste de valeurs traduisible** : les descriptions se traduisent via `SE63` ou le bouton *Traduction* de `SE11`, et la liste déroulante suivra automatiquement la langue de connexion.

📸 *Capture 8-02 : valeurs fixes du domaine*

---

## Étape 2 – Élément de données

`SE11` → **Type de données** → `ZE_SRC_STATUS` → *Créer* → *Élément de données*.

| Champ | Valeur |
|---|---|
| Description courte | `Statut de traitement sourcing` |
| Domaine | `ZD_SRC_STATUS` |
| Libellé court (10) | `Statut` |
| Libellé moyen (20) | `Statut traitement` |
| Libellé long (40) | `Statut de traitement` |
| Titre (55) | `Statut de traitement` |

Activer.

---

## Étape 3 – Table de persistance du statut

`SE11` → **Table de base de données** → `ZMM_SRC_STATUS`.

**Livraison et maintenance** : classe de livraison `A`, *Affichage/maintenance autorisés*.

| Zone | Clé | Init. | Type | Description |
|---|---|---|---|---|
| `MANDT` | ✔ | ✔ | `MANDT` | Mandant |
| `BANFN` | ✔ | ✔ | `BANFN` | Demande d'achat |
| `BNFPO` | ✔ | ✔ | `BNFPO` | Poste |
| `SRC_STATUS` | | | `ZE_SRC_STATUS` | Statut de traitement |
| `CHANGED_BY` | | | `SYUNAME` | Dernier utilisateur |
| `CHANGED_AT` | | | `TIMESTAMPL` | Dernière modification |

Paramètres techniques : `APPL1`, catégorie de taille `0`. Activer.

Générateur de maintenance (`SE54`) : groupe d'autorisations `&NC&`, groupe de fonctions `ZFG_MM_SRC_STATUS`, une étape, écran `0001`.

`SM30` → saisir deux ou trois lignes de test correspondant à des DA `ZCTL` existantes :

| BANFN | BNFPO | SRC_STATUS |
|---|---|---|
| 10000123 | 00010 | `IP` |
| 10000123 | 00020 | `CL` |

> Grâce au domaine, `SM30` refuse déjà toute valeur hors `OP`/`IP`/`CL`/`CA` : c'est le premier niveau de contrôle, indépendant de l'UI.

📸 *Capture 8-03 : SM30 avec l'aide à la saisie du domaine*

---

## Étape 4 – Vue d'aide à la saisie

### Variante A (recommandée) — à partir de `I_DomainFixedValue`

ADT → package `ZMM_SOURCING` → **New → Data Definition** → `ZI_SrcStatusVH`, template *Define View Entity*.

```abap
@EndUserText.label: 'Aide a la saisie - statut de traitement sourcing'
@AccessControl.authorizationCheck: #NOT_REQUIRED
@Metadata.ignorePropagatedAnnotations: true
@ObjectModel.resultSet.sizeCategory: #XS
@ObjectModel.representativeKey: 'SrcProcessingStatus'
@ObjectModel.dataCategory: #TEXT
@Search.searchable: true
define view entity ZI_SrcStatusVH
  as select from I_DomainFixedValue
{
      @EndUserText.label: 'Statut'
      @ObjectModel.text.element: ['SrcProcessingStatusText']
      @Search.defaultSearchElement: true
  key cast( DomainValue as ze_src_status ) as SrcProcessingStatus,

      @EndUserText.label: 'Désignation du statut'
      @Semantics.text: true
      _DomainFixedValueText[ 1: Language = $session.system_language ].DomainText
                                           as SrcProcessingStatusText
}
where
  SAPDataDictionaryDomain = 'ZD_SRC_STATUS'
```

### Variante B (repli) — à partir de `DD07L` / `DD07T`

À utiliser si `I_DomainFixedValue` n'existe pas sur votre release.

```abap
@EndUserText.label: 'Aide a la saisie - statut de traitement sourcing'
@AccessControl.authorizationCheck: #NOT_REQUIRED
@Metadata.ignorePropagatedAnnotations: true
@ObjectModel.resultSet.sizeCategory: #XS
@ObjectModel.representativeKey: 'SrcProcessingStatus'
@ObjectModel.dataCategory: #TEXT
@Search.searchable: true
define view entity ZI_SrcStatusVH
  as select from    dd07l as FixedValue
    left outer join dd07t as ValueText
      on  ValueText.domname    = FixedValue.domname
      and ValueText.as4local   = FixedValue.as4local
      and ValueText.as4vers    = FixedValue.as4vers
      and ValueText.valpos     = FixedValue.valpos
      and ValueText.ddlanguage = $session.system_language
{
      @EndUserText.label: 'Statut'
      @ObjectModel.text.element: ['SrcProcessingStatusText']
      @Search.defaultSearchElement: true
  key cast( FixedValue.domvalue_l as ze_src_status ) as SrcProcessingStatus,

      @EndUserText.label: 'Désignation du statut'
      @Semantics.text: true
      ValueText.ddtext as SrcProcessingStatusText
}
where
      FixedValue.domname  = 'ZD_SRC_STATUS'
  and FixedValue.as4local = 'A'
```

**Activer**, puis **Data Preview** : vous devez voir exactement quatre lignes avec leurs textes dans votre langue.

### À quoi sert chaque annotation

| Annotation | Rôle | Symptôme si elle manque |
|---|---|---|
| `@ObjectModel.resultSet.sizeCategory: #XS` | Produit `sap:value-list="fixed-values"` | Boîte de dialogue de recherche au lieu d'une liste déroulante |
| `@ObjectModel.representativeKey: 'SrcProcessingStatus'` | Désigne la clé représentative de la liste | Aide à la saisie incomplète, mauvais paramètre renvoyé |
| `@ObjectModel.text.element: ['…Text']` | Associe le texte à la clé → `sap:text` dans les métadonnées | La liste n'affiche que les codes `OP`, `IP`… |
| `@Semantics.text: true` | Déclare l'élément comme texte | Idem |
| `@ObjectModel.dataCategory: #TEXT` | Qualifie la vue comme liste de codes | Rendu dégradé selon les contrôles |
| `@Search.searchable` / `@Search.defaultSearchElement` | Recherche dans la liste | Pas de saisie semi-automatique |
| `$session.system_language` | Texte dans la langue de l'utilisateur | Textes vides ou en anglais uniquement |
| `cast( … as ze_src_status )` | Aligne le type de la liste sur celui du champ | Avertissement ou erreur de compatibilité de types |

> **Si le `cast` est refusé à l'activation** : retirez-le et exposez la valeur brute, puis vérifiez que l'aide à la saisie fonctionne malgré la différence de longueur. En dernier recours, définissez le domaine en `CHAR 10`.

📸 *Capture 8-04 : Data Preview de la vue d'aide à la saisie (4 lignes)*

---

## Étape 5 – Vue de consommation de l'aide à la saisie

L'entité **exposée dans le service** doit elle aussi porter `@ObjectModel.resultSet.sizeCategory`.

```abap
@EndUserText.label: 'Statut de traitement - consommation'
@AccessControl.authorizationCheck: #NOT_REQUIRED
@Metadata.allowExtensions: true
@ObjectModel.resultSet.sizeCategory: #XS
@ObjectModel.representativeKey: 'SrcProcessingStatus'
@ObjectModel.dataCategory: #TEXT
@Search.searchable: true
define view entity ZC_SrcStatusVH
  as select from ZI_SrcStatusVH as Status
{
      @ObjectModel.text.element: ['SrcProcessingStatusText']
      @Search.defaultSearchElement: true
  key Status.SrcProcessingStatus,

      @Semantics.text: true
      Status.SrcProcessingStatusText
}
```

Activer.

---

## Étape 6 – Intégration dans le modèle de l'application

### 6.1 Vue d'interface — ajouter le statut

Dans `ZI_PurReqnItemSourcing` (document 04 REV2, étape 3), ajouter la jointure et la zone :

```abap
define view entity ZI_PurReqnItemSourcing
  as select from I_PurchaseRequisitionItemAPI01 as PurReqnItem

    left outer join zmm_src_status as SrcStatus
      on  SrcStatus.banfn = PurReqnItem.PurchaseRequisition
      and SrcStatus.bnfpo = PurReqnItem.PurchaseRequisitionItem

  association [0..1] to I_Supplier as _Supplier
    on $projection.FixedSupplier = _Supplier.Supplier
{
  key PurReqnItem.PurchaseRequisition,
  key PurReqnItem.PurchaseRequisitionItem,
      …
      --- Statut de traitement (table Z)
      SrcStatus.src_status as SrcProcessingStatus,
      …
      _Supplier
}
where
  PurReqnItem.PurchaseRequisitionType = 'ZCTL'
```

> La jointure gauche conserve les postes sans ligne de statut : `SrcProcessingStatus` est alors vide. Pour forcer une valeur par défaut, utilisez `cast( coalesce( SrcStatus.src_status, 'OP' ) as ze_src_status ) as SrcProcessingStatus` — à tester, l'expression dépend de la version du compilateur.

### 6.2 Vue de consommation — texte, criticité et association

Dans `ZC_PurReqnItemSourcing` :

```abap
define view entity ZC_PurReqnItemSourcing
  as select from ZI_PurReqnItemSourcing as Item

  association [0..*] to ZC_SourceOfSupplyCandidate as _SourceCandidate
    on  $projection.Material = _SourceCandidate.Material
    and $projection.Plant    = _SourceCandidate.Plant

  association [0..1] to ZC_SrcStatusVH as _SrcStatus
    on  $projection.SrcProcessingStatus = _SrcStatus.SrcProcessingStatus
{
  key Item.PurchaseRequisition,
  key Item.PurchaseRequisitionItem,
      …
      --- Statut
      @ObjectModel.text.element: ['SrcProcessingStatusText']
      Item.SrcProcessingStatus,

      @Semantics.text: true
      _SrcStatus.SrcProcessingStatusText as SrcProcessingStatusText,

      // 1 = rouge, 2 = orange, 3 = vert, 0 = neutre
      case Item.SrcProcessingStatus
        when 'CA' then 1
        when 'IP' then 2
        when 'CL' then 3
        else 0
      end as SrcProcessingStatusCriticality,
      …
      /* Associations */
      _SourceCandidate,
      _SrcStatus
}
```

Activer dans l'ordre : `ZI_SrcStatusVH` → `ZC_SrcStatusVH` → `ZI_PurReqnItemSourcing` → `ZC_PurReqnItemSourcing`.

> `@ObjectModel.text.element` se place dans la **vue**, pas dans la metadata extension : les annotations `@ObjectModel.*` ne sont pas extensibles en DDLX. Voir [annexe C](#annexe-c--où-placer-chaque-annotation).

---

## Étape 7 – Annotations UI

Dans la metadata extension `ZC_PurReqnItemSourcing` :

```abap
  @UI.lineItem:       [{ position: 125, label: 'Statut', importance: #HIGH,
                         criticality: 'SrcProcessingStatusCriticality' }]
  @UI.identification: [{ position: 130, label: 'Statut' }]
  @UI.selectionField: [{ position: 60 }]
  @UI.textArrangement: #TEXT_ONLY
  @Consumption.valueHelpDefinition: [{
    entity: { name: 'ZC_SrcStatusVH', element: 'SrcProcessingStatus' } }]
  @Consumption.filter.multipleSelections: true
  SrcProcessingStatus;

  @UI.hidden: true
  SrcProcessingStatusText;

  @UI.hidden: true
  SrcProcessingStatusCriticality;
```

| Annotation | Effet |
|---|---|
| `@Consumption.valueHelpDefinition` | Rattache la liste de valeurs au champ |
| `@UI.selectionField` | Place le champ dans la barre de filtres — **sans elle, aucune liste déroulante visible dans une app en lecture seule** |
| `@UI.textArrangement: #TEXT_ONLY` | Affiche « En cours » plutôt que `IP` |
| `criticality` | Pastille de couleur dans la colonne |
| `@Consumption.filter.multipleSelections: true` | Sélection multiple dans le filtre (`MultiComboBox`) |

Activer.

---

## Étape 8 – Exposition dans le service et republication

### 8.1 Service definition

```abap
@EndUserText.label: 'Determination source appro - DA ZCTL'
define service ZUI_PR_SOURCING {
  expose ZC_PurReqnItemSourcing     as PurReqnItemSourcing;
  expose ZC_SourceOfSupplyCandidate as SourceOfSupplyCandidate;
  expose ZC_SrcStatusVH             as SrcStatusVH;      // ← indispensable
  expose I_Plant                    as Plant;
  expose I_Product                  as Product;
}
```

**L'oubli de cette ligne est la première cause d'échec** : l'annotation `Common.ValueList` pointe alors vers un entity set absent du service, et l'interface n'affiche rien, sans message d'erreur.

### 8.2 Réactivation et purge

```text
1. Activer : ZI_SrcStatusVH → ZC_SrcStatusVH → ZI_PurReqnItemSourcing
             → ZC_PurReqnItemSourcing → DDLX → SRVD → SRVB
2. Service binding ZUI_PR_SOURCING_O2 → bouton Publish si ADT le propose
3. /IWFND/CACHE_CLEANUP      (métadonnées hub)
4. /IWBEP/CACHE_CLEANUP      (métadonnées backend)
5. /UI2/INVALIDATE_GLOBAL_CACHES
6. Navigateur : rechargement forcé (Ctrl+Maj+R)
```

> Les métadonnées OData sont mises en cache des deux côtés. Tant que les caches ne sont pas purgés, vous testez l'ancienne version du service et vous concluez à tort que les annotations ne fonctionnent pas.

---

## Étape 9 – Contrôle du `$metadata`

Ouvrir dans le navigateur :

```text
/sap/opu/odata/sap/ZUI_PR_SOURCING_O2/$metadata
```

Quatre contrôles, dans cet ordre :

| # | Ce qu'il faut trouver | Si absent |
|---|---|---|
| 1 | `<EntitySet Name="SrcStatusVH" …/>` | L'entité n'est pas exposée → étape 8.1 |
| 2 | Sur la propriété du champ : `sap:value-list="fixed-values"` | `sizeCategory` absente de la vue exposée, ou cache non purgé |
| 3 | Sur la même propriété : `sap:text="SrcProcessingStatusText"` | `@ObjectModel.text.element` ou `@Semantics.text` manquants |
| 4 | Un bloc `<Annotations Target="…SrcProcessingStatus">` avec `Common.ValueList`, un paramètre `ValueListParameterInOut` sur le code et un `ValueListParameterDisplayOnly` sur le texte | `@Consumption.valueHelpDefinition` mal orthographiée (nom de vue ou d'élément) |

Si la propriété porte `sap:value-list="standard"` au lieu de `fixed-values`, la liaison fonctionne mais le rendu sera une **boîte de dialogue** : seule l'annotation `sizeCategory` est en cause.

Test des données de la liste :

```text
/sap/opu/odata/sap/ZUI_PR_SOURCING_O2/SrcStatusVH?$format=json
```

Quatre entrées attendues, avec leurs textes.

📸 *Capture 8-05 : `$metadata` montrant `sap:value-list="fixed-values"`*

---

## Étape 10 – Test dans l'application

| # | Test | Attendu |
|---|---|---|
| T1 | Ouvrir l'application, barre de filtres | Champ **Statut** présent |
| T2 | Cliquer dans le champ | Liste déroulante à 4 entrées, libellés en français |
| T3 | Sélectionner « En cours » puis *Exécuter* | Seuls les postes `IP` sont listés |
| T4 | Colonne Statut du tableau | Texte affiché, pastille de couleur selon la criticité |
| T5 | Changer la langue de connexion en anglais | Libellés traduits (si la traduction du domaine est faite) |
| T6 | Object Page d'un poste | Statut affiché en lecture, sans liste déroulante — comportement normal en V1 |
| T7 | Console `F12` → onglet Réseau | Appel `SrcStatusVH` en 200, aucune 404 |

📸 *Capture 8-06 : liste déroulante ouverte dans la barre de filtres*
📸 *Capture 8-07 : colonne Statut avec pastilles de criticité*

---

## Étape 11 – Rendre le champ modifiable

En lecture seule, la liste déroulante n'existe que dans les filtres. Pour l'obtenir **sur le champ** (Object Page en édition), il faut un comportement RAP — c'est le prolongement du document 06.

```abap
" Behavior definition de l'entité d'interface
unmanaged implementation in class zbp_i_purreqnitemsourcing unique;
strict ( 1 );

define behavior for ZI_PurReqnItemSourcing alias PurReqnItem
lock master
authorization master ( instance )
{
  update;
  field ( readonly ) PurchaseRequisition, PurchaseRequisitionItem;
  field ( mandatory ) SrcProcessingStatus;
}
```

```abap
" Behavior projection
projection;
strict ( 1 );

define behavior for ZC_PurReqnItemSourcing alias PurReqnItemSourcing
{
  use update;
  use field ( mandatory ) SrcProcessingStatus;
}
```

Côté implémentation : méthode `update` du behavior pool qui écrit dans `ZMM_SRC_STATUS` (`MODIFY` dans la phase `save`, sans `COMMIT WORK` — voir document 06, annexe A).

Conséquences à accepter :

| Sujet | Impact |
|---|---|
| Vue de consommation | Doit devenir `define root view entity … provider contract transactional_query as projection on …` (document 04 REV2, annexe C) |
| Verrouillage | `FOR LOCK` à implémenter (`ENQUEUE_EMEBANE` ou verrou propre sur la table Z) |
| Autorisation | `get_instance_authorizations` en activité `02` |
| UI | Le champ devient une `ComboBox` en mode édition, alimentée par la même liste |

---

## Étape 12 – Transport

| Ordre | Objets |
|---|---|
| Workbench | `R3TR DOMA ZD_SRC_STATUS`, `R3TR DTEL ZE_SRC_STATUS`, `R3TR TABL ZMM_SRC_STATUS`, `R3TR FUGR ZFG_MM_SRC_STATUS`, `R3TR DDLS ZI_SrcStatusVH` et `ZC_SrcStatusVH`, `DDLS` modifiées (`ZI_`/`ZC_PurReqnItemSourcing`), `R3TR DDLX ZC_PURREQNITEMSOURCING`, `R3TR SRVD` et `R3TR SRVB` |
| Customizing | Entrées de `ZMM_SRC_STATUS` si vous livrez un jeu initial, traductions du domaine |

Après import dans le système cible : purger les caches (étape 8.2), puis rejouer les tests T1 à T5. **Les valeurs fixes du domaine suivent le transport du domaine** : aucune saisie manuelle n'est nécessaire dans la cible.

---

## Annexe A – Pourquoi une liste déroulante ne sort pas : les 12 causes

| # | Cause | Contrôle | Correction |
|---|---|---|---|
| 1 | Entité d'aide à la saisie non exposée dans la service definition | `$metadata` : `EntitySet` absent | Étape 8.1 |
| 2 | `@ObjectModel.resultSet.sizeCategory` posée sur la vue d'interface mais pas sur la vue exposée | `sap:value-list="standard"` | Poser l'annotation sur les deux |
| 3 | Valeur `#S` au lieu de `#XS` | Activation ou rendu KO | Utiliser `#XS` |
| 4 | Caches de métadonnées non purgés | Ancien `$metadata` | `/IWFND/CACHE_CLEANUP` + `/IWBEP/CACHE_CLEANUP` |
| 5 | Nom de vue ou d'élément erroné dans `valueHelpDefinition` | Pas de bloc `Common.ValueList` | Respecter la casse exacte des alias CDS |
| 6 | Champ absent de la barre de filtres et application en lecture seule | Aucun endroit où saisir | Ajouter `@UI.selectionField` ou rendre le champ modifiable (étape 11) |
| 7 | Textes non annotés | Liste avec codes seulement | `@ObjectModel.text.element` + `@Semantics.text: true` |
| 8 | Texte non filtré sur la langue | Lignes dupliquées ou vides | `$session.system_language` |
| 9 | Clé représentative non déclarée | Aide incomplète | `@ObjectModel.representativeKey` |
| 10 | Types incompatibles entre le champ et la clé de la liste | Avertissement d'activation, liste inopérante | `cast( … as ze_src_status )` |
| 11 | Domaine sans valeurs fixes actives | Liste vide, `Data Preview` vide | `SE11`, vérifier `AS4LOCAL = 'A'` |
| 12 | Binding OData V4 avec des annotations pensées pour V2 | Rendu en dialogue | `Common.ValueListWithFixedValues` en V4 |

---

## Annexe B – Variantes

| Besoin | Solution |
|---|---|
| Valeurs maintenues par le métier sans transport | Remplacer le domaine par une **table de codes Z** (code typé `ZE_SRC_STATUS` + table de textes) et pointer la vue d'aide à la saisie dessus ; le reste du document est inchangé |
| Liste déroulante dans une boîte de paramètres d'action RAP | Même annotation `@Consumption.valueHelpDefinition` sur l'élément de l'**entité abstraite** des paramètres (document 06, étape 2) |
| Validation d'entrée | `useForValidation: true` dans `valueHelpDefinition` si votre release l'accepte, sinon contrôle dans la behavior definition |
| Liste filtrée par un autre champ | `additionalBinding` avec `localElement` / `element` et `usage: #FILTER_AND_RESULT` |
| Plus d'une vingtaine de valeurs | Retirer `sizeCategory` : la boîte de dialogue de recherche est plus confortable qu'une liste déroulante longue |
| Migration en OData V4 | Remplacer le rendu attendu : `Common.ValueListWithFixedValues` est généré à partir de la même annotation CDS, mais nécessite une version SAPUI5 plus récente côté FLP |

---

## Annexe C – Où placer chaque annotation

| Annotation | Vue (DDL) | Metadata extension (DDLX) |
|---|---|---|
| `@ObjectModel.resultSet.sizeCategory` | ✔ (vue d'aide à la saisie) | ✗ |
| `@ObjectModel.representativeKey` | ✔ | ✗ |
| `@ObjectModel.text.element` | ✔ | ✗ non extensible |
| `@Semantics.text` | ✔ | ✗ |
| `@Consumption.valueHelpDefinition` | ✔ possible | ✔ **recommandé** |
| `@Consumption.filter.*` | ✔ possible | ✔ recommandé |
| `@UI.*` | à éviter | ✔ |
| `@Search.*` | ✔ | ✗ |
| `@Metadata.allowExtensions` | ✔ obligatoire sur la vue annotée en DDLX | — |

Règle simple : **ce qui décrit le modèle reste dans la vue, ce qui décrit l'écran va dans la metadata extension.**

---

## Captures à réaliser

| N° | Écran | Fichier suggéré |
|---|---|---|
| 8-01 | `I_DomainFixedValue` dans ADT | `img/08-01-domainfixedvalue.png` |
| 8-02 | Valeurs fixes du domaine | `img/08-02-domaine.png` |
| 8-03 | `SM30` avec aide à la saisie | `img/08-03-sm30.png` |
| 8-04 | Data Preview de la vue VH | `img/08-04-preview-vh.png` |
| 8-05 | `$metadata` avec `fixed-values` | `img/08-05-metadata.png` |
| 8-06 | Liste déroulante dans le filtre | `img/08-06-dropdown.png` |
| 8-07 | Colonne Statut avec criticité | `img/08-07-criticality.png` |

---

## Références

- SAP Help – *Simple Value Help* : « Value Help as Dropdown List », `@ObjectModel : { resultSet.sizeCategory: #XS }` produit `sap:value-list="fixed-values"` dans le `$metadata`
- SAP Help – *ABAP RESTful Application Programming Model : Providing Value Help*
- SAP-samples – *abap-platform-fiori-feature-showcase*, chapitre *Value Help* (`/DMO/FSA_I_Criticality`, `additionalBinding`, `useForValidation`)
- SAP Community – *Fixed value List with CDS annotations* (annotation inopérante avant NW 7.52)
- SAP Community – *Using Domain Fixed Values as Value Help in SAP RAP Fiori Elements* (`DD07L` / `DD07T`)
- sapdev.eu – *Domain Fixed Values as CDS Value Help in S/4HANA* (`I_DomainFixedValue`, `_DomainFixedValueText`, `cast` vers l'élément de données)
- Documents internes : `04_App_RAP_Determination_Source_Appro_REV2.md`, `06_App_RAP_V2_Action_Affectation_Source.md`
