# Mode opératoire — Création d'un service RAP

## Suivi des demandes d'achat

**ABAP RESTful Application Programming Model**
CDS · Behavior Definition · Behavior Implementation · Service Binding
**SAP S/4HANA (Private Cloud and On-Premise) 2021 — FPS02**

| Attribut | Valeur |
|---|---|
| Référence du document | MO-RAP-PURREQ-2021FPS02-v1.0 |
| Documents liés | [MO Activation](MO%20Activation.md) · [MO Extension](MO%20Extension.md) |
| Objet métier créé | Suivi des demandes d'achat — objet métier RAP transactionnel |
| Scénario RAP | Managed, avec draft, sur table de persistance Z |
| Version produit | SAP S/4HANA 2021 FPS02 — Private Cloud / On-Premise |
| Plateforme ABAP | SAP_BASIS 756 (ABAP 7.56) |
| Outils | Eclipse + ABAP Development Tools · VS Code + SAP Fiori tools · SAP GUI |
| Auteur | … |
| Vérifié par | … |
| Approuvé par | … |
| Date de rédaction | … |
| Statut | Version de travail |

> **Convention de ce document**
> Chaque emplacement de copie d'écran est signalé par un bloc `📸 COPIE D'ÉCRAN N°XX`.
> Déposez vos images dans `images/RAP-PURREQ/` puis remplacez la ligne indiquée par le lien Markdown correspondant.

---

## Sommaire

- [1. Objet et périmètre](#1-objet-et-périmètre)
- [2. Principes du modèle RAP](#2-principes-du-modèle-rap)
- [3. Prérequis et outillage](#3-prérequis-et-outillage)
- [4. Synoptique de la démarche](#4-synoptique-de-la-démarche)
- [Partie A — Couche de persistance](#partie-a--couche-de-persistance)
- [Partie B — Couche modèle de données](#partie-b--couche-modèle-de-données)
- [Partie C — Couche comportement](#partie-c--couche-comportement)
- [Partie D — Couche service](#partie-d--couche-service)
- [Partie E — Exposition dans le Launchpad](#partie-e--exposition-dans-le-launchpad)
- [Partie F — Recette, transport et exploitation](#partie-f--recette-transport-et-exploitation)
- [Annexe A — Outils et transactions](#annexe-a--outils-et-transactions)
- [Annexe B — Diagnostic des incidents fréquents](#annexe-b--diagnostic-des-incidents-fréquents)
- [Annexe C — Glossaire RAP](#annexe-c--glossaire-rap)
- [Annexe D — Index des copies d'écran](#annexe-d--index-des-copies-décran)
- [Annexe E — Historique des versions](#annexe-e--historique-des-versions)

---

## 1. Objet et périmètre

### 1.1 Objet

Ce mode opératoire décrit la création complète d'un service OData au moyen du modèle de programmation ABAP RESTful (RAP) sur SAP S/4HANA 2021 FPS02, depuis la table de persistance jusqu'à la tuile dans le SAP Fiori Launchpad. Le code de chaque couche est fourni intégralement.

### 1.2 Cas d'usage retenu

L'objet métier créé permet de suivre des demandes d'achat : pour un poste de demande d'achat existant, l'utilisateur enregistre un statut de suivi, un commentaire et une date d'échéance. L'objet lit les données de la demande d'achat par association vers les vues standard, mais conserve ses propres données dans une table dédiée : le standard n'est jamais modifié.

Le scénario retenu est un objet métier **managed avec gestion du draft**. Il couvre la création, la modification, la suppression, les déterminations, les validations et une action métier : c'est le scénario de référence pour un développement neuf sur données propres.

### 1.3 Convention de nommage

| Couche | Objet | Type |
|---|---|---|
| Persistance | `ZMM_PR_TRACK` | Table de base de données |
| Persistance | `ZMM_PR_TRACK_D` | Table de draft |
| Modèle | `ZI_PurReqTracking` | Vue CDS d'interface |
| Modèle | `ZC_PurReqTracking` | Vue CDS de projection |
| Modèle | `ZC_PurReqTracking` | Extension de métadonnées |
| Comportement | `ZI_PurReqTracking` | Behavior definition |
| Comportement | `ZC_PurReqTracking` | Behavior definition de projection |
| Comportement | `ZBP_I_PurReqTracking` | Behavior implementation (classe) |
| Service | `ZUI_PurReqTracking` | Service definition |
| Service | `ZUI_PurReqTracking_O4` | Service binding OData V4 UI |
| Messages | `ZMM_PR_TRACK` | Classe de messages |
| Package | `Z_MM_PUR_RAP` | Package de développement |

### 1.4 Hors périmètre

- Scénario unmanaged et scénario managed avec sauvegarde non standard.
- Extensibilité de l'objet métier créé par un tiers.
- Publication d'API et scénarios d'intégration inter-systèmes.
- Développement sur SAP BTP ABAP Environment : le document vise l'ABAP on-stack.
- Mise en place d'une chaîne CI/CD et gestion des sources avec abapGit.

---

## 2. Principes du modèle RAP

### 2.1 Architecture en couches

| Ordre | Couche | Objet | Rôle |
|---|---|---|---|
| 1 | Persistance | Table de base de données | Stockage des données propres à l'objet |
| 2 | Modèle de données | Vue CDS d'interface | Modèle de données stable et réutilisable |
| 3 | Modèle de données | Vue CDS de projection | Vue orientée consommation, exposée au service |
| 4 | Modèle de données | Extension de métadonnées | Annotations d'interface utilisateur |
| 5 | Comportement | Behavior definition | Opérations, validations, déterminations, actions |
| 6 | Comportement | Behavior implementation | Code ABAP des règles de gestion |
| 7 | Service | Service definition | Périmètre exposé |
| 8 | Service | Service binding | Protocole et publication |

### 2.2 Choix du scénario

| Scénario | Quand l'utiliser | Effort |
|---|---|---|
| Managed | Développement neuf sur une table propre : le framework gère la persistance | Faible |
| Managed avec sauvegarde non standard | Données propres mais logique de sauvegarde spécifique | Moyen |
| Unmanaged | Réutilisation d'une logique existante (fonctions, BAPI, classes) | Élevé |
| Read-only | Exposition d'un modèle de lecture, sans écriture | Très faible |

> **ℹ️ Scénario retenu**
> Managed avec draft. Le draft permet à l'utilisateur de commencer une saisie, de la quitter et d'y revenir : c'est le comportement attendu par les applications SAP Fiori elements transactionnelles.

### 2.3 RAP ou Service Builder

| Critère | RAP | Service Builder (SEGW) |
|---|---|---|
| Développement neuf | Recommandé | Déconseillé |
| Protocole | OData V2 et V4 | OData V2 |
| Volume de code | Faible — le framework prend en charge le générique | Élevé |
| Draft et verrouillage | Nativement gérés | À développer |
| Extension d'un service SAP existant | Selon la nature du service | Voie classique |
| Compétence requise | CDS, EML, ABAP objet | ABAP objet |

---

## 3. Prérequis et outillage

### 3.1 Prérequis techniques

| Élément | Exigence |
|---|---|
| Plateforme | SAP S/4HANA 2021 FPS02 — ABAP Platform 2021 (`SAP_BASIS 756`) |
| Base de données | SAP HANA |
| Outil de développement | Eclipse avec les ABAP Development Tools — obligatoire pour RAP |
| Système | Ouvert aux modifications, clé de développeur disponible |
| Package | Package de développement Z créé et rattaché à une couche logicielle |
| Transport | Ordre de workbench et ordre de customizing créés |

> **⚠️ Point bloquant fréquent**
> Les objets RAP — behavior definition, service definition, service binding — ne sont pas éditables depuis SAP GUI. L'installation des ABAP Development Tools est un prérequis absolu, pas une commodité.

### 3.2 Autorisations nécessaires

| Domaine | Objets requis |
|---|---|
| Développement | `S_DEVELOP` sur les types d'objets DDLS, BDEF, SRVD, SRVB, CLAS, TABL |
| Transport | `S_TRANSPRT` |
| Administration Gateway | `/IWFND/RT_ADMIN` — publication et activation des services |
| Administration ICF | `S_ICF_ADMIN` |
| Contenu Launchpad | Accès au Launchpad Designer et à `PFCG` |

### 3.3 Préparation du poste et du package

1. Installer Eclipse et le plug-in ABAP Development Tools, puis créer le projet ABAP vers le système de développement.
2. Créer le package de développement Z destiné à l'objet métier.
3. Créer une classe de messages dédiée et y définir les messages utilisés par les validations.
4. Créer l'ordre de transport de workbench qui portera l'ensemble des objets.

> **📸 COPIE D'ÉCRAN N°01** — Eclipse ADT : projet ABAP connecté et package de développement créé
> *Remplacer cette ligne par :* `![Copie 01](images/RAP-PURREQ/capture-01.png)`

> **📸 COPIE D'ÉCRAN N°02** — Transaction `SE91` : classe de messages du projet et messages de validation
> *Remplacer cette ligne par :* `![Copie 02](images/RAP-PURREQ/capture-02.png)`

---

## 4. Synoptique de la démarche

| N° | Étape | Objet créé | Outil |
|---|---|---|---|
| A1 | Créer la table de persistance | `ZMM_PR_TRACK` | Eclipse ADT |
| A2 | Créer la table de draft | `ZMM_PR_TRACK_D` | Eclipse ADT |
| B1 | Créer la vue CDS d'interface | `ZI_PurReqTracking` | Eclipse ADT |
| B2 | Créer la vue CDS de projection | `ZC_PurReqTracking` | Eclipse ADT |
| B3 | Créer l'extension de métadonnées | `ZC_PurReqTracking` | Eclipse ADT |
| C1 | Créer la behavior definition | `ZI_PurReqTracking` | Eclipse ADT |
| C2 | Créer la behavior definition de projection | `ZC_PurReqTracking` | Eclipse ADT |
| C3 | Implémenter le comportement | `ZBP_I_PurReqTracking` | Eclipse ADT |
| D1 | Créer la service definition | `ZUI_PurReqTracking` | Eclipse ADT |
| D2 | Créer et publier le service binding | `ZUI_PurReqTracking_O4` | Eclipse ADT |
| D3 | Activer le service côté Gateway | — | SAP GUI |
| D4 | Tester le service | — | Aperçu, EML |
| E1 | Générer l'application Fiori elements | — | VS Code |
| E2 | Déployer l'application | — | VS Code |
| E3 | Publier la tuile et affecter le rôle | — | SAP GUI |
| F | Recette, transport, exploitation | — | — |

---

## Partie A — Couche de persistance

### A1 — Créer la table de persistance

#### Points d'attention

- La clé technique est un identifiant universel : c'est le choix recommandé pour un objet métier RAP avec draft.
- Les cinq champs d'administration sont requis par le scénario managed : ils portent la traçabilité et les jetons de concurrence.
- Le champ d'horodatage de dernière modification sert de jeton de concurrence global, le champ d'instance locale de jeton d'instance.

#### Mode opératoire

1. Dans Eclipse, clic droit sur le package puis créer un objet de type « Database Table ».
2. Nommer la table et saisir la définition ci-dessous.
3. Activer l'objet et vérifier l'absence d'avertissement.
4. Contrôler la création physique de la table via la transaction `SE11`.

#### Code — table `ZMM_PR_TRACK`

```abap
@EndUserText.label : 'Suivi des demandes d''achat'
@AbapCatalog.enhancementCategory : #NOT_EXTENSIBLE
@AbapCatalog.tableCategory : #TRANSPARENT
@AbapCatalog.deliveryClass : #A
@AbapCatalog.dataMaintenance : #RESTRICTED
define table zmm_pr_track {
  key client            : abap.clnt not null;
  key tracking_uuid     : sysuuid_x16 not null;
  purchase_req          : banfn;
  purchase_req_item     : bnfpo;
  tracking_status       : abap.char(2);
  comments              : abap.char(250);
  target_date           : abap.dats;
  created_by            : abp_creation_user;
  created_at            : abp_creation_tstmpl;
  last_changed_by       : abp_lastchange_user;
  last_changed_at       : abp_lastchange_tstmpl;
  local_last_changed_at : abp_locinst_lastchange_tstmpl;
}
```

> **📸 COPIE D'ÉCRAN N°03** — Eclipse ADT : création de la table de persistance
> *Remplacer cette ligne par :* `![Copie 03](images/RAP-PURREQ/capture-03.png)`

> **📸 COPIE D'ÉCRAN N°04** — Eclipse ADT : table activée, sans avertissement
> *Remplacer cette ligne par :* `![Copie 04](images/RAP-PURREQ/capture-04.png)`

---

### A2 — Créer la table de draft

La table de draft stocke les instances en cours de saisie. Elle reprend la structure de la table de base et ajoute l'include d'administration du draft. Eclipse propose une correction rapide qui la génère à partir de la behavior definition : la définition ci-dessous permet de la créer directement, ou de contrôler ce qui a été généré.

#### Code — table `ZMM_PR_TRACK_D`

```abap
@EndUserText.label : 'Table draft - suivi des demandes d''achat'
@AbapCatalog.enhancementCategory : #NOT_EXTENSIBLE
@AbapCatalog.tableCategory : #TRANSPARENT
@AbapCatalog.deliveryClass : #A
@AbapCatalog.dataMaintenance : #RESTRICTED
define table zmm_pr_track_d {
  key client            : abap.clnt not null;
  key tracking_uuid     : sysuuid_x16 not null;
  purchase_req          : banfn;
  purchase_req_item     : bnfpo;
  tracking_status       : abap.char(2);
  comments              : abap.char(250);
  target_date           : abap.dats;
  created_by            : abp_creation_user;
  created_at            : abp_creation_tstmpl;
  last_changed_by       : abp_lastchange_user;
  last_changed_at       : abp_lastchange_tstmpl;
  local_last_changed_at : abp_locinst_lastchange_tstmpl;
  include sych_bdl_draft_admin_inc;
}
```

> **📸 COPIE D'ÉCRAN N°05** — Eclipse ADT : table de draft créée et activée
> *Remplacer cette ligne par :* `![Copie 05](images/RAP-PURREQ/capture-05.png)`

> **ℹ️ Alternative**
> Si vous créez d'abord la behavior definition, Eclipse signale la table de draft manquante et propose de la générer. Cette voie garantit la cohérence des types entre les deux tables.

---

## Partie B — Couche modèle de données

### B1 — Créer la vue CDS d'interface

#### Rôle de la vue

La vue d'interface porte le modèle de données stable de l'objet métier. Elle expose les champs de la table, déclare les associations vers les vues standard des demandes d'achat et porte les annotations sémantiques qui permettent au framework de reconnaître les champs d'administration.

#### Mode opératoire

1. Créer un objet de type « Data Definition » dans le package.
2. Choisir le modèle de vue de type entité racine.
3. Saisir la définition ci-dessous, puis activer.
4. Contrôler le résultat par un aperçu de données.
5. Vérifier dans Eclipse les noms de champs exacts des vues standard référencées avant activation.

#### Code — vue `ZI_PurReqTracking`

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
{
  key tracking_uuid              as TrackingUUID,
      purchase_req               as PurchaseRequisition,
      purchase_req_item          as PurchaseRequisitionItem,
      tracking_status            as TrackingStatus,
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
      _PurReqItem
}
```

