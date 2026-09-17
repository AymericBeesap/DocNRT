# Extension d'une application SAP Fiori standard via Key User Extensibility (in-app)

**Contexte :** SAP S/4HANA Cloud Private Edition 2021 FPS02 (SAP_UI 7.56, SAPUI5 1.96.x)
**Application concernée :** F0842A — *Gérer les commandes d'achat* (et ME21N / ME22N / ME23N)
**Outils :** applications Fiori d'extensibilité (*Custom Fields and Logic*, *Configure Software Packages*, *Register Extensions for Transport*) + *Adapter l'UI* dans le Launchpad
**Résultat :** un **champ personnalisé d'en-tête** « Référence interne » visible et saisissable dans F0842A (et ME21N), ajouté à l'écran **sans code**, rendu **obligatoire pour les commandes NB** par une logique key user, et transportable.

---

## Sommaire

1. [Objectif et résultat attendu](#1-objectif-et-résultat-attendu)
2. [Prérequis techniques](#2-prérequis-techniques)
3. [Étape 1 – Paramétrer l'Adaptation Transport Organizer](#étape-1--paramétrer-ladaptation-transport-organizer)
4. [Étape 2 – Configurer le package logiciel](#étape-2--configurer-le-package-logiciel)
5. [Étape 3 – Créer le champ personnalisé](#étape-3--créer-le-champ-personnalisé)
6. [Étape 4 – Activer le champ pour les interfaces](#étape-4--activer-le-champ-pour-les-interfaces)
7. [Étape 5 – Publier et vérifier la génération](#étape-5--publier-et-vérifier-la-génération)
8. [Étape 6 – Enregistrer l'extension pour le transport](#étape-6--enregistrer-lextension-pour-le-transport)
9. [Étape 7 – Adapter l'UI de F0842A](#étape-7--adapter-lui-de-f0842a)
10. [Étape 8 – Créer la logique personnalisée](#étape-8--créer-la-logique-personnalisée)
11. [Étape 9 – Tester](#étape-9--tester)
12. [Étape 10 – Transporter](#étape-10--transporter)
13. [Comparatif des trois approches](#comparatif-des-trois-approches)
14. [Dépannage](#dépannage)
15. [Liste des captures d'écran à réaliser](#liste-des-captures-décran-à-réaliser)
16. [Références](#références)

---

## 1. Objectif et résultat attendu

| # | Élément | Outil | Visible sans données |
|---|---|---|---|
| A | Champ d'en-tête `ZZ1_REFINTERNE` (Texte, 20) – libellé « Référence interne » | *Custom Fields and Logic* | — |
| B | Champ activé pour F0842A et pour ME21N/ME22N/ME23N | Onglet *UIs and Reports* | — |
| C | Champ ajouté dans la section « Informations générales » de la commande | *Adapter l'UI* | Oui (commande en création, champ vide) |
| D | Champ disponible comme filtre de la liste | *Adapter les filtres* | Oui (barre de filtres) |
| E | Référence interne obligatoire si type = NB | *Custom Logic* (BAdI key user) | — |
| F | Transport vers QAS / PRD | *Configure Software Packages* + *Register Extensions for Transport* | — |

### Écran attendu (maquette – commande en création, sans données)

```text
┌───────────────────────────────────────────────────────────────────────────────┐
│ ◀  Nouvelle commande d'achat                                         🔍  👤  │
├───────────────────────────────────────────────────────────────────────────────┤
│  Informations générales                                                       │
│   Type de commande      [ NB  ▾ ]        Fournisseur          [          ]    │
│   Organisation d'achats [       ]        Groupe d'acheteurs   [          ]    │
│   Société               [       ]      ┏ Référence interne    [          ] ┓  │
│                                        ┗━━━━━━━━━━━━ ajouté (C) ━━━━━━━━━━┛  │
│  Postes (0)                                                                   │
├───────────────────────────────────────────────────────────────────────────────┤
│                                                   [ Commander ]  [ Annuler ]  │
└───────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Prérequis techniques

### 2.1 Objets et services

| Élément | Action | Transaction |
|---|---|---|
| Adaptation Transport Organizer (ATO) | Paramétré dans le mandant de développement | `S_ATO_SETUP` (note SAP 2807979) |
| Services ICF des apps de transport | `NW_APS_ATO_CONF`, `NW_APS_ATO_REGI`, `NW_APS_ATO_LIB`, `NW_APS_LIB` actifs | `SICF` |
| Application *Custom Fields and Logic* (F1481) | Services OData et ICF actifs (liste exacte : fiche de l'app) | `/IWFND/MAINT_SERVICES`, `SICF` |
| *Configure Software Packages* (F1590) | Services actifs | idem |
| *Register Extensions for Transport* (F1589) | Services actifs | idem |
| Launchpad | Nœud `/sap/bc/ui2/flp` actif | `SICF` |
| App F0842A | `MM_PUR_PO_MAINT_V2_SRV` actif | `/IWFND/MAINT_SERVICES` |
| Package transportable | `ZMM_KEYUSER` (couche de transport Z) | `SE21` |
| Ordre Workbench | `S4DK9xxxxx` | `SE09` |

> Les numéros d'application F1590 / F1589 et les catalogues peuvent varier selon la version : vérifiez-les dans la *SAP Fiori Apps Reference Library* pour 2021 FPS02.

### 2.2 Rôles et autorisations

| Utilisateur | Rôle / Catalogue | Pourquoi |
|---|---|---|
| Key user | **`SAP_UI_FLEX_KEY_USER`** | Active l'entrée *Adapter l'UI* et l'enregistrement de changements key user |
| | Rôle Z contenant le catalogue métier **`SAP_CORE_BC_EXT`** (*Extensibility*) — ou le modèle `SAP_BR_EXTENSIBILITY_SPEC` s'il existe dans votre release | Accès à *Custom Fields and Logic* et aux apps d'extensibilité |
| | Catalogue / rôle de *Register Extensions for Transport* et *Configure Software Packages* (ex. `SAP_NW_APS_EXT_ATO_PK_AI_APP` selon version) | Transport des extensions |
| | `S_TRANSPRT`, `S_DEVELOP` sur `ZMM_KEYUSER` | Enregistrement des objets générés dans l'OT |
| | Rôle dérivé de `SAP_BR_PURCHASER` | Ouvrir F0842A pour l'adapter |
| Utilisateur final | Rôle acheteur | Utiliser le champ |

Transactions : `PFCG`, `SU01`, `SU53`.

### 2.3 Transactions et apps utilisées

| Transaction / App | Usage |
|---|---|
| `S_ATO_SETUP` | Paramétrage ATO |
| `SE21`, `SE09`, `SE10` | Package, ordres |
| F1481 *Custom Fields and Logic* | Champs et logique personnalisés |
| F1590 *Configure Software Packages* | Rattacher package + OT |
| F1589 *Register Extensions for Transport* | Affecter les extensions au package |
| *Adapter l'UI* (menu utilisateur du FLP) | Placer le champ à l'écran |
| `SE11` | Contrôle technique (`EKKO`) |
| `/IWFND/CACHE_CLEANUP`, `/IWBEP/CACHE_CLEANUP` | Purge des caches de métadonnées OData |
| `/UI2/INVALIDATE_GLOBAL_CACHES` | Purge des caches FLP |
| `ME21N` / `ME23N` | Test SAP GUI |

---

## Étape 1 – Paramétrer l'Adaptation Transport Organizer

1. `S_ATO_SETUP` (mandant de développement, avec un utilisateur ayant les droits d'administration).
2. Renseigner (valeurs indicatives, voir note 2807979) :

| Champ | Valeur recommandée | Commentaire |
|---|---|---|
| Configuration | **Standard** (transportable) | « Local only » ne permet que des tests |
| Préfixe objets locaux (sandbox) | `YY1_` | Objets de test non transportés |
| Préfixe objets transportables | `ZZ1_` | Préfixe de nos champs / logiques |
| Package local (sandbox) | `TEST_YY_KEY_USER` (proposé) | |
| Package transportable par défaut | `ZMM_KEYUSER` ou package dédié | |

3. **Enregistrer**. Vérifier le message de succès.

```text
S_ATO_SETUP – Paramétrage de l'Adaptation Transport Organizer
┌───────────────────────────────────────────────────────────────┐
│ Configuration              (•) Standard   ( ) Local only       │
│ Préfixe sandbox            [YY1_]                              │
│ Préfixe transportable      [ZZ1_]                              │
│ Package sandbox            [TEST_YY_KEY_USER   ]               │
│ Package transportable      [ZMM_KEYUSER        ]               │
│                                             [ Enregistrer ]    │
└───────────────────────────────────────────────────────────────┘
```

> ⚠️ Le paramétrage ATO est structurant : le définir une fois, en accord avec l'équipe Basis, avant toute création de champ.

📸 *Capture 3-01 : S_ATO_SETUP*

---

## Étape 2 – Configurer le package logiciel

App **Configure Software Packages** (F1590) :

1. Cliquer sur **Ajouter un enregistrement** (*Add Registration*) → sélectionner le package `ZMM_KEYUSER` (créé au préalable en `SE21`).
2. Sélectionner la ligne → **Assigner un ordre** → choisir l'ordre Workbench `S4DK9xxxxx`.
3. Activer **Gestion automatique des tâches** (*Automatic Task Handling*) — ou **Gestion automatique des ordres** si votre gouvernance l'autorise.
4. **Sauvegarder**.

| Colonne | Valeur |
|---|---|
| Package | `ZMM_KEYUSER` |
| Ordre de transport | `S4DK9xxxxx` |
| Gestion automatique des tâches | Activée |
| Statut | Enregistrement des modifications actif |

📸 *Capture 3-02 : Configure Software Packages*

---

## Étape 3 – Créer le champ personnalisé

App **Custom Fields and Logic** (F1481, FR : *Zones personnalisées et logique*) → onglet **Custom Fields** → bouton **+** (*Créer*).

Boîte **Nouvelle zone** (*New Field*) :

| Champ | Valeur | Remarque |
|---|---|---|
| Contexte métier (*Business Context*) | **Procurement: Purchasing Document** (`MM_PURDOC_HEADER`) | En-tête du document d'achat. Pour le poste : *Procurement: Purchasing Document Item* (`MM_PURDOC_ITEM`) |
| Libellé (*Label*) | `Référence interne` | |
| Identificateur (*Identifier*) | `ZZ1_REFINTERNE` | Préfixe imposé par l'ATO |
| Type | **Texte** (*Text*) | Autres : Nombre, Montant avec devise, Date, Case à cocher, Liste de codes… |
| Longueur | `20` | |
| Info-bulle (*Tooltip*) | `Référence interne de la demande d'achat` | |

Cliquer sur **Créer et éditer** (*Create and Edit*).

```text
┌─────────────────────────── Nouvelle zone ───────────────────────────┐
│ Contexte métier  [ Procurement: Purchasing Document           ▾ ]   │
│ Libellé          [ Référence interne                            ]   │
│ Identificateur   [ ZZ1_REFINTERNE                               ]   │
│ Type             [ Texte ▾ ]      Longueur [ 20 ]                   │
│ Info-bulle       [ Référence interne de la demande d'achat      ]   │
│                        [ Créer et éditer ] [ Créer ] [ Annuler ]    │
└─────────────────────────────────────────────────────────────────────┘
```

Sur la page de détail (statut **Brouillon**), les onglets disponibles sont typiquement : *Informations générales*, *Traductions*, *UIs and Reports*, *Scénarios métier*, *Modèles d'e-mail*, *Modèles de formulaire*, *Utilisation*.

- **Traductions** : ajouter `EN` → `Internal Reference`.

> Informations techniques (contexte `MM_PURDOC_HEADER`) : le champ est stocké dans `EKKO` via l'include `EKKO_INCL_EEW_PS`, avec le suffixe de contexte **`PDH`**. Le nom technique final sera donc `ZZ1_REFINTERNE_PDH`.

📸 *Capture 3-03 : boîte « Nouvelle zone »*

---

## Étape 4 – Activer le champ pour les interfaces

Onglet **UIs and Reports** :

| Interface / Source de données | Action |
|---|---|
| **Manage Purchase Orders** (service `MM_PUR_PO_MAINT_V2_SRV`) | **Activer l'utilisation** (*Enable Usage*) |
| Create / Change / Display Purchase Order – Advanced (SAP GUI `ME21N`/`ME22N`/`ME23N`) | Activer l'utilisation |
| Rapports / sources analytiques d'achats | Uniquement si besoin |

Onglet **Scénarios métier** : aucun pour ce cas (utile par exemple pour propager vers la réception de marchandises).

Cliquer sur **Sauvegarder** (*Save*).

```text
UIs and Reports
┌────────────────────────────────────────────────────────────┬──────────────┐
│ Interface / Rapport                                        │ Statut       │
├────────────────────────────────────────────────────────────┼──────────────┤
│ Manage Purchase Orders  (MM_PUR_PO_MAINT_V2_SRV)            │ ✔ Activé     │
│ Purchase Order – Advanced (SAP GUI)                        │ ✔ Activé     │
│ …                                                          │ [Activer]    │
└────────────────────────────────────────────────────────────┴──────────────┘
```

📸 *Capture 3-04 : onglet UIs and Reports*

---

## Étape 5 – Publier et vérifier la génération

1. Cliquer sur **Publier** (*Publish*). Le statut passe à **Publication en cours** puis **Publié** (quelques minutes : extension de tables, de vues CDS, de structures et de services OData).
2. Actualiser la liste jusqu'au statut **Publié**.
3. Contrôles techniques (optionnels mais recommandés) :

| Contrôle | Transaction | Attendu |
|---|---|---|
| Zone en base | `SE11` → `EKKO` → include `EKKO_INCL_EEW_PS` | Zone `ZZ1_REFINTERNE_PDH` |
| Métadonnées OData | `/IWFND/GW_CLIENT` → `/sap/opu/odata/sap/MM_PUR_PO_MAINT_V2_SRV/$metadata` | Propriété `ZZ1_REFINTERNE_PDH` dans `C_PurchaseOrderTPType` |
| Caches | `/IWFND/CACHE_CLEANUP` et `/IWBEP/CACHE_CLEANUP` | À exécuter si la propriété n'apparaît pas |

📸 *Capture 3-05 : statut « Publié »*

---

## Étape 6 – Enregistrer l'extension pour le transport

App **Register Extensions for Transport** (F1589) :

1. Rechercher `ZZ1_REFINTERNE`.
2. Sélectionner la ligne → **Réaffecter au package** (*Reassign to Package*) → `ZMM_KEYUSER`.
3. Vérifier les colonnes : *Transportable* = Oui, *Ordre* = `S4DK9xxxxx`.

> À refaire pour la logique personnalisée (Étape 8). Les objets créés avant l'enregistrement du package sont dans le package local et ne sont **pas** transportés tant qu'ils ne sont pas réaffectés.

📸 *Capture 3-06 : Register Extensions for Transport*

---

## Étape 7 – Adapter l'UI de F0842A

### 7.1 Ajouter le champ sur la page de la commande

1. Ouvrir le FLP avec l'utilisateur key user → tuile **Gérer les commandes d'achat**.
2. Cliquer sur **Créer** (commande en brouillon, champs vides) — ou ouvrir une commande existante.
3. Menu utilisateur (avatar en haut à droite) → **Adapter l'UI** (*Adapt UI*). Confirmer le message d'information éventuel.
4. La barre d'adaptation apparaît :

```text
┌───────────────────────────────────────────────────────────────────────────────┐
│ [Adaptation de l'UI | Navigation]        ↶  ↷   [Réinitialiser] [Publier]     │
│                                                          [Sauvegarder et quitter] │
└───────────────────────────────────────────────────────────────────────────────┘
```

5. Mode **Adaptation de l'UI** : survoler le groupe **Informations générales** → clic droit → **Ajouter : Zone** (*Add: Field*).
6. Boîte **Zones disponibles** : rechercher `Référence` → cocher **Référence interne** → **OK**.
7. Faire glisser le champ à la position souhaitée (ex. sous *Société*).
8. (Optionnel) clic droit → **Renommer** pour ajuster le libellé à l'écran.
9. Cliquer sur **Publier** → sélectionner le package `ZMM_KEYUSER` et l'ordre `S4DK9xxxxx` → **OK**.
10. **Sauvegarder et quitter**.

Le champ est immédiatement visible, **vide**, sur la commande en création.

### 7.2 Ajouter le champ comme filtre de la liste (visible sans données)

1. Revenir sur la List Report de F0842A (aucune recherche lancée).
2. **Adapter les filtres** → rechercher *Référence interne* → cocher → **OK**.
3. Enregistrer une vue (*Standard ▾* → **Enregistrer sous**) nommée `Acheteurs – Référence interne`, cochée **Publique** et **Définir par défaut** (selon vos droits et votre politique de variantes).

> Pour la List Report, la personnalisation des filtres/colonnes via vue publique est l'approche la plus simple. Les changements *Adapter l'UI* sont eux appliqués à tous les utilisateurs après publication.

📸 *Capture 3-07 : menu utilisateur – Adapter l'UI*
📸 *Capture 3-08 : boîte « Zones disponibles »*
📸 *Capture 3-09 : champ visible sur la commande en création*
📸 *Capture 3-10 : filtre « Référence interne » dans la barre de filtres vide*

---

## Étape 8 – Créer la logique personnalisée

App **Custom Fields and Logic** → onglet **Custom Logic** → **+** (*Créer*).

Boîte **Nouvelle implémentation d'amélioration** :

| Champ | Valeur |
|---|---|
| Contexte métier | **Procurement: Purchasing Document** |
| Définition BAdI | **Check of Purchase Order Before Saving** (`BD_MMPUR_FINAL_CHECK_PO`) |
| Description de l'implémentation | `Contrôle référence interne NB` |
| ID d'implémentation | `ZZ1_CHK_REFINTERNE` (préfixe ATO) |

→ **Créer**. L'éditeur s'ouvre sur l'onglet **Logique brouillon** (*Draft Logic*), avec la documentation de la BAdI accessible via **Afficher la documentation**.

### Code

```abap
* Logique key user – Contrôle avant sauvegarde de la commande d'achat
* Règle : la référence interne est obligatoire pour les commandes de type NB
DATA ls_message LIKE LINE OF messages.

IF purchaseorder-purchaseordertype = 'NB'
   AND purchaseorder-zz1_refinterne_pdh IS INITIAL.

  ls_message-messagetype      = 'E'.
  ls_message-messagevariable1 = 'Référence interne obligatoire (type NB)'.
  APPEND ls_message TO messages.

ENDIF.
```

Points d'attention :
- Utiliser `Ctrl+Espace` après `purchaseorder-` pour sélectionner le nom exact du champ personnalisé (avec suffixe `_PDH`) et des zones standard.
- Le langage est l'**ABAP restreint key user** : pas d'appel de classes/fonctions Z, pas de `SELECT` sur des tables non publiées. Pour ces besoins, voir le document 2.
- Seuls `MESSAGETYPE` et `MESSAGEVARIABLE1` (texte, 50 caractères max.) sont exploités ; le texte saisi ici n'est pas traduit automatiquement.

### Tester dans l'éditeur

1. Bouton **Tester** (*Test*) → saisir des valeurs de test pour `PURCHASEORDER` (type `NB`, référence vide) → **Exécuter**.
2. Vérifier que `MESSAGES` contient une ligne `E`.
3. Refaire avec une référence renseignée → `MESSAGES` vide.

### Publier

1. **Sauvegarder le brouillon** puis **Publier**.
2. Le statut passe à **Publié** ; l'onglet **Logique publiée** montre le code actif.
3. **Register Extensions for Transport** : réaffecter `ZZ1_CHK_REFINTERNE` au package `ZMM_KEYUSER`.

> Si le document 2 (implémentation développeur de la même BAdI) est déjà en place sur ce système : vérifier en `SE18` si la BAdI est à **utilisation multiple**. Sinon, ne garder qu'une seule implémentation active.

📸 *Capture 3-11 : boîte « Nouvelle implémentation d'amélioration »*
📸 *Capture 3-12 : éditeur Custom Logic avec le code*
📸 *Capture 3-13 : environnement de test*

---

## Étape 9 – Tester

| # | Interface | Type | Référence interne | Résultat attendu |
|---|---|---|---|---|
| T1 | F0842A | NB | vide | ⛔ `Référence interne obligatoire (type NB)` |
| T2 | F0842A | NB | `DA-2026-001` | ✅ Commande créée, valeur affichée en consultation |
| T3 | F0842A | autre type | vide | ✅ Commande créée |
| T4 | ME21N | NB | vide | ⛔ Même message |
| T5 | ME23N | — | — | Valeur visible dans l'onglet des champs personnalisés de l'en-tête |
| T6 | F0842A (liste) | — | filtre `DA-2026-001` | La commande T2 est retrouvée |

Déroulé F0842A :
1. **Créer** → type `NB`, fournisseur, organisation d'achats, groupe d'acheteurs, société, un poste.
2. Laisser **Référence interne** vide → **Commander** → message bloquant dans le popover.
3. Renseigner la référence → **Commander** → commande créée.

📸 *Capture 3-14 : message bloquant dans F0842A*
📸 *Capture 3-15 : champ dans ME23N*

---

## Étape 10 – Transporter

1. **Register Extensions for Transport** : vérifier que *le champ*, *la logique* et *les adaptations UI* sont dans `ZMM_KEYUSER` et sur l'ordre `S4DK9xxxxx`.
2. `SE09` → contrôler le contenu de l'ordre (objets générés par le framework d'extensibilité et changements UI du Layered Repository).
3. Libérer les tâches puis l'ordre → import via `STMS` (ou processus du fournisseur Private Cloud).
4. **Dans le système cible** : aucune republication n'est nécessaire ; purger les caches si le champ n'apparaît pas (`/IWFND/CACHE_CLEANUP`, `/IWBEP/CACHE_CLEANUP`, `/UI2/INVALIDATE_GLOBAL_CACHES`).
5. Les **vues publiques** (variantes de filtres) sont stockées séparément : les recréer dans le système cible ou les transporter selon la note SAP 3196205.

> **Suppression d'un champ publié** : possible uniquement sous conditions (plus d'utilisation, données à purger). Choisir le type et la longueur avec soin dès la création.

---

## Comparatif des trois approches

| Critère | 1 – Adaptation Project | 2 – BAdI développeur | 3 – Key User |
|---|---|---|---|
| Profil | Développeur UI5 | Développeur ABAP | Key user / consultant fonctionnel |
| Outil | BAS / VS Code | SE19 / ADT | Apps Fiori + Adapter l'UI |
| Portée | UI uniquement (nouvelle variante d'app) | Logique backend (Fiori + GUI) | Champs, UI, logique restreinte |
| Code | JavaScript / XML | ABAP complet | ABAP restreint |
| Tables / classes Z | Non | Oui | Non |
| Application standard modifiée | Non (variante séparée) | Non (point d'extension) | Oui (changements appliqués à l'app) |
| Nouvelle tuile FLP | Oui | Non | Non |
| Transport | OT (BSP) | OT Workbench + Customizing | ATO + OT |
| Stabilité en upgrade | Bonne (Safe Mode) à vérifier (fragments/contrôleurs) | Bonne sur BAdI publiée | Très bonne |

---

## Dépannage

| Symptôme | Cause probable | Solution |
|---|---|---|
| *Adapter l'UI* absent du menu utilisateur | Rôle `SAP_UI_FLEX_KEY_USER` manquant | `PFCG` / `SU01`, se reconnecter |
| App *Custom Fields and Logic* en erreur | Services OData/ICF inactifs, ATO non paramétré | Fiche F1481, `S_ATO_SETUP` |
| Identificateur impossible à saisir / préfixe inattendu | Paramétrage ATO | `S_ATO_SETUP` |
| Statut bloqué sur « Publication en cours » | Erreur de génération | Consulter le journal de l'app (onglet / lien *Journal*), `SLG1` |
| Champ absent de la boîte *Zones disponibles* | Interface non activée, cache OData | Étape 4 ; `/IWFND/CACHE_CLEANUP`, `/IWBEP/CACHE_CLEANUP` ; recharger le navigateur |
| Champ visible mais non sauvegardé | Activation manquante pour l'interface de saisie | Vérifier *UIs and Reports* |
| Bouton *Publier* grisé dans Adapter l'UI | Pas de package/ordre transportable, autorisations | Étape 2, `S_TRANSPRT` |
| Logique non déclenchée | Implémentation non publiée, autre implémentation active | Statut *Publié* ; `SE18` (utilisation multiple) |
| Message affiché sans texte | Texte non placé dans `MESSAGEVARIABLE1` | Voir code Étape 8 |
| Champ obligatoire non contrôlé dans ME21N | Contrôle par field control uniquement | Utiliser une BAdI de contrôle (KBA 3555492) |
| Objets non transportés | Restés dans le package local | *Register Extensions for Transport* → Réaffecter |

---

## Liste des captures d'écran à réaliser

| N° | Écran | Fichier suggéré |
|---|---|---|
| 3-01 | `S_ATO_SETUP` | `img/03-01-ato.png` |
| 3-02 | Configure Software Packages | `img/03-02-packages.png` |
| 3-03 | Nouvelle zone | `img/03-03-new-field.png` |
| 3-04 | UIs and Reports | `img/03-04-uis.png` |
| 3-05 | Statut Publié | `img/03-05-published.png` |
| 3-06 | Register Extensions for Transport | `img/03-06-register.png` |
| 3-07 | Menu Adapter l'UI | `img/03-07-adapt-ui.png` |
| 3-08 | Zones disponibles | `img/03-08-available-fields.png` |
| 3-09 | Champ sur commande en création | `img/03-09-field-empty.png` |
| 3-10 | Filtre dans la liste vide | `img/03-10-filter.png` |
| 3-11 | Nouvelle implémentation | `img/03-11-new-logic.png` |
| 3-12 | Éditeur Custom Logic | `img/03-12-logic-code.png` |
| 3-13 | Test de la logique | `img/03-13-logic-test.png` |
| 3-14 | Message dans F0842A | `img/03-14-fiori-error.png` |
| 3-15 | Champ dans ME23N | `img/03-15-me23n.png` |

---

## Références

- Note SAP 2807979 – Information to setup Adaptation Transport Organizer for S/4HANA On Premise via `S_ATO_SETUP`
- Note SAP 2660797 – Configuration des apps de transport d'extensions (on-premise)
- Note SAP 3196205 – Migration des variantes/vues d'apps UI5 entre systèmes
- KBA SAP 3268822 – Field Control in Purchase Orders and Purchase Requisitions
- KBA SAP 3555492 – Mandatory Custom Field Check not executed in ME21N
- SAP Help – *Extensibility* (Custom Fields and Logic, Configure Software Packages, Register Extensions for Transport)
- SAP Learning – *Get Started with In-App Extensibility in SAP S/4HANA* (chapitre *Transporting Extensions*)
- SAP Community – *Key User Extensibility on SAP S/4HANA Cloud and SAP S/4HANA On-Premise – UI Adaptations*
- SAP Fiori Apps Reference Library – F1481, F1589, F1590, F0842A
