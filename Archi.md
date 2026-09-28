# Architecture technique, outillage et runtime — 4 cas d'usage Fiori / S/4HANA

**Contexte :** SAP S/4HANA Cloud Private Edition 2021 FPS02 (ABAP Platform 2021 / SAP_BASIS 7.56, SAP_UI 7.56, SAPUI5 1.96.x), Fiori Launchpad embarqué (ABAP).
**Objet :** situer les briques logicielles, les services, les objets et les rôles pour quatre situations :

| Cas | Situation |
|---|---|
| **1** | Activation d'une tuile standard + configuration launchpad + rôle |
| **2** | Adaptation project d'une application standard + déploiement + launchpad + rôle |
| **3** | Service RAP (sans UI) + rôle |
| **4** | Application spécifique + déploiement + launchpad + rôle |

---

## Sommaire

1. [Vue d'ensemble du paysage](#1-vue-densemble-du-paysage)
2. [Les briques et où elles s'exécutent](#2-les-briques-et-où-elles-sexécutent)
3. [Prérequis logiciels et extensions](#3-prérequis-logiciels-et-extensions)
4. [Chaîne d'exécution d'une application Fiori](#4-chaîne-dexécution-dune-application-fiori)
5. [Cas 1 – Activation d'une tuile standard](#cas-1--activation-dune-tuile-standard)
6. [Cas 2 – Adaptation project](#cas-2--adaptation-project)
7. [Cas 3 – Service RAP sans UI](#cas-3--service-rap-sans-ui)
8. [Cas 4 – Application spécifique](#cas-4--application-spécifique)
9. [Tableau comparatif des quatre cas](#9-tableau-comparatif-des-quatre-cas)
10. [Matrice des transactions](#10-matrice-des-transactions)
11. [Matrice des rôles et autorisations](#11-matrice-des-rôles-et-autorisations)
12. [Flux réseau, ports et URLs](#12-flux-réseau-ports-et-urls)
13. [Transport DEV → QAS → PRD par type d'objet](#13-transport-dev--qas--prd-par-type-dobjet)
14. [Checklists de mise en place](#14-checklists-de-mise-en-place)
15. [Pièges fréquents](#15-pièges-fréquents)

---

## 1. Vue d'ensemble du paysage

```text
 POSTE DE TRAVAIL                        SAP BTP (optionnel)              S/4HANA PCE 2021 FPS02
┌──────────────────────┐                ┌──────────────────────┐        ┌───────────────────────────────────┐
│ Navigateur           │══ HTTPS 443 ═══╪══════════════════════╪═══════▶│ ICF  /sap/bc/ui2/flp   (Launchpad)│
│ (Chrome / Edge)      │                │                      │        │ ICF  /sap/bc/ui5_ui5   (apps BSP) │
│                      │                │                      │        │ ICF  /sap/bc/lrep      (flex)     │
│ SAP GUI for Windows  │══ DIAG 32xx ═══╪══════════════════════╪═══════▶│ Transactions (SE80, PFCG, /UI2/…) │
│                      │                │                      │        │                                   │
│ Eclipse + ADT        │══ /sap/bc/adt ═╪══════════════════════╪═══════▶│ ABAP Development Tools backend    │
│                      │                │                      │        │                                   │
│ VS Code              │══ /sap/opu/… ══╪══════════════════════╪═══════▶│ Gateway embedded (OData V2/V4)    │
│  + SAP Fiori tools   │══ /sap/bc/lrep╪═══════════════════════╪═══════▶│ LREP + SAPUI5 ABAP Repository     │
│  + Node.js           │                │                      │        │                                   │
└──────────────────────┘                │  SAP Business        │        │ CDS / RAP / BOPF / ABAP           │
            │                           │  Application Studio  │        │ SAP HANA                          │
            └── (variante BAS) ────────▶│  (Dev Space Fiori)   │        └───────────────────────────────────┘
                                        └──────────┬───────────┘                        ▲
                                                   │                                    │
                                                   └── SAP Cloud Connector ═════════════┘
                                                        (hôte virtuel, ports exposés)
```

Points structurants :

- **Tout s'exécute côté ABAP en production** : le Launchpad, les applications SAPUI5 (stockées en BSP), les services OData et la logique métier sont dans le serveur S/4HANA. BTP et BAS ne servent qu'au **développement**.
- **Le navigateur est le seul runtime client** : il télécharge SAPUI5 et les applications depuis l'ICF, puis dialogue en OData.
- **Le poste de développement n'est pas dans la chaîne d'exécution** : après déploiement, VS Code, BAS et Eclipse peuvent être éteints.

---

## 2. Les briques et où elles s'exécutent

| Brique | Rôle | Où elle s'exécute | Accès / transaction |
|---|---|---|---|
| **SAP Fiori Launchpad (FLP)** | Page d'entrée, résolution des intents, affichage des tuiles | ICF du serveur ABAP (`/sap/bc/ui2/flp`) | Navigateur |
| **Contenu launchpad** | Catalogues, tuiles, target mappings, espaces, pages | Tables ABAP client-spécifiques | `/UI2/FLPAM`, `/UI2/FLPD_CUST`, `/UI2/FLPCM_CONF` |
| **SAPUI5 ABAP Repository** | Stockage des applications UI5 sous forme d'applications BSP | Serveur ABAP (`/sap/bc/ui5_ui5/sap/<bsp>`) | `SE80` → Application BSP |
| **Bibliothèque SAPUI5** | Runtime JavaScript 1.96 livré par SAP_UI | Servie par l'ICF `/sap/public/bc/ui5_ui5/resources` | — |
| **LREP (Layered Repository)** | Changements flex : adaptations key user, variantes d'application, variantes de vue | Serveur ABAP (`/sap/bc/lrep`) | Via RTA / adaptation project |
| **App index** | Index des applications UI5 déployées, utilisé par le FLP et les outils | Serveur ABAP | `/UI5/APP_INDEX_CALCULATE` |
| **SAP Gateway (embedded)** | Exposition OData V2 et V4 | Serveur ABAP (`/sap/opu/odata`, `/sap/opu/odata4`) | `/IWFND/MAINT_SERVICES` |
| **Modèle CDS / RAP / BOPF** | Modèle de données et logique métier | Serveur ABAP + SAP HANA | ADT, `SE11` |
| **PFCG / autorisations** | Ce que l'utilisateur voit et peut appeler | Serveur ABAP | `PFCG`, `SU01` |
| **ADT (ABAP Development Tools)** | Développement ABAP, CDS, RAP | Eclipse sur le poste, backend `/sap/bc/adt` | Eclipse |
| **SAP Fiori tools** | Génération, preview, déploiement d'applications UI5 | VS Code ou BAS + Node.js sur le poste / le dev space | VS Code / BAS |
| **SAP Cloud Connector** | Tunnel BTP → réseau interne | Machine dédiée on-premise | `https://<cc>:8443` |

---

## 3. Prérequis logiciels et extensions

### 3.1 Poste développeur ABAP / CDS / RAP (cas 3, et modèle du cas 4)

| Composant | Détail |
|---|---|
| **Eclipse IDE for Java Developers** | Version figurant dans la matrice de support ADT en cours (voir `tools.hana.ondemand.com`) |
| **JDK 64 bits** | Version exigée par la release Eclipse installée |
| **ABAP Development Tools (ADT)** | Installé depuis l'update site `https://tools.hana.ondemand.com/latest` |
| **SAP GUI for Windows** | Requis par ADT pour ouvrir les transactions et certains éditeurs classiques |
| **SAPCRYPTOLIB / certificats** | Pour les connexions HTTPS et SNC selon votre paysage |
| Côté serveur | Nœud ICF `/sap/bc/adt` actif, autorisation `S_ADT_RES`, `S_DEVELOP` |

### 3.2 Poste développeur UI5 / Fiori tools (cas 2 et 4)

| Composant | Détail |
|---|---|
| **Visual Studio Code** | Version courante |
| **Extension « SAP Fiori tools – Extension Pack »** | Regroupe Application Modeler, Service Modeler, Guided Development, XML Annotation Language Server, UI5 Language Assistant, Adaptation Project Generator |
| **Node.js LTS** | Version supportée par les Fiori tools ; `npm` opérationnel |
| **Accès au registre npm** | `registry.npmjs.org` (ou miroir interne d'entreprise) |
| **Git** | Versionnement des projets UI5 (les objets ABAP restent dans les ordres de transport) |
| Optionnel | Extension SAP Fiori Tools – Guided Answers, ESLint, Prettier |
| Côté serveur | `/sap/bc/ui5_ui5`, `/sap/bc/lrep`, `/sap/bc/ui2/app_index`, `/sap/bc/adt`, service `/UI5/ABAP_REPOSITORY_SRV` actifs |

### 3.3 Variante SAP Business Application Studio

| Composant | Détail |
|---|---|
| **Abonnement BAS** | Souscription dans le sous-compte BTP |
| **Dev Space type « SAP Fiori »** | Inclut les Fiori tools et le générateur d'adaptation project |
| **SAP Cloud Connector** | Mappage d'accès vers le S/4HANA, ressources `/sap/bc/ui5_ui5`, `/sap/bc/lrep`, `/sap/bc/ui2`, `/sap/bc/adt`, `/sap/opu/odata`, `/sap/public/bc` |
| **Destination BTP** | Type HTTP, Proxy `OnPremise`, propriétés `WebIDEEnabled=true`, `WebIDEUsage=odata_abap,dev_abap,ui5_execute_abap`, `sap-client`, `HTML5.DynamicDestination=true` |

### 3.4 Administrateur Launchpad / autorisations (les quatre cas)

| Composant | Détail |
|---|---|
| **SAP GUI** | `/UI2/FLPAM`, `/UI2/FLPD_CUST`, `/UI2/FLPCM_CONF`, `PFCG`, `SU01`, `/IWFND/MAINT_SERVICES`, `SICF` |
| **Navigateur** | Apps *Gérer les espaces / les pages du launchpad*, Fiori Apps Reference Library |
| **Accès STMS** | Souvent réservé à l'équipe Basis ou au fournisseur PCE |

### 3.5 Utilisateur final

Navigateur à jour, accès HTTPS au serveur, rôle PFCG portant le catalogue métier et l'espace. Aucun logiciel spécifique.

---

## 4. Chaîne d'exécution d'une application Fiori

Séquence commune aux cas 1, 2 et 4 :

```text
 1  Navigateur ──▶ GET /sap/bc/ui2/flp?sap-client=100
                    │
 2                  ├─▶ FLP charge le contenu launchpad autorisé par les RÔLES PFCG
                    │    (catalogues métiers → tuiles + target mappings → espaces / pages)
                    │
 3  Clic sur tuile ─┴─▶ intent  #SemanticObject-action
                    │
 4                  ├─▶ Résolution par le target mapping :
                    │      • type d'application (SAPUI5 Fiori App, transaction, Web Dynpro…)
                    │      • ID de composant SAPUI5
                    │      • chemin ICF de l'application BSP
                    │
 5                  ├─▶ GET /sap/bc/ui5_ui5/sap/<bsp>/Component.js + manifest.json
                    │      (+ /sap/bc/lrep/flex/data pour une variante ou des adaptations)
                    │
 6                  └─▶ Appels OData : /sap/opu/odata/sap/<SERVICE>/$metadata puis données
                              │
 7                            └─▶ Gateway embedded → CDS / RAP / BOPF → SAP HANA
                                     avec contrôle S_SERVICE + objets d'autorisation métier + DCL
```

Trois niveaux d'autorisation indépendants, à ne pas confondre :

| Niveau | Objet | Effet si manquant |
|---|---|---|
| **Voir la tuile** | Catalogue métier dans le rôle PFCG | La tuile n'apparaît pas |
| **Démarrer le service** | `S_SERVICE` sur le service OData | L'application s'ouvre mais reste vide / erreur 403 |
| **Voir les données** | Objets métier (`M_BANF_*`, `M_BEST_*`…) + DCL | Liste vide ou message d'autorisation |

---

## Cas 1 – Activation d'une tuile standard

**Situation :** l'application est livrée par SAP, il n'y a **aucun développement**. Tout est configuration.

### Architecture

```text
Navigateur ──▶ FLP ──▶ target mapping SAP ──▶ BSP livré par SAP ──▶ service OData SAP ──▶ CDS SAP
                 ▲                                                        ▲
                 │                                                        │
        rôle PFCG (catalogue métier)                            service activé dans /IWFND/MAINT_SERVICES
```

### Outils utilisés

| Outil | Usage |
|---|---|
| Navigateur | Fiori Apps Reference Library (fiche de l'app : service OData, BSP, catalogues, rôle modèle) |
| SAP GUI | `SICF`, `/IWFND/MAINT_SERVICES`, `/UI2/FLPAM` ou `/UI2/FLPD_CUST`, `/UI2/FLPCM_CONF`, `PFCG`, `SU01` |
| Navigateur (apps admin) | *Gérer les espaces du launchpad*, *Gérer les pages du launchpad* |

Aucun Eclipse, aucun VS Code, aucun Node.js.

### Objets touchés

| Objet | Livré par SAP | À créer / adapter |
|---|---|---|
| Application BSP | ✔ | — |
| Service OData | ✔ | Activation dans le hub (`/IWFND/MAINT_SERVICES`, alias `LOCAL`) |
| Nœud ICF | ✔ | Activation (`SICF`) |
| Catalogue technique SAP | ✔ | — |
| Catalogue métier | Modèles SAP | Copie ou catalogue Z référençant tuile + target mapping |
| Espace / page | Modèles SAP | Affectation des tuiles |
| Rôle PFCG | Modèles `SAP_BR_*` | Rôle Z dérivé, avec autorisations métier |

### Séquence de mise en place

```text
1. Fiche de l'app (Apps Reference Library) : service OData, BSP, catalogues, rôle modèle
2. SICF        : activer les nœuds ICF requis
3. /IWFND/MAINT_SERVICES : activer le service OData, alias LOCAL, tester (/$metadata)
4. /UI5/APP_INDEX_CALCULATE : indexer les applications
5. /UI2/FLPCM_CONF ou /UI2/FLPD_CONF : catalogue métier Z, référencer tuile + target mapping SAP
6. Espaces / pages : placer la tuile
7. PFCG : rôle Z (menu = catalogue + espace), générer les autorisations, compléter les objets métier
8. SU01 : affecter le rôle
9. Test : #SemanticObject-action dans le FLP
```

### Où ça s'exécute

| Élément | Emplacement |
|---|---|
| Application UI5 | BSP SAP, `/sap/bc/ui5_ui5/sap/<bsp>` |
| Logique métier | ABAP standard |
| Configuration | Tables launchpad du client |
| Développement | Néant |

---

## Cas 2 – Adaptation project

**Situation :** on part d'une application SAP et on crée une **variante d'application** (nouvel ID) avec des adaptations UI. L'application standard reste intacte.

### Architecture

```text
 DÉVELOPPEMENT                                    EXÉCUTION
┌───────────────────────────┐                   ┌─────────────────────────────────────────────┐
│ VS Code + Fiori tools     │                   │ Navigateur                                  │
│  (ou BAS + Cloud Connector)│                  │   │                                         │
│  générateur d'adaptation  │                   │   ▼                                         │
│  éditeur d'adaptation     │── HTTPS ────────▶ │ FLP ──▶ target mapping de la VARIANTE        │
│  preview local (Node.js)  │   charge l'app    │   │        ID = customer.<variante>          │
│  déploiement              │   depuis le       │   ▼                                         │
└───────────────────────────┘   backend         │ BSP de la variante (descripteur + changes)   │
             │                                  │   +                                          │
             │  npm run deploy                  │ BSP de l'application SAP d'origine           │
             └── /UI5/ABAP_REPOSITORY_SRV ────▶ │   +                                          │
                                                │ LREP (/sap/bc/lrep) : fusion des changements │
                                                │   ▼                                          │
                                                │ service OData SAP inchangé                   │
                                                └─────────────────────────────────────────────┘
```

Le runtime **fusionne** : application de base SAP + descripteur de variante + changements flex. C'est pourquoi la variante suit automatiquement les évolutions du standard.

### Outils utilisés

| Outil | Usage |
|---|---|
| VS Code + SAP Fiori tools + Node.js | Génération, éditeur d'adaptation, preview, déploiement |
| (ou BAS + Cloud Connector + destination) | Même chose, hébergé dans BTP |
| SAP GUI | `SE80`, `SE09`, `/UI5/APP_INDEX_CALCULATE`, `/UI2/FLPAM`, `PFCG` |
| Eclipse + ADT | Non requis (utile pour lire le package et les ordres) |

### Objets créés

| Objet | Type | Remarque |
|---|---|---|
| Projet d'adaptation | Fichiers locaux | `manifest.appdescr_variant`, `changes/fragments`, `changes/coding` |
| Application BSP de la variante | `R3TR WAPA` | Nom ≤ 15 caractères, package Z, ordre Workbench |
| Target mapping + tuile | Contenu launchpad | **URL vide, ID = ID de la variante**, semantic object/action propres |
| Rôle | PFCG | Catalogue métier contenant la variante |

### Où ça s'exécute

| Phase | Emplacement |
|---|---|
| Édition, preview | Poste de travail (ou dev space BAS) — serveur Node.js local, données réelles via le backend |
| Exécution finale | Serveur ABAP : BSP variante + LREP + BSP standard |
| Service OData | Celui de l'application SAP, inchangé |

---

## Cas 3 – Service RAP sans UI

**Situation :** on expose un modèle de données ou une logique métier en OData, sans application Fiori. Consommateurs possibles : une application Fiori développée plus tard, un outil externe, une intégration.

### Architecture

```text
 DÉVELOPPEMENT                                   EXÉCUTION
┌──────────────────────────┐                   ┌────────────────────────────────────────────┐
│ Eclipse + ADT            │                   │ Consommateur : app Fiori, Postman,         │
│  CDS (DDLS)              │── /sap/bc/adt ──▶ │ intégration, Integration Suite…            │
│  DCL, BDEF, classes      │                   │        │ HTTPS + OData                      │
│  Service Definition      │                   │        ▼                                   │
│  Service Binding         │── Publish ──────▶ │ Gateway embedded                            │
└──────────────────────────┘                   │   V2 : /sap/opu/odata/sap/<BINDING>        │
                                               │   V4 : /sap/opu/odata4/sap/<binding>/…     │
                                               │        ▼                                   │
                                               │ RAP runtime → CDS → SAP HANA               │
                                               │   contrôle : S_SERVICE + DCL + AUTHORITY-  │
                                               │   CHECK dans le behavior pool              │
                                               └────────────────────────────────────────────┘
```

### Outils utilisés

| Outil | Usage |
|---|---|
| Eclipse + ADT | Tout le modèle : CDS, DCL, BDEF, behavior pool, SRVD, SRVB, *Data Preview*, aperçu Fiori elements intégré |
| SAP GUI | `/IWFND/MAINT_SERVICES` (vérification), `SE11`, `SE91`, `SE09`, `PFCG`, `SU01`, `/IWFND/ERROR_LOG` |
| Navigateur / Postman | Test de `$metadata` et des données |
| VS Code | Non requis à ce stade |

### Objets créés

| Objet | Type | Rôle |
|---|---|---|
| Vues d'interface et de consommation | `DDLS` | Modèle de données |
| Contrôle d'accès | `DCLS` | Filtrage par autorisation (`aspect pfcg_auth`) |
| Behavior definition + behavior pool | `BDEF` + `CLAS` | Uniquement si écriture ou actions |
| Service definition | `SRVD` | Périmètre exposé |
| Service binding | `SRVB` | Protocole : OData V2 UI, V4 UI ou API |
| Ordre Workbench | — | Transport |

### Autorisations spécifiques

| Niveau | Objet |
|---|---|
| Démarrer le service | `S_SERVICE` (le nom technique dépend du binding) |
| Lire les données | DCL de la vue + objets métier (`M_BANF_*`…) |
| Exécuter une action | Contrôle explicite dans le behavior pool (`AUTHORITY-CHECK`, `get_instance_authorizations`) |

### Où ça s'exécute

Tout côté serveur ABAP. Aucun artefact sur le poste de travail, aucun déploiement de fichier : la publication du *local service endpoint* depuis ADT suffit à enregistrer le service dans le hub local.

---

## Cas 4 – Application spécifique

**Situation :** application SAPUI5 développée pour un besoin propre, généralement Fiori elements au-dessus d'un service RAP (cas 3).

### Architecture

```text
 DÉVELOPPEMENT                                          EXÉCUTION
┌────────────────────────┐  ┌───────────────────────┐  ┌──────────────────────────────────────┐
│ Eclipse + ADT          │  │ VS Code + Fiori tools │  │ Navigateur                           │
│  modèle CDS / RAP      │  │  générateur d'app     │  │   │                                  │
│  SRVD + SRVB (Publish) │  │  annotations locales  │  │   ▼                                  │
└──────────┬─────────────┘  │  extensions (ext/)    │  │ FLP ──▶ target mapping Z             │
           │                │  preview Node.js      │  │   │      semantic object Z + action   │
           │                └──────────┬────────────┘  │   ▼                                  │
           │ service OData Z           │ npm run deploy│ BSP Z /sap/bc/ui5_ui5/sap/<bsp>      │
           └───────────────────────────┴──────────────▶│   ▼                                  │
                                                       │ service OData Z → RAP → CDS → HANA   │
                                                       └──────────────────────────────────────┘
```

### Outils utilisés

| Phase | Outil |
|---|---|
| Modèle et service | Eclipse + ADT |
| Application UI5 | VS Code + SAP Fiori tools + Node.js (ou BAS) |
| Déploiement | `npm run deploy` → `/UI5/ABAP_REPOSITORY_SRV` |
| Launchpad et rôle | SAP GUI : `/UI2/SEMOBJ`, `/UI2/FLPAM`, `/UI2/FLPD_CONF`, `PFCG`, `SU01` |
| Versionnement | Git pour le projet UI5 ; ordres de transport pour l'ABAP |

### Objets créés

| Couche | Objets |
|---|---|
| Données / service | `DDLS`, `DDLX`, `DCLS`, éventuellement `BDEF` + `CLAS`, `SRVD`, `SRVB` |
| UI | Projet UI5 local → application BSP `R3TR WAPA` |
| Launchpad | **Objet sémantique Z** (`/UI2/SEMOBJ`), catalogue technique, target mapping, tuile, catalogue métier, espace / page |
| Sécurité | Rôle PFCG : catalogue, `S_SERVICE` du service Z, objets métier |

### Spécificité par rapport au cas 2

| Sujet | Cas 2 (adaptation) | Cas 4 (application spécifique) |
|---|---|---|
| Application de base | SAP | Aucune, tout est créé |
| Objet sémantique | Existant (SAP) | **À créer** dans `/UI2/SEMOBJ` |
| Service OData | SAP | Z, à créer et publier |
| Fusion LREP au runtime | Oui | Non |
| Sensibilité aux upgrades | Moyenne (dépend du standard) | Faible (code maîtrisé) |

---

## 9. Tableau comparatif des quatre cas

| Critère | Cas 1 — Tuile standard | Cas 2 — Adaptation project | Cas 3 — Service RAP | Cas 4 — Application spécifique |
|---|---|---|---|---|
| Profil | Consultant / admin FLP | Développeur UI5 | Développeur ABAP | Développeur ABAP + UI5 |
| Eclipse + ADT | Non | Non | **Oui** | **Oui** |
| VS Code + Fiori tools + Node.js | Non | **Oui** | Non | **Oui** |
| BAS + Cloud Connector | Non | Alternative à VS Code | Non | Alternative à VS Code |
| SAP GUI | Oui | Oui | Oui | Oui |
| Objets ABAP créés | Aucun (config) | `WAPA` | `DDLS`, `DCLS`, `SRVD`, `SRVB`, (`BDEF`, `CLAS`) | Les deux colonnes précédentes réunies |
| Objet sémantique | SAP | SAP (nouvelle action) | Sans objet | **Z, à créer** |
| Service OData | SAP, à activer | SAP, inchangé | **Z, à publier** | **Z, à publier** |
| Déploiement de fichiers | Non | Oui (BSP) | Non | Oui (BSP) |
| Contenu launchpad | Oui | Oui | Non | Oui |
| Rôle PFCG | Oui | Oui | Oui (`S_SERVICE` + métier) | Oui |
| Transport | Customizing (+ Workbench selon contenu) | Workbench + Customizing | Workbench | Workbench + Customizing |
| Impact upgrade | Nul | À retester | Faible (BAdI / vues released) | Faible |

---

## 10. Matrice des transactions

| Transaction / App | Cas 1 | Cas 2 | Cas 3 | Cas 4 | Usage |
|---|:--:|:--:|:--:|:--:|---|
| `SICF` | ✔ | ✔ | ✔ | ✔ | Activation des nœuds ICF |
| `/IWFND/MAINT_SERVICES` | ✔ | ✔ | ✔ | ✔ | Activation / contrôle des services OData |
| `/IWFND/ERROR_LOG`, `/IWBEP/ERROR_LOG` | ✔ | ✔ | ✔ | ✔ | Diagnostic OData |
| `/UI5/APP_INDEX_CALCULATE` | ✔ | ✔ | | ✔ | Index des applications |
| `/UI2/FLPAM` ou `/UI2/FLPD_CUST` | ✔ | ✔ | | ✔ | Catalogue technique, target mapping, tuile |
| `/UI2/FLPD_CONF` / `/UI2/FLPCM_CONF` | ✔ | ✔ | | ✔ | Catalogue métier |
| `/UI2/SEMOBJ` | | | | ✔ | Objet sémantique Z |
| `/UI2/INVALIDATE_GLOBAL_CACHES` | ✔ | ✔ | | ✔ | Purge des caches FLP |
| `/IWFND/CACHE_CLEANUP`, `/IWBEP/CACHE_CLEANUP` | ✔ | ✔ | ✔ | ✔ | Purge des métadonnées |
| `SE80` | | ✔ | ✔ | ✔ | BSP, packages |
| `SE11` / `SE91` | | | ✔ | ✔ | Tables, classes de messages |
| `SE09` / `SE10` | ✔ | ✔ | ✔ | ✔ | Ordres de transport |
| `PFCG` / `SU01` / `SU53` | ✔ | ✔ | ✔ | ✔ | Rôles et autorisations |
| `STMS` | ✔ | ✔ | ✔ | ✔ | Import (souvent Basis / fournisseur PCE) |
| ADT (Eclipse) | | | ✔ | ✔ | CDS, RAP, services |

---

## 11. Matrice des rôles et autorisations

### Par profil

| Profil | Autorisations clés | Cas concernés |
|---|---|---|
| **Développeur ABAP** | `S_DEVELOP` (`DDLS`, `DCLS`, `DDLX`, `SRVD`, `SRVB`, `BDEF`, `CLAS`, `DEVC`), `S_TRANSPRT`, `S_ADT_RES` | 3, 4 |
| **Développeur UI5** | `S_DEVELOP` (OBJTYPE `WAPA`, package Z), `S_TRANSPRT`, `S_SERVICE` sur `/UI5/ABAP_REPOSITORY_SRV`, `S_ADT_RES` | 2, 4 |
| **Key user (adaptation runtime)** | `SAP_UI_FLEX_KEY_USER` | hors périmètre ici (voir document 03) |
| **Administrateur FLP** | `SAP_UI2_ADMIN_700` ou équivalent, accès `/UI2/*` | 1, 2, 4 |
| **Administrateur Gateway** | Droits `/IWFND/MAINT_SERVICES` | 1, 3, 4 |
| **Utilisateur final** | Catalogue métier dans le rôle + `S_SERVICE` + objets métier (`M_BANF_*`, `M_BEST_*`…) | 1, 2, 4 |

### Composition d'un rôle utilisateur (identique dans les cas 1, 2 et 4)

```text
Rôle PFCG  Z_BR_<domaine>
├─ Menu
│   ├─ Launchpad Catalog  : Z_BC_<domaine>       → rend la tuile visible
│   └─ Launchpad Space    : Z_SP_<domaine>       → place la tuile dans une page
└─ Autorisations
    ├─ S_SERVICE          : service OData de l'application
    ├─ Objets métier      : M_BANF_* / M_BEST_* … en 01 / 02 / 03
    └─ S_TCODE / S_START  : selon le type d'application cible
```

---

## 12. Flux réseau, ports et URLs

| Flux | Origine → cible | Port / protocole | Cas |
|---|---|---|---|
| Utilisateur → FLP | Navigateur → S/4 | HTTPS 443 (ou 44300 en interne) | 1, 2, 4 |
| Développeur ADT | Eclipse → `/sap/bc/adt` | HTTPS | 3, 4 |
| Fiori tools | VS Code → `/sap/opu/odata`, `/sap/bc/lrep`, `/sap/bc/ui5_ui5`, `/sap/bc/ui2/app_index` | HTTPS | 2, 4 |
| SAP GUI | Poste → serveur | DIAG 32xx | tous |
| BAS → backend | Dev space → Cloud Connector → hôte virtuel | HTTPS via tunnel | 2, 4 (variante BAS) |
| Administration Cloud Connector | Navigateur → CC | HTTPS 8443 | 2, 4 (variante BAS) |

URLs utiles :

```text
Launchpad               /sap/bc/ui2/flp?sap-client=100#<SemObj>-<action>
Application BSP         /sap/bc/ui5_ui5/sap/<bsp>/manifest.json
Service OData V2        /sap/opu/odata/sap/<SERVICE>/$metadata
Service OData V4        /sap/opu/odata4/sap/<binding>/srvd/sap/<service_definition>/0001/
Version SAPUI5          /sap/public/bc/ui5_ui5/resources/sap-ui-version.json
Infos techniques FLP    Ctrl+Alt+Shift+P dans l'application
```

---

## 13. Transport DEV → QAS → PRD par type d'objet

| Objet | Type d'ordre | Remarques |
|---|---|---|
| Vues CDS, DCL, BDEF, classes, SRVD, SRVB | Workbench | Vérifier ensuite l'enregistrement du service dans `/IWFND/MAINT_SERVICES` de la cible |
| Application BSP (spécifique ou variante) | Workbench | `R3TR WAPA` ; recalculer l'app index après import |
| Objet sémantique (`/UI2/SEMOBJ`) | Customizing | Client-spécifique |
| Catalogues, tuiles, target mappings | Customizing (client) ou Workbench selon la portée | Dépend de votre configuration `/UI2/*` |
| Espaces et pages | Customizing | Via les apps d'administration ou les transactions dédiées |
| Rôles PFCG | Workbench (transport de rôles) | Ou recréation manuelle selon la gouvernance |
| Activation des services OData SAP | Non transportable en tant que telle | À refaire ou à traiter par liste de tâches dans chaque système |
| Changements key user (LREP) | Workbench via l'ATO / la publication RTA | Voir document 03 |

**Séquence d'import recommandée** : service et modèle → applications BSP → objet sémantique → contenu launchpad → rôles.
**Après chaque import** : `/UI5/APP_INDEX_CALCULATE`, `/IWFND/CACHE_CLEANUP`, `/IWBEP/CACHE_CLEANUP`, `/UI2/INVALIDATE_GLOBAL_CACHES`.

---

## 14. Checklists de mise en place

### Poste développeur complet (cas 3 et 4)

- [ ] SAP GUI installé et système configuré
- [ ] JDK 64 bits conforme à la release Eclipse
- [ ] Eclipse + ADT depuis `tools.hana.ondemand.com`
- [ ] Projet ABAP créé et connecté (`/sap/bc/adt` actif, `S_ADT_RES`)
- [ ] Node.js LTS + accès npm
- [ ] VS Code + SAP Fiori tools Extension Pack
- [ ] Système déclaré (`Fiori: Add SAP System`) et testé (`Fiori: Open Environment Check`)
- [ ] Package Z et ordre de transport disponibles
- [ ] Certificats du serveur acceptés

### Poste administrateur (cas 1, et fin des cas 2 et 4)

- [ ] Accès `/UI2/*`, `PFCG`, `SU01`, `/IWFND/MAINT_SERVICES`, `SICF`
- [ ] Convention de nommage arrêtée : catalogues `Z_TC_*` / `Z_BC_*`, rôles `Z_BR_*`, objets sémantiques Z
- [ ] Procédure de purge de caches connue
- [ ] Circuit de transport identifié (Basis interne ou fournisseur PCE)

### Variante BAS (cas 2 et 4)

- [ ] Souscription BAS et dev space type SAP Fiori
- [ ] Cloud Connector opérationnel, ressources exposées
- [ ] Destination BTP avec `WebIDEUsage` complet, test de connexion OK

---

## 15. Pièges fréquents

| Symptôme | Cause habituelle | Cas |
|---|---|---|
| La tuile n'apparaît pas | Catalogue absent du rôle, cache FLP, espace non affecté | 1, 2, 4 |
| La tuile apparaît, l'application ne s'ouvre pas | Target mapping incorrect, app index non recalculé | 1, 2, 4 |
| Variante qui ouvre l'application standard | Target mapping avec l'ID du composant SAP ou une URL renseignée | 2 |
| Application vide, erreur 403 | `S_SERVICE` manquant ou service non activé | 1, 3, 4 |
| Liste vide sans erreur | Autorisations métier ou DCL restrictif | 3, 4 |
| Liste d'applications vide dans le générateur | Ressources Cloud Connector incomplètes, `WebIDEUsage` partiel, app index | 2, 4 |
| Déploiement refusé | `S_DEVELOP` sur `WAPA`, package ou ordre invalide | 2, 4 |
| Service introuvable après transport | Enregistrement Gateway non transporté | 3, 4 |
| Modifications invisibles après redéploiement | Cache navigateur, app index, caches FLP | 2, 4 |
| Fonctionne en DEV, pas en QAS | Paramétrage métier absent en QAS, contenu launchpad non transporté | tous |

---

## Documents liés

| Document | Contenu |
|---|---|
| `01_Extension_UI_Adaptation_Project_F0842A.md` | Cas 2, pas à pas dans BAS |
| `02_Extension_BAdI_F0842A.md` | Extensibilité développeur côté logique métier |
| `03_Key_User_Extensibility_F0842A.md` | Extensibilité key user (hors périmètre de ce document) |
| `04_App_RAP_Determination_Source_Appro_REV2.md` | Cas 3 et 4, modèle CDS et application |
| `05_Adaptation_Project_F1048_Navigation_VSCode.md` | Cas 2 dans VS Code, avec transport vers la qualité |
| `06_App_RAP_V2_Action_Affectation_Source.md` | Cas 3 enrichi : behavior definition et action RAP |