> **📸 COPIE D'ÉCRAN N°06** — Eclipse ADT : vue CDS d'interface, code source
> *Remplacer cette ligne par :* `![Copie 06](images/RAP-PURREQ/capture-06.png)`

> **📸 COPIE D'ÉCRAN N°07** — Eclipse ADT : aperçu de données de la vue d'interface
> *Remplacer cette ligne par :* `![Copie 07](images/RAP-PURREQ/capture-07.png)`

> **⚠️ À vérifier**
> Les noms des vues standard et de leurs champs peuvent différer selon la version. Ouvrir la définition de la vue standard dans Eclipse pour confirmer les noms avant d'activer, plutôt que de recopier l'exemple tel quel.

---

### B2 — Créer la vue CDS de projection

La vue de projection est la couche de consommation : c'est elle qui est exposée par le service. La clause de contrat de fournisseur indique au framework qu'il s'agit d'une projection transactionnelle. L'autorisation d'extension de métadonnées permet de déporter les annotations d'interface dans un objet séparé.

#### Code — vue `ZC_PurReqTracking`

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
      TrackingStatus,
      Comments,
      TargetDate,
      Material,
      PurReqItemText,
      LastChangedAt,
      LocalLastChangedAt
}
```

> **📸 COPIE D'ÉCRAN N°08** — Eclipse ADT : vue CDS de projection, code source
> *Remplacer cette ligne par :* `![Copie 08](images/RAP-PURREQ/capture-08.png)`

---

### B3 — Créer l'extension de métadonnées

Les annotations d'interface utilisateur sont isolées dans une extension de métadonnées : le modèle de données reste lisible et les évolutions d'affichage n'impactent pas la vue de projection. Ces annotations pilotent directement le rendu de l'application SAP Fiori elements générée en partie E.

#### Mode opératoire

1. Créer un objet de type « Metadata Extension » portant le nom de la vue de projection.
2. Déclarer la couche de métadonnées.
3. Définir l'en-tête de l'objet, les facettes, puis les annotations champ par champ.
4. Activer, puis contrôler le rendu par l'aperçu du service binding une fois la partie D réalisée.

#### Code — extension de métadonnées `ZC_PurReqTracking`

```abap
@Metadata.layer: #CORE

