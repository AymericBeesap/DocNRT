# Extension UI d'une application SAP Fiori standard via Adaptation Project

**Contexte :** SAP S/4HANA Cloud Private Edition 2021 FPS02 (ABAP Platform 2021, SAP_UI 7.56, SAPUI5 1.96.x)
**Application étendue :** F0842A — *Gérer les commandes d'achat* (*Manage Purchase Orders*)
**Outil :** SAP Business Application Studio (BAS) + SAP Fiori tools (alternative VS Code en §2.6)
**Résultat :** une **variante d'application** (app variant) déployée dans le SAPUI5 ABAP Repository, avec un bouton personnalisé et des libellés modifiés, **visibles sans aucune donnée et en preview local**.

---

## Sommaire

1. [Objectif et résultat attendu](#1-objectif-et-résultat-attendu)
2. [Prérequis](#2-prérequis)
3. [Étape 1 – Vérifier l'application de référence](#étape-1--vérifier-lapplication-de-référence)
4. [Étape 2 – Préparer le Dev Space BAS](#étape-2--préparer-le-dev-space-bas)
5. [Étape 3 – Générer l'Adaptation Project](#étape-3--générer-ladaptation-project)
6. [Étape 4 – Comprendre la structure du projet](#étape-4--comprendre-la-structure-du-projet)
7. [Étape 5 – Ouvrir l'éditeur d'adaptation](#étape-5--ouvrir-léditeur-dadaptation)
8. [Étape 6 – Adaptations simples (Safe Mode actif)](#étape-6--adaptations-simples-safe-mode-actif)
9. [Étape 7 – Ajouter un bouton via fragment XML](#étape-7--ajouter-un-bouton-via-fragment-xml)
10. [Étape 8 – Ajouter une extension de contrôleur](#étape-8--ajouter-une-extension-de-contrôleur)
11. [Étape 9 – Preview local (sans données)](#étape-9--preview-local-sans-données)
12. [Étape 10 – Déployer dans l'ABAP Repository](#étape-10--déployer-dans-labap-repository)
13. [Étape 11 – Vérifications côté backend](#étape-11--vérifications-côté-backend)
14. [Étape 12 – Configuration du Fiori Launchpad](#étape-12--configuration-du-fiori-launchpad)
15. [Étape 13 – Test dans le Launchpad](#étape-13--test-dans-le-launchpad)
16. [Cycle de vie et transport](#cycle-de-vie-et-transport)
17. [Dépannage](#dépannage)
18. [Liste des captures d'écran à réaliser](#liste-des-captures-décran-à-réaliser)
19. [Références](#références)

---

## 1. Objectif et résultat attendu

### Pourquoi F0842A ?

| Critère | F0842A – Gérer les commandes d'achat |
|---|---|
| Technologie | SAP Fiori elements **V2** (List Report + Object Page, draft) |
| ID de composant SAPUI5 | `ui.ssuite.s2p.mm.pur.po.manage.st.s1` |
| Nom technique BSP | `MM_PO_MANAGES1` |
| Service OData | `MM_PUR_PO_MAINT_V2_SRV` (entité principale `C_PurchaseOrderTP`) |
| Intent (à confirmer dans la fiche) | `PurchaseOrder-manage` |
| Rôle métier modèle | `SAP_BR_PURCHASER` (Acheteur) |
| Compatible Adaptation Project | Oui (app Fiori elements « flex enabled ») |

C'est l'une des applications standard les plus utilisées comme base d'adaptation project : la page *List Report* affiche la barre de filtres et la barre d'outils du tableau **même quand aucune commande n'est chargée**, ce qui permet de voir l'extension immédiatement en preview.

### Ce que l'on va réaliser

| # | Adaptation | Type | Mode Safe | Visible sans données |
|---|---|---|---|---|
| A | Renommer le bouton standard « Créer » en « Nouvelle commande » | Changement UI (rename) | Oui | Oui |
| B | Ajouter un bouton « Contrôle Z » dans la barre d'outils du tableau | Fragment XML | Non | Oui |
| C | Afficher une boîte de message au clic (nb. de lignes sélectionnées) | Controller extension | Non | Oui (affiche 0) |
| D | Changer le titre de l'application (variante) | Descriptor change (i18n) | — | Oui |

### Écran attendu (maquette)

```text
┌──────────────────────────────────────────────────────────────────────────────────┐
│ ◀  SAP   Gérer les commandes d'achat – Variante Z                         🔍  👤 │
├──────────────────────────────────────────────────────────────────────────────────┤
│ Standard ▾                                                                       │
│ Rechercher [          ]  Fournisseur [       ]  Groupe d'achat [      ]  ...    │
│                                              [Exécuter]  [Adapter les filtres]   │
├──────────────────────────────────────────────────────────────────────────────────┤
│ Commandes d'achat           ┏━━━━━━━━━━━━┓ [Nouvelle commande] [...]   ⚙  ⤓     │
│                             ┃ Contrôle Z ┃  ◀── ajouté (B)   ◀── renommé (A)   │
│                             ┗━━━━━━━━━━━━┛                                       │
│ ──────────────────────────────────────────────────────────────────────────────── │
│ Commande d'achat │ Fournisseur │ Groupe d'achat │ Valeur nette │ Statut          │
│                                                                                  │
│          Pour commencer, définissez les filtres et choisissez « Exécuter ».      │
└──────────────────────────────────────────────────────────────────────────────────┘
```

> Les boutons standard exacts de la barre d'outils dépendent du niveau de support package. Le principe reste identique.

---

## 2. Prérequis

### 2.1 Système backend

| Élément | Valeur attendue / Action | Transaction |
|---|---|---|
| Version | S/4HANA 2021 FPS02, SAP_UI 7.56 | `SPAM` / *Système → Statut → Composants* |
| Version SAPUI5 | 1.96.x (minimum requis pour adaptation project : 1.71) | FLP → *À propos* ou `Ctrl+Alt+Shift+P` |
| Service OData de l'app | `MM_PUR_PO_MAINT_V2_SRV` actif | `/IWFND/MAINT_SERVICES` |
| Service de déploiement | `/UI5/ABAP_REPOSITORY_SRV` actif | `/IWFND/MAINT_SERVICES` |
| Index des applications | Calculé et à jour | `/UI5/APP_INDEX_CALCULATE` (programme du même nom via `SA38`) |
| Déploiement Gateway | Embedded (standard en Private Cloud) | `/IWFND/MAINT_SERVICES` → alias système |

**Nœuds SICF à activer** (`SICF`, hôte `default_host`) :

| Chemin ICF | Rôle |
|---|---|
| `/sap/bc/ui5_ui5` | SAPUI5 ABAP Repository (applications BSP) |
| `/sap/bc/lrep` | Layered Repository (stockage des changements flex) |
| `/sap/bc/ui2/app_index` | Index des applications (liste des apps dans le générateur) |
| `/sap/bc/ui2/flp` | Fiori Launchpad |
| `/sap/bc/adt` | ABAP Development Tools (lecture packages / OT par BAS) |
| `/sap/opu/odata/ui5/abap_repository_srv` | Déploiement |
| `/sap/opu/odata/sap/mm_pur_po_maint_v2_srv` | Service de l'application |
| `/sap/public/bc/ui5_ui5`, `/sap/public/bc/ui2` | Ressources SAPUI5 et thèmes |

### 2.2 Objets ABAP de support

| Objet | Valeur exemple | Transaction |
|---|---|---|
| Package | `ZMM_UI_EXT` (composant applicatif `MM-PUR-PO`, couche de transport du projet) | `SE21` ou `SE80` |
| Ordre de transport | Ordre Workbench `S4DK9xxxxx` | `SE09` / `SE10` |

> Pour un premier test sans transport, vous pouvez déployer dans `$TMP` (objet local, non transportable).

### 2.3 Autorisations

| Utilisateur | Rôle / Objets d'autorisation | Pourquoi |
|---|---|---|
| Développeur (utilisateur technique de la destination ou utilisateur nominatif) | `S_DEVELOP` (DEVCLASS `ZMM_UI_EXT` ou `$TMP`, OBJTYPE `WAPA`, ACTVT 01/02/03/06/07) | Création/écrasement de l'application BSP |
| | `S_TRANSPRT` (TTYPE `DTRA`, `TASK`, ACTVT 01/02/03) | Enregistrement dans l'OT |
| | `S_SERVICE` pour `/UI5/ABAP_REPOSITORY_SRV` et `MM_PUR_PO_MAINT_V2_SRV` | Démarrage des services OData |
| | `S_ADT_RES` | Accès aux ressources ADT utilisées par BAS |
| | Rôle dérivé de `SAP_BR_PURCHASER` (au moins lecture : `M_BEST_BSA`, `M_BEST_EKG`, `M_BEST_EKO`, `M_BEST_WRK`) | Le preview exécute réellement l'app sur le backend |
| Administrateur FLP | `SAP_UI2_ADMIN_700` (ou équivalent client) | Catalogues, tuiles, target mappings |
| Utilisateur final | Rôle `Z_BR_PURCHASER_EXT` (créé en Étape 12) | Accès à la tuile de la variante |

Transactions : `PFCG` (rôles), `SU01` (affectation), `SU53` / `STAUTHTRACE` (analyse d'autorisation).

### 2.4 Connectivité BAS ↔ S/4HANA : SAP Cloud Connector

Dans l'administration du Cloud Connector (`https://<cc-host>:8443`) → *Cloud To On-Premise* → *Mappage d'accès* :

| Champ | Valeur exemple |
|---|---|
| Back-end Type | ABAP System |
| Protocol | HTTPS |
| Internal Host / Port | `s4hdev.corp.local` / `44300` |
| Virtual Host / Port | `s4h-dev` / `44300` |
| Principal Type | None (Basic Auth) ou X.509 (Principal Propagation) |
| Host In Request Header | Use Virtual Host |

**Ressources** (onglet *Resources*) : pour un système de développement, le plus simple est `/sap/` avec *Path and all sub-paths*. Sinon, exposer au minimum : `/sap/opu/odata`, `/sap/bc/ui5_ui5`, `/sap/bc/lrep`, `/sap/bc/ui2`, `/sap/bc/adt`, `/sap/public/bc`.

> Symptôme typique d'une ressource manquante : la liste des applications est vide dans le générateur.

### 2.5 Destination SAP BTP

SAP BTP Cockpit → sous-compte → *Connectivity* → *Destinations* → *New Destination* :

| Propriété | Valeur |
|---|---|
| Name | `S4H_DEV_100` |
| Type | `HTTP` |
| URL | `http://s4h-dev:44300` (hôte virtuel du Cloud Connector) |
| Proxy Type | `OnPremise` |
| Authentication | `BasicAuthentication` (User / Password) ou `PrincipalPropagation` |
| Location ID | À renseigner si le Cloud Connector en utilise un |

**Propriétés additionnelles :**

| Clé | Valeur |
|---|---|
| `sap-client` | `100` |
| `WebIDEEnabled` | `true` |
| `WebIDEUsage` | `odata_abap,dev_abap,ui5_execute_abap` |
| `HTML5.DynamicDestination` | `true` |
| `HTML5.Timeout` | `60000` |

Cliquer sur **Check Connection** : le résultat attendu est un code HTTP 200 ou 401 (401 = joignable mais authentification requise, ce qui est normal sans identifiants).

### 2.6 Alternative : VS Code

SAP Fiori tools pour VS Code propose aussi le générateur d'adaptation project pour les systèmes ABAP on-premise (commande `Fiori: Open Adaptation Project Generator`). La connexion se fait alors directement par URL système (pas de Cloud Connector). SAP positionne toutefois BAS comme outil stratégique pour ce scénario. La suite du document décrit BAS ; les écrans sont très proches dans VS Code.

---

## Étape 1 – Vérifier l'application de référence

1. Ouvrir la **SAP Fiori Apps Reference Library** → rechercher `F0842A` → filtrer sur *SAP S/4HANA* version *2021 FPS02*.
2. Onglet **Implementation Information** :
   - section *Configuration* : noter le service OData, le nom BSP (`MM_PO_MANAGES1`), le composant SAPUI5, l'intent (semantic object / action), les catalogues techniques et métiers ;
   - section *Extensibility* : vérifier la mention des possibilités d'extension UI.
3. Contrôler le manifest dans le navigateur :

   ```text
   https://<host>:<port>/sap/bc/ui5_ui5/sap/mm_po_manages1/manifest.json?sap-client=100
   ```

   Vérifier :
   - `"sap.app" → "id": "ui.ssuite.s2p.mm.pur.po.manage.st.s1"`
   - `"sap.ui5" → "flexEnabled": true` (condition pour les adaptations)
   - `"sap.ui.generic.app" → "pages"` (confirme Fiori elements V2)

4. Lancer l'application standard depuis le FLP avec un utilisateur acheteur pour confirmer qu'elle fonctionne **avant** toute adaptation.

📸 *Capture 1-01 : fiche F0842A – Implementation Information*
📸 *Capture 1-02 : manifest.json avec `flexEnabled`*

---

## Étape 2 – Préparer le Dev Space BAS

1. SAP BTP Cockpit → *Instances and Subscriptions* → **SAP Business Application Studio** → *Go to Application*.
2. **Create Dev Space** :

   | Champ | Valeur |
   |---|---|
   | Dev Space Name | `FioriAdaptation` |
   | Type | **SAP Fiori** |
   | Additional SAP Extensions | laisser les valeurs par défaut (SAP Fiori tools, Adaptation Project inclus) |

3. Cliquer sur **Create Dev Space**, attendre le statut *RUNNING*, puis ouvrir le Dev Space.

📸 *Capture 1-03 : écran « Create Dev Space » type SAP Fiori*

---

## Étape 3 – Générer l'Adaptation Project

1. Dans BAS : **File → New Project from Template** (ou *Template Wizard* via la palette de commandes `F1`).
2. Choisir la tuile **Adaptation Project** → *Start*.
3. Renseigner les écrans du générateur (les libellés peuvent légèrement varier selon la version de SAP Fiori tools) :

**Écran « Target environment / System »**

| Champ | Valeur |
|---|---|
| Target environment | `ABAP` |
| System | `S4H_DEV_100` (destination) |
| Username / Password | Si la destination n'embarque pas d'identifiants |
| Application | **Manage Purchase Orders** (`ui.ssuite.s2p.mm.pur.po.manage.st.s1`) |

> Si l'application n'apparaît pas ou est signalée « non supportée » : voir [Dépannage](#dépannage).

**Écran « Project Attributes »**

| Champ | Valeur | Remarque |
|---|---|---|
| Project Name | `zmmpoext` | Minuscules, sans caractères spéciaux |
| Application Title | `Gérer les commandes d'achat – Variante Z` | Titre de la variante |
| Namespace | `customer.zmmpoext` | Généré automatiquement ; devient l'**ID de la variante** |
| Target Folder | `/home/user/projects` | |
| SAPUI5 Version | Version du système (1.96.x) | Ne pas forcer une version supérieure |
| Enable TypeScript | Non | |

**Écran « Deployment Configuration »** (si proposé ; sinon il sera fait en Étape 10)

| Champ | Valeur |
|---|---|
| SAPUI5 ABAP Repository | `ZMM_PO_EXT` (≤ 15 caractères, commence par Z ou Y) |
| Deployment Description | `Variante Z Gérer commandes d'achat` |
| Package | `ZMM_UI_EXT` |
| Transport Request | `S4DK9xxxxx` |

4. Cliquer sur **Finish**. BAS génère le projet et installe les dépendances npm.

📸 *Capture 1-04 : sélection du système et de l'application*
📸 *Capture 1-05 : Project Attributes renseignés*

---

## Étape 4 – Comprendre la structure du projet

```text
zmmpoext/
├── package.json                  scripts npm (start, start-editor, build, deploy)
├── ui5.yaml                      configuration du preview (middleware fiori-tools)
├── ui5-deploy.yaml               configuration du déploiement ABAP
└── webapp/
    ├── manifest.appdescr_variant descripteur de la variante (≠ manifest.json)
    ├── i18n/
    │   └── i18n.properties       textes de la variante
    └── changes/                  (créé au premier changement)
        ├── *.change              changements flex (rename, addXML, ...)
        ├── fragments/            fragments XML ajoutés
        └── coding/               extensions de contrôleur (.js)
```

Exemple de `manifest.appdescr_variant` généré :

```json
{
  "fileName": "manifest",
  "layer": "CUSTOMER_BASE",
  "fileType": "appdescr_variant",
  "reference": "ui.ssuite.s2p.mm.pur.po.manage.st.s1",
  "id": "customer.zmmpoext",
  "namespace": "apps/ui.ssuite.s2p.mm.pur.po.manage.st.s1/appVariants/customer.zmmpoext/",
  "version": "0.1.0",
  "content": [
    {
      "changeType": "appdescr_ui5_addNewModelEnhanceWith",
      "content": {
        "modelId": "i18n",
        "bundleUrl": "i18n/i18n.properties",
        "supportedLocales": [""],
        "fallbackLocale": ""
      }
    },
    {
      "changeType": "appdescr_app_setTitle",
      "content": {},
      "texts": { "i18n": "i18n/i18n.properties" }
    }
  ]
}
```

Points clés :
- `reference` = ID de l'app standard (jamais modifié) ;
- `id` = ID de la variante, réutilisé dans la configuration FLP ;
- `layer` = `CUSTOMER_BASE` (couche développeur).

**Adaptation D – titre de la variante** : ouvrir `webapp/i18n/i18n.properties` et vérifier/modifier la clé générée :

```properties
# Titre de la variante (clé générée par le wizard, conserver son nom)
customer.zmmpoext_sap.app.title=Gérer les commandes d'achat – Variante Z
```

> Ne pas modifier les fichiers `.change` à la main sauf besoin précis : ils sont générés par l'éditeur.

---

## Étape 5 – Ouvrir l'éditeur d'adaptation

1. Clic droit sur `webapp/manifest.appdescr_variant` → **Open Adaptation Editor** (selon version : *Open SAPUI5 Visual Editor*, ou via la page *Application Info* du projet).
   Alternative terminal : `npm run start-editor`.
2. L'éditeur charge l'application **depuis le backend** via la destination : la List Report s'affiche, généralement vide (aucune recherche lancée).
3. Barre d'outils de l'éditeur :

| Élément | Usage |
|---|---|
| **UI Adaptation** / **Navigation** | Basculer entre mode édition (clic droit sur les contrôles) et mode navigation (utiliser l'app normalement) |
| **Safe Mode** (interrupteur) | Actif = uniquement les changements « key user » stables ; Inactif = fragments, controller extensions |
| Undo / Redo | Annuler / rétablir |
| Save | Écrit les fichiers `.change` dans `webapp/changes` |

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│ [UI Adaptation | Navigation]   Safe Mode [●━━]   ↶  ↷          [ Save ]    │
├─────────────────────────────────────────────────────────────────────────────┤
│            (application F0842A chargée dans un iFrame)                      │
│   Clic droit sur un contrôle →  Rename / Remove / Add: Fragment /           │
│                                 Extend With Controller / Settings           │
└─────────────────────────────────────────────────────────────────────────────┘
```

📸 *Capture 1-06 : éditeur ouvert, List Report vide*

---

## Étape 6 – Adaptations simples (Safe Mode actif)

### A – Renommer le bouton « Créer »

1. Mode **UI Adaptation**, **Safe Mode activé**.
2. Survoler la barre d'outils du tableau → clic droit sur le bouton **Créer** (*Create*).
3. Choisir **Rename**, saisir `Nouvelle commande`, valider par `Entrée`.
4. Cliquer sur **Save**.

Résultat : un fichier `webapp/changes/id_xxxxxxxx_rename.change` est créé :

```json
{
  "changeType": "rename",
  "layer": "CUSTOMER_BASE",
  "fileType": "change",
  "reference": "ui.ssuite.s2p.mm.pur.po.manage.st.s1",
  "content": {},
  "texts": { "newText": { "value": "Nouvelle commande", "type": "XBUT" } },
  "selector": { "id": "…--addEntry", "idIsLocal": true }
}
```

### (Optionnel) Masquer un bouton standard

Clic droit sur le bouton → **Remove**. Le contrôle est masqué (pas supprimé). À n'utiliser qu'en accord avec le métier.

📸 *Capture 1-07 : menu contextuel « Rename » sur le bouton Créer*

---

## Étape 7 – Ajouter un bouton via fragment XML

1. Désactiver **Safe Mode** (confirmer l'avertissement).
2. Clic droit sur la **barre d'outils du tableau** (zone vide à côté du titre « Commandes d'achat ») → **Add: Fragment**.
3. Boîte de dialogue *Add Fragment* :

   | Champ | Valeur |
   |---|---|
   | Control type | `sap.m.OverflowToolbar` (détecté) |
   | Target Aggregation | `content` |
   | Index | Position souhaitée (ex. `2`, juste avant les boutons standard) |
   | Fragment Name | `ZBtnControle` |

4. Cliquer sur **Create**. Le fichier `webapp/changes/fragments/ZBtnControle.fragment.xml` est créé avec un contenu d'exemple.
5. Remplacer son contenu par :

```xml
<core:FragmentDefinition xmlns:core="sap.ui.core" xmlns="sap.m">
    <Button id="btnZControle"
            text="Contrôle Z"
            icon="sap-icon://inspection"
            type="Emphasized"
            tooltip="Contrôle spécifique de la variante Z"
            press=".extension.customer.zmmpoext.ZListReportExt.onPressZControle"/>
</core:FragmentDefinition>
```

6. Enregistrer (`Ctrl+S`) et **Save** dans l'éditeur. Le bouton apparaît immédiatement dans la barre d'outils, même sans données.

> L'attribut `press` référence l'extension de contrôleur créée à l'étape suivante. Le format est `.extension.<nom complet de l'extension>.<méthode>`. Tant que l'extension n'existe pas, le clic ne fait rien (erreur console).

📸 *Capture 1-08 : boîte « Add Fragment »*
📸 *Capture 1-09 : bouton « Contrôle Z » visible dans le tableau vide*

---

## Étape 8 – Ajouter une extension de contrôleur

1. Toujours Safe Mode désactivé : clic droit sur n'importe quelle zone de la List Report → **Extend With Controller**.
2. Boîte de dialogue :

   | Champ | Valeur |
   |---|---|
   | Controller Name | `ZListReportExt` |

3. **Create** → le fichier `webapp/changes/coding/ZListReportExt.js` est généré et un fichier `.change` de type `codeExt` référence la vue `sap.suite.ui.generic.template.ListReport.view.ListReport`.
4. Ouvrir `ZListReportExt.js`. **Vérifier le nom passé à `ControllerExtension.extend(...)`** : il doit correspondre exactement à celui utilisé dans l'attribut `press` du fragment (ici `customer.zmmpoext.ZListReportExt`). Adapter le fragment si le nom généré diffère.
5. Remplacer le contenu par :

```javascript
/**
 * Extension du contrôleur de la List Report F0842A – Variante Z
 * Fonctionne sans données : affiche le nombre de lignes sélectionnées (0 si tableau vide).
 */
sap.ui.define([
    "sap/ui/core/mvc/ControllerExtension",
    "sap/m/MessageBox"
], function (ControllerExtension, MessageBox) {
    "use strict";

    return ControllerExtension.extend("customer.zmmpoext.ZListReportExt", {

        override: {
            /**
             * Appelé à l'initialisation de la vue List Report
             */
            onInit: function () {
                // Point d'entrée pour initialiser un modèle JSON local, un log, etc.
                jQuery.sap.log.info("Variante Z F0842A : extension chargée");
            }
        },

        /**
         * Handler du bouton « Contrôle Z » (déclaré dans ZBtnControle.fragment.xml)
         */
        onPressZControle: function () {
            var oView = this.base.getView();
            var iSelected = this._getSelectedCount(oView);

            MessageBox.information(
                "Variante Z active.\n" +
                "Lignes sélectionnées : " + iSelected + "\n" +
                "Application de référence : ui.ssuite.s2p.mm.pur.po.manage.st.s1",
                { title: "Contrôle Z" }
            );
        },

        /**
         * Retrouve la SmartTable de la page et compte les lignes sélectionnées,
         * que le tableau interne soit un sap.m.Table (responsive) ou un sap.ui.table.Table (grid).
         */
        _getSelectedCount: function (oView) {
            var aSmartTables = oView.findAggregatedObjects(true, function (oControl) {
                return oControl.isA("sap.ui.comp.smarttable.SmartTable");
            });
            if (!aSmartTables.length) {
                return 0;
            }
            var oInner = aSmartTables[0].getTable();
            if (oInner && typeof oInner.getSelectedContexts === "function") {
                return oInner.getSelectedContexts().length;     // sap.m.Table
            }
            if (oInner && typeof oInner.getSelectedIndices === "function") {
                return oInner.getSelectedIndices().length;      // sap.ui.table.*
            }
            return 0;
        }
    });
});
```

6. Enregistrer, recharger l'éditeur (bouton *refresh* ou rouvrir), passer en mode **Navigation** et cliquer sur **Contrôle Z** : la boîte de message s'affiche avec `Lignes sélectionnées : 0`.

```text
          ┌──────────────────────────────────────────────┐
          │ ⓘ  Contrôle Z                                │
          ├──────────────────────────────────────────────┤
          │ Variante Z active.                           │
          │ Lignes sélectionnées : 0                     │
          │ Application de référence :                   │
          │ ui.ssuite.s2p.mm.pur.po.manage.st.s1         │
          │                                  [  OK  ]    │
          └──────────────────────────────────────────────┘
```

📸 *Capture 1-10 : fichier `ZListReportExt.js` dans BAS*
📸 *Capture 1-11 : MessageBox affichée en mode Navigation*

---

## Étape 9 – Preview local (sans données)

1. Clic droit sur `manifest.appdescr_variant` → **Preview Application** (ou terminal : `npm run start`).
2. Choisir le script `start` si BAS le demande. Un nouvel onglet s'ouvre sur un FLP *sandbox* local (`/test/flp.html#app-preview` selon la version des outils).
3. Vérifications **sans lancer de recherche** :

| Contrôle | Attendu |
|---|---|
| Titre de l'onglet / en-tête | `Gérer les commandes d'achat – Variante Z` |
| Barre d'outils du tableau | Bouton **Contrôle Z** présent |
| Bouton standard | Libellé **Nouvelle commande** |
| Clic sur Contrôle Z | MessageBox avec `Lignes sélectionnées : 0` |
| Console navigateur (`F12`) | Message `Variante Z F0842A : extension chargée`, aucune erreur rouge liée à `customer.zmmpoext` |

4. (Facultatif) Cliquer sur **Exécuter** pour charger de vraies commandes et sélectionner des lignes : le compteur s'incrémente.

> Le preview utilise le service OData **réel** du backend via la destination : l'utilisateur doit avoir les autorisations de lecture achats (sinon le tableau reste vide avec une erreur 403, mais le bouton reste visible).

📸 *Capture 1-12 : preview local, tableau vide, bouton visible*

---

## Étape 10 – Déployer dans l'ABAP Repository

### Option A – Assistant

1. Clic droit sur `manifest.appdescr_variant` → **Adaptation Project → Open Deployment Wizard** (selon version : *Deploy Adaptation Project* dans le Template Wizard).
2. Renseigner :

   | Champ | Valeur |
   |---|---|
   | System | `S4H_DEV_100` |
   | SAPUI5 ABAP Repository | `ZMM_PO_EXT` |
   | Description | `Variante Z Gérer commandes d'achat` |
   | Package | `ZMM_UI_EXT` |
   | Transport Request | `S4DK9xxxxx` |

3. **Finish** → suivre les logs dans le terminal (`Deployment successful`).

### Option B – Ligne de commande

`ui5-deploy.yaml` (extrait indicatif, généré par l'outil) :

```yaml
specVersion: "3.0"
metadata:
  name: customer.zmmpoext
type: application
builder:
  customTasks:
    - name: deploy-to-abap
      afterTask: generateCachebusterInfo
      configuration:
        target:
          destination: S4H_DEV_100
        app:
          name: ZMM_PO_EXT
          description: Variante Z Gerer commandes d'achat
          package: ZMM_UI_EXT
          transport: S4DK9xxxxx
        exclude:
          - /test/
```

Commande :

```bash
npm run deploy
```

📸 *Capture 1-13 : assistant de déploiement renseigné*
📸 *Capture 1-14 : terminal « Deployment successful »*

---

## Étape 11 – Vérifications côté backend

| Vérification | Transaction / URL | Attendu |
|---|---|---|
| Application BSP créée | `SE80` → *Application BSP* → `ZMM_PO_EXT` | Fichiers `manifest.appdescr_variant`, `changes/…`, `i18n/…` |
| Contenu de l'OT | `SE09` → `S4DK9xxxxx` | Objet `R3TR WAPA ZMM_PO_EXT` |
| Index des applications | `/UI5/APP_INDEX_CALCULATE` (package ou application `ZMM_PO_EXT`) | Exécution sans erreur |
| Caches FLP | `/UI2/INVALIDATE_GLOBAL_CACHES`, `/UI2/INVALIDATE_CLIENT_CACHES` | À lancer si la variante n'est pas reconnue |

---

## Étape 12 – Configuration du Fiori Launchpad

La variante est une **nouvelle application** (nouvel ID) : elle nécessite sa propre tuile et son propre target mapping. SAP recommande une **combinaison semantic object / action unique** plutôt qu'un paramètre `sap-appvar-id`.

### 12.1 Catalogue technique (Launchpad App Manager)

Transaction **`/UI2/FLPAM`** (Launchpad App Manager ; alternative dépréciée : `/UI2/FLPD_CUST`).

1. Créer le catalogue technique `Z_TC_MM_PO_EXT` (titre : *Variantes achats Z*).
2. Méthode recommandée : **copier l'app descriptor item de F0842A** depuis le catalogue technique SAP qui le contient (voir fiche de l'app), puis adapter :

| Champ | Valeur |
|---|---|
| Fiori ID | `F0842A_ZEXT` |
| Application Type | SAPUI5 Fiori App |
| Semantic Object | `PurchaseOrder` |
| Action | `manageZext` |
| **SAPUI5 Component ID** | **`customer.zmmpoext`** (ID de la variante) |
| Titre | `Gérer les commandes d'achat – Variante Z` |
| Device Types | Desktop, Tablet |

3. Tuile (*App Launcher – Static*) :

| Champ | Valeur |
|---|---|
| Title | `Commandes d'achat` |
| Subtitle | `Variante Z` |
| Icon | `sap-icon://inspection` |
| Semantic Object / Action | `PurchaseOrder` / `manageZext` |

> **Si vous utilisez le Launchpad Designer** (`/UI2/FLPD_CUST`) : target mapping de type *SAPUI5 Fiori App*, champ **URL laissé vide**, champ **ID = `customer.zmmpoext`**. Mettre l'ID de l'application standard ou remplir l'URL est la cause d'erreur la plus fréquente.

### 12.2 Catalogue métier et espace

1. Launchpad Designer client (`/UI2/FLPD_CONF`) ou Launchpad Content Manager (`/UI2/FLPCM_CONF`) : créer le catalogue métier `Z_BC_MM_PO_EXT` et y **référencer** la tuile et le target mapping de `Z_TC_MM_PO_EXT`.
2. Si vous utilisez Spaces & Pages : app *Gérer les espaces du launchpad* / *Gérer les pages du launchpad* → ajouter la tuile dans une page de l'espace acheteur.

### 12.3 Rôle PFCG

`PFCG` → rôle `Z_BR_PURCHASER_EXT` (copie ou complément du rôle acheteur) :

1. Onglet *Menu* → *Insérer* → *SAP Fiori Launchpad* → **Launchpad Catalog** → `Z_BC_MM_PO_EXT`.
2. Ajouter l'espace (Launchpad Space) ou le groupe correspondant.
3. Onglet *Autorisations* : générer, conserver les autorisations achats (`M_BEST_*`) et `S_SERVICE` pour `MM_PUR_PO_MAINT_V2_SRV`.
4. `SU01` : affecter le rôle à l'utilisateur de test.

📸 *Capture 1-15 : app descriptor item dans /UI2/FLPAM (Component ID = variante)*
📸 *Capture 1-16 : rôle PFCG avec catalogue et espace*

---

## Étape 13 – Test dans le Launchpad

1. URL :

   ```text
   https://<host>:<port>/sap/bc/ui2/flp?sap-client=100&sap-language=FR#PurchaseOrder-manageZext
   ```

2. Vérifier la tuile *Commandes d'achat – Variante Z*, puis les mêmes points qu'au §Étape 9.
3. Outils utiles :

| Raccourci / paramètre | Usage |
|---|---|
| `Ctrl+Alt+Shift+P` | Informations techniques (ID composant, version UI5, variante) |
| `Ctrl+Alt+Shift+S` | Diagnostics / Support Assistant |
| `?sap-ui-debug=true` | Sources non minifiées pour déboguer `ZListReportExt.js` |
| `F12` → *Sources* → `lrep/flex/modules` | Présence du code de l'extension dans le navigateur |

4. Vérifier que l'application **standard** F0842A fonctionne toujours, sans modification.

---

## Cycle de vie et transport

- **Transport** : l'OT contient `R3TR WAPA ZMM_PO_EXT`. Libérer via `SE09`, importer via `STMS`. La configuration FLP (catalogues, pages) est transportée séparément (ordres Workbench/Customizing selon cross-client ou client).
- **Modification ultérieure** : rouvrir le projet dans BAS, modifier, puis **redéployer** dans le même `ZMM_PO_EXT` (même OT ou nouvel OT).
- **Mise à niveau (FPS/SPS/release)** : l'app standard évolue ; la variante hérite automatiquement des évolutions. Retester la variante dans BAS après chaque upgrade (les changements en Safe Mode sont les plus stables ; fragments et controller extensions sont à vérifier).
- **Git** : versionner le projet (hors `node_modules`) dans votre dépôt de modes opératoires ou un dépôt dédié.

---

## Dépannage

| Symptôme | Cause probable | Solution |
|---|---|---|
| Liste des applications vide dans le générateur | Ressources Cloud Connector incomplètes, index des apps non calculé, `WebIDEUsage` incomplet | Exposer `/sap/bc/ui2`, `/sap/bc/lrep`, `/sap/bc/ui5_ui5` ; exécuter `/UI5/APP_INDEX_CALCULATE` ; vérifier `dev_abap` |
| Application listée comme non supportée | App non « flex enabled » ou version UI5 < 1.71 | Vérifier le manifest ; choisir une autre app |
| Erreur 403 à l'ouverture de l'éditeur dans BAS | Trop de ports exposés dans le Dev Space | Palette `F1` → *Ports: Preview* → *Ports: Unexpose* sur un port inutilisé |
| Clic sur « Contrôle Z » sans effet | Nom de l'extension différent entre `press` et `ControllerExtension.extend` | Aligner les deux noms ; vérifier la console |
| Déploiement : *ZIP archive contains disallowed files or has an incorrect structure* | Correctif manquant | Note SAP 3073188 |
| Déploiement : *500 Internal server error* / *Transport check could not be performed* | Correctif manquant | Note SAP 3243791 |
| FLP : « Impossible d'ouvrir l'application » | ID de target mapping = ID standard, URL renseignée, rôle manquant | ID = `customer.zmmpoext`, URL vide ; vérifier `PFCG` ; vider les caches `/UI2/INVALIDATE_*` |
| Changements non visibles après redéploiement | Cache navigateur / app index | `/UI5/APP_INDEX_CALCULATE`, `/UI2/INVALIDATE_GLOBAL_CACHES`, rechargement forcé |
| Tableau vide avec erreur 403 en preview | Autorisations achats manquantes | `SU53` sur l'utilisateur de la destination |
| Libellé i18n affiché brut | Clé absente de `i18n.properties` de la variante | Ajouter la clé ou utiliser un texte en dur pour le test |

---

## Liste des captures d'écran à réaliser

| N° | Écran | Fichier suggéré |
|---|---|---|
| 1-01 | Fiche F0842A – Implementation Information | `img/01-01-fiche-f0842a.png` |
| 1-02 | `manifest.json` avec `flexEnabled` | `img/01-02-manifest.png` |
| 1-03 | Création Dev Space SAP Fiori | `img/01-03-devspace.png` |
| 1-04 | Générateur : système + application | `img/01-04-wizard-system.png` |
| 1-05 | Générateur : Project Attributes | `img/01-05-wizard-attributes.png` |
| 1-06 | Éditeur d'adaptation ouvert | `img/01-06-editor.png` |
| 1-07 | Rename du bouton Créer | `img/01-07-rename.png` |
| 1-08 | Boîte Add Fragment | `img/01-08-add-fragment.png` |
| 1-09 | Bouton visible, tableau vide | `img/01-09-button-empty.png` |
| 1-10 | Code `ZListReportExt.js` | `img/01-10-controller.png` |
| 1-11 | MessageBox en mode Navigation | `img/01-11-messagebox.png` |
| 1-12 | Preview local | `img/01-12-preview.png` |
| 1-13 | Assistant de déploiement | `img/01-13-deploy-wizard.png` |
| 1-14 | Log de déploiement | `img/01-14-deploy-log.png` |
| 1-15 | `/UI2/FLPAM` – app descriptor item | `img/01-15-flpam.png` |
| 1-16 | Rôle `PFCG` | `img/01-16-pfcg.png` |

Insertion dans ce document : `![Capture 1-09](img/01-09-button-empty.png)`

---

## Références

- SAP Help – *Deploy the Adaptation Project to the ABAP Repository* (SAP Business Application Studio)
- SAP Help – *Configuring Target Mappings with the Launchpad Designer*, chapitre « Settings for SAPUI5 App Variants »
- SAP Help – *Extending an SAP Fiori Application for an On-Premise System* (valable pour Private Cloud Edition)
- SAP Tutorials – *Work with SAPUI5 Adaptation Projects* (groupe de 3 tutoriels)
- SAP Community – *Extending SAP-delivered SAP Fiori elements apps using adaptation projects to create SAP S/4HANA app variants*
- SAP Fiori Apps Reference Library – F0842A
- Notes SAP : 3073188, 3243791
- KBA SAP 3074554 – *How to Create a Tile for Adaptation Project?*
