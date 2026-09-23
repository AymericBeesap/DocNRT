# Application « Détermination Source Appro » — V2 : action RAP et annotation `#FOR_ACTION`

**Contexte :** SAP S/4HANA Cloud Private Edition 2021 FPS02 (ABAP Platform 2021 / SAP_BASIS 7.56, SAPUI5 1.96.x)
**Point de départ :** V1 livrée (document 04) — service RAP en lecture seule, application Fiori elements OData V2, pop-up front-end de consultation des sources.
**Objet de la V2 :** transformer le modèle en **RAP business object unmanaged** portant une action `AssignSourceOfSupply`, exposée par l'annotation **`@UI.lineItem: [{ type: #FOR_ACTION }]`**, qui affecte réellement la source d'approvisionnement au poste de DA via `BAPI_PR_CHANGE`.

---

## Sommaire

1. [Périmètre et delta V1 → V2](#1-périmètre-et-delta-v1--v2)
2. [Décisions d'architecture](#2-décisions-darchitecture)
3. [Prérequis](#3-prérequis)
4. [Étape 1 – Classe de messages](#étape-1--classe-de-messages)
5. [Étape 2 – Entité abstraite des paramètres](#étape-2--entité-abstraite-des-paramètres)
6. [Étape 3 – Behavior definition](#étape-3--behavior-definition)
7. [Étape 4 – Behavior implementation](#étape-4--behavior-implementation)
8. [Étape 5 – Behavior projection](#étape-5--behavior-projection)
9. [Étape 6 – Annotation `#FOR_ACTION`](#étape-6--annotation-for_action)
10. [Étape 7 – Service definition et binding](#étape-7--service-definition-et-binding)
11. [Étape 8 – Nettoyage du front V1](#étape-8--nettoyage-du-front-v1)
12. [Étape 9 – Tests](#étape-9--tests)
13. [Étape 10 – Transport et mise en qualité](#étape-10--transport-et-mise-en-qualité)
14. [Annexe A – Règles RAP unmanaged à respecter](#annexe-a--règles-rap-unmanaged-à-respecter)
15. [Annexe B – Variantes et évolutions](#annexe-b--variantes-et-évolutions)
16. [Dépannage](#dépannage)
17. [Captures à réaliser](#captures-à-réaliser)
18. [Références](#références)

---

## 1. Périmètre et delta V1 → V2

| Élément | V1 (doc 04) | V2 (ce document) |
|---|---|---|
| Modèle | Vues CDS + projections, **pas de behavior definition** | + **BDEF unmanaged**, behavior pool, behavior projection |
| Écriture | Aucune | `BAPI_PR_CHANGE` dans la phase de sauvegarde RAP |
| Déclenchement UI | Bouton custom du manifest + fragment XML | **`@UI.lineItem: [{ type: #FOR_ACTION }]`** → boîte de paramètres générée |
| Contrôle d'activation du bouton | `requiresSelection` côté front | **Feature control d'instance** (`%action-…` désactivé si source déjà affectée) |
| Autorisation | DCL en lecture | DCL + **instance authorization** sur `M_BANF_WRK` en modification |
| Verrouillage | Sans objet | `lock master` + `ENQUEUE_EMEBANE` |
| Messages | `MessageToast` front | Messages RAP (`reported`) affichés dans le message popover |
| Sources candidates | Pop-up front-end | Aide à la saisie du paramètre + facette de l'Object Page (conservée) |

### Maquette V2 — List Report et boîte de paramètres

```text
┌────────────────────────────────────────────────────────────────────────────────────┐
│ Postes de DA (12)                     ┏━━━━━━━━━━━━━━━━━━━━━━━┓   ⚙  ⤓            │
│                                       ┃ Affecter une source   ┃ ◀ DataFieldForAction│
│                                       ┗━━━━━━━━━━━━━━━━━━━━━━━┛   (grisé si aucune  │
│ ────────────────────────────────────────────────────────────────  ligne cochée ou   │
│ ☑ 10000123/10 │ TG11 │ 1010 │ 100 PC │ 12.10.2026 │ ⬤ Aucune      source déjà posée)│
│ ☐ 10000123/20 │ TG12 │ 1010 │  50 PC │ 05.10.2026 │ ⬤ 17300001                     │
└────────────────────────────────────────────────────────────────────────────────────┘
                                   │ clic
                                   ▼
        ┌────────────────────────────────────────────────────────┐
        │  Affecter une source                                   │
        ├────────────────────────────────────────────────────────┤
        │  Fournisseur           [ 17300001        ] 🔍          │
        │  Contrat               [                 ] 🔍          │
        │  Poste de contrat      [                 ]             │
        │  Fiche info achat      [                 ] 🔍          │
        │  Division de livraison [                 ]             │
        ├────────────────────────────────────────────────────────┤
        │                              [ Affecter ]  [ Annuler ] │
        └────────────────────────────────────────────────────────┘
```

---

## 2. Décisions d'architecture

| Décision | Justification |
|---|---|
| **BO `unmanaged`** | La persistance est la demande d'achat standard (`EBAN`), gérée par le noyau MM. Le runtime managed nécessiterait une table persistante propre : inadapté. L'unmanaged encapsule l'API existante. |
| **`BAPI_PR_CHANGE`** | API standard de modification de DA, qui déclenche les contrôles MM, la stratégie de libération et les versions. Sur 2021 on-premise, la demande d'achat n'est pas exposée comme BO RAP standard modifiable : pas d'EML possible. |
| **Action instanciée avec `result [1] $self`** | Permet à SAP Fiori elements de rafraîchir la ligne après exécution. Impose d'implémenter `FOR READ`. |
| **Paramètres dans une entité abstraite** | La boîte de dialogue de saisie est générée par le framework à partir de l'entité abstraite : plus de fragment XML à maintenir. |
| **Binding OData V2 conservé** | Continuité avec la V1 et richesse des annotations sur SAPUI5 1.96. Fiori elements V2 génère bien une boîte de paramètres pour un `DataFieldForAction` avec paramètres. La bascule V4 est traitée en [Annexe B](#annexe-b--variantes-et-évolutions). |
| **Appel du BAPI dans la phase `save`** | Règle RAP : l'action prépare et contrôle, la sauvegarde écrit. Aucun `COMMIT WORK` explicite : le framework le déclenche. |

> ⚠️ **Conséquence sur la V1** : la vue de consommation `ZC_PurReqnItemSourcing` doit impérativement porter `provider contract transactional_query` et être une `define root view entity`. Le repli « vue de consommation simple » évoqué dans le document 04 §5.1 n'est plus utilisable.

---

## 3. Prérequis

| Élément | Contrôle | Où |
|---|---|---|
| V1 opérationnelle | Application, service `ZUI_PR_SOURCING_O2`, données de test `ZCTL` | Document 04 |
| Projection racine | `define root view entity ZC_PurReqnItemSourcing provider contract transactional_query` | ADT |
| `BAPI_PR_CHANGE` | Existe, structures `BAPIMEREQITEMIMP` / `BAPIMEREQITEMX` | `SE37`, `SE11` |
| Objet de verrouillage | `EMEBANE` (FM `ENQUEUE_EMEBANE` / `DEQUEUE_EMEBANE`, arguments `BANFN`, `BNFPO`) | `SE11` / `SE37` |
| Objets d'autorisation DA | `M_BANF_WRK`, `M_BANF_EKG`, `M_BANF_EKO`, `M_BANF_BSA` | `SU21` |
| Jeu de test | DA `ZCTL` **sans** source, DA `ZCTL` **avec** source, DA en cours de libération | `ME53N` |
| Autorisations développeur | `S_DEVELOP` sur `BDEF`, `CLAS`, `DDLS`, `DDLX`, `MSAG`, `SRVD`, `SRVB` dans `ZMM_SOURCING` | `PFCG` |
| Ordre de transport | Nouvel ordre Workbench `S4DK9yyyyy` « V2 action affectation source » | `SE09` |

---

## Étape 1 – Classe de messages

`SE91` → classe `ZMM_PR_SRC` (package `ZMM_SOURCING`) :

| N° | Texte | Usage |
|---|---|---|
| `001` | `Indiquez au moins une source d''approvisionnement` | Aucun paramètre renseigné |
| `002` | `Poste &1/&2 : une source est deja affectee` | Garde-fou serveur (le bouton est déjà grisé) |
| `003` | `Poste &1/&2 introuvable ou supprime` | Clé invalide |
| `004` | `DA &1 verrouillee par l''utilisateur &2` | Conflit de verrou |
| `005` | `Source affectee au poste &1/&2` | Message de succès (type `S`) |
| `006` | `Echec de l''affectation pour la DA &1 : &2` | Retour BAPI en erreur |
| `007` | `Contrat &1 poste &2 : saisissez les deux valeurs ensemble` | Contrôle de cohérence |

Traduction : `SE91` → *Aller à → Traduction* (ou `SE63`).

---

## Étape 2 – Entité abstraite des paramètres

ADT : clic droit sur le package → **New → Other ABAP Repository Object → Core Data Services → Data Definition** → nom `ZD_AssignSourceOfSupply`, template *Define Abstract Entity*.

```abap
@EndUserText.label: 'Parametres affectation source appro'
define abstract entity ZD_AssignSourceOfSupply
{
      @EndUserText.label: 'Fournisseur'
      @Consumption.valueHelpDefinition: [{
        entity: { name: 'ZC_SourceOfSupplyCandidate', element: 'Supplier' } }]
      FixedSupplier          : lifnr;

      @EndUserText.label: 'Contrat'
      @Consumption.valueHelpDefinition: [{
        entity: { name: 'ZC_SourceOfSupplyCandidate', element: 'PurchaseAgreement' } }]
      PurchaseContract       : konnr;

      @EndUserText.label: 'Poste de contrat'
      PurchaseContractItem   : ktpnr;

      @EndUserText.label: 'Fiche info achat'
      PurchasingInfoRecord   : infnr;

      @EndUserText.label: 'Division de livraison'
      SupplyingPlant         : reswk;
}
```

Activer.

> **À vérifier** : les éléments de données `lifnr`, `konnr`, `ktpnr`, `infnr`, `reswk` doivent exister tels quels dans votre système (`SE11`). Si `reswk` n'est pas disponible, utiliser `werks_d`.
>
> **Aide à la saisie** : elle n'est rendue dans la boîte de paramètres que si l'entité de value help est exposée dans la service definition (étape 7). Avec le binding OData V2, le comportement de l'aide à la saisie sur un paramètre d'action est plus limité qu'en V4 : à valider sur votre système, sinon la saisie reste libre et le contrôle se fait côté serveur.

---

## Étape 3 – Behavior definition

Clic droit sur `ZI_PurReqnItemSourcing` → **New Behavior Definition** :

| Champ | Valeur |
|---|---|
| Name | `ZI_PurReqnItemSourcing` (identique à la vue) |
| Description | `Affectation source appro - implementation unmanaged` |
| Implementation Type | **Unmanaged** |

```abap
unmanaged implementation in class zbp_i_purreqnitemsourcing unique;
strict ( 1 );

define behavior for ZI_PurReqnItemSourcing alias PurReqnItem
lock master
authorization master ( instance )
{
  // Aucun create / update / delete : le BO reste non modifiable directement.
  // Seule l'action ci-dessous modifie la DA, via l'API standard.

  action ( features : instance ) AssignSourceOfSupply
         parameter ZD_AssignSourceOfSupply
         result [1] $self;
}
```

Activer : ADT propose un **quick fix** (`Ctrl+1`) pour générer la classe d'implémentation `ZBP_I_PURREQNITEMSOURCING` avec les squelettes des méthodes.

> `strict ( 1 )` est le mode compatible le plus large sur 2021. Si votre système accepte `strict ( 2 )`, le préférer : les contrôles syntaxiques y sont plus stricts et plus proches des releases récentes.

---

## Étape 4 – Behavior implementation

Classe `ZBP_I_PURREQNITEMSOURCING`, onglet *Local Types* (`CCIMP`).

### 4.1 Déclarations

```abap
CLASS lhc_purreqnitem DEFINITION INHERITING FROM cl_abap_behavior_handler.

  PUBLIC SECTION.
    TYPES: BEGIN OF ty_assignment,
             purchase_requisition      TYPE banfn,
             purchase_requisition_item TYPE bnfpo,
             fixed_supplier            TYPE lifnr,
             purchase_contract         TYPE konnr,
             purchase_contract_item    TYPE ktpnr,
             purchasing_info_record    TYPE infnr,
             supplying_plant           TYPE reswk,
           END OF ty_assignment,
           tt_assignment TYPE STANDARD TABLE OF ty_assignment WITH EMPTY KEY.

    " Tampon partagé entre la phase d'interaction et la phase de sauvegarde
    CLASS-DATA gt_assignment TYPE tt_assignment.

  PRIVATE SECTION.

    METHODS get_instance_authorizations FOR INSTANCE AUTHORIZATION
      IMPORTING keys REQUEST requested_authorizations FOR PurReqnItem RESULT result.

    METHODS get_instance_features FOR INSTANCE FEATURES
      IMPORTING keys REQUEST requested_features FOR PurReqnItem RESULT result.

    METHODS read FOR READ
      IMPORTING keys FOR READ PurReqnItem RESULT result.

    METHODS lock FOR LOCK
      IMPORTING keys FOR LOCK PurReqnItem.

    METHODS AssignSourceOfSupply FOR MODIFY
      IMPORTING keys FOR ACTION PurReqnItem~AssignSourceOfSupply RESULT result.

ENDCLASS.
```

### 4.2 Lecture (obligatoire en unmanaged)

```abap
  METHOD read.

    SELECT FROM ZI_PurReqnItemSourcing
      FIELDS *
      FOR ALL ENTRIES IN @keys
      WHERE PurchaseRequisition     = @keys-PurchaseRequisition
        AND PurchaseRequisitionItem = @keys-PurchaseRequisitionItem
      INTO CORRESPONDING FIELDS OF TABLE @result.

    " Clés demandées mais absentes -> signalées comme non trouvées
    LOOP AT keys INTO DATA(ls_key).
      IF NOT line_exists( result[ PurchaseRequisition     = ls_key-PurchaseRequisition
                                  PurchaseRequisitionItem = ls_key-PurchaseRequisitionItem ] ).
        APPEND VALUE #( %tky = ls_key-%tky ) TO failed-purreqnitem.
      ENDIF.
    ENDLOOP.

  ENDMETHOD.
```

### 4.3 Verrouillage

```abap
  METHOD lock.

    LOOP AT keys INTO DATA(ls_key).

      CALL FUNCTION 'ENQUEUE_EMEBANE'
        EXPORTING
          mode_eban = 'E'
          mandt     = sy-mandt
          banfn     = ls_key-PurchaseRequisition
          bnfpo     = ls_key-PurchaseRequisitionItem
          _scope    = '1'          " verrou libéré à la fin de la LUW RAP
        EXCEPTIONS
          foreign_lock   = 1
          system_failure = 2
          OTHERS         = 3.

      IF sy-subrc <> 0.
        APPEND VALUE #( %tky = ls_key-%tky ) TO failed-purreqnitem.
        APPEND VALUE #( %tky = ls_key-%tky
                        %msg = new_message( id       = 'ZMM_PR_SRC'
                                            number   = '004'
                                            v1       = ls_key-PurchaseRequisition
                                            v2       = sy-msgv1
                                            severity = if_abap_behv_message=>severity-error ) )
               TO reported-purreqnitem.
      ENDIF.

    ENDLOOP.

  ENDMETHOD.
```

> Le mode `E` est ré-attribuable au même propriétaire de LUW : le verrou posé ici n'empêche pas `BAPI_PR_CHANGE` de poser le sien dans la même transaction.

### 4.4 Contrôle d'activation du bouton (feature control)

```abap
  METHOD get_instance_features.

    READ ENTITIES OF ZI_PurReqnItemSourcing IN LOCAL MODE
      ENTITY PurReqnItem
        FIELDS ( SourceOfSupplyIsAssigned )
        WITH CORRESPONDING #( keys )
      RESULT DATA(lt_item)
      FAILED failed.

    result = VALUE #( FOR ls_item IN lt_item
                      ( %tky = ls_item-%tky
                        %action-AssignSourceOfSupply =
                          COND #( WHEN ls_item-SourceOfSupplyIsAssigned = 'X'
                                  THEN if_abap_behv=>fc-o-disabled
                                  ELSE if_abap_behv=>fc-o-enabled ) ) ).

  ENDMETHOD.
```

### 4.5 Autorisation d'instance

```abap
  METHOD get_instance_authorizations.

    READ ENTITIES OF ZI_PurReqnItemSourcing IN LOCAL MODE
      ENTITY PurReqnItem
        FIELDS ( Plant PurchasingGroup )
        WITH CORRESPONDING #( keys )
      RESULT DATA(lt_item)
      FAILED failed.

    LOOP AT lt_item INTO DATA(ls_item).

      AUTHORITY-CHECK OBJECT 'M_BANF_WRK'
        ID 'WERKS' FIELD ls_item-Plant
        ID 'ACTVT' FIELD '02'.

      APPEND VALUE #( %tky = ls_item-%tky
                      %action-AssignSourceOfSupply =
                        COND #( WHEN sy-subrc = 0 THEN if_abap_behv=>auth-allowed
                                                  ELSE if_abap_behv=>auth-unauthorized ) )
             TO result.

    ENDLOOP.

  ENDMETHOD.
```

### 4.6 Action

```abap
  METHOD AssignSourceOfSupply.

    LOOP AT keys INTO DATA(ls_key).

      DATA(ls_param) = ls_key-%param.

      " 1. Au moins une source renseignée
      IF ls_param-FixedSupplier        IS INITIAL
     AND ls_param-PurchaseContract     IS INITIAL
     AND ls_param-PurchasingInfoRecord IS INITIAL
     AND ls_param-SupplyingPlant       IS INITIAL.
        APPEND VALUE #( %tky = ls_key-%tky ) TO failed-purreqnitem.
        APPEND VALUE #( %tky = ls_key-%tky
                        %msg = new_message( id = 'ZMM_PR_SRC' number = '001'
                                            severity = if_abap_behv_message=>severity-error ) )
               TO reported-purreqnitem.
        CONTINUE.
      ENDIF.

      " 2. Contrat et poste de contrat vont ensemble
      IF ( ls_param-PurchaseContract IS NOT INITIAL AND ls_param-PurchaseContractItem IS INITIAL )
      OR ( ls_param-PurchaseContract IS INITIAL     AND ls_param-PurchaseContractItem IS NOT INITIAL ).
        APPEND VALUE #( %tky = ls_key-%tky ) TO failed-purreqnitem.
        APPEND VALUE #( %tky = ls_key-%tky
                        %msg = new_message( id = 'ZMM_PR_SRC' number = '007'
                                            v1 = |{ ls_param-PurchaseContract }|
                                            v2 = |{ ls_param-PurchaseContractItem }|
                                            severity = if_abap_behv_message=>severity-error ) )
               TO reported-purreqnitem.
        CONTINUE.
      ENDIF.

      " 3. État courant du poste
      READ ENTITIES OF ZI_PurReqnItemSourcing IN LOCAL MODE
        ENTITY PurReqnItem
          ALL FIELDS WITH VALUE #( ( %tky = ls_key-%tky ) )
        RESULT DATA(lt_item).

      IF lt_item IS INITIAL.
        APPEND VALUE #( %tky = ls_key-%tky ) TO failed-purreqnitem.
        APPEND VALUE #( %tky = ls_key-%tky
                        %msg = new_message( id = 'ZMM_PR_SRC' number = '003'
                                            v1 = ls_key-PurchaseRequisition
                                            v2 = |{ ls_key-PurchaseRequisitionItem }|
                                            severity = if_abap_behv_message=>severity-error ) )
               TO reported-purreqnitem.
        CONTINUE.
      ENDIF.

      DATA(ls_item) = lt_item[ 1 ].

      IF ls_item-SourceOfSupplyIsAssigned = 'X'.
        APPEND VALUE #( %tky = ls_key-%tky ) TO failed-purreqnitem.
        APPEND VALUE #( %tky = ls_key-%tky
                        %msg = new_message( id = 'ZMM_PR_SRC' number = '002'
                                            v1 = ls_key-PurchaseRequisition
                                            v2 = |{ ls_key-PurchaseRequisitionItem }|
                                            severity = if_abap_behv_message=>severity-error ) )
               TO reported-purreqnitem.
        CONTINUE.
      ENDIF.

      " 4. Mise en tampon pour la phase de sauvegarde
      APPEND VALUE #( purchase_requisition      = ls_key-PurchaseRequisition
                      purchase_requisition_item = ls_key-PurchaseRequisitionItem
                      fixed_supplier            = ls_param-FixedSupplier
                      purchase_contract         = ls_param-PurchaseContract
                      purchase_contract_item    = ls_param-PurchaseContractItem
                      purchasing_info_record    = ls_param-PurchasingInfoRecord
                      supplying_plant           = ls_param-SupplyingPlant )
             TO gt_assignment.

      " 5. Résultat renvoyé à l'UI ; Fiori elements relit l'entité après la sauvegarde
      APPEND VALUE #( %tky   = ls_key-%tky
                      %param = CORRESPONDING #( ls_item ) ) TO result.

      APPEND VALUE #( %tky = ls_key-%tky
                      %msg = new_message( id = 'ZMM_PR_SRC' number = '005'
                                          v1 = ls_key-PurchaseRequisition
                                          v2 = |{ ls_key-PurchaseRequisitionItem }|
                                          severity = if_abap_behv_message=>severity-success ) )
             TO reported-purreqnitem.

    ENDLOOP.

  ENDMETHOD.
```

### 4.7 Classe de sauvegarde

Toujours dans `CCIMP` :

```abap
CLASS lsc_zi_purreqnitemsourcing DEFINITION INHERITING FROM cl_abap_behavior_saver.
  PROTECTED SECTION.
    METHODS check_before_save REDEFINITION.
    METHODS save              REDEFINITION.
    METHODS cleanup           REDEFINITION.
    METHODS cleanup_finalize  REDEFINITION.
ENDCLASS.

CLASS lsc_zi_purreqnitemsourcing IMPLEMENTATION.

  METHOD check_before_save.
    " Emplacement recommandé pour un contrôle final bloquant.
    " Si BAPI_PR_CHANGE expose le paramètre TESTRUN sur votre release,
    " l'appeler ici en simulation et remonter les erreurs dans reported/failed.
  ENDMETHOD.

  METHOD save.

    DATA: lt_item   TYPE STANDARD TABLE OF bapimereqitemimp,
          lt_itemx  TYPE STANDARD TABLE OF bapimereqitemx,
          lt_return TYPE STANDARD TABLE OF bapiret2.

    CHECK lhc_purreqnitem=>gt_assignment IS NOT INITIAL.

    LOOP AT lhc_purreqnitem=>gt_assignment INTO DATA(ls_assign)
         GROUP BY ( pr = ls_assign-purchase_requisition )
         INTO DATA(ls_group).

      CLEAR: lt_item, lt_itemx, lt_return.

      LOOP AT GROUP ls_group INTO DATA(ls_line).

        APPEND VALUE #( preq_item  = ls_line-purchase_requisition_item
                        fixed_vend = ls_line-fixed_supplier
                        agreement  = ls_line-purchase_contract
                        agmt_item  = ls_line-purchase_contract_item
                        info_rec   = ls_line-purchasing_info_record )
               TO lt_item.

        APPEND VALUE #( preq_item  = ls_line-purchase_requisition_item
                        preq_itemx = abap_true
                        fixed_vend = abap_true
                        agreement  = abap_true
                        agmt_item  = abap_true
                        info_rec   = abap_true )
               TO lt_itemx.

      ENDLOOP.

      CALL FUNCTION 'BAPI_PR_CHANGE'
        EXPORTING
          number  = ls_group-pr
        TABLES
          return  = lt_return
          pritem  = lt_item
          pritemx = lt_itemx.

      " Pas de BAPI_TRANSACTION_COMMIT ni de COMMIT WORK :
      " le COMMIT est déclenché par le framework RAP en fin de LUW.

      LOOP AT lt_return INTO DATA(ls_return) WHERE type CA 'AEX'.
        " Journalisation applicative : à cette phase, les messages ne remontent plus à l'UI
        MESSAGE ID ls_return-id TYPE 'I' NUMBER ls_return-number
          WITH ls_return-message_v1 ls_return-message_v2
               ls_return-message_v3 ls_return-message_v4
          INTO DATA(lv_text).
        " Exemple : écriture dans le log applicatif (BAL / SLG1)
      ENDLOOP.

    ENDLOOP.

    CLEAR lhc_purreqnitem=>gt_assignment.

  ENDMETHOD.

  METHOD cleanup.
    CLEAR lhc_purreqnitem=>gt_assignment.
  ENDMETHOD.

  METHOD cleanup_finalize.
    CLEAR lhc_purreqnitem=>gt_assignment.
  ENDMETHOD.

ENDCLASS.
```

Activer la classe (`Ctrl+F3`), puis la behavior definition.

> **À vérifier dans `SE11`** : la structure `BAPIMEREQITEMIMP` de votre release contient-elle `INFO_REC` et une zone de division de livraison (`SUPPL_PLNT` / `SUPPLYING_PLANT`) ? Adapter les champs alimentés et leur pendant dans `BAPIMEREQITEMX`. Ne renseigner dans `…X` que les zones réellement transmises.

📸 *Capture 6-01 : BDEF activée*
📸 *Capture 6-02 : classe de comportement dans ADT*

---

## Étape 5 – Behavior projection

Clic droit sur `ZC_PurReqnItemSourcing` → **New Behavior Definition** → *Projection* :

```abap
projection;
strict ( 1 );

define behavior for ZC_PurReqnItemSourcing alias PurReqnItemSourcing
{
  use action AssignSourceOfSupply;
}
```

Activer. Sans cette projection, l'action existe dans le BO mais n'est pas exposée dans le service.

---

## Étape 6 – Annotation `#FOR_ACTION`

Modifier la metadata extension `ZC_PurReqnItemSourcing` (document 04, étape 6). Sur l'élément `PurchaseRequisition` :

```abap
  @UI.lineItem: [
    { position: 10, label: 'Demande d''achat', importance: #HIGH },
    { type: #FOR_ACTION,
      dataAction: 'AssignSourceOfSupply',
      label: 'Affecter une source',
      invocationGrouping: #CHANGE_SET }
  ]
  @UI.identification: [
    { position: 10 },
    { type: #FOR_ACTION,
      dataAction: 'AssignSourceOfSupply',
      label: 'Affecter une source' }
  ]
  @UI.fieldGroup: [{ qualifier: 'General', position: 10 }]
  PurchaseRequisition;
```

| Emplacement | Rendu |
|---|---|
| `@UI.lineItem` `#FOR_ACTION` | Bouton dans la barre d'outils du tableau, actif après sélection d'une ou plusieurs lignes |
| `@UI.identification` `#FOR_ACTION` | Bouton dans l'en-tête de l'Object Page |
| `invocationGrouping: #CHANGE_SET` | Les lignes sélectionnées sont traitées dans un seul change set : tout réussit ou tout échoue |

Activer la metadata extension.

> **Rappel V2 (OData)** : en SAP Fiori elements pour OData V2, un `DataFieldForAction` s'affiche comme bouton de barre d'outils avec sélection obligatoire. Le bouton **par ligne** (`inline`) est une capacité OData V4.

---

## Étape 7 – Service definition et binding

1. Compléter la service definition `ZUI_PR_SOURCING` si les aides à la saisie du paramètre pointent vers de nouvelles entités :

```abap
define service ZUI_PR_SOURCING {
  expose ZC_PurReqnItemSourcing     as PurReqnItemSourcing;
  expose ZC_SourceOfSupplyCandidate as SourceOfSupplyCandidate;
  expose I_Supplier                 as Supplier;
  expose I_Product                  as Product;
  expose I_PlantStdVH               as PlantVH;
  expose I_ProductStdVH             as ProductVH;
}
```

2. Réactiver la service definition puis le service binding `ZUI_PR_SOURCING_O2`.
3. **Republier** le *local service endpoint* si ADT le demande après l'ajout de l'action.
4. Contrôles :

| Contrôle | Attendu |
|---|---|
| `…/ZUI_PR_SOURCING_O2/$metadata` | `<FunctionImport Name="AssignSourceOfSupply" …>` avec les paramètres de `ZD_AssignSourceOfSupply` et les clés du poste |
| Métadonnées de l'entité | Annotation `DataFieldForAction` pointant sur l'action |
| Contrôle d'activation | Propriété générée pour le feature control de l'action (nom variable selon le binding) — sinon, le bouton reste actif et le garde-fou du §4.6/message 002 joue |
| Aperçu ADT (*Preview*) | Bouton **Affecter une source** dans la barre d'outils |

📸 *Capture 6-03 : `$metadata` avec le function import*
📸 *Capture 6-04 : aperçu Fiori depuis ADT avec le bouton*

---

## Étape 8 – Nettoyage du front V1

La pop-up front-end n'a plus de raison d'être : la boîte de paramètres est générée par le framework.

1. Dans `webapp/manifest.json`, **supprimer** le bloc de la custom action :

```diff
- "extends": {
-   "extensions": {
-     "sap.ui.controllerExtensions": {
-       "sap.suite.ui.generic.template.ListReport.view.ListReport": {
-         "controllerName": "zmmprsourcing.ext.controller.ListReportExt",
-         "sap.ui.generic.app": {
-           "PurReqnItemSourcing": {
-             "EntitySet": "PurReqnItemSourcing",
-             "Actions": { "openSourceDialog": { … } }
-           }
-         }
-       }
-     }
-   }
- },
```

2. Supprimer `webapp/ext/controller/ListReportExt.controller.js` et `webapp/ext/fragment/SourceOfSupplyDialog.fragment.xml`, ainsi que les clés i18n devenues inutiles (`ASSIGN_SOURCE`, `SRC_DIALOG_TITLE`, `SRC_SELECT_ONE`).
3. **Conserver** la facette `#LINEITEM_REFERENCE` sur `_SourceCandidate` : la liste des sources candidates reste utile dans l'Object Page avant de lancer l'action.
4. Ajouter les libellés de la boîte de paramètres si vous souhaitez les surcharger : ils proviennent par défaut des `@EndUserText.label` de l'entité abstraite.
5. Retester en local (`npm start`) puis redéployer :

```bash
npm run deploy
```

6. Backend : `/UI5/APP_INDEX_CALCULATE`, `/IWFND/CACHE_CLEANUP`, `/IWBEP/CACHE_CLEANUP`, `/UI2/INVALIDATE_GLOBAL_CACHES`.

> Si vous souhaitez garder une consultation des sources sans lancer l'action, conservez le fragment et la custom action : les deux mécanismes peuvent coexister.

📸 *Capture 6-05 : boîte de paramètres générée*

---

## Étape 9 – Tests

| # | Cas | Attendu |
|---|---|---|
| T1 | Aucune ligne sélectionnée | Bouton **Affecter une source** grisé |
| T2 | Ligne sans source sélectionnée | Bouton actif, boîte de paramètres à l'ouverture |
| T3 | Ligne avec source déjà affectée | Bouton grisé (feature control) ; si forcé par appel direct : message `002` |
| T4 | Paramètres tous vides | Message `001`, aucune écriture |
| T5 | Contrat sans poste de contrat | Message `007` |
| T6 | Fournisseur valide | Message de succès `005`, `EBAN-FLIEF` alimenté (`ME53N` / `SE16N` sur `EBAN`), indicateur *Source affectée* au vert après rafraîchissement |
| T7 | Sélection multiple (3 lignes) | Un seul change set ; toutes traitées ou aucune |
| T8 | DA ouverte en parallèle dans `ME52N` | Message `004`, aucune écriture |
| T9 | Utilisateur sans `M_BANF_WRK` sur la division | Action non autorisée (bouton indisponible / message d'autorisation) |
| T10 | Fournisseur bloqué ou inexistant | Le BAPI refuse ; contrôler le comportement et, si nécessaire, déplacer le contrôle dans `check_before_save` |
| T11 | DA en stratégie de libération | Comportement standard MM respecté (le BAPI applique les règles) |
| T12 | Annotations V1 | Statuts colorés, progression, onglets, filtres : inchangés |

Outils de diagnostic : point d'arrêt externe dans `AssignSourceOfSupply` et dans `save`, `/IWFND/ERROR_LOG`, `/IWBEP/ERROR_LOG`, `ST22`, `SM12` (verrous), `SE16N` sur `EBAN`, historique des modifications dans `ME53N`.

📸 *Capture 6-06 : message de succès dans le popover*
📸 *Capture 6-07 : `EBAN-FLIEF` alimenté après l'action*

---

## Étape 10 – Transport et mise en qualité

### Objets nouveaux ou modifiés

| Objet | Type | Statut |
|---|---|---|
| `ZMM_PR_SRC` | `R3TR MSAG` | Nouveau |
| `ZD_AssignSourceOfSupply` | `R3TR DDLS` | Nouveau |
| `ZI_PurReqnItemSourcing` | `R3TR BDEF` | Nouveau |
| `ZBP_I_PURREQNITEMSOURCING` | `R3TR CLAS` | Nouveau |
| `ZC_PurReqnItemSourcing` | `R3TR BDEF` (projection) | Nouveau |
| `ZC_PurReqnItemSourcing` | `R3TR DDLX` | Modifié (annotation `#FOR_ACTION`) |
| `ZC_PurReqnItemSourcing` | `R3TR DDLS` | Modifié si le `provider contract` a été ajouté |
| `ZUI_PR_SOURCING` / `ZUI_PR_SOURCING_O2` | `R3TR SRVD` / `R3TR SRVB` | Modifiés |
| `ZMM_PR_SRC` (BSP) | `R3TR WAPA` | Modifié (nettoyage front) |

### Séquence

1. `SE09` : libérer les tâches puis l'ordre `S4DK9yyyyy`.
2. `STMS` : importer en qualité **après** les ordres de la V1 (documents 04 et 05).
3. Actions post-import en qualité :
   - `/IWFND/MAINT_SERVICES` : service présent et actif ; republier le binding si l'action n'apparaît pas dans `$metadata` ;
   - `/UI5/APP_INDEX_CALCULATE` ;
   - `/IWFND/CACHE_CLEANUP`, `/IWBEP/CACHE_CLEANUP`, `/UI2/INVALIDATE_GLOBAL_CACHES` ;
   - `PFCG` : compléter le rôle `Z_BR_SOURCING` avec `M_BANF_WRK` / `M_BANF_EKG` en activité **02** (modification) ;
   - rejouer la grille de tests T1 à T12 sur des DA de qualité.

### Retour arrière

| Niveau | Procédure |
|---|---|
| Désactiver le bouton sans transport | `SM30`/table de paramétrage si vous en ajoutez une ; sinon, forcer `fc-o-disabled` dans `get_instance_features` |
| Retirer l'action de l'UI | Retirer l'entrée `#FOR_ACTION` de la DDLX, réactiver, purger les caches |
| Revenir à la V1 complète | Désactiver la behavior projection, retirer l'action de la BDEF, réactiver le service ; les vues et annotations V1 restent valides |

---

## Annexe A – Règles RAP unmanaged à respecter

| Règle | Pourquoi |
|---|---|
| Aucun `COMMIT WORK` ni `BAPI_TRANSACTION_COMMIT` dans le behavior pool | Le framework RAP pilote la LUW ; un commit manuel casse la transaction et peut provoquer un dump |
| Aucun `ROLLBACK WORK` | Idem ; utiliser `failed` / `reported` pour refuser une opération |
| Pas de dialogue (`POPUP_*`, `MESSAGE` sans `INTO`, `CALL SCREEN`) | Le code s'exécute dans un contexte OData sans interface |
| Contrôles bloquants au plus tôt | Dans l'action ou `check_before_save` : après, les messages ne remontent plus à l'utilisateur |
| Tampon en `CLASS-DATA` vidé dans `cleanup` / `cleanup_finalize` | Évite qu'une requête suivante rejoue des affectations fantômes |
| Messages typés `new_message( … )` avec classe de messages | Traduisibles, contrairement à `new_message_with_text` |
| Lecture systématique de l'état courant avant d'écrire | Le tampon peut être obsolète si plusieurs requêtes s'enchaînent |
| Volume | Le BAPI est appelé une fois par DA : regrouper les postes (`GROUP BY`) comme dans le code ci-dessus |

---

## Annexe B – Variantes et évolutions

| Variante | Quand l'utiliser | Impact |
|---|---|---|
| **Action statique** (`static action`) | Affectation de masse indépendante d'une ligne | Pas de verrou ni de feature control d'instance ; rendu comme bouton global |
| **Basculer en OData V4** | Montée de version SAPUI5 (≥ 1.108 côté FLP) | Boîte de paramètres plus riche, aides à la saisie plus fiables, bouton par ligne (`inline`), `Core.OperationAvailable` standardisé |
| **Ajouter une validation RAP** | Contrôles métiers avant sauvegarde | `validation checkSource on save { field … }` déclarée dans la BDEF |
| **Ajouter une determination** | Proposer automatiquement la source de la liste des sources | `determination proposeSource on modify { create; }` |
| **Journal applicatif** | Traçabilité des affectations | BAL (`BAL_LOG_CREATE`) dans `save`, consultable en `SLG1` |
| **Historique Z** | Reporting sur les affectations | Table `ZMM_SRC_LOG` alimentée dans `save`, exposée par une vue CDS |
| **Union des sources** | Fiches info achat et contrats en plus de la liste des sources | `union all` dans `ZI_SourceOfSupplyCandidate` avec une zone `SourceType` |

---

## Dépannage

| Symptôme | Cause probable | Solution |
|---|---|---|
| Activation BDEF : *behavior definition not found for projection* | Projection sans `provider contract transactional_query` ou BDEF interface inactive | Corriger la vue de consommation, activer dans l'ordre : vue → BDEF → projection → BDEF projection |
| Activation : méthode manquante dans la classe | Squelettes non régénérés | `Ctrl+1` sur l'erreur → quick fix ADT |
| Le bouton n'apparaît pas | DDLX non activée, action non exposée dans la projection, cache Gateway | Vérifier `use action`, `$metadata`, purger les caches |
| Le bouton reste actif sur une ligne déjà affectée | Feature control non transmis par le binding V2 | Le garde-fou serveur (message `002`) joue ; envisager la bascule V4 |
| Boîte de paramètres sans aide à la saisie | Entité de value help non exposée, ou limitation V2 | Exposer l'entité dans la service definition ; sinon contrôle serveur |
| Message de succès affiché mais `EBAN` inchangé | Retour BAPI en erreur non remonté | Point d'arrêt dans `save`, analyser `lt_return` ; déplacer le contrôle en `check_before_save` |
| Dump `BEHAVIOR_ILLEGAL_STATEMENT` ou transaction incohérente | `COMMIT WORK` dans le behavior pool | Le supprimer (Annexe A) |
| Message `004` systématique | Verrou résiduel | `SM12` : identifier et supprimer le verrou, vérifier `_scope` |
| L'action réussit pour un utilisateur, échoue pour un autre | Autorisation `M_BANF_WRK` en activité `02` absente | Compléter le rôle |
| Rafraîchissement de la ligne absent après l'action | `result [1] $self` non alimenté | Vérifier l'ajout dans `result` (§4.6) |

---

## Captures à réaliser

| N° | Écran | Fichier suggéré |
|---|---|---|
| 6-01 | BDEF activée | `img/06-01-bdef.png` |
| 6-02 | Classe de comportement | `img/06-02-behavior-pool.png` |
| 6-03 | `$metadata` avec l'action | `img/06-03-metadata.png` |
| 6-04 | Aperçu ADT avec le bouton | `img/06-04-preview.png` |
| 6-05 | Boîte de paramètres générée | `img/06-05-dialog.png` |
| 6-06 | Message de succès | `img/06-06-message.png` |
| 6-07 | `EBAN` après affectation | `img/06-07-eban.png` |

---

## Références

- SAP Help – *ABAP RESTful Application Programming Model* : *Unmanaged Business Objects*, *Actions*, *Feature Control*, *Saver Class*
- SAP Help – *CDS Annotations* : `@UI.lineItem` `type: #FOR_ACTION`, `dataAction`, `invocationGrouping`
- SAPUI5 Demo Kit – *Actions* (SAP Fiori elements pour OData V2) : boîtes de paramètres et messages
- `SE37` → `BAPI_PR_CHANGE` (documentation du module fonction) ; `SE11` → `BAPIMEREQITEMIMP`, `BAPIMEREQITEMX`
- `SE11` → objet de verrouillage `EMEBANE`
- Documents internes : `04_App_RAP_Determination_Source_Appro.md`, `05_Adaptation_Project_F1048_Navigation_VSCode.md`