@UI.headerInfo: {
  typeName: 'Suivi de DA',
  typeNamePlural: 'Suivis de DA',
  title: { type: #STANDARD, value: 'PurchaseRequisition' },
  description: { value: 'PurReqItemText' }
}

annotate entity ZC_PurReqTracking with
{
  @UI.facet: [ { id: 'General', purpose: #STANDARD,
                 type: #IDENTIFICATION_REFERENCE,
                 label: 'Informations générales', position: 10 } ]

  @UI.hidden: true
  TrackingUUID;

  @UI: { lineItem: [ { position: 10, label: 'Demande d''achat' } ],
         identification: [ { position: 10 } ],
         selectionField: [ { position: 10 } ] }
  PurchaseRequisition;

  @UI: { lineItem: [ { position: 20, label: 'Poste' } ],
         identification: [ { position: 20 } ] }
  PurchaseRequisitionItem;

  @UI: { lineItem: [ { position: 30, label: 'Statut' } ],
         identification: [ { position: 30 } ],
         selectionField: [ { position: 20 } ] }
  TrackingStatus;

  @UI: { lineItem: [ { position: 40, label: 'Échéance' } ],
         identification: [ { position: 40 } ] }
  TargetDate;

  @UI: { identification: [ { position: 50 } ] }
  Comments;

  @UI: { lineItem: [ { position: 50, label: 'Article' } ] }
  @UI.identification: [ { position: 60 } ]
  Material;
}
```

> **📸 COPIE D'ÉCRAN N°09** — Eclipse ADT : extension de métadonnées, code source
> *Remplacer cette ligne par :* `![Copie 09](images/RAP-PURREQ/capture-09.png)`

| Annotation | Effet dans l'application |
|---|---|
| `headerInfo` | Titre et description de l'objet dans la page de détail |
| `facet` | Structuration en sections de la page de détail |
| `lineItem` | Colonnes de la liste |
| `identification` | Champs de la section de détail |
| `selectionField` | Filtres de la barre de recherche |
| `hidden` | Masquage d'un champ technique |

---

## Partie C — Couche comportement

### C1 — Créer la behavior definition

#### Rôle

La behavior definition déclare ce que l'objet métier sait faire : opérations autorisées, gestion du verrouillage et de la concurrence, déterminations, validations, actions, et correspondance entre les champs de la vue et les colonnes de la table.

#### Mode opératoire

1. Dans Eclipse, clic droit sur la vue d'interface puis créer la behavior definition.
2. Choisir le type d'implémentation **managed**.
3. Saisir la définition ci-dessous.
4. Utiliser les corrections rapides proposées par Eclipse pour générer la classe d'implémentation et la table de draft manquantes.
5. Activer.

#### Code — behavior definition `ZI_PurReqTracking`

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
  field ( mandatory )                     PurchaseRequisition, PurchaseRequisitionItem;
  field ( readonly )                      Material, PurReqItemText,
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

  draft action Activate optimized;
  draft action Discard;
  draft action Edit;
  draft action Resume;
  draft determine action Prepare
  {
    validation validateRequisition;
    validation validateTargetDate;
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

> **📸 COPIE D'ÉCRAN N°10** — Eclipse ADT : behavior definition, code source
> *Remplacer cette ligne par :* `![Copie 10](images/RAP-PURREQ/capture-10.png)`

> **📸 COPIE D'ÉCRAN N°11** — Eclipse ADT : correction rapide générant la classe d'implémentation
> *Remplacer cette ligne par :* `![Copie 11](images/RAP-PURREQ/capture-11.png)`

#### Lecture des clauses principales

| Clause | Signification |
|---|---|
| `managed implementation in class` | Le framework gère la persistance ; la classe porte les règles de gestion |
| `with draft` / `draft table` | Active la gestion du draft et désigne sa table |
| `lock master total etag` | Déclare le verrouillage et le jeton de concurrence global |
| `etag master` | Jeton de concurrence au niveau de l'instance |
| `authorization master ( global )` | Contrôle d'autorisation global, implémenté dans la classe |
| `field ( numbering : managed )` | Génération automatique de la clé technique |
| `determination … on modify` | Dérivation automatique déclenchée à la modification |
| `validation … on save` | Contrôle déclenché à l'enregistrement |
| `action ( features : instance )` | Action métier dont la disponibilité dépend de l'instance |
| `mapping for` | Correspondance entre champs de la vue et colonnes de la table |

> **ℹ️ Syntaxe stricte**
> Si la syntaxe numérotée du mode strict n'est pas reconnue par votre niveau de support, la remplacer par la forme simple sans paramètre. Le mode strict est fortement recommandé : il fait remonter dès l'activation des erreurs qui, sinon, n'apparaîtraient qu'à l'exécution.

---

### C2 — Créer la behavior definition de projection

La projection expose au service le sous-ensemble du comportement rendu disponible aux consommateurs. Elle ne redéfinit rien : elle référence ce qui existe déjà dans la behavior definition d'interface.

#### Code — behavior definition `ZC_PurReqTracking`

```abap
projection;
strict ( 1 );
use draft;

define behavior for ZC_PurReqTracking alias PurReqTracking
{
  use create;
  use update;
  use delete;

  use action markAsCompleted;

  use action Activate;
  use action Discard;
  use action Edit;
  use action Resume;
  use action Prepare;
}
```

> **📸 COPIE D'ÉCRAN N°12** — Eclipse ADT : behavior definition de projection, code source
> *Remplacer cette ligne par :* `![Copie 12](images/RAP-PURREQ/capture-12.png)`

---

### C3 — Implémenter le comportement

#### Structure de la classe

La classe d'implémentation contient une classe locale de gestion, dont chaque méthode correspond à une déclaration de la behavior definition. La déclaration doit être strictement alignée sur la définition, faute de quoi l'activation échoue.

```abap
CLASS lhc_PurReqTracking DEFINITION INHERITING FROM cl_abap_behavior_handler.

  PRIVATE SECTION.

    METHODS get_global_authorizations FOR GLOBAL AUTHORIZATION
      IMPORTING REQUEST requested_authorizations FOR PurReqTracking
      RESULT result.

    METHODS get_instance_features FOR INSTANCE FEATURES
      IMPORTING keys REQUEST requested_features FOR PurReqTracking
      RESULT result.

    METHODS setInitialStatus FOR DETERMINE ON MODIFY
      IMPORTING keys FOR PurReqTracking~setInitialStatus.

    METHODS validateRequisition FOR VALIDATE ON SAVE
      IMPORTING keys FOR PurReqTracking~validateRequisition.

    METHODS validateTargetDate FOR VALIDATE ON SAVE
      IMPORTING keys FOR PurReqTracking~validateTargetDate.

    METHODS markAsCompleted FOR MODIFY
      IMPORTING keys FOR ACTION PurReqTracking~markAsCompleted
      RESULT result.

ENDCLASS.
```

> **📸 COPIE D'ÉCRAN N°13** — Eclipse ADT : classe d'implémentation, section de déclaration
> *Remplacer cette ligne par :* `![Copie 13](images/RAP-PURREQ/capture-13.png)`

#### C3.1 Détermination — statut initial

La détermination affecte un statut par défaut à la création, sans intervention de l'utilisateur. Elle lit d'abord les instances concernées, écarte celles déjà valorisées, puis écrit la valeur en mode local.

```abap
METHOD setInitialStatus.

  READ ENTITIES OF ZI_PurReqTracking IN LOCAL MODE
    ENTITY PurReqTracking
      FIELDS ( TrackingStatus )
      WITH CORRESPONDING #( keys )
    RESULT DATA(lt_tracking).

  DELETE lt_tracking WHERE TrackingStatus IS NOT INITIAL.
  CHECK lt_tracking IS NOT INITIAL.

  MODIFY ENTITIES OF ZI_PurReqTracking IN LOCAL MODE
    ENTITY PurReqTracking
      UPDATE FIELDS ( TrackingStatus )
      WITH VALUE #( FOR ls IN lt_tracking
                    ( %tky          = ls-%tky
                      TrackingStatus = 'OP' ) )
    REPORTED DATA(lt_reported).

