# Mode opératoire — Détermination de la source d'approvisionnement en RAP

## Réalisation cible du socle commun sur ABAP RESTful Application Programming Model

| Attribut | Valeur |
|---|---|
| Référence du document | MO-SRCAPPRO-RAP-2021FPS02-v1.0 |
| Documents liés | [MO Socle source d'approvisionnement (maquettes)](Spec-SourceAppro.md) · [MO Cockpit réservations](Spec.md) |
| Objets créés | 8 tables (3 applicatives, 5 de customizing), 1 tranche de numéros, 1 objet d'autorisation, vues CDS d'interface, 2 business objects RAP, 4 projections, 4 services OData V4, 4 applications Fiori elements V4, 2 rôles |
| Périmètre | Détermination de la source par poste de DA (Alternative 1 et Alternative 2), commande d'achat / de transfert, commande petite caisse, création d'un besoin de réapprovisionnement |
| Architecture | 100 % ABAP — RAP managed avec sauvegarde non managée, OData V4, Fiori elements V4, **sans draft** |
| Version produit | SAP S/4HANA 2021 FPS02 — SAPUI5 1.96 |
| Package | `ZRESA_SRC` |
| Outils | Eclipse + ABAP Development Tools · VS Code + SAP Fiori tools · SAP GUI |
| Auteur | … |
| Vérifié par | … |
| Date de rédaction | … |
| Statut | Version de travail |

> **Convention de ce document**
> - Chaque emplacement de copie d'écran est signalé par un bloc `📸 COPIE D'ÉCRAN N°XX`. Déposer les images dans
>   `images/RESA-SRC-RAP/`.
> - Toute valeur **non confirmée** est écrite `⟨ENTRE_CHEVRONS⟩` et reprise dans
>   l'[annexe C — Paramètres à renseigner](#annexe-c--paramètres-à-renseigner). Le code qui en contient ne
>   s'active pas tant qu'elles ne sont pas remplacées : c'est volontaire.
> - Les encadrés **⚠️ À vérifier** signalent un nom de champ standard ou une capacité RAP à contrôler dans le
>   système avant activation, comme dans le MO du cockpit (§A3).

---

## Sommaire

- [1. Objet et décisions de conception](#1-objet-et-décisions-de-conception)
- [2. Prérequis](#2-prérequis)
- [3. Architecture et synoptique](#3-architecture-et-synoptique)
- [Partie A — Persistance](#partie-a--persistance)
- [Partie B — Customizing](#partie-b--customizing)
- [Partie C — Vues CDS d'interface](#partie-c--vues-cds-dinterface)
- [Partie D — Business objects RAP](#partie-d--business-objects-rap)
- [Partie E — Implémentation ABAP](#partie-e--implémentation-abap)
- [Partie F — Projections et annotations](#partie-f--projections-et-annotations)
- [Partie G — Services OData V4](#partie-g--services-odata-v4)
- [Partie H — Applications Fiori elements V4](#partie-h--applications-fiori-elements-v4)
- [Partie I — Rôles, autorisations et launchpad](#partie-i--rôles-autorisations-et-launchpad)
- [Partie J — Recette, transport et exploitation](#partie-j--recette-transport-et-exploitation)
- [Annexe A — Diagnostic des incidents fréquents](#annexe-a--diagnostic-des-incidents-fréquents)
- [Annexe B — Correspondance maquette V2 → RAP](#annexe-b--correspondance-maquette-v2--rap)
- [Annexe C — Paramètres à renseigner](#annexe-c--paramètres-à-renseigner)
- [Annexe D — Index des copies d'écran](#annexe-d--index-des-copies-décran)
- [Annexe E — Historique des versions](#annexe-e--historique-des-versions)

---

## 1. Objet et décisions de conception

### 1.1 Objet

Les maquettes du [MO Socle source d'approvisionnement](Spec-SourceAppro.md) restituent la détermination de la
source sur des vues CDS classiques publiées en OData V2, avec des actions simulées par le mock server. Ce mode
opératoire décrit la **réalisation réelle** de ce socle, **exclusivement en RAP** : la persistance du choix, la
création des documents aval et les contrôles sont portés par des business objects, et non par un programme ou
des function imports codés à la main.

Il reste valable **quel que soit l'arbitrage** entre les deux alternatives : un seul business object porte la
logique, deux projections l'exposent.

### 1.2 Décisions retenues

| Sujet | Décision | Conséquence |
|---|---|---|
| Périmètre | **Un BO de détermination, deux projections** (Alternative 1 : toutes les DA ; Alternative 2 : réservations) + un BO de création de besoin | Logique écrite une seule fois ; chaque tuile a ses propres annotations |
| Protocole | **OData V4**, Fiori elements V4 | Les quatre applications maquettes V2 sont régénérées en V4 |
| Draft | **Sans draft** | Les actions écrivent directement ; pas de tables draft ; verrou pessimiste sur la DA |
| Document aval | Fournisseur EDI → **CA `ZDI5`** ; entrepôt et centre voisin → **commande de transfert**, type **paramétré** ; petite caisse → **document `ZPC`** | Table `ZTRESA_SRCDOCTY` |
| Création CA / STO | **`BAPI_PO_CREATE1` en phase de sauvegarde RAP**, sans `COMMIT` | Sauvegarde non managée (`with unmanaged save`) |
| Création DA (Alt. 1) | **`BAPI_PR_CREATE`** en phase de sauvegarde | BO `ZR_ResaReplenRequest` |
| Plafond petite caisse | **Par division** (montant + devise) | Table `ZTRESA_PCLIMIT` |
| Numérotation `ZPC` | **Tranche de numéros SNRO** `ZRESA_PC`, numérotation tardive | `adjust_numbers` |
| Autorisations | Objets **standard achats** (`M_BANF_WRK`, `M_BEST_WRK`, `M_BEST_BSA`) ; objet **Z** `ZRESA_PC` pour la petite caisse uniquement | DCL + contrôles d'instance |
| Disponibilité fournisseur | **Table Z existante alimentée par `ZC10`** | Nom et champs : `⟨TABLE_ZC10⟩` à renseigner |
| Centres voisins | Table Z division → voisins (rang, délai) | `ZTRESA_NEIGHB` |
| Entrepôts | Table Z division → entrepôts livreurs (rang, délai) | `ZTRESA_WHSE` |
| Règle de proposition | **Aucune proposition automatique** (inchangé) : comparatif, choix manuel | Le champ de rang est restitué mais n'ordonne rien d'autorité |

### 1.3 Ce que le RAP remplace dans la maquette

| Maquette (V2) | Réalisation RAP |
|---|---|
| `@OData.publish: true` sur les vues de consommation | Service definition + service binding OData V4 |
| Function import `AssignSource` | Action d'instance `SelectSource` sur la ligne d'option de source |
| Function import `ConvertToPurchaseOrder` | Action d'instance sur le poste, création réelle en sauvegarde |
| Function import `CreatePettyCashOrder` | Action d'instance + création par association de l'entité `PettyCash` |
| Function imports `CreatePurchaseRequisition` / `CreateReplenishmentOrder` | Actions statiques du BO `ZR_ResaReplenRequest` |
| Plafond petite caisse en dur (200) | `ZTRESA_PCLIMIT` |
| Disponibilité EDI en dur (`'A'`) | `⟨TABLE_ZC10⟩` |
| Entrepôts filtrés sur `like 'WH%'` | `ZTRESA_WHSE` |
| Vue `ZI_ResaStoreNeighbour` « à créer » | `ZTRESA_NEIGHB` |
| Écart assumé : boutons de création dans l'écran de détermination | **Résolu** : une projection et une extension de métadonnées par tuile |

### 1.4 Hors périmètre

- Alimentation de `⟨TABLE_ZC10⟩` par le message `ZC10` (existant, non modifié).
- Intégration des réservations par IDoc `Z_CREA_PR` et champ de réservation dans la DA (supposés en place, cf. [MO Cockpit](Spec.md) §2.1).
- Scénario commande client (§2 du MO maquettes) : le BO pourra être appelé depuis ce scénario par EML, sans modification.
- Circuit de justificatif, imputation comptable et validation de la petite caisse (question ouverte n°3 du MO maquettes).
- Règle de priorisation automatique des sources.

---

## 2. Prérequis

### 2.1 Prérequis fonctionnels

| Élément | Exigence | Responsable |
|---|---|---|
| Champ de réservation | Présent dans la DA et exposé dans `I_PurchaseRequisitionItem` (`ZZReservation`) | Déjà traité par le MO cockpit |
| Types de document | `ZDI5` et types de commande de transfert existants et paramétrés (MM) | Fonctionnel MM |
| Type de DA de réapprovisionnement manuel | Existant, par division | Fonctionnel MM — `⟨TYPE_DA_REAPPRO⟩` |
| Organisation d'achat / groupe d'acheteurs par division et type de source | Connus | Fonctionnel MM |
| Transfert entre divisions | Paramétrage STO en place (division livreuse, type de livraison) | Fonctionnel MM / SD |
| Table `ZC10` | Nom, champs fournisseur / article / (division) / disponibilité, signification des codes | Équipe interface — `⟨TABLE_ZC10⟩` |
| Plafonds petite caisse | Montant et devise par division | Métier |
| Centres voisins et entrepôts | Liste par division, rang, délai | Métier / supply chain |

### 2.2 Prérequis techniques

| Élément | Exigence |
|---|---|
| Plateforme | SAP S/4HANA 2021 FPS02 (ABAP 7.56) |
| ADT | Version à jour, compatible avec le niveau 7.56 (éditeurs *Behavior Definition*, *Service Binding*) |
| VS Code | SAP Fiori tools, générateur *List Report Page* OData V4 |
| Autorisations développeur | `S_DEVELOP`, `S_TRANSPRT`, `S_NUMBER` (SNRO), SU21, PFCG, `/IWFND/V4_ADMIN` |
| Package / transport | Package `ZRESA_SRC`, un ordre workbench, un ordre customizing |

### 2.3 Relevés à faire avant de coder

Ne rien deviner : chaque ligne ci-dessous conditionne le code des parties C à E.

| N° | Relevé | Où | Utilisé en |
|---|---|---|---|
| R1 | Noms réels dans `I_PurchaseRequisitionItem` : `ZZReservation`, `PurchaseOrder`, `PurchaseOrderItem`, `DeliveryDate`, `PurchaseRequisitionPrice`, unité de prix, indicateur de suppression, indicateur de clôture | ADT, *Open Data Preview* + *Element Info* | C4, D1 |
| R2 | Vue de données organisationnelles de la fiche info (nom supposé `I_PurgInfoRecdOrgPlntData`) : champs division, organisation d'achat, catégorie, prix net, délai prévisionnel | ADT | C1 |
| R3 | Vue de stock utilisée pour entrepôt et centre voisin (ex. `I_MaterialStock_2`) et champ de stock libre ; décision stock libre / ATP (question ouverte n°2 du MO maquettes) | ADT + métier | C2 |
| R4 | Objet de verrouillage de la table `EBAN` et noms de ses paramètres | SE11 → *Objets de verrouillage*, recherche sur la table `EBAN` | E1 |
| R5 | `⟨TABLE_ZC10⟩` : nom, champs, codes de disponibilité | SE11 | C1 |
| R6 | Capacités RAP du système : `strict`, `provider contract`, `late numbering` en managed, `union` dans une *view entity*, fonction `get_numeric_value` | Créer un objet de test dans `$TMP` | C, D, F |
| R7 | Format attendu par `BAPI_PR_CREATE` et `BAPI_PO_CREATE1` pour la date de livraison et l'unité (interne / externe / ISO) | SE37, exécution test sans commit | E5 |

> **⚠️ Capacités RAP à confirmer (R6)**
> Les éléments de syntaxe `strict;`, `provider contract transactional_query`, `late numbering` sur une entité
> managed et `union all` dans une `define view entity` sont utilisés dans ce document. S'ils sont refusés par
> le compilateur du système :
> - `strict;` → retirer la ligne (la sémantique ne change pas) ;
> - `provider contract transactional_query` → retirer la clause ;
> - `union all` en view entity → créer la vue d'union en vue classique (`define view` + `@AbapCatalog.sqlViewName`), les view entities peuvent la consommer ;
> - `late numbering` → voir le repli en [annexe A](#annexe-a--diagnostic-des-incidents-fréquents).

> **📸 COPIE D'ÉCRAN N°01** — ADT : aperçu de `I_PurchaseRequisitionItem`, champs relevés en R1
> *Remplacer cette ligne par :* `![Copie 01](images/RESA-SRC-RAP/capture-01.png)`

> **📸 COPIE D'ÉCRAN N°02** — SE11 : objet de verrouillage de `EBAN` et ses paramètres (R4)
> *Remplacer cette ligne par :* `![Copie 02](images/RESA-SRC-RAP/capture-02.png)`

---

## 3. Architecture et synoptique

### 3.1 Vue d'ensemble

```
CUSTOMIZING (classe C)   ZTRESA_SRCDOCTY  ZTRESA_REPLCFG  ZTRESA_PCLIMIT  ZTRESA_NEIGHB  ZTRESA_WHSE
EXISTANT                 ⟨TABLE_ZC10⟩     I_PurchaseRequisitionItem   fiches info   stock
                                  │
INTERFACE (C)            ZI_ResaSrcEdi   ZI_ResaSrcWhse   ZI_ResaSrcStore      (division × article × source)
                                  └──────────┬──────────┘
                                   ZI_ResaSrcCandidate (union)  ───►  ZI_ResaSrcCandidateVH
                                             │
                                   ZI_ResaSourceAvail  (poste de DA × source, + CASH)
                                             │
                                   ZI_ResaSourceCount  (nombre de sources par poste)
                                             │
BO DÉTERMINATION (D)     ZR_ResaPurReqSource ─┬─ composition ─► ZR_ResaSourceOption  (lecture seule)
  persistance A          (ZTRESA_SRCSEL)      └─ composition ─► ZR_ResaPettyCash     (ZTRESA_PCORDER, SNRO)
BO CRÉATION (D)          ZR_ResaReplenRequest (ZTRESA_REPLREQ)
                                             │
PROJECTIONS (F)          Alt. 1 : ZC_ResaPurReqSourceAll  (+ ZC_ResaSourceOptAll, ZC_ResaPettyCashAll)
                                  ZC_ResaReplenRequestPR
                         Alt. 2 : ZC_ResaPurReqSourceResa (+ ZC_ResaSourceOptResa, ZC_ResaPettyCashResa)
                                  ZC_ResaReplenRequestPO
                                             │
SERVICES V4 (G)          ZUI_RESASRC_ALT1   ZUI_RESAREPL_ALT1   ZUI_RESASRC_ALT2   ZUI_RESAREPL_ALT2
                                             │
APPLICATIONS (H)         zlr.alt1source     zlr.alt1creation    zlr.alt2source     zlr.alt2creation
```

### 3.2 Pourquoi « managed with unmanaged save »

| Contrainte | Réponse |
|---|---|
| La racine est le **poste de DA standard** : la ligne de choix de source (`ZTRESA_SRCSEL`) n'existe pas tant que l'utilisateur n'a rien retenu | La sauvegarde managée ne sait faire qu'un `UPDATE` d'une ligne existante ; la sauvegarde non managée fait un `MODIFY` (création ou mise à jour) |
| La commande doit être **réellement créée** par `BAPI_PO_CREATE1` | Appel dans `save_modified`, seul endroit où une mise à jour hors BO est autorisée |
| Le tampon transactionnel, les actions, le contrôle des fonctionnalités et des autorisations restent standard | Partie « managed » conservée |

### 3.3 Cycle de vie d'un poste

| `SourceStatus` | Libellé | Atteint par | Actions possibles |
|---|---|---|---|
| `TODO` | À traiter | Poste de DA ouvert sans choix | Retenir une source |
| `SEL` | Source choisie | `SelectSource` | Retenir une autre source, convertir (si source ≠ petite caisse), petite caisse (si éligible) |
| `REQ` | Conversion demandée | `ConvertToPurchaseOrder` (état transitoire dans la requête) | — |
| `ERR` | Échec de conversion | Retour en erreur de la BAPI | Retenir une source, convertir à nouveau |
| `CONV` | Approvisionnement lancé | CA / STO créée, ou document `ZPC` créé | — |

### 3.4 Synoptique des étapes

| N° | Étape | Objet | Partie |
|---|---|---|---|
| A1 | Créer les domaines et éléments de données | `ZRESA_SRCTYPE`, `ZRESA_PROCSTAT`, `ZRESA_PCSTAT` | A |
| A2 | Créer les tables applicatives | `ZTRESA_SRCSEL`, `ZTRESA_PCORDER`, `ZTRESA_REPLREQ` | A |
| A3 | Créer la tranche de numéros | `ZRESA_PC` | A |
| A4 | Créer la classe de messages | `ZRESA_SRC` | A |
| B1 | Créer les tables de customizing | 5 tables `ZTRESA_…` | B |
| B2 | Générer la maintenance SM30 | Groupe de fonctions `ZRESA_SRC_TMG` | B |
| B3 | Saisir le paramétrage initial | Ordre de customizing | B |
| C1 à C6 | Vues CDS d'interface et contrôle d'accès | `ZI_Resa…` | C |
| D1 à D5 | Vues du BO, entités abstraites, behavior definitions | `ZR_Resa…`, `ZD_Resa…` | D |
| E1 à E6 | Classes d'implémentation et classes utilitaires | `ZBP_R_…`, `ZCL_RESA_SRC_…` | E |
| F1 à F4 | Projections, behavior de projection, extensions de métadonnées, DCL | `ZC_Resa…` | F |
| G1 à G3 | Service definitions, bindings, publication | `ZUI_…` | G |
| H1 à H5 | Génération et configuration des applications | 4 applications | H |
| I1 à I5 | Objet d'autorisation, rôles, catalogues, tuiles | PFCG, launchpad | I |
| J | Recette, transport, exploitation | — | J |

---

<!-- SUITE -->
