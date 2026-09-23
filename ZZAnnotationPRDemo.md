# Adaptation Project F1048 dans VS Code — bouton de navigation vers « Détermination Source Appro »

**Contexte :** SAP S/4HANA Cloud Private Edition 2021 FPS02 (SAP_UI 7.56, SAPUI5 1.96.x)
**Application étendue :** F1048 — *Process Purchase Requisitions* (Traiter les demandes d'achat)
**Outil :** Visual Studio Code + SAP Fiori tools
**Objectif :** ajouter un bouton dans la List Report de F1048 qui ouvre l'application spécifique `ZSourceOfSupply-determine` (document 04) en transmettant la DA sélectionnée, puis **déployer sur DEV** et **transporter vers QUALITÉ**.

> Ce document suppose que l'application Z du **document 04** est déployée sur DEV (BSP `ZMM_PR_SRC`, objet sémantique `ZSourceOfSupply`, action `determine`).

---

## Sommaire

1. [Objectif et maquette](#1-objectif-et-maquette)
2. [Prérequis](#2-prérequis)
3. [Étape 1 – Vérifier F1048](#étape-1--vérifier-f1048)
4. [Étape 2 – Préparer VS Code](#étape-2--préparer-vs-code)
5. [Étape 3 – Générer l'adaptation project](#étape-3--générer-ladaptation-project)
6. [Étape 4 – Structure du projet](#étape-4--structure-du-projet)
7. [Étape 5 – Ouvrir l'éditeur d'adaptation](#étape-5--ouvrir-léditeur-dadaptation)
8. [Étape 6 – Ajouter le bouton (fragment)](#étape-6--ajouter-le-bouton-fragment)
9. [Étape 7 – Extension de contrôleur et navigation croisée](#étape-7--extension-de-contrôleur-et-navigation-croisée)
10. [Étape 8 – Preview local](#étape-8--preview-local)
11. [Étape 9 – Déploiement sur DEV](#étape-9--déploiement-sur-dev)
12. [Étape 10 – Configuration du Launchpad DEV](#étape-10--configuration-du-launchpad-dev)
13. [Étape 11 – Tests sur DEV](#étape-11--tests-sur-dev)
14. [Étape 12 – Transport DEV → QUALITÉ](#étape-12--transport-dev--qualité)
15. [Étape 13 – Recette sur QUALITÉ](#étape-13--recette-sur-qualité)
16. [Dépannage](#dépannage)
17. [Captures à réaliser](#captures-à-réaliser)
18. [Références](#références)

---

## 1. Objectif et maquette

| # | Élément | Détail |
|---|---|---|
| A | Bouton `Source appro (Z)` dans la barre d'outils du tableau de F1048 | Fragment XML ajouté par l'adaptation project |
| B | Navigation vers l'app Z | `CrossApplicationNavigation.toExternal` vers `#ZSourceOfSupply-determine` |
| C | Transmission du contexte | Paramètre `PurchaseRequisition` (et poste si disponible) |
| D | Comportement sans sélection | Navigation sans filtre + message d'information |
| E | Application standard inchangée | La variante est une application distincte avec sa propre tuile |

```text
┌──────────────────────────────────────────────────────────────────────────────────┐
│ ◀  Traiter les demandes d'achat – Variante Z                            🔍  👤  │
├──────────────────────────────────────────────────────────────────────────────────┤
│ Standard ▾                                                                       │
│ Demande d'achat [      ]  Division [     ]  Groupe acheteurs [    ]   [Exécuter] │
├──────────────────────────────────────────────────────────────────────────────────┤
│ Demandes d'achat (0)       ┏━━━━━━━━━━━━━━━━━━┓  [Affecter une source] [...]  ⚙  │
│                            ┃ Source appro (Z) ┃ ◀ ajouté                         │
│                            ┗━━━━━━━━━━━━━━━━━━┛                                  │
│ ──────────────────────────────────────────────────────────────────────────────── │
│ DA / Poste │ Article │ Division │ Quantité │ Date de livraison │ Statut          │
│                                                                                  │
│            Pour commencer, définissez les filtres et choisissez « Exécuter ».    │
└──────────────────────────────────────────────────────────────────────────────────┘
                                      │ clic
                                      ▼
            #ZSourceOfSupply-determine?PurchaseRequisition=10000123
```

Le bouton est visible **même sans données**, comme pour l'adaptation project du document 01.

---

## 2. Prérequis

### 2.1 Poste de travail

| Élément | Version / action |
|---|---|
| Visual Studio Code | Version courante |
| Extension | **SAP Fiori tools – Extension Pack** (inclut l'Adaptation Project Generator et l'Adaptation Editor) |
| Node.js | Version LTS supportée par les Fiori tools |
| Accès réseau | HTTPS direct vers le système DEV (pas de Cloud Connector nécessaire en VS Code) |
| Certificat | Si certificat auto-signé : importer le certificat ou utiliser `ignoreCertErrors` en développement |

### 2.2 Système DEV

| Élément | Valeur | Transaction |
|---|---|---|
| SAPUI5 | 1.96.x (minimum requis en VS Code : 1.72) | `Ctrl+Alt+Shift+P` dans le FLP |
| Nœuds ICF | `/sap/bc/ui5_ui5`, `/sap/bc/lrep`, `/sap/bc/ui2/app_index`, `/sap/bc/adt`, `/sap/opu/odata` | `SICF` |
| Service de déploiement | `/UI5/ABAP_REPOSITORY_SRV` actif | `/IWFND/MAINT_SERVICES` |
| Service de F1048 | `MM_PUR_PR_PROCESS_SRV` actif | `/IWFND/MAINT_SERVICES` |
| Application Z cible | BSP `ZMM_PR_SRC` déployée, intent `ZSourceOfSupply-determine` opérationnel | Document 04 |
| Package | `ZMM_UI_EXT` (ou `ZMM_SOURCING`) | `SE21` |
| Ordre de transport | Workbench `S4DK9xxxxx` | `SE09` |

### 2.3 Autorisations

| Profil | Objets |
|---|---|
| Développeur | `S_DEVELOP` (OBJTYPE `WAPA`, DEVCLASS `ZMM_UI_EXT`, ACTVT 01/02/03/06/07), `S_TRANSPRT`, `S_SERVICE` (`/UI5/ABAP_REPOSITORY_SRV`), `S_ADT_RES` |
| Administrateur FLP | `SAP_UI2_ADMIN_700` ou équivalent |
| Testeur | Rôle acheteur (`SAP_BR_PURCHASER` dérivé) + rôle `Z_BR_SOURCING` (app cible) + rôle de la variante |
| Transport | `S_CTS_ADMI` / autorisations `STMS` selon votre gouvernance (souvent l'équipe Basis / le fournisseur PCE) |

---

## Étape 1 – Vérifier F1048

| Information | Valeur attendue | Où |
|---|---|---|
| App ID | `F1048` – *Process Purchase Requisitions* | Fiori Apps Reference Library |
| Composant SAPUI5 | `ui.ssuite.s2p.mm.pur.pr.prcss.s1` | Fiche de l'app / manifest |
| Application BSP | `MM_PR_PRCS1` (URL `/sap/bc/ui5_ui5/sap/mm_pr_prcs1`) | `SE80` |
| Service OData | `MM_PUR_PR_PROCESS_SRV` | `/IWFND/MAINT_SERVICES` |
| Rôle métier | `SAP_BR_PURCHASER` | Fiche de l'app |
| Intent standard | Semantic object `PurchaseRequisition`, action à relever dans le target mapping | `/UI2/FLPD_CUST` |
| Extensibilité UI | `"flexEnabled": true` dans le manifest | `/sap/bc/ui5_ui5/sap/mm_pr_prcs1/manifest.json` |

> Si votre système expose aussi **F1048A** (*Process Purchase Requisitions – Version 2*, OData V4 `MM_PUR_PROCESS_PR_V4`), vérifiez laquelle est réellement utilisée par les acheteurs : l'adaptation project doit porter sur l'application affichée dans leur espace.

📸 *Capture 5-01 : fiche F1048*
📸 *Capture 5-02 : manifest avec `flexEnabled`*

---

## Étape 2 – Préparer VS Code

1. Installer **SAP Fiori tools – Extension Pack** depuis la marketplace.
2. Palette `Ctrl+Shift+P` → `Fiori: Open Environment Check` → onglet *Environment* : vérifier Node.js, les extensions et la connexion système.
3. Déclarer le système : palette → `Fiori: Add SAP System` :

| Champ | Valeur |
|---|---|
| System type | ABAP On Premise |
| System name | `S4H_DEV_100` |
| URL | `https://<host>:<port>` |
| Client | `100` |
| Username / Password | utilisateur développeur |

4. Contrôle : `Fiori: Open Environment Check` → *System* → `S4H_DEV_100` → test de connexion vert (catalogue des services et LREP accessibles).

📸 *Capture 5-03 : Environment Check*

---

## Étape 3 – Générer l'adaptation project

Palette → **`Fiori: Open Adaptation Project Generator`**.

**Écran 1 – Système et application**

| Champ | Valeur |
|---|---|
| System | `S4H_DEV_100` |
| Username / Password | si non mémorisés |
| Application | **Process Purchase Requisitions** (`ui.ssuite.s2p.mm.pur.pr.prcss.s1`) |

**Écran 2 – Attributs du projet**

| Champ | Valeur |
|---|---|
| Project Name | `zprsourcingnav` |
| Application Title | `Traiter les demandes d'achat – Variante Z` |
| Namespace | `customer.zprsourcingnav` (généré, = **ID de la variante**) |
| Target Folder | dossier de travail local |
| SAPUI5 Version | version du système (1.96.x) |
| Enable TypeScript | Non |
| Add deployment configuration | **Oui** |

**Écran 3 – Déploiement**

| Champ | Valeur |
|---|---|
| SAPUI5 ABAP Repository | `ZPR_SRC_NAV` (≤ 15 caractères) |
| Description | `Variante Z F1048 navigation sourcing` |
| Package | `ZMM_UI_EXT` |
| Transport Request | `S4DK9xxxxx` |

→ **Finish**. VS Code ouvre le projet et installe les dépendances.

📸 *Capture 5-04 : générateur — sélection de l'application*
📸 *Capture 5-05 : attributs du projet*

---

## Étape 4 – Structure du projet

```text
zprsourcingnav/
├── package.json
├── ui5.yaml                      configuration du preview (middleware adp)
├── ui5-deploy.yaml               configuration du déploiement ABAP
└── webapp/
    ├── manifest.appdescr_variant descripteur de la variante
    ├── i18n/i18n.properties
    └── changes/
        ├── fragments/            fragments XML (étape 6)
        ├── coding/               extensions de contrôleur (étape 7)
        └── *.change              changements flex générés
```

`manifest.appdescr_variant` :

```json
{
  "fileName": "manifest",
  "layer": "CUSTOMER_BASE",
  "fileType": "appdescr_variant",
  "reference": "ui.ssuite.s2p.mm.pur.pr.prcss.s1",
  "id": "customer.zprsourcingnav",
  "namespace": "apps/ui.ssuite.s2p.mm.pur.pr.prcss.s1/appVariants/customer.zprsourcingnav/",
  "version": "0.1.0",
  "content": [ ... ]
}
```

`i18n.properties` — ajouter les libellés utilisés par le fragment :

```properties
BTN_SOURCE_Z=Source appro (Z)
MSG_NO_SELECTION=Aucune ligne sélectionnée : ouverture sans filtre
MSG_NO_FLP=Navigation disponible uniquement dans le SAP Fiori launchpad
```

---

## Étape 5 – Ouvrir l'éditeur d'adaptation

1. Ouvrir la page **Application Information** : palette → `Fiori: Open Application Info`.
2. Cliquer sur **Open Adaptation Editor** (ou palette → `Fiori: Open Adaptation Editor`). Un onglet navigateur s'ouvre avec l'application F1048 chargée depuis DEV.
3. Repères de l'éditeur :

```text
┌────────────────────────────────────────────────────────────────────────────┐
│ [UI Adaptation | Navigation]   Safe Mode [●━━]   ↶ ↷              [Save]   │
├────────────────────────────────────────────────────────────────────────────┤
│                 F1048 chargée (List Report, tableau vide)                  │
│  Clic droit → Rename / Remove / Add: Fragment / Extend With Controller     │
└────────────────────────────────────────────────────────────────────────────┘
```

> Le **Safe Mode** doit être **désactivé** pour ajouter un fragment et une extension de contrôleur.

📸 *Capture 5-06 : éditeur d'adaptation dans VS Code / navigateur*

---

## Étape 6 – Ajouter le bouton (fragment)

1. Clic droit sur la **barre d'outils du tableau** → **Add: Fragment**.

| Champ | Valeur |
|---|---|
| Target Aggregation | `content` |
| Index | position souhaitée (ex. `1`) |
| Fragment Name | `ZBtnSourceAppro` |

2. **Create** → `webapp/changes/fragments/ZBtnSourceAppro.fragment.xml` est généré. Remplacer son contenu :

```xml
<core:FragmentDefinition xmlns:core="sap.ui.core" xmlns="sap.m">
    <Button id="btnZSourceAppro"
            text="{i18n>BTN_SOURCE_Z}"
            icon="sap-icon://supplier"
            type="Transparent"
            tooltip="Ouvrir l'application de détermination de la source d'approvisionnement"
            press=".extension.customer.zprsourcingnav.ZListReportExt.onNavToSourcing"/>
</core:FragmentDefinition>
```

3. Enregistrer, puis **Save** dans l'éditeur : le bouton apparaît immédiatement, tableau vide compris.

> Si la clé i18n s'affiche brute, vérifier que `i18n.properties` de la variante contient bien `BTN_SOURCE_Z` ; pour un test rapide, remplacer par un texte en dur.

📸 *Capture 5-07 : boîte « Add Fragment »*
📸 *Capture 5-08 : bouton visible sans données*

---

## Étape 7 – Extension de contrôleur et navigation croisée

1. Clic droit dans la vue → **Extend With Controller** → nom `ZListReportExt` → **Create**.
2. Ouvrir `webapp/changes/coding/ZListReportExt.js` et **vérifier le nom** passé à `ControllerExtension.extend(...)` : il doit être identique à celui utilisé dans l'attribut `press` du fragment.
3. Remplacer le contenu :

```javascript
/**
 * Variante Z de F1048 – navigation vers l'application
 * « Détermination source appro » (intent ZSourceOfSupply-determine).
 */
sap.ui.define([
    "sap/ui/core/mvc/ControllerExtension",
    "sap/m/MessageToast",
    "sap/m/MessageBox"
], function (ControllerExtension, MessageToast, MessageBox) {
    "use strict";

    return ControllerExtension.extend("customer.zprsourcingnav.ZListReportExt", {

        override: {
            onInit: function () {
                jQuery.sap.log.info("Variante Z F1048 : extension chargée");
            }
        },

        /**
         * Handler du bouton « Source appro (Z) ».
         * Transmet la demande d'achat sélectionnée en paramètre d'URL.
         */
        onNavToSourcing: function () {
            var oView   = this.base.getView();
            var oParams = {};
            var aRows   = this._getSelectedObjects(oView);

            if (aRows.length) {
                // Plusieurs lignes possibles : le paramètre accepte un tableau de valeurs
                oParams.PurchaseRequisition = aRows.map(function (oRow) {
                    return oRow.PurchaseRequisition;
                });
            } else {
                MessageToast.show("Aucune ligne sélectionnée : ouverture sans filtre");
            }

            if (!(sap.ushell && sap.ushell.Container)) {
                MessageBox.information(
                    "Navigation disponible uniquement dans le SAP Fiori launchpad.\n" +
                    "Cible : #ZSourceOfSupply-determine");
                return;
            }

            sap.ushell.Container.getServiceAsync("CrossApplicationNavigation")
                .then(function (oCrossAppNav) {
                    oCrossAppNav.toExternal({
                        target: {
                            semanticObject: "ZSourceOfSupply",
                            action: "determine"
                        },
                        params: oParams
                    });
                })
                .catch(function (oError) {
                    MessageBox.error("Navigation impossible : " + oError);
                });
        },

        /**
         * Récupère les lignes sélectionnées du tableau,
         * quelle que soit la variante de tableau utilisée par l'application.
         */
        _getSelectedObjects: function (oView) {
            var aSmartTables = oView.findAggregatedObjects(true, function (oControl) {
                return oControl.isA("sap.ui.comp.smarttable.SmartTable");
            });
            if (!aSmartTables.length) {
                return [];
            }
            var oInner = aSmartTables[0].getTable();
            var aContexts = [];

            if (oInner && typeof oInner.getSelectedContexts === "function") {
                aContexts = oInner.getSelectedContexts();                       // sap.m.Table
            } else if (oInner && typeof oInner.getSelectedIndices === "function") {
                aContexts = oInner.getSelectedIndices().map(function (iIndex) { // sap.ui.table.*
                    return oInner.getContextByIndex(iIndex);
                });
            }

            return aContexts.filter(Boolean).map(function (oCtx) {
                return oCtx.getObject();
            });
        }
    });
});
```

4. Enregistrer, recharger l'éditeur, passer en mode **Navigation** et cliquer sur le bouton : hors Launchpad, la boîte d'information s'affiche (comportement attendu en preview local).

> **Variante « navigation depuis une ligne »** : pour un lien cliquable dans une colonne plutôt qu'un bouton de barre d'outils, utiliser `@Consumption.semanticObject: 'ZSourceOfSupply'` côté service standard — impossible ici (service SAP), d'où le choix du bouton.
>
> **Récupération du paramètre côté application Z** : le FLP transmet `PurchaseRequisition` dans l'URL ; SAP Fiori elements l'applique automatiquement comme filtre si la zone porte le même nom dans le service Z — c'est le cas (`PurchaseRequisition`). Sinon, ajouter un `SelectionVariant` de navigation.

📸 *Capture 5-09 : `ZListReportExt.js`*
📸 *Capture 5-10 : message en preview local*

---

## Étape 8 – Preview local

```bash
cd zprsourcingnav
npm install
npm start
```

| Contrôle | Attendu |
|---|---|
| Titre | `Traiter les demandes d'achat – Variante Z` |
| Barre d'outils du tableau | Bouton **Source appro (Z)** présent, tableau vide |
| Clic sans sélection (hors FLP) | Boîte d'information avec l'intent cible |
| Console `F12` | Message `Variante Z F1048 : extension chargée`, aucune erreur |
| Après *Exécuter* + sélection | Aucun blocage, le handler récupère les lignes (à vérifier au débogueur) |

> La navigation croisée réelle ne peut être testée que dans le Launchpad (étape 11) : le sandbox local ne connaît pas l'intent `ZSourceOfSupply-determine`.

📸 *Capture 5-11 : preview local*

---

## Étape 9 – Déploiement sur DEV

### Option A – Ligne de commande

```bash
npm run deploy
```

`ui5-deploy.yaml` (extrait) :

```yaml
specVersion: "3.0"
metadata:
  name: customer.zprsourcingnav
type: application
builder:
  customTasks:
    - name: deploy-to-abap
      afterTask: generateCachebusterInfo
      configuration:
        target:
          url: https://<host>:<port>
          client: "100"
        app:
          name: ZPR_SRC_NAV
          description: Variante Z F1048 navigation sourcing
          package: ZMM_UI_EXT
          transport: S4DK9xxxxx
        exclude:
          - /test/
```

### Option B – Assistant

Clic droit sur `manifest.appdescr_variant` → **Adaptation Project → Deploy** (selon la version des Fiori tools) et renseigner les mêmes valeurs.

### Contrôles backend

| Contrôle | Transaction | Attendu |
|---|---|---|
| Application BSP | `SE80` → *Application BSP* `ZPR_SRC_NAV` | `manifest.appdescr_variant`, `changes/…` |
| Ordre | `SE09` → `S4DK9xxxxx` | `R3TR WAPA ZPR_SRC_NAV` |
| Index | `/UI5/APP_INDEX_CALCULATE` | Exécution sans erreur |

📸 *Capture 5-12 : log de déploiement*

---

## Étape 10 – Configuration du Launchpad DEV

La variante est une application distincte : elle a besoin de son propre target mapping et de sa tuile.

### 10.1 Catalogue technique

`/UI2/FLPAM` (ou `/UI2/FLPD_CUST`) :

| Champ | Valeur |
|---|---|
| Catalogue technique | `Z_TC_MM_SOURCING` (réutilisé du document 04) |
| Application Type | `SAPUI5 Fiori App` |
| Semantic Object | `PurchaseRequisition` |
| Action | `processZ` |
| **ID (composant SAPUI5)** | **`customer.zprsourcingnav`** |
| URL | laissée **vide** (règle des app variants) |
| Fiori ID | `F1048_ZEXT` |
| Device Types | Desktop, Tablet |

Tuile *App Launcher – Static* :

| Champ | Valeur |
|---|---|
| Title | `Traiter les demandes d'achat` |
| Subtitle | `Variante Z` |
| Icon | `sap-icon://sales-order` |
| Semantic Object / Action | `PurchaseRequisition` / `processZ` |

### 10.2 Catalogue métier, espace et rôle

1. Catalogue métier `Z_BC_MM_SOURCING` : référencer la tuile et le target mapping.
2. Espace / page acheteurs : ajouter la tuile (apps *Gérer les espaces / les pages du launchpad*).
3. `PFCG` → rôle `Z_BR_SOURCING` (ou rôle acheteur dérivé) :
   - catalogue `Z_BC_MM_SOURCING` (variante F1048 **et** application Z du document 04),
   - autorisations `S_SERVICE` pour `MM_PUR_PR_PROCESS_SRV` et `ZUI_PR_SOURCING_O2`,
   - autorisations DA (`M_BANF_*`).
4. `SU01` : affecter le rôle au testeur.

📸 *Capture 5-13 : target mapping de la variante*

---

## Étape 11 – Tests sur DEV

| # | Test | Attendu |
|---|---|---|
| T1 | Ouvrir `#PurchaseRequisition-processZ` | Variante affichée, titre `– Variante Z` |
| T2 | Sans données | Bouton **Source appro (Z)** visible |
| T3 | Clic sans sélection | Message « ouverture sans filtre » puis ouverture de l'app Z |
| T4 | *Exécuter*, sélectionner 1 ligne, clic | App Z ouverte et filtrée sur la DA |
| T5 | Sélectionner 2 lignes, clic | App Z ouverte avec les deux valeurs dans le filtre |
| T6 | Bouton *Retour* du navigateur | Retour à la variante, filtres conservés |
| T7 | Ouvrir l'application standard F1048 | Inchangée, sans le bouton Z |
| T8 | Utilisateur sans rôle `Z_BR_SOURCING` | La tuile de la variante n'apparaît pas |

Outils : `Ctrl+Alt+Shift+P` (infos techniques, ID de la variante), `?sap-ui-debug=true`, onglet *Réseau* de `F12` pour voir la requête `lrep/flex/data`.

📸 *Capture 5-14 : variante dans le FLP*
📸 *Capture 5-15 : navigation aboutie vers l'app Z*

---

## Étape 12 – Transport DEV → QUALITÉ

### 12.1 Inventaire des ordres

| Ordre | Type | Contenu | Origine |
|---|---|---|---|
| OT-1 | Workbench | `DEVC ZMM_SOURCING`, `DDLS`/`DCLS`/`DDLX` du modèle, `SRVD ZUI_PR_SOURCING`, `SRVB ZUI_PR_SOURCING_O2`, `WAPA ZMM_PR_SRC` | Document 04 |
| OT-2 | Workbench | `WAPA ZPR_SRC_NAV` (variante F1048) | Ce document |
| OT-3 | Customizing | Objet sémantique `ZSourceOfSupply` (`/UI2/SEMOBJ`), contenu FLP client-spécifique (catalogues, tuiles, target mappings, espaces/pages) | Documents 04 et 05 |
| OT-4 | Workbench | Rôles PFCG (si vos rôles sont transportés) | `PFCG` → *Rôle → Transport* |

### 12.2 Ordre d'import

```text
1. OT-1  (service Z + application Z)       ──▶ sinon la navigation n'a pas de cible
2. OT-2  (variante de F1048)
3. OT-3  (objet sémantique + contenu FLP)
4. OT-4  (rôles)
```

### 12.3 Procédure

1. `SE09` sur DEV : libérer les **tâches** puis les **ordres** (contrôler l'onglet *Objets* avant libération).
2. `STMS` → file d'attente d'import de QAS → importer les ordres dans l'ordre ci-dessus (ou transmettre la demande à l'équipe Basis / au fournisseur PCE).
3. Contrôler les logs d'import : code retour 0 ou 4 acceptable (avertissements de génération), **8 ou 12 à analyser**.

### 12.4 Actions post-import sur QUALITÉ

| # | Action | Transaction |
|---|---|---|
| 1 | Vérifier le service OData Z (alias `LOCAL`, nœud ICF vert) ; si absent, l'ajouter ou republier le service binding | `/IWFND/MAINT_SERVICES` |
| 2 | Recalculer l'index des applications (`ZMM_PR_SRC`, `ZPR_SRC_NAV`) | `/UI5/APP_INDEX_CALCULATE` |
| 3 | Purger les caches Gateway | `/IWFND/CACHE_CLEANUP`, `/IWBEP/CACHE_CLEANUP` |
| 4 | Purger les caches FLP | `/UI2/INVALIDATE_GLOBAL_CACHES`, `/UI2/INVALIDATE_CLIENT_CACHES` |
| 5 | Vérifier l'objet sémantique `ZSourceOfSupply` | `/UI2/SEMOBJ` |
| 6 | Vérifier target mappings et tuiles | `/UI2/FLPAM` ou `/UI2/FLPD_CUST` |
| 7 | Vérifier les rôles et les affecter aux testeurs | `PFCG`, `SU01` |
| 8 | Contrôler l'existence de DA de type `ZCTL` et d'entrées `EORD` dans QAS | `ME53N`, `ME03` |

> **Point de vigilance** : le paramétrage métier (type de document `ZCTL`, liste des sources) est souvent absent de QAS au premier import. Le prévoir avec l'équipe fonctionnelle, sinon l'application Z s'affichera vide et la recette conclura à tort à un défaut technique.

---

## Étape 13 – Recette sur QUALITÉ

| # | Cas | Résultat attendu | OK/KO |
|---|---|---|---|
| R1 | Tuile *Traiter les demandes d'achat – Variante Z* visible pour le rôle testeur | Affichée | |
| R2 | Tuile *Détermination source appro* visible | Affichée | |
| R3 | Variante F1048 : bouton Z présent sans données | Visible | |
| R4 | Navigation avec une DA sélectionnée | App Z filtrée sur la DA | |
| R5 | App Z : liste des DA `ZCTL` | Données cohérentes avec `ME53N` | |
| R6 | App Z : annotations (statuts colorés, progression, onglets, filtres F4) | Conformes au document 04 | |
| R7 | App Z : pop-up des sources candidates | Sources cohérentes avec `ME03` | |
| R8 | Application standard F1048 inchangée | Aucun bouton Z | |
| R9 | Utilisateur sans autorisation division | Aucune donnée, pas de dump (contrôle DCL) | |
| R10 | Performances de la liste (volume réel QAS) | Temps de réponse acceptable | |

Journaux à consulter en cas d'anomalie : `/IWFND/ERROR_LOG`, `/IWBEP/ERROR_LOG`, `ST22`, `SLG1`, console navigateur.

---

## Dépannage

| Symptôme | Cause probable | Solution |
|---|---|---|
| L'application n'apparaît pas dans le générateur | Index des apps non calculé, droits, URL/mandant erronés | `/UI5/APP_INDEX_CALCULATE`, revérifier `Fiori: Add SAP System` |
| Message « application non supportée » | Application non `flexEnabled` ou UI5 < 1.72 | Vérifier le manifest ; envisager une extension par annotation/BAdI |
| Erreur de certificat au lancement | Certificat auto-signé | Importer le certificat, ou `FIORI_TOOLS_DISABLE_CERT_VALIDATION`/`ignoreCertErrors` en développement uniquement |
| Bouton sans effet | Nom d'extension différent entre `press` et `ControllerExtension.extend` | Aligner les deux, vérifier la console |
| `sap.ushell is undefined` | Exécution hors Launchpad | Comportement prévu : la boîte d'information s'affiche (code de l'étape 7) |
| Navigation ouvre « Application introuvable » | Intent cible absent, rôle non affecté, cache | Vérifier `/UI2/SEMOBJ`, le target mapping de l'app Z, le rôle, purger les caches |
| App Z ouverte mais non filtrée | Nom de paramètre ≠ nom de zone du service Z | Utiliser exactement `PurchaseRequisition`, ou définir un `SelectionVariant` de navigation |
| Déploiement : *ZIP archive contains disallowed files* | Correctif manquant | Note SAP 3073188 |
| Déploiement : *Transport check could not be performed* | Correctif manquant | Note SAP 3243791 |
| Variante OK sur DEV, KO sur QAS | Caches, index, ordre d'import, contenu FLP non transporté | Dérouler §12.4 dans l'ordre |
| Après upgrade : bouton disparu ou erreur JS | Structure de la vue standard modifiée | Rouvrir le projet, revalider le fragment et l'extension, redéployer |

---

## Captures à réaliser

| N° | Écran | Fichier suggéré |
|---|---|---|
| 5-01 | Fiche F1048 | `img/05-01-fiche-f1048.png` |
| 5-02 | `manifest.json` `flexEnabled` | `img/05-02-manifest.png` |
| 5-03 | Environment Check VS Code | `img/05-03-envcheck.png` |
| 5-04 | Générateur — application | `img/05-04-gen-app.png` |
| 5-05 | Générateur — attributs | `img/05-05-gen-attr.png` |
| 5-06 | Éditeur d'adaptation | `img/05-06-editor.png` |
| 5-07 | Add Fragment | `img/05-07-add-fragment.png` |
| 5-08 | Bouton sans données | `img/05-08-button.png` |
| 5-09 | Extension de contrôleur | `img/05-09-controller.png` |
| 5-10 | Message en preview local | `img/05-10-preview-msg.png` |
| 5-11 | Preview local | `img/05-11-preview.png` |
| 5-12 | Log de déploiement | `img/05-12-deploy.png` |
| 5-13 | Target mapping variante | `img/05-13-targetmapping.png` |
| 5-14 | Variante dans le FLP | `img/05-14-flp.png` |
| 5-15 | Navigation vers l'app Z | `img/05-15-navigation.png` |

---

## Références

- SAP Community – *Introducing SAPUI5 Adaptation Projects in Visual Studio Code* (version UI5 minimale, commandes)
- SAP Help – *Deploy the Adaptation Project to the ABAP Repository*
- SAP Help – *Configuring Target Mappings*, chapitre « Settings for SAPUI5 App Variants » (URL vide, ID = variante)
- SAPUI5 Demo Kit – *Cross Application Navigation* (`toExternal`, `getServiceAsync`)
- SAP Fiori Apps Reference Library – F1048 / F1048A
- Notes SAP : 3073188, 3243791 ; KBA 3074554
- Documents internes : `01_Extension_UI_Adaptation_Project_F0842A.md`, `04_App_RAP_Determination_Source_Appro.md`