ENDMETHOD.
```

> **📸 COPIE D'ÉCRAN N°14** — Eclipse ADT : code de la détermination
> *Remplacer cette ligne par :* `![Copie 14](images/RAP-PURREQ/capture-14.png)`

#### C3.2 Validation — existence de la demande d'achat

La validation vérifie que le poste de demande d'achat référencé existe réellement. En cas d'échec, l'instance est ajoutée à la structure des échecs et un message est ajouté à la structure des messages, avec désignation du champ fautif pour que l'interface le mette en évidence.

```abap
METHOD validateRequisition.

  READ ENTITIES OF ZI_PurReqTracking IN LOCAL MODE
    ENTITY PurReqTracking
      FIELDS ( PurchaseRequisition PurchaseRequisitionItem )
      WITH CORRESPONDING #( keys )
    RESULT DATA(lt_tracking).

  LOOP AT lt_tracking INTO DATA(ls_tracking).

    SELECT SINGLE @abap_true
      FROM i_purchaserequisitionitem
      WHERE purchaserequisition     = @ls_tracking-PurchaseRequisition
        AND purchaserequisitionitem = @ls_tracking-PurchaseRequisitionItem
      INTO @DATA(lv_exists).

    IF lv_exists = abap_false.

      APPEND VALUE #( %tky = ls_tracking-%tky ) TO failed-purreqtracking.

      APPEND VALUE #(
        %tky                          = ls_tracking-%tky
        %element-PurchaseRequisition   = if_abap_behv=>mk-on
        %msg = new_message(
                 id       = 'ZMM_PR_TRACK'
                 number   = '001'
                 severity = if_abap_behv_message=>severity-error
                 v1       = ls_tracking-PurchaseRequisition
                 v2       = ls_tracking-PurchaseRequisitionItem ) )
        TO reported-purreqtracking.

    ENDIF.

  ENDLOOP.

