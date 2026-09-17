# Extension d'une application SAP Fiori standard via BAdI (extensibilité développeur)

**Contexte :** SAP S/4HANA Cloud Private Edition 2021 FPS02 (ABAP Platform 2021, SAP_BASIS 7.56)
**Application concernée :** F0842A — *Gérer les commandes d'achat* (et transactions ME21N / ME22N)
**Outils :** SAP GUI (`SE18`, `SE19`, `SE11`, `SE91`…) ou ABAP Development Tools (ADT, Eclipse)
**Résultat :** un **contrôle métier avant sauvegarde** de la commande d'achat, paramétrable par table Z, avec message traduisible, actif dans l'app Fiori **et** dans SAP GUI.

---

## Sommaire

1. [Objectif et choix de la BAdI](#1-objectif-et-choix-de-la-badi)
2. [Prérequis et autorisations](#2-prérequis-et-autorisations)
3. [Étape 1 – Analyser la BAdI](#étape-1--analyser-la-badi)
4. [Étape 2 – Package et ordre de transport](#étape-2--package-et-ordre-de-transport)
5. [Étape 3 – Classe de messages](#étape-3--classe-de-messages)
6. [Étape 4 – Table de paramétrage Z](#étape-4--table-de-paramétrage-z)
7. [Étape 5 – Créer l'implémentation (SE19)](#étape-5--créer-limplémentation-se19)
8. [Étape 6 – Coder la logique](#étape-6--coder-la-logique)
9. [Étape 7 – Activer](#étape-7--activer)
10. [Étape 8 – Tester dans Fiori et SAP GUI](#étape-8--tester-dans-fiori-et-sap-gui)
11. [Étape 9 – Déboguer](#étape-9--déboguer)
12. [Étape 10 – Transporter](#étape-10--transporter)
13. [Annexe A – Variante classique ME_PROCESS_PO_CUST](#annexe-a--variante-classique-me_process_po_cust)
14. [Annexe B – Réaliser la même implémentation dans ADT](#annexe-b--réaliser-la-même-implémentation-dans-adt)
15. [Dépannage](#dépannage)
16. [Liste des captures d'écran à réaliser](#liste-des-captures-décran-à-réaliser)
17. [Références](#références)

---

## 1. Objectif et choix de la BAdI

### Règle métier implémentée

> Pour les couples **société / type de commande** déclarés actifs dans la table `ZMM_PO_CHK`, chaque poste de commande d'achat **doit avoir un article**. Sinon, la sauvegarde est bloquée avec le message :
> `Poste 00010 : article obligatoire (type NB)`

Cette règle est volontairement simple et vérifiable. L'intérêt de l'extensibilité développeur (par rapport au key user, cf. document 3) est montré par l'usage d'une **table Z** et d'une **classe de messages traduisible**, impossibles en logique key user.

### Comparatif des BAdIs disponibles pour la commande d'achat

| BAdI | Spot / Type | Déclenchement | Fiori F0842A | ME21N | Commentaire |
|---|---|---|---|---|---|
| **`BD_MMPUR_FINAL_CHECK_PO`** | `ES_MMPUR_PROCESS_PO_CLOUD` (nouvelle BAdI, publiée) | Avant sauvegarde | Oui | Oui | **Retenue** : messages remontés à l'app Fiori, API stable |
| `ME_PROCESS_PO_CUST` | BAdI classique | Traitement en-tête/poste, check, post | Partiel | Oui | Très puissante, mais des remontées de messages vers F0842A incomplètes sont signalées par la communauté (voir Annexe A) |
| `MM_PUR_S4_PO_MODIFY_HEADER` / `_ITEM` | `ES_MMPUR_PROCESS_PO_CLOUD` | Modification | Oui | Oui | Pour **valoriser** des champs (surtout custom fields) |
| `MM_PUR_S4_PO_FLDCNTRL_SIMPLE` | `ES_MMPUR_PROCESS_PO_CLOUD` | Contrôle de zones | Oui (liste de zones limitée) | Oui | Masquer / rendre obligatoire certaines zones de poste |

> **Points clés de `BD_MMPUR_FINAL_CHECK_PO`** : paramètres `PURCHASEORDER` (en-tête), `PURCHASEORDERITEMS` (table des postes, sans ligne d'en-tête) et `MESSAGES` (retour). La BAdI ne gère pas l'intégralité des attributs de message : **seuls `MESSAGETYPE` et `MESSAGEVARIABLE1` sont exploités**, `MESSAGEVARIABLE1` contenant le texte du message (50 caractères maximum). On construit donc le texte avec `MESSAGE … INTO` pour conserver la traduction SE91.

### Écran attendu (maquette F0842A)

```text
┌───────────────────────────────────────────────────────────────────────────────┐
│ ◀  Nouvelle commande d'achat                                         🔍  👤  │
├───────────────────────────────────────────────────────────────────────────────┤
│ Type de commande : NB     Fournisseur : 17300001    Société : 1010            │
│ ── Postes ─────────────────────────────────────────────────────────────────── │
│  Poste │ Article │ Texte court        │ Quantité │ Prix net                   │
│  10    │         │ Prestation diverse │ 1 UN     │ 150,00 EUR                 │
├───────────────────────────────────────────────────────────────────────────────┤
│ ⛔ 1                                               [ Commander ]  [ Annuler ]  │
└───────────────────────────────────────────────────────────────────────────────┘
      ┌────────────────────────────────────────────────┐
      │ Messages                                       │
      │ ⛔ Poste 00010 : article obligatoire (type NB)  │
      └────────────────────────────────────────────────┘
```

---

## 2. Prérequis et autorisations

### Système

| Élément | Valeur | Transaction |
|---|---|---|
| Release | S/4HANA 2021 FPS02 | *Système → Statut* |
| App F0842A opérationnelle | Service `MM_PUR_PO_MAINT_V2_SRV` actif | `/IWFND/MAINT_SERVICES` |
| Existence de la BAdI | `BD_MMPUR_FINAL_CHECK_PO` visible | `SE18` |
| Client de développement | Modifications d'objets Repository autorisées | `SCC4` |
| Clé développeur / enregistrement | Selon la gouvernance de votre Private Cloud | SAP for Me |

### Autorisations développeur (rôle Z à construire dans `PFCG`)

| Objet | Champs principaux | Usage |
|---|---|---|
| `S_DEVELOP` | DEVCLASS `ZMM_EXT`, OBJTYPE `ENHO`, `CLAS`, `TABL`, `MSAG`, `FUGR`, `DEBUG` ; ACTVT 01/02/03/06/07 (+ 03 pour débogage) | Création des objets et débogage |
| `S_TRANSPRT` | TTYPE `DTRA`, `CUST`, `TASK` ; ACTVT 01/02/03 | OT Workbench et Customizing |
| `S_TABU_DIS` / `S_TABU_NAM` | Groupe d'autorisation `&NC&` ou `ZMM_PO_CHK` | Maintenance de la table via `SM30` |
| `S_ADT_RES` | — | Si ADT est utilisé |

### Utilisateur de test

Rôle dérivé de `SAP_BR_PURCHASER` (catalogues contenant F0842A et « Créer une commande d'achat – avancé ») + objets `M_BEST_BSA`, `M_BEST_EKG`, `M_BEST_EKO`, `M_BEST_WRK` en création/modification.

### Transactions utilisées dans ce document

| Transaction | Usage |
|---|---|
| `SE18` | Afficher une définition de BAdI |
| `SE19` | Créer / éditer une implémentation de BAdI |
| `SE20` | Afficher un enhancement spot |
| `SE80` / `SE21` | Package |
| `SE09` / `SE10` | Ordres de transport |
| `SE91` | Classe de messages |
| `SE63` | Traduction des messages |
| `SE11` | Table de base de données |
| `SE54` | Générateur de maintenance de table |
| `SM30` | Maintenance des entrées de la table |
| `SE24` | Classe d'implémentation |
| `ME21N` / `ME22N` / `ME23N` | Test SAP GUI |
| `/IWFND/ERROR_LOG`, `/IWBEP/ERROR_LOG` | Erreurs OData |
| `ST22` | Dumps |
| `SAT` | Trace d'exécution |
| `/IWFND/GW_CLIENT` | Appels OData manuels |

---

## Étape 1 – Analyser la BAdI

1. `SE18` → option **BAdI Name** → `BD_MMPUR_FINAL_CHECK_PO` → **Afficher**.
2. Relever sur l'écran de définition :

   | Information | Où | Attendu / À noter |
   |---|---|---|
   | Enhancement Spot | En-tête | `ES_MMPUR_PROCESS_PO_CLOUD` |
   | Utilisation multiple | Onglet *Propriétés* | À noter (impacte la coexistence avec le document 3) |
   | Dépendance filtre | Noeud *Filtre* | À noter (normalement sans filtre) |
   | Interface | Noeud *Interface* | Nom de l'interface et **nom de la méthode** |
   | Signature | Double-clic sur la méthode | `PURCHASEORDER`, `PURCHASEORDERITEMS`, `MESSAGES` |
   | Classe exemple / par défaut | Noeud *Implementation* | Éventuelle classe d'exemple |
   | Documentation | Bouton *Documentation* | Limites des messages |

3. **Double-cliquer sur le type de chaque paramètre** pour ouvrir la structure dans `SE11` et **noter les noms de zones exacts** utilisés dans le code (ex. société, type de commande, numéro de poste, article).
4. `SE20` → Enhancement Spot `ES_MMPUR_PROCESS_PO_CLOUD` → **Afficher** : liste des BAdIs sœurs (`MM_PUR_S4_PO_MODIFY_HEADER`, `MM_PUR_S4_PO_MODIFY_ITEM`, `MM_PUR_S4_PO_FLDCNTRL_SIMPLE`, `MM_PUR_S4_PO_CHECK_ALL_ITEMS`, etc. selon le niveau de SP).

```text
SE18 – Business Add-In : afficher la définition BD_MMPUR_FINAL_CHECK_PO
┌──────────────────────────────────────────────────────────────────────┐
│ Enhancement Spot    ES_MMPUR_PROCESS_PO_CLOUD                        │
│ BAdI Definition     BD_MMPUR_FINAL_CHECK_PO                          │
│ ├─ Propriétés       [ ] Filtre  [?] Utilisation multiple            │
│ ├─ Interface        IF_…                                             │
│ │   └─ Méthode      … (PURCHASEORDER, PURCHASEORDERITEMS, MESSAGES)  │
│ └─ Implémentations  (liste des implémentations existantes)           │
└──────────────────────────────────────────────────────────────────────┘
```

**Astuce – retrouver une BAdI appelée par Fiori** : placer un point d'arrêt externe sur l'instruction ABAP `CALL BADI` (Débogueur → *Points d'arrêt → Point d'arrêt à l'instruction*), puis exécuter l'action dans F0842A. Pour les BAdIs classiques : point d'arrêt sur `CL_EXITHANDLER=>GET_INSTANCE`.

📸 *Capture 2-01 : SE18 – définition de la BAdI*
📸 *Capture 2-02 : signature de la méthode*

---

## Étape 2 – Package et ordre de transport

1. `SE21` → **Créer** :

   | Champ | Valeur |
   |---|---|
   | Package | `ZMM_EXT` |
   | Description | `Extensions achats – commande d'achat` |
   | Composant applicatif | `MM-PUR-PO` |
   | Composant logiciel | `HOME` |
   | Couche de transport | Couche Z de votre landscape |
   | Type de package | Non-main package |

2. `SE09` → créer un **ordre Workbench** `S4DK9xxxxx` (« BAdI contrôle commande d'achat ») et un **ordre Customizing** pour les entrées de table.

📸 *Capture 2-03 : création du package*

---

## Étape 3 – Classe de messages

1. `SE91` → Classe de messages `ZMM_PO` → **Créer**.
2. Onglet *Attributs* : texte bref `Messages extensions commande d'achat`, package `ZMM_EXT`.
3. Onglet *Messages* :

   | N° | Texte (FR) | Autodocument. |
   |---|---|---|
   | `001` | `Poste &1 : article obligatoire (type &2)` | Coché |

   > Le texte final doit tenir en **50 caractères** après remplacement des variables (limite de `MESSAGEVARIABLE1`).

4. Sauvegarder, affecter à l'OT.
5. Traduction : `SE91` → *Aller à → Traduction* (ou `SE63`) → langue **EN** : `Item &1: material is mandatory (type &2)`.

📸 *Capture 2-04 : SE91 message 001*

---

## Étape 4 – Table de paramétrage Z

### 4.1 Création de la table (`SE11`)

`SE11` → Table de base de données `ZMM_PO_CHK` → **Créer**.

**Onglet Livraison et maintenance**

| Champ | Valeur |
|---|---|
| Description | `Contrôle article obligatoire par société / type de commande` |
| Classe de livraison | `C` (table de paramétrage) |
| Data Browser / Table View Maint. | `Affichage/maintenance autorisés` |

**Onglet Zones**

| Zone | Clé | Init. | Type de données | Description |
|---|---|---|---|---|
| `MANDT` | ✔ | ✔ | `MANDT` | Mandant |
| `BUKRS` | ✔ | ✔ | `BUKRS` | Société |
| `BSART` | ✔ | ✔ | `ESART` | Type de document d'achat |
| `ACTIVE` | | | `XFELD` | Contrôle actif |

**Paramètres techniques** (`Aller à → Paramètres techniques`) : type de données `APPL2`, catégorie de taille `0`, mise en mémoire tampon non autorisée.

**Clés externes** (recommandé) : `BUKRS` → `T001`, `BSART` → `T161`.

Enregistrer, affecter au package `ZMM_EXT` et à l'OT, puis **Activer** (`Ctrl+F3`).

### 4.2 Générateur de maintenance (`SE54` ou `SE11 → Utilitaires → Générateur de maintenance de tables`)

| Champ | Valeur |
|---|---|
| Groupe d'autorisations | `&NC&` (ou groupe dédié) |
| Groupe de fonctions | `ZFG_MM_PO_CHK` |
| Package | `ZMM_EXT` |
| Type de maintenance | Une étape |
| Écran d'aperçu | `0001` (bouton *Chercher numéros d'écran*) |
| Routine d'enregistrement | Enregistrement standard |

Cliquer sur **Créer**, puis enregistrer dans l'OT Workbench.

### 4.3 Saisie des entrées (`SM30`)

`SM30` → `ZMM_PO_CHK` → **Maintenir** → *Nouvelles entrées* :

| Société | Type | Actif |
|---|---|---|
| `1010` (à adapter) | `NB` | ✔ |

Sauvegarder dans l'**OT Customizing**.

📸 *Capture 2-05 : SE11 – zones de ZMM_PO_CHK*
📸 *Capture 2-06 : SM30 – entrée 1010 / NB*

---

## Étape 5 – Créer l'implémentation (SE19)

1. `SE19` → section **Créer l'implémentation** :

```text
SE19 – Business Add-Ins : écran initial des implémentations
┌────────────────────────────────────────────────────────────────────────┐
│ Éditer l'implémentation                                                │
│   ( ) Nouvelle BAdI    Implémentation d'amélioration  [            ]   │
│   ( ) BAdI classique   Nom de l'implémentation        [            ]   │
│ Créer l'implémentation                                                 │
│   (•) Nouvelle BAdI    Spot d'amélioration  [ES_MMPUR_PROCESS_PO_CLOUD]│
│   ( ) BAdI classique   Nom de la BAdI       [                        ] │
│                                                   [ Créer impl. ]      │
└────────────────────────────────────────────────────────────────────────┘
```

   | Champ | Valeur |
   |---|---|
   | Radio | **Nouvelle BAdI** (*New BAdI*) |
   | Spot d'amélioration (*Enhancement Spot*) | `ES_MMPUR_PROCESS_PO_CLOUD` |

   → **Créer impl.** (*Create Impl.*)

2. Popup **Créer une implémentation d'amélioration** (*Create Enhancement Implementation*) :

   | Champ | Valeur |
   |---|---|
   | Implémentation d'amélioration | `ZEI_MM_PO_FINAL_CHECK` |
   | Texte bref | `Contrôles commande d'achat avant sauvegarde` |
   | Implémentation composite | (vide ou `ZCEI_MM_PUR` si vous regroupez vos extensions achats) |

   → Valider, puis choisir le package `ZMM_EXT` et l'OT Workbench.

3. Popup **Créer une implémentation de BAdI** (*Create BAdI Implementation*) :

   | Colonne | Valeur |
   |---|---|
   | Implémentation BAdI | `ZBI_MM_PO_FINAL_CHECK` |
   | Classe d'implémentation | `ZCL_MM_PO_FINAL_CHECK` |
   | Définition BAdI | `BD_MMPUR_FINAL_CHECK_PO` (F4 : le spot contient plusieurs BAdIs) |

   → Valider.

4. Si le système propose une classe d'exemple : choisir **Créer une classe vide** (ou copier l'exemple pour s'en inspirer). Confirmer la création de `ZCL_MM_PO_FINAL_CHECK` dans le package et l'OT.

5. L'écran de l'implémentation s'ouvre :

```text
Implémentation d'amélioration ZEI_MM_PO_FINAL_CHECK        (onglet Éléments)
┌──────────────────────────────┬─────────────────────────────────────────────┐
│ ▾ ZBI_MM_PO_FINAL_CHECK      │ Implémentation BAdI  ZBI_MM_PO_FINAL_CHECK  │
│    • Classe d'implémentation │ Définition BAdI      BD_MMPUR_FINAL_CHECK_PO│
│                              │ Comportement d'exécution                    │
│                              │   [x] L'implémentation est active           │
│                              │   [ ] Implémentation par défaut             │
│                              │ Effet dans le mandant : implémentation      │
│                              │ appelée                                     │
└──────────────────────────────┴─────────────────────────────────────────────┘
```

6. Vérifier que **« L'implémentation est active »** (*Implementation is active*) est cochée.

📸 *Capture 2-07 : SE19 écran initial*
📸 *Capture 2-08 : popup « Créer une implémentation de BAdI »*
📸 *Capture 2-09 : écran de l'implémentation (active)*

---

## Étape 6 – Coder la logique

1. Dans l'arborescence, double-cliquer sur **Classe d'implémentation** → la classe `ZCL_MM_PO_FINAL_CHECK` s'ouvre (`SE24`) avec la méthode de l'interface.
2. Double-cliquer sur la méthode : le squelette `METHOD … ENDMETHOD.` est **généré par le système** (ne pas modifier la ligne `METHOD`).
3. Coller le corps suivant **entre** `METHOD` et `ENDMETHOD` :

```abap
*----------------------------------------------------------------------*
* Contrôle avant sauvegarde de la commande d'achat
* Règle : pour les couples société / type actifs dans ZMM_PO_CHK,
*         chaque poste doit avoir un article.
* Appelé par : F0842A (Gérer les commandes d'achat) et ME21N/ME22N
*----------------------------------------------------------------------*
    DATA: ls_message LIKE LINE OF messages,
          lv_text    TYPE string.

*   1. Le contrôle est-il actif pour cette société et ce type ?
    SELECT SINGLE @abap_true
      FROM zmm_po_chk
      WHERE bukrs  = @purchaseorder-companycode
        AND bsart  = @purchaseorder-purchaseordertype
        AND active = @abap_true
      INTO @DATA(lv_active).

    IF lv_active <> abap_true.
      RETURN.
    ENDIF.

*   2. Contrôle poste par poste
*      (PURCHASEORDERITEMS n'a pas de ligne d'en-tête : LOOP ... INTO)
    LOOP AT purchaseorderitems INTO DATA(ls_item).

      IF ls_item-material IS INITIAL.

*       Texte construit depuis SE91 pour conserver la traduction
        MESSAGE e001(zmm_po)
          WITH ls_item-purchaseorderitem
               purchaseorder-purchaseordertype
          INTO lv_text.

        CLEAR ls_message.
        ls_message-messagetype      = 'E'.       " A, E, W ou I
        ls_message-messagevariable1 = lv_text.   " 50 caractères max.
        APPEND ls_message TO messages.

      ENDIF.

    ENDLOOP.
```

4. **Adapter les noms de zones** si ceux relevés à l'Étape 1 diffèrent (`companycode`, `purchaseordertype`, `purchaseorderitem`, `material`). Astuce : `Ctrl+Espace` après `purchaseorder-` propose les zones disponibles.
5. **Vérifier** (`Ctrl+F2`) puis **Sauvegarder**.

> **Règles de codage à respecter dans une BAdI de contrôle** : pas de `COMMIT WORK`, pas d'appel de dialogue (`POPUP_*`), pas d'instruction `MESSAGE` sans `INTO` (elle interromprait le flux OData), lectures en base limitées et indexées (la BAdI est appelée à chaque sauvegarde).

📸 *Capture 2-10 : code dans l'éditeur de classe*

---

## Étape 7 – Activer

1. Dans `SE24` : **Activer** la classe (`Ctrl+F3`) → sélectionner tous les objets inactifs (classe, méthodes).
2. Revenir dans `SE19` → implémentation `ZEI_MM_PO_FINAL_CHECK` → **Activer** (`Ctrl+F3`).
3. Contrôle : `SE19` → *Éditer l'implémentation* → `ZEI_MM_PO_FINAL_CHECK` → **Afficher** → statut *Actif*, runtime « Implémentation appelée ».
4. Contrôle transversal : `SE18` → `BD_MMPUR_FINAL_CHECK_PO` → *Implémentation → Aperçu* → `ZBI_MM_PO_FINAL_CHECK` listée et active.

> Si la BAdI est à **utilisation unique** et qu'une autre implémentation active existe (par exemple une logique key user du document 3), une seule sera exécutée : désactiver l'une des deux ou fusionner la logique.

---

## Étape 8 – Tester dans Fiori et SAP GUI

### Cas de test

| # | Société | Type | Poste | Article | Entrée ZMM_PO_CHK | Résultat attendu |
|---|---|---|---|---|---|---|
| T1 | 1010 | NB | 10 | vide (texte court + groupe de marchandises) | Active | ⛔ Erreur, sauvegarde bloquée |
| T2 | 1010 | NB | 10 | Article renseigné | Active | ✅ Commande créée |
| T3 | 1010 | NB | 10 | vide | `ACTIVE` décoché | ✅ Commande créée |
| T4 | Autre société | NB | 10 | vide | Pas d'entrée | ✅ Commande créée |
| T5 | 1010 | NB | 10 (article) + 20 (vide) | mixte | Active | ⛔ Erreur sur le poste 00020 uniquement |

### Test Fiori (F0842A)

1. FLP → tuile **Gérer les commandes d'achat** → **Créer**.
2. Renseigner type `NB`, fournisseur, organisation d'achats, groupe d'acheteurs, société `1010`.
3. Ajouter un poste **sans article** : texte court, groupe de marchandises, quantité, prix, division.
4. Cliquer sur **Commander** (*Order*) : le compteur de messages ⛔ s'affiche en pied de page ; le popover contient le message `Poste 00010 : article obligatoire (type NB)`.
5. Renseigner un article → **Commander** → la commande est créée.

### Test SAP GUI (ME21N)

1. `ME21N` → mêmes données → **Contrôler** (`Ctrl+Shift+F4`) puis **Sauvegarder**.
2. Le message apparaît dans la liste des messages de la commande ; la sauvegarde est refusée.

📸 *Capture 2-11 : F0842A – popover de messages*
📸 *Capture 2-12 : ME21N – message bloquant*

---

## Étape 9 – Déboguer

### Point d'arrêt externe (appels HTTP / OData)

1. `SE24` → `ZCL_MM_PO_FINAL_CHECK` → méthode → positionner le curseur sur la première ligne exécutable.
2. **Utilitaires → Paramètres → Éditeur ABAP → Débogage** : utilisateur = votre utilisateur de test (celui qui se connecte au FLP).
3. **Point d'arrêt externe** (`Ctrl+Shift+F9`, icône avec un personnage).
4. Déclencher **Commander** dans F0842A : la session de débogage s'ouvre dans SAP GUI.

### Outils complémentaires

| Outil | Usage |
|---|---|
| `/IWFND/ERROR_LOG` | Erreurs côté hub Gateway (réponse OData en erreur) |
| `/IWBEP/ERROR_LOG` | Erreurs côté backend OData |
| `ST22` | Dump ABAP (ex. zone inexistante, conversion) |
| `SAT` | Trace de performance (vérifier le coût de la BAdI) |
| `/IWFND/GW_CLIENT` | Rejouer une requête OData du service `MM_PUR_PO_MAINT_V2_SRV` |
| `F12` navigateur → *Réseau* → requête `$batch` | Contenu de la réponse et messages `sap-message` |

📸 *Capture 2-13 : débogueur arrêté dans la méthode*

---

## Étape 10 – Transporter

### Contenu attendu des ordres (`SE09`)

| OT | Objets |
|---|---|
| Workbench | `R3TR DEVC ZMM_EXT` (si nouveau), `R3TR MSAG ZMM_PO`, `R3TR TABL ZMM_PO_CHK`, `R3TR FUGR ZFG_MM_PO_CHK`, `R3TR TOBJ ZMM_PO_CHKS` (objet de maintenance), `R3TR ENHO ZEI_MM_PO_FINAL_CHECK`, `R3TR CLAS ZCL_MM_PO_FINAL_CHECK` |
| Customizing | `R3TR TABU ZMM_PO_CHK` (entrées) |

### Ordre d'import recommandé

1. OT Workbench (table, messages, classe, implémentation).
2. OT Customizing (entrées de paramétrage).
3. Traductions (si ordre séparé via `SE63`).

Libération : `SE09` → libérer les tâches puis l'ordre. Import : `STMS` (ou processus de votre fournisseur de Private Cloud).

---

## Annexe A – Variante classique ME_PROCESS_PO_CUST

À utiliser lorsque la logique doit intervenir **pendant la saisie** (méthodes `PROCESS_*`) ou modifier des zones standard. La documentation SAP de `MM_PUR_S4_PO_FLDCNTRL_SIMPLE` rappelle d'ailleurs qu'en on-premise, `ME_PROCESS_PO_CUST` (SE18/SE19) est l'outil le plus puissant.

> **Attention Fiori** : un retour communautaire indique que des messages levés avec les macros dans `ME_PROCESS_PO_CUST` s'affichent dans ME21N mais pas toujours dans F0842A (seul un message générique « postes en erreur » remonte). Tester systématiquement dans les deux interfaces ; pour un contrôle bloquant côté Fiori, privilégier `BD_MMPUR_FINAL_CHECK_PO`.

### Création

1. `SE19` → *Créer l'implémentation* → radio **BAdI classique** → Nom de la BAdI `ME_PROCESS_PO_CUST` → **Créer impl.**
2. Nom de l'implémentation : `ZME_PROCESS_PO_CUST` → texte bref → package `ZMM_EXT` → OT.
3. Onglet **Interface** : classe d'implémentation proposée `ZCL_IM_ME_PROCESS_PO_CUST`.
4. Méthodes disponibles : `INITIALIZE`, `OPEN`, `PROCESS_HEADER`, `PROCESS_ITEM`, `PROCESS_SCHEDULE`, `PROCESS_ACCOUNT`, `CHECK`, `POST`, `CLOSE`, `FIELDSELECTION_HEADER`, `FIELDSELECTION_ITEM` (et variantes `_REFKEYS`).
5. Double-cliquer sur `CHECK` pour coder, puis activer la classe et **activer l'implémentation** (bouton *Activer* de SE19).

### Exemple de code – méthode CHECK

```abap
METHOD if_ex_me_process_po_cust~check.

  INCLUDE mm_messages_mac.   " macros de gestion des messages achats

  DATA: ls_header TYPE mepoheader,
        lt_items  TYPE purchase_order_items,
        ls_item   TYPE purchase_order_item,
        ls_data   TYPE mepoitem.

  ls_header = im_header->get_data( ).

* Contrôle actif pour cette société / ce type ?
  SELECT SINGLE @abap_true
    FROM zmm_po_chk
    WHERE bukrs  = @ls_header-bukrs
      AND bsart  = @ls_header-bsart
      AND active = @abap_true
    INTO @DATA(lv_active).
  CHECK lv_active = abap_true.

  lt_items = im_header->get_items( ).

  LOOP AT lt_items INTO ls_item.
    ls_data = ls_item-item->get_data( ).

    CHECK ls_data-loekz IS INITIAL.          " ignorer les postes supprimés

    IF ls_data-matnr IS INITIAL.
      mmpur_context 901.                     " contexte client (900-999)
      mmpur_business_obj_id ls_data-id.      " rattache le message au poste
      mmpur_message_forced 'E' 'ZMM_PO' '001'
                           ls_data-ebelp ls_header-bsart '' ''.
      ls_item-item->invalidate( ).           " marque le poste en erreur
      ch_failed = abap_true.                 " bloque la sauvegarde
    ENDIF.
  ENDLOOP.

ENDMETHOD.
```

Pour retirer le message une fois l'erreur corrigée (logique placée dans `PROCESS_ITEM`) : `mmpur_remove_msg_by_context <id objet> 901.`

📸 *Capture 2-A1 : SE19 – BAdI classique, onglet Interface*

---

## Annexe B – Réaliser la même implémentation dans ADT

1. Eclipse → *Project Explorer* → projet ABAP du système → package `ZMM_EXT`.
2. Clic droit → **New → Other ABAP Repository Object** → *Enhancements* → **BAdI Implementation** → *Next*.
3. Assistant :

   | Champ | Valeur |
   |---|---|
   | Name | `ZEI_MM_PO_FINAL_CHECK` |
   | Description | `Contrôles commande d'achat avant sauvegarde` |
   | Enhancement Spot | `ES_MMPUR_PROCESS_PO_CLOUD` (bouton *Browse*) |

4. *Next* → OT → *Finish*. L'éditeur de l'implémentation d'amélioration s'ouvre.
5. Bouton **Add BAdI Implementation** :

   | Champ | Valeur |
   |---|---|
   | BAdI Definition | `BD_MMPUR_FINAL_CHECK_PO` |
   | BAdI Implementation | `ZBI_MM_PO_FINAL_CHECK` |

6. Dans la section *Implementing Class* : lien **Implementing Class** → *Create* → `ZCL_MM_PO_FINAL_CHECK`. ADT génère la classe avec `INTERFACES` et la méthode vide.
7. Coller le code de l'Étape 6, **Activer** (`Ctrl+F3`) la classe puis l'implémentation d'amélioration.
8. Vérifier dans l'éditeur que *Implementation is active* est coché.

---

## Dépannage

| Symptôme | Cause probable | Solution |
|---|---|---|
| La BAdI n'est pas appelée | Implémentation inactive, classe inactive, entrée `ZMM_PO_CHK` absente | `SE19` statut ; `SE24` activer ; `SM30` |
| Message affiché mais texte vide / seul le n° de poste s'affiche | Utilisation de `MESSAGEID` / `MESSAGENUMBER` au lieu du texte | Construire le texte avec `MESSAGE … INTO` et le mettre dans `MESSAGEVARIABLE1` |
| Texte tronqué | Limite de 50 caractères | Raccourcir le message SE91 |
| Dump `ITAB_ILLEGAL_COMPONENT` / zone inconnue | Nom de zone de structure incorrect | Vérifier la structure du paramètre dans `SE11` |
| Message OK dans ME21N, absent dans F0842A | Logique placée dans `ME_PROCESS_PO_CUST` | Déplacer le contrôle bloquant dans `BD_MMPUR_FINAL_CHECK_PO` |
| Deux logiques de contrôle, une seule s'exécute | BAdI à utilisation unique | Une seule implémentation active |
| Débogage externe ne s'arrête pas | Mauvais utilisateur dans les paramètres de débogage | Utilitaires → Paramètres → Débogage → utilisateur du FLP |
| La commande s'ouvre dans « Créer une commande d'achat – avancé » | Fonctionnalité non supportée par F0842A (KBA 2372209) | Tester dans ME21N ; la BAdI y est également appelée |

---

## Liste des captures d'écran à réaliser

| N° | Écran | Fichier suggéré |
|---|---|---|
| 2-01 | `SE18` définition | `img/02-01-se18.png` |
| 2-02 | Signature de la méthode | `img/02-02-signature.png` |
| 2-03 | `SE21` package | `img/02-03-package.png` |
| 2-04 | `SE91` message 001 | `img/02-04-se91.png` |
| 2-05 | `SE11` table | `img/02-05-se11.png` |
| 2-06 | `SM30` entrée | `img/02-06-sm30.png` |
| 2-07 | `SE19` écran initial | `img/02-07-se19.png` |
| 2-08 | Popup création implémentation | `img/02-08-popup.png` |
| 2-09 | Implémentation active | `img/02-09-active.png` |
| 2-10 | Code de la méthode | `img/02-10-code.png` |
| 2-11 | F0842A popover messages | `img/02-11-fiori-error.png` |
| 2-12 | ME21N message | `img/02-12-me21n.png` |
| 2-13 | Débogueur | `img/02-13-debug.png` |
| 2-A1 | `SE19` BAdI classique | `img/02-A1-classic.png` |

---

## Références

- `SE18` → documentation de `BD_MMPUR_FINAL_CHECK_PO` et de `MM_PUR_S4_PO_FLDCNTRL_SIMPLE` (dans votre système)
- SAP Business Accelerator Hub – BAdIs *Private Edition* (`PCE_…`)
- KBA SAP 3268822 – Field Control in Purchase Orders and Purchase Requisitions
- KBA SAP 3555492 – Mandatory Custom Field Check (recommande `MM_PUR_S4_PO_MODIFY_HEADER` ou `BD_MMPUR_FINAL_CHECK_PO` pour les contrôles)
- KBA SAP 2372209 – Purchase order opens in the Advanced (WEBGUI) app
- SAP Community – *Why message displayed shows no text, only the variable 1? (BAdI BD_MMPUR_FINAL_CHECK_PO)*
- SAP Community – *SAP PO Error Handling using BADI ME_PROCESS_PO_CUST*