ENDMETHOD.
```

> **📸 COPIE D'ÉCRAN N°15** — Eclipse ADT : code de la validation d'existence
> *Remplacer cette ligne par :* `![Copie 15](images/RAP-PURREQ/capture-15.png)`

#### C3.3 Validation — cohérence de la date d'échéance

```abap
METHOD validateTargetDate.

  READ ENTITIES OF ZI_PurReqTracking IN LOCAL MODE
    ENTITY PurReqTracking
      FIELDS ( TargetDate )
      WITH CORRESPONDING #( keys )
    RESULT DATA(lt_tracking).

  LOOP AT lt_tracking INTO DATA(ls_tracking).

    IF ls_tracking-TargetDate IS NOT INITIAL
       AND ls_tracking-TargetDate < cl_abap_context_info=>get_system_date( ).

      APPEND VALUE #( %tky = ls_tracking-%tky ) TO failed-purreqtracking.

      APPEND VALUE #(
        %tky                 = ls_tracking-%tky
        %element-TargetDate  = if_abap_behv=>mk-on
        %msg = new_message(
                 id       = 'ZMM_PR_TRACK'
                 number   = '002'
                 severity = if_abap_behv_message=>severity-error ) )
        TO reported-purreqtracking.

    ENDIF.

  ENDLOOP.

ENDMETHOD.
```

> **📸 COPIE D'ÉCRAN N°16** — Eclipse ADT : code de la validation de date
> *Remplacer cette ligne par :* `![Copie 16](images/RAP-PURREQ/capture-16.png)`

#### C3.4 Action métier — clôturer le suivi

L'action met à jour le statut puis relit les instances pour renvoyer l'état à jour au consommateur. Ce renvoi permet à l'application de rafraîchir l'écran sans appel supplémentaire.

```abap
METHOD markAsCompleted.

  MODIFY ENTITIES OF ZI_PurReqTracking IN LOCAL MODE
    ENTITY PurReqTracking
      UPDATE FIELDS ( TrackingStatus )
      WITH VALUE #( FOR key IN keys
                    ( %tky           = key-%tky
                      TrackingStatus = 'CL' ) )
    FAILED   failed
    REPORTED reported.

  READ ENTITIES OF ZI_PurReqTracking IN LOCAL MODE
    ENTITY PurReqTracking
      ALL FIELDS WITH CORRESPONDING #( keys )
    RESULT DATA(lt_tracking).

  result = VALUE #( FOR ls IN lt_tracking
                    ( %tky   = ls-%tky
                      %param = ls ) ).

ENDMETHOD.
```

> **📸 COPIE D'ÉCRAN N°17** — Eclipse ADT : code de l'action métier
> *Remplacer cette ligne par :* `![Copie 17](images/RAP-PURREQ/capture-17.png)`

#### C3.5 Disponibilité dynamique de l'action

L'action ne doit pas être proposée sur une instance déjà clôturée. La méthode de caractéristiques d'instance renvoie l'état d'activation pour chaque clé demandée.

```abap
METHOD get_instance_features.

  READ ENTITIES OF ZI_PurReqTracking IN LOCAL MODE
    ENTITY PurReqTracking
      FIELDS ( TrackingStatus )
      WITH CORRESPONDING #( keys )
    RESULT DATA(lt_tracking)
    FAILED failed.

  result = VALUE #( FOR ls IN lt_tracking
                    ( %tky = ls-%tky
                      %action-markAsCompleted =
                        COND #( WHEN ls-TrackingStatus = 'CL'
                                THEN if_abap_behv=>fc-o-disabled
                                ELSE if_abap_behv=>fc-o-enabled ) ) ).

ENDMETHOD.
```

> **📸 COPIE D'ÉCRAN N°18** — Eclipse ADT : code des caractéristiques d'instance
> *Remplacer cette ligne par :* `![Copie 18](images/RAP-PURREQ/capture-18.png)`

#### C3.6 Contrôle d'autorisation global

Le contrôle global est évalué une fois par opération, indépendamment des instances. L'objet d'autorisation utilisé est à choisir selon la politique de sécurité du projet.

```abap
METHOD get_global_authorizations.

  IF requested_authorizations-%create = if_abap_behv=>mk-on.
    AUTHORITY-CHECK OBJECT 'M_BANF_EKG'
      ID 'ACTVT' FIELD '01'
      ID 'EKGRP' DUMMY.
    result-%create = COND #( WHEN sy-subrc = 0
                             THEN if_abap_behv=>auth-allowed
                             ELSE if_abap_behv=>auth-unauthorized ).
  ENDIF.

  IF requested_authorizations-%update = if_abap_behv=>mk-on.
    AUTHORITY-CHECK OBJECT 'M_BANF_EKG'
      ID 'ACTVT' FIELD '02'
      ID 'EKGRP' DUMMY.
    result-%update = COND #( WHEN sy-subrc = 0
                             THEN if_abap_behv=>auth-allowed
                             ELSE if_abap_behv=>auth-unauthorized ).
  ENDIF.

ENDMETHOD.
```

> **📸 COPIE D'ÉCRAN N°19** — Eclipse ADT : code du contrôle d'autorisation
> *Remplacer cette ligne par :* `![Copie 19](images/RAP-PURREQ/capture-19.png)`

> **ℹ️ Choix de l'objet d'autorisation**
> L'exemple s'appuie sur un objet du domaine achat. Selon la gouvernance du projet, il peut être préférable de créer un objet d'autorisation dédié à l'objet métier plutôt que de réutiliser un objet standard dont la sémantique diffère.

---

## Partie D — Couche service

### D1 — Créer la service definition

La service definition déclare le périmètre exposé. Seules les entités listées sont accessibles depuis l'extérieur : c'est le point de contrôle du périmètre fonctionnel du service.

```abap
@EndUserText.label: 'Service de suivi des demandes d''achat'
define service ZUI_PurReqTracking {
  expose ZC_PurReqTracking as PurReqTracking;
}
```

> **📸 COPIE D'ÉCRAN N°20** — Eclipse ADT : service definition, code source
> *Remplacer cette ligne par :* `![Copie 20](images/RAP-PURREQ/capture-20.png)`

---

### D2 — Créer et publier le service binding

#### Mode opératoire

1. Clic droit sur la service definition puis créer un objet de type « Service Binding ».
2. Choisir le type de liaison : OData V4 pour une interface utilisateur.
3. Nommer le binding, valider puis activer.
4. Dans l'éditeur du binding, choisir **Publish** pour publier le service local.
5. Relever l'URL du service affichée dans l'éditeur : elle sert à tous les tests ultérieurs.
6. Vérifier que l'entité exposée apparaît bien dans l'arborescence du binding.

| Type de liaison | Usage |
|---|---|
| OData V4 — UI | Applications SAP Fiori elements récentes ; scénario retenu ici |
| OData V2 — UI | Compatibilité avec des outils ou applications existants en V2 |
| OData V4 — Web API | Intégration technique, sans annotations d'interface |

> **📸 COPIE D'ÉCRAN N°21** — Eclipse ADT : création du service binding, choix du protocole
> *Remplacer cette ligne par :* `![Copie 21](images/RAP-PURREQ/capture-21.png)`

> **📸 COPIE D'ÉCRAN N°22** — Eclipse ADT : service binding publié, entité exposée et URL du service
> *Remplacer cette ligne par :* `![Copie 22](images/RAP-PURREQ/capture-22.png)`

---

### D3 — Activer le service côté Gateway

La publication depuis Eclipse enregistre le service localement. Un contrôle côté serveur reste nécessaire, en particulier sur un système où les services d'infrastructure OData version 4 n'ont jamais été utilisés.

1. Vérifier l'activation du nœud ICF racine des services OData version 4 via la transaction `SICF`.
2. Contrôler l'enregistrement du groupe de services dans la transaction d'administration des services version 4.
3. Vérifier l'absence d'erreur dans le journal des erreurs Gateway.
4. En cas de déploiement avec serveur front-end séparé, publier également le service sur le hub.

```
/sap/opu/odata4/   →   nœud racine des services OData V4 dans SICF
```

> **📸 COPIE D'ÉCRAN N°23** — Transaction `SICF` : nœud racine des services OData V4 actif
> *Remplacer cette ligne par :* `![Copie 23](images/RAP-PURREQ/capture-23.png)`

> **📸 COPIE D'ÉCRAN N°24** — Transaction d'administration des services V4 : groupe de services enregistré
> *Remplacer cette ligne par :* `![Copie 24](images/RAP-PURREQ/capture-24.png)`

---

### D4 — Tester le service

#### D4.1 Aperçu depuis Eclipse

1. Dans l'éditeur du service binding, choisir l'aperçu de l'entité exposée.
2. Vérifier l'ouverture de l'application de prévisualisation dans le navigateur.
3. Créer une instance de test, contrôler le fonctionnement du draft, des validations et de l'action.
4. Vérifier la persistance en base après activation du draft.

> **📸 COPIE D'ÉCRAN N°25** — Navigateur : aperçu du service, liste vide et bouton de création
> *Remplacer cette ligne par :* `![Copie 25](images/RAP-PURREQ/capture-25.png)`

> **📸 COPIE D'ÉCRAN N°26** — Navigateur : aperçu du service, saisie d'une instance en mode draft
> *Remplacer cette ligne par :* `![Copie 26](images/RAP-PURREQ/capture-26.png)`

> **📸 COPIE D'ÉCRAN N°27** — Navigateur : aperçu du service, message d'erreur émis par une validation
> *Remplacer cette ligne par :* `![Copie 27](images/RAP-PURREQ/capture-27.png)`

> **📸 COPIE D'ÉCRAN N°28** — Navigateur : aperçu du service, instance créée et action de clôture disponible
> *Remplacer cette ligne par :* `![Copie 28](images/RAP-PURREQ/capture-28.png)`

> **📸 COPIE D'ÉCRAN N°29** — Transaction `SE16N` : table `ZMM_PR_TRACK`, instance persistée
> *Remplacer cette ligne par :* `![Copie 29](images/RAP-PURREQ/capture-29.png)`

> **⚠️ Nature de l'aperçu**
> La prévisualisation du service binding est un outil de développement. Elle ne constitue pas une application publiable et ne doit jamais être communiquée aux utilisateurs finaux comme point d'entrée.

#### D4.2 Test du modèle de métadonnées

1. Ouvrir l'URL du service suivie du chemin des métadonnées dans un navigateur authentifié.
2. Vérifier la présence de l'entité, de ses propriétés et de l'action déclarée.
3. Contrôler la remontée des annotations d'interface utilisateur.

> **📸 COPIE D'ÉCRAN N°30** — Navigateur : document de métadonnées du service RAP
> *Remplacer cette ligne par :* `![Copie 30](images/RAP-PURREQ/capture-30.png)`

#### D4.3 Test par EML

Le langage de manipulation d'entités permet de tester l'objet métier sans passer par le protocole OData, ce qui isole les problèmes de logique des problèmes d'exposition.

```abap
REPORT zmm_pr_track_test.

DATA(lv_uuid) = cl_system_uuid=>create_uuid_x16_static( ).

MODIFY ENTITIES OF ZI_PurReqTracking
  ENTITY PurReqTracking
    CREATE FIELDS ( PurchaseRequisition PurchaseRequisitionItem TargetDate )
    WITH VALUE #( ( %cid                    = 'C1'
                    PurchaseRequisition     = '0010000123'
                    PurchaseRequisitionItem = '00010'
                    TargetDate              = '20261231' ) )
  MAPPED   DATA(ls_mapped)
  FAILED   DATA(ls_failed)
  REPORTED DATA(ls_reported).

IF ls_failed IS INITIAL.
  COMMIT ENTITIES RESPONSE OF ZI_PurReqTracking
    FAILED   DATA(ls_commit_failed)
    REPORTED DATA(ls_commit_reported).
  WRITE / 'Instance créée.'.
ELSE.
  LOOP AT ls_reported-purreqtracking INTO DATA(ls_msg).
    WRITE / ls_msg-%msg->if_message~get_text( ).
  ENDLOOP.
ENDIF.
```

> **📸 COPIE D'ÉCRAN N°31** — Eclipse ADT : exécution du programme de test EML et résultat
> *Remplacer cette ligne par :* `![Copie 31](images/RAP-PURREQ/capture-31.png)`

---

## Partie E — Exposition dans le Launchpad

Le service publié n'est pas encore accessible aux utilisateurs. Il faut générer une application SAP Fiori elements qui le consomme, la déployer, puis la publier comme n'importe quelle application, selon la démarche décrite dans le mode opératoire d'activation.

### E1 — Générer l'application SAP Fiori elements

1. Ouvrir Visual Studio Code et lancer le générateur d'application SAP Fiori.
2. Choisir le modèle de liste avec page de détail, en version OData V4.
3. Sélectionner comme source de données le système ABAP puis le service publié en partie D.
4. Sélectionner l'entité principale exposée par le service.
5. Renseigner le nom du module, le titre et l'espace de noms de l'application.
6. Ajouter la configuration Launchpad : objet sémantique, action et titre de la tuile.
7. Terminer la génération et lancer l'application en local pour contrôle.

> **📸 COPIE D'ÉCRAN N°32** — VS Code : générateur d'application SAP Fiori, choix du modèle
> *Remplacer cette ligne par :* `![Copie 32](images/RAP-PURREQ/capture-32.png)`

> **📸 COPIE D'ÉCRAN N°33** — VS Code : sélection du service RAP comme source de données
> *Remplacer cette ligne par :* `![Copie 33](images/RAP-PURREQ/capture-33.png)`

> **📸 COPIE D'ÉCRAN N°34** — VS Code : configuration Launchpad, objet sémantique et action
> *Remplacer cette ligne par :* `![Copie 34](images/RAP-PURREQ/capture-34.png)`

> **📸 COPIE D'ÉCRAN N°35** — Navigateur : application générée, exécutée en local
> *Remplacer cette ligne par :* `![Copie 35](images/RAP-PURREQ/capture-35.png)`

> **ℹ️ Rôle des annotations**
> L'application générée ne contient quasiment aucune logique : colonnes, filtres et sections de détail proviennent des annotations écrites en partie B. Toute correction d'affichage se fait dans l'extension de métadonnées, pas dans le code de l'application.

### E2 — Déployer l'application

1. Lancer la commande de déploiement de l'application.
2. Renseigner le nom du repository SAPUI5 ABAP, le package et l'ordre de transport.
3. Contrôler la fin du déploiement dans le journal.
4. Vérifier la création de l'application BSP dans le système.
5. Activer le nœud ICF de la nouvelle application via la transaction `SICF`.

> **📸 COPIE D'ÉCRAN N°36** — VS Code : journal de déploiement de l'application
> *Remplacer cette ligne par :* `![Copie 36](images/RAP-PURREQ/capture-36.png)`

> **📸 COPIE D'ÉCRAN N°37** — Transaction `SICF` : nœud ICF de l'application déployée, activé
> *Remplacer cette ligne par :* `![Copie 37](images/RAP-PURREQ/capture-37.png)`

### E3 — Publier la tuile et affecter le rôle

1. Lancer la transaction `/UI2/FLPD_CUST` et créer un catalogue Z dédié.
2. Créer la tuile et renseigner titre, sous-titre et icône.
3. Créer le target mapping en reprenant l'objet sémantique et l'action déclarés en E1, et en pointant vers l'application déployée.
4. Créer ou compléter le rôle PFCG, y ajouter le catalogue puis le groupe ou l'espace.
5. Compléter le rôle avec l'autorisation d'exécution du service.
6. Affecter le rôle à l'utilisateur de test et lancer la comparaison utilisateur.
7. Invalider les caches du Launchpad.
8. Ouvrir le Launchpad et vérifier l'affichage et le fonctionnement de la tuile.

> **📸 COPIE D'ÉCRAN N°38** — Launchpad Designer : tuile et target mapping de l'application
> *Remplacer cette ligne par :* `![Copie 38](images/RAP-PURREQ/capture-38.png)`

> **📸 COPIE D'ÉCRAN N°39** — Transaction `PFCG` : catalogue ajouté au rôle et profil généré
> *Remplacer cette ligne par :* `![Copie 39](images/RAP-PURREQ/capture-39.png)`

> **📸 COPIE D'ÉCRAN N°40** — Launchpad : tuile visible et application ouverte sur des données réelles
> *Remplacer cette ligne par :* `![Copie 40](images/RAP-PURREQ/capture-40.png)`

---

## Partie F — Recette, transport et exploitation

### F1 — Fiche de recette

| N° | Point de contrôle | Résultat attendu | OK / KO |
|---|---|---|---|
| 1 | Tables de base et de draft activées | Actives | ☐ |
| 2 | Vue d'interface activée, aperçu de données correct | Conforme | ☐ |
| 3 | Vue de projection activée | Active | ☐ |
| 4 | Extension de métadonnées activée | Active | ☐ |
| 5 | Behavior definition activée sans avertissement | Conforme | ☐ |
| 6 | Classe d'implémentation activée | Active | ☐ |
| 7 | Service definition et binding publiés | Publiés | ☐ |
| 8 | Métadonnées du service accessibles | HTTP 200 | ☐ |
| 9 | Création d'une instance en mode draft | Fonctionnelle | ☐ |
| 10 | Détermination du statut initial | Conforme | ☐ |
| 11 | Validation d'existence de la demande d'achat | Message correct | ☐ |
| 12 | Validation de la date d'échéance | Message correct | ☐ |
| 13 | Action de clôture et désactivation conditionnelle | Conforme | ☐ |
| 14 | Contrôle d'autorisation global | Conforme | ☐ |
| 15 | Persistance en base après activation | Conforme | ☐ |
| 16 | Modification concurrente correctement rejetée | Conforme | ☐ |
| 17 | Application déployée et tuile visible | Visible | ☐ |
| 18 | Comportement identique pour un second utilisateur | Conforme | ☐ |

> **📸 COPIE D'ÉCRAN N°41** — Synthèse de recette : application RAP en fonctionnement dans le Launchpad
> *Remplacer cette ligne par :* `![Copie 41](images/RAP-PURREQ/capture-41.png)`

### F2 — Transport

| Objet | Mode de propagation |
|---|---|
| Tables de base et de draft | Ordre de workbench |
| Vues CDS et extension de métadonnées | Ordre de workbench |
| Behavior definition et classe d'implémentation | Ordre de workbench |
| Service definition et service binding | Ordre de workbench ; publication à contrôler dans le système cible |
| Classe de messages | Ordre de workbench |
| Application SAPUI5 déployée | Ordre de workbench ; activation du nœud ICF manuelle |
| Catalogue, tuile et target mapping | Ordre de customizing |
| Rôle PFCG | Ordre de customizing |

> **⚠️ Après import**
> Activer les nœuds ICF, contrôler la publication du service binding, invalider les caches du Launchpad, puis rejouer la fiche de recette avec un utilisateur représentatif.

### F3 — Exploitation et supervision

| Sujet | Point de vigilance |
|---|---|
| Croissance de la table de draft | Prévoir la purge des drafts abandonnés selon la politique de rétention |
| Journal des erreurs | Superviser le journal des erreurs Gateway sur le service |
| Performance | Contrôler le plan d'exécution des vues CDS sur les volumes réels |
| Autorisations | Rejouer les scénarios d'autorisation après toute évolution du rôle |
| Montée de version | Retester l'activation des objets RAP et le comportement du service |

---

## Annexe A — Outils et transactions

| Outil / Transaction | Usage |
|---|---|
| Eclipse ADT | Création et activation de tous les objets RAP |
| `SE11` | Contrôle des tables générées |
| `SE16N` | Affichage du contenu des tables de base et de draft |
| `SE24` | Contrôle de la classe d'implémentation |
| `SE91` | Classe de messages |
| `SE80` | Application BSP déployée |
| `SICF` | Activation des nœuds ICF, dont la racine des services OData V4 |
| `/IWFND/MAINT_SERVICE` | Services OData version 2 |
| `/IWFND/GW_CLIENT` | Test des services OData version 2 |
| `/IWFND/ERROR_LOG` | Journal des erreurs Gateway |
| `/UI2/FLPD_CUST` | Launchpad Designer |
| `/UI2/INVALIDATE_CACHES` | Invalidation des caches du Launchpad |
| `PFCG` | Rôles et autorisations |
| `ST22` | Analyse des dumps ABAP |
| VS Code + SAP Fiori tools | Génération et déploiement de l'application Fiori elements |

---

## Annexe B — Diagnostic des incidents fréquents

| Symptôme | Cause probable | Action corrective |
|---|---|---|
| La behavior definition ne s'active pas | Déclaration de méthode non alignée sur la définition | Utiliser la correction rapide d'Eclipse pour régénérer les déclarations |
| Erreur sur la table de draft | Table absente ou structure divergente | Régénérer la table de draft depuis la behavior definition |
| Le champ clé reste vide à la création | Numérotation managed non déclarée | Compléter la clause de numérotation dans la behavior definition |
| La modification n'est pas persistée | Écriture réalisée hors du mode local | Reprendre l'instruction de modification en mode local |
| Les messages ne remontent pas dans l'interface | Instance absente de la structure des échecs | Alimenter les deux structures, échecs et messages |
| L'action reste toujours active | Caractéristiques d'instance non implémentées | Implémenter la méthode de caractéristiques d'instance |
| Le service binding ne se publie pas | Service definition inactive ou entité non exposée | Réactiver la chaîne complète puis republier |
| Erreur HTTP 404 sur l'URL du service | Nœud ICF racine inactif | Activer le nœud racine des services OData V4 |
| L'aperçu affiche une liste sans colonne | Annotations d'interface absentes | Compléter l'extension de métadonnées puis réactiver |
| La tuile n'apparaît pas | Catalogue non affecté au rôle ou cache non invalidé | Compléter le rôle, régénérer le profil, vider les caches |
| Conflit de modification concurrente | Jeton de concurrence mal déclaré | Contrôler les clauses de verrouillage et de jeton |

---

## Annexe C — Glossaire RAP

| Terme | Définition |
|---|---|
| Objet métier | Ensemble cohérent de données et de comportements exposé comme une unité |
| Vue d'interface | Modèle de données stable, réutilisable, non exposé directement |
| Vue de projection | Vue orientée consommation, exposée par le service |
| Behavior definition | Déclaration du comportement de l'objet métier |
| Behavior implementation | Classe portant le code des règles de gestion |
| Détermination | Dérivation automatique d'une valeur déclenchée par un événement |
| Validation | Contrôle bloquant déclenché par un événement |
| Action | Opération métier explicite, déclenchée par l'utilisateur |
| Draft | État intermédiaire d'une instance en cours de saisie, non encore persistée |
| EML | Langage de manipulation des entités, utilisé pour piloter l'objet métier en ABAP |
| Service definition | Déclaration du périmètre exposé |
| Service binding | Liaison entre le périmètre exposé et un protocole |

---

## Annexe D — Index des copies d'écran

| N° | Chapitre / Étape | Contenu attendu |
|---|---|---|
| 01 à 02 | Chapitre 3 | Projet ABAP et classe de messages |
| 03 à 04 | Étape A1 | Table de persistance |
| 05 | Étape A2 | Table de draft |
| 06 à 07 | Étape B1 | Vue CDS d'interface et aperçu de données |
| 08 | Étape B2 | Vue CDS de projection |
| 09 | Étape B3 | Extension de métadonnées |
| 10 à 11 | Étape C1 | Behavior definition et génération de la classe |
| 12 | Étape C2 | Behavior definition de projection |
| 13 à 19 | Étape C3 | Code de la classe d'implémentation |
| 20 | Étape D1 | Service definition |
| 21 à 22 | Étape D2 | Service binding et publication |
| 23 à 24 | Étape D3 | Activation côté Gateway |
| 25 à 31 | Étape D4 | Tests de l'objet métier et du service |
| 32 à 35 | Étape E1 | Génération de l'application Fiori elements |
| 36 à 37 | Étape E2 | Déploiement |
| 38 à 40 | Étape E3 | Tuile, rôle et Launchpad |
| 41 | Partie F | Synthèse de recette |

---

## Annexe E — Historique des versions

| Version | Date | Auteur | Nature des modifications |
|---|---|---|---|
| 1.0 | … | … | Création du document |
| | | | |
