# Mode opératoire — Application CAP sur SAP BTP

## Suivi des demandes d'achat

**Cloud Application Programming Model · SAP HANA Cloud**
CDS · Node.js · XSUAA · Cloud Foundry
**Système source : SAP S/4HANA 2021 FPS02**

| Attribut | Valeur |
|---|---|
| Référence du document | MO-CAP-BTP-PURREQ-v1.0 |
| Documents liés | [MO Service RAP](MO%20Service%20RAP.md) · [MO Cockpit OVP](MO%20Cockpit%20OVP.md) |
| Objet créé | Application CAP de suivi des demandes d'achat |
| Plateforme d'exécution | SAP BTP — environnement Cloud Foundry |
| Persistance | SAP HANA Cloud — conteneur HDI dédié |
| Système source | SAP S/4HANA 2021 FPS02, accès par OData |
| Outils | VS Code · `@sap/cds-dk` · Cloud Foundry CLI · MTA Build Tool |
| Auteur | … |
| Vérifié par | … |
| Approuvé par | … |
| Date de rédaction | … |
| Statut | Version de travail |

> **Convention de ce document**
> Chaque emplacement de copie d'écran est signalé par un bloc `📸 COPIE D'ÉCRAN N°XX`.
> Déposez vos images dans `images/CAP-BTP/` puis remplacez la ligne indiquée par le lien Markdown correspondant.

---

## Sommaire

- [1. Objet et architecture](#1-objet-et-architecture)
- [2. Prérequis](#2-prérequis)
- [3. Synoptique de la démarche](#3-synoptique-de-la-démarche)
- [Partie A — Service CAP](#partie-a--service-cap)
- [Partie B — Intégration à S/4HANA](#partie-b--intégration-à-s4hana)
- [Partie C — Persistance HANA Cloud](#partie-c--persistance-hana-cloud)
- [Partie D — Sécurité et autorisations](#partie-d--sécurité-et-autorisations)
- [Partie E — Interface utilisateur](#partie-e--interface-utilisateur)
- [Partie F — Construction et déploiement](#partie-f--construction-et-déploiement)
- [Partie G — Publication dans le launchpad](#partie-g--publication-dans-le-launchpad)
- [Partie H — Recette et exploitation](#partie-h--recette-et-exploitation)
- [Annexe A — Commandes utiles](#annexe-a--commandes-utiles)
- [Annexe B — Diagnostic des incidents fréquents](#annexe-b--diagnostic-des-incidents-fréquents)
- [Annexe C — Index des copies d'écran](#annexe-c--index-des-copies-décran)
- [Annexe D — Historique des versions](#annexe-d--historique-des-versions)

---

## 1. Objet et architecture

### 1.1 Objet

Ce mode opératoire décrit la création d'une application de suivi des demandes d'achat au moyen du **Cloud Application Programming Model**, son exécution sur SAP BTP en environnement Cloud Foundry, sa persistance dans SAP HANA Cloud et son intégration au système S/4HANA source.

### 1.2 Deux bases de données distinctes

Point de compréhension déterminant : **SAP HANA Cloud n'est pas la base de données de votre S/4HANA**. L'application CAP déploie ses tables dans un conteneur HDI qui lui est propre, sur une instance HANA Cloud distincte. Elle ne lit jamais directement les tables de S/4HANA : les données du système source sont consommées par appel OData.

| Élément | Où il réside | Mode d'accès |
|---|---|---|
| Données de suivi | Conteneur HDI sur HANA Cloud | Accès direct par CAP |
| Demandes d'achat | Base de S/4HANA | Service OData, via destination |
| Logique de suivi | Service Node.js sur Cloud Foundry | Exécution CAP |
| Interface | Référentiel HTML5 sur BTP | Servie par l'approuter |
| Authentification | Service XSUAA | Jetons et rôles |

> **⚠️ Conséquence**
> Toute information provenant de S/4HANA transite par un service OData et une destination. Sans connectivité opérationnelle vers le système source, l'application fonctionnera sur ses seules données propres.

### 1.3 Comparaison avec les voies ABAP

| Critère | RAP | Overview Page | CAP sur BTP |
|---|---|---|---|
| Lieu d'exécution | Backend ABAP | Backend ABAP | SAP BTP |
| Persistance | Base S/4HANA | Base S/4HANA | HANA Cloud dédiée |
| Langage | ABAP | Annotations | CDS et Node.js |
| Infrastructure requise | Aucune | Aucune | Abonnement BTP |
| Accès aux données S/4 | Direct | Direct | Par OData |
| Cycle de livraison | Transports | Transports | Build et déploiement |

### 1.4 Hors périmètre

- Souscription et paramétrage initial du sous-compte BTP.
- Installation et configuration du Cloud Connector, qui relève de l'infrastructure.
- Mise en place d'une chaîne d'intégration et de livraison continues.
- Multi-tenance et distribution de l'application à plusieurs clients.
- Migration des données existantes vers le nouveau modèle.

---

## 2. Prérequis

### 2.1 Souscriptions et services BTP

| Service | Usage | Disponible |
|---|---|---|
| Environnement Cloud Foundry | Exécution du service CAP | ☐ |
| SAP HANA Cloud | Persistance des données de suivi | ☐ |
| Authorization and Trust Management | Authentification et rôles | ☐ |
| Destination | Déclaration des systèmes distants | ☐ |
| Connectivity | Passerelle vers le Cloud Connector | ☐ |
| HTML5 Application Repository | Hébergement de l'interface | ☐ |
| Service de portail ou Work Zone | Launchpad et tuiles | ☐ |

> **⚠️ Point d'attention budgétaire**
> Une instance HANA Cloud est facturée au temps d'exécution. Prévoir son arrêt automatique hors heures ouvrées sur les environnements de développement, faute de quoi la consommation devient rapidement significative.

### 2.2 Connectivité vers S/4HANA

| Élément | Exigence |
|---|---|
| Cloud Connector | Installé, connecté au sous-compte, en état vert |
| Ressources exposées | Chemins des services OData du système source autorisés |
| Destination | Créée dans le sous-compte, pointant vers le Cloud Connector |
| Authentification | Utilisateur technique, principal propagation ou autre mécanisme retenu |
| Service source | Service OData du système S/4HANA enregistré et actif |

> **📸 COPIE D'ÉCRAN N°01** — Cockpit BTP : attributions de service du sous-compte
> *Remplacer cette ligne par :* `![Copie 01](images/CAP-BTP/capture-01.png)`

> **📸 COPIE D'ÉCRAN N°02** — Cloud Connector : connexion au sous-compte en état opérationnel
> *Remplacer cette ligne par :* `![Copie 02](images/CAP-BTP/capture-02.png)`

> **📸 COPIE D'ÉCRAN N°03** — Cloud Connector : ressources exposées du système S/4HANA
> *Remplacer cette ligne par :* `![Copie 03](images/CAP-BTP/capture-03.png)`

### 2.3 Poste de développement

```bash
# Outillage global
npm install --global @sap/cds-dk mbt
npm install --global @sap/approuter

# Client Cloud Foundry
cf --version
cf login -a https://api.cf.<region>.hana.ondemand.com

# Création du projet
cds init purreq-cockpit --add nodejs
cd purreq-cockpit
npm install
```

| Outil | Rôle |
|---|---|
| Node.js | Exécution du service CAP |
| `@sap/cds-dk` | Outils de développement CAP en ligne de commande |
| Cloud Foundry CLI | Connexion et déploiement sur BTP |
| MTA Build Tool | Construction de l'archive de déploiement |
| VS Code + SAP CDS Language Support | Édition des modèles CDS |
| VS Code + SAP Fiori tools | Génération de l'interface |

> **📸 COPIE D'ÉCRAN N°04** — Terminal : versions des outils installés
> *Remplacer cette ligne par :* `![Copie 04](images/CAP-BTP/capture-04.png)`

> **📸 COPIE D'ÉCRAN N°05** — Terminal : connexion Cloud Foundry réussie
> *Remplacer cette ligne par :* `![Copie 05](images/CAP-BTP/capture-05.png)`

---

## 3. Synoptique de la démarche

| N° | Étape | Livrable | Partie |
|---|---|---|---|
| A1 | Initialiser le projet | Arborescence CAP | A |
| A2 | Définir le modèle de données | `db/schema.cds` | A |
| A3 | Définir le service | `srv/purreq-service.cds` | A |
| A4 | Implémenter la logique | `srv/purreq-service.js` | A |
| A5 | Charger des données de test | Fichiers CSV | A |
| A6 | Exécuter en local | Service opérationnel | A |
| B1 | Importer le modèle S/4HANA | `srv/external` | B |
| B2 | Déclarer le service distant | `package.json` | B |
| B3 | Consommer le service source | Logique d'intégration | B |
| C1 | Créer l'instance HANA Cloud | Instance active | C |
| C2 | Configurer la persistance | Conteneur HDI | C |
| C3 | Tester en mode hybride | Liaison locale | C |
| D1 | Définir les rôles | `xs-security.json` | D |
| D2 | Restreindre le service | Annotations d'autorisation | D |
| E1 | Générer l'interface | Application Fiori elements | E |
| E2 | Annoter l'interface | `annotations.cds` | E |
| F1 | Construire l'archive | Fichier MTAR | F |
| F2 | Déployer | Application en ligne | F |
| G1 | Publier dans le launchpad | Tuile et rôles | G |
| H | Recette et exploitation | — | H |

---

## Partie A — Service CAP

### A1 — Initialiser le projet

1. Créer le projet avec l'outil en ligne de commande, en choisissant le moteur Node.js.
2. Installer les dépendances.
3. Ouvrir le dossier dans VS Code et vérifier la reconnaissance des fichiers CDS.
4. Initialiser le dépôt de sources et effectuer un premier enregistrement.

| Dossier | Contenu |
|---|---|
| `db` | Modèle de données et données de test |
| `srv` | Définitions de service et logique applicative |
| `app` | Interfaces utilisateur |
| `package.json` | Dépendances et configuration CAP |

> **📸 COPIE D'ÉCRAN N°06** — VS Code : arborescence du projet CAP initialisé
> *Remplacer cette ligne par :* `![Copie 06](images/CAP-BTP/capture-06.png)`

---

### A2 — Définir le modèle de données

Le modèle s'écrit en CDS. Les aspects réutilisables fournis par le framework apportent l'identifiant technique et les champs d'administration : il est inutile de les redéclarer.

#### Code — `db/schema.cds`

```cds
namespace zpur.tracking;

using { cuid, managed, Currency, CodeList } from '@sap/cds/common';

/**
 * Suivi d'un poste de demande d'achat.
 */
entity PurReqTrackings : cuid, managed {
  @title: 'Demande d''achat'
  @mandatory
  purchaseRequisition     : String(10);

  @title: 'Poste'
  @mandatory
  purchaseRequisitionItem : String(5);

  @title: 'Statut de suivi'
  status                  : Association to TrackingStatuses;

  @title: 'Commentaire'
  comments                : String(250);

  @title: 'Date d''échéance'
  targetDate              : Date;

  @title: 'Montant'
  amount                  : Decimal(15,2);

  currency                : Currency;

  @title: 'Article'
  material                : String(40);
}

/**
 * Liste de codes des statuts de suivi.
 */
entity TrackingStatuses : CodeList {
  key code    : String(2);
      criticality : Integer;
}
```

| Aspect | Apport |
|---|---|
| `cuid` | Clé technique de type identifiant universel |
| `managed` | Champs de création et de dernière modification |
| `Currency` | Devise avec liste de valeurs associée |
| `CodeList` | Liste de codes traduisible, avec libellé et description |

> **📸 COPIE D'ÉCRAN N°07** — VS Code : modèle de données CDS
> *Remplacer cette ligne par :* `![Copie 07](images/CAP-BTP/capture-07.png)`

---

### A3 — Définir le service

Le service expose des projections du modèle. L'activation du draft procure le même comportement de saisie progressive que dans les applications RAP. L'entité d'agrégation alimentera les indicateurs de l'interface.

#### Code — `srv/purreq-service.cds`

```cds
using { zpur.tracking as db } from '../db/schema';

service PurReqTrackingService @(path: '/purreq') {

  @odata.draft.enabled
  entity PurReqTrackings as projection on db.PurReqTrackings
  actions {
    @Common.SideEffects: { TargetProperties: ['in/status_code'] }
    action markAsCompleted() returns PurReqTrackings;
  };

  @readonly
  entity TrackingStatuses as projection on db.TrackingStatuses;

  @readonly
  entity OpenTrackingsByStatus as
    select from db.PurReqTrackings {
      key status.code   as statusCode,
          count(*)      as itemCount : Integer,
          sum(amount)   as totalAmount : Decimal(15,2)
    }
    group by status.code;
}
```

> **📸 COPIE D'ÉCRAN N°08** — VS Code : définition du service CDS
> *Remplacer cette ligne par :* `![Copie 08](images/CAP-BTP/capture-08.png)`

---

### A4 — Implémenter la logique applicative

La logique s'écrit en JavaScript, par des gestionnaires d'événements. Les équivalences avec le modèle ABAP sont directes.

| Notion RAP | Équivalent CAP | Mécanisme |
|---|---|---|
| Détermination | Gestionnaire `before` | Modification des données entrantes |
| Validation | Gestionnaire `before` | Appel de la méthode d'erreur |
| Action | Gestionnaire `on` | Traitement et valeur de retour |
| Contrôle d'autorisation | Annotation de restriction | Déclaratif |
| EML | Requêtes CQL | `SELECT`, `UPDATE`, `INSERT` |

#### Code — `srv/purreq-service.js`

```javascript
const cds = require('@sap/cds');

module.exports = class PurReqTrackingService extends cds.ApplicationService {

  async init() {

    const { PurReqTrackings } = this.entities;

    // Détermination : statut initial à la création
    this.before('CREATE', PurReqTrackings, (req) => {
      if (!req.data.status_code) {
        req.data.status_code = 'OP';
      }
    });

    // Validation : date d'échéance non antérieure à aujourd'hui
    this.before(['CREATE', 'UPDATE'], PurReqTrackings, (req) => {
      const { targetDate } = req.data;
      if (targetDate && new Date(targetDate) < new Date().setHours(0, 0, 0, 0)) {
        req.error({
          code: 'TARGET_DATE_IN_PAST',
          message: 'La date d\'échéance ne peut pas être antérieure à aujourd\'hui.',
          target: 'targetDate',
          status: 400
        });
      }
    });

    // Action métier : clôturer le suivi
    this.on('markAsCompleted', PurReqTrackings, async (req) => {
      const { ID } = req.params[0];

      const current = await SELECT.one.from(PurReqTrackings).where({ ID });
      if (!current) return req.error(404, 'Suivi introuvable.');
      if (current.status_code === 'CL') {
        return req.error(400, 'Ce suivi est déjà clôturé.');
      }

      await UPDATE(PurReqTrackings).set({ status_code: 'CL' }).where({ ID });
      return SELECT.one.from(PurReqTrackings).where({ ID });
    });

    return super.init();
  }
};
```

> **📸 COPIE D'ÉCRAN N°09** — VS Code : implémentation du service
> *Remplacer cette ligne par :* `![Copie 09](images/CAP-BTP/capture-09.png)`

---

### A5 — Charger des données de test

Les fichiers CSV placés dans le dossier de données sont chargés automatiquement au déploiement. Le nom du fichier reprend l'espace de noms et le nom de l'entité.

```
# db/data/zpur.tracking-TrackingStatuses.csv
code;name;descr;criticality
OP;Ouvert;Suivi en cours;2
CL;Clôturé;Suivi terminé;3
CA;Annulé;Suivi annulé;0

# db/data/zpur.tracking-PurReqTrackings.csv
ID;purchaseRequisition;purchaseRequisitionItem;status_code;targetDate;amount;currency_code
11111111-1111-1111-1111-111111111111;0010000123;00010;OP;2026-12-31;1500.00;EUR
22222222-2222-2222-2222-222222222222;0010000124;00010;CL;2026-06-30;3200.50;EUR
```

> **📸 COPIE D'ÉCRAN N°10** — VS Code : fichiers de données de test
> *Remplacer cette ligne par :* `![Copie 10](images/CAP-BTP/capture-10.png)`

> **⚠️ Attention en production**
> Les données CSV sont rechargées à chaque déploiement et écrasent le contenu existant des tables concernées. Les réserver aux listes de codes et aux environnements de développement.

---

### A6 — Exécuter en local

```bash
# Exécution locale avec base en mémoire
cds watch

# Exécution avec persistance SQLite
cds add sqlite
cds deploy --to sqlite
cds watch

# Simulation du service S/4HANA en local
cds mock API_PURCHASEREQ

# Exécution hybride : code local, HANA Cloud distante
cds bind --to purreq-cockpit-db
cds watch --profile hybrid
```

1. Lancer l'exécution locale et ouvrir la page d'accueil du service.
2. Vérifier la présence des entités exposées.
3. Tester la création d'une instance et le déclenchement de la validation.
4. Tester l'action de clôture.
5. Contrôler le document de métadonnées.

> **📸 COPIE D'ÉCRAN N°11** — Navigateur : page d'accueil du service CAP local
> *Remplacer cette ligne par :* `![Copie 11](images/CAP-BTP/capture-11.png)`

> **📸 COPIE D'ÉCRAN N°12** — Navigateur : lecture d'une entité et données de test
> *Remplacer cette ligne par :* `![Copie 12](images/CAP-BTP/capture-12.png)`

> **📸 COPIE D'ÉCRAN N°13** — Terminal : message d'erreur émis par la validation
> *Remplacer cette ligne par :* `![Copie 13](images/CAP-BTP/capture-13.png)`

---

## Partie B — Intégration à S/4HANA

### B1 — Importer le modèle du service source

CAP ne devine pas la structure du service distant : il faut lui fournir son modèle. Celui-ci s'obtient à partir du document de métadonnées du service OData, converti en modèle CDS.

```bash
# Récupérer le fichier EDMX du service S/4HANA
#   API Business Hub, ou :
#   /sap/opu/odata/sap/<SERVICE>/$metadata

# Importer le modèle dans le projet CAP
cds import ./API_PURCHASEREQ_PROCESS_SRV.edmx \
  --as cds \
  --into srv/external/API_PURCHASEREQ.cds

# Contrôler le modèle importé
cds compile srv/external/API_PURCHASEREQ.cds --to sql
```

> **📸 COPIE D'ÉCRAN N°14** — Navigateur : document de métadonnées du service S/4HANA source
> *Remplacer cette ligne par :* `![Copie 14](images/CAP-BTP/capture-14.png)`

> **📸 COPIE D'ÉCRAN N°15** — VS Code : modèle importé dans le dossier des services externes
> *Remplacer cette ligne par :* `![Copie 15](images/CAP-BTP/capture-15.png)`

---

### B2 — Déclarer le service distant

La déclaration associe le modèle importé à une destination. Le nom de la destination doit correspondre exactement à celle définie dans le sous-compte BTP.

#### Code — `package.json`

```json
{
  "name": "purreq-cockpit",
  "dependencies": {
    "@sap/cds": "^8",
    "@sap/xssec": "^4",
    "express": "^4",
    "@cap-js/hana": "^1"
  },
  "cds": {
    "requires": {
      "db": {
        "kind": "hana-cloud"
      },
      "auth": {
        "kind": "xsuaa"
      },
      "API_PURCHASEREQ": {
        "kind": "odata-v2",
        "model": "srv/external/API_PURCHASEREQ",
        "credentials": {
          "destination": "S4H_PURREQ",
          "path": "/sap/opu/odata/sap/API_PURCHASEREQ_PROCESS_SRV"
        }
      }
    },
    "hana": {
      "deploy-format": "hdbtable"
    }
  }
}
```

| Clé | Rôle |
|---|---|
| `kind` | Protocole du service distant |
| `model` | Chemin du modèle importé |
| `credentials.destination` | Nom de la destination dans le sous-compte |
| `credentials.path` | Chemin du service dans le système source |

> **📸 COPIE D'ÉCRAN N°16** — Cockpit BTP : destination vers le système S/4HANA
> *Remplacer cette ligne par :* `![Copie 16](images/CAP-BTP/capture-16.png)`

---

### B3 — Consommer le service source

La connexion au service distant s'établit une fois à l'initialisation. Les requêtes s'écrivent ensuite dans la même syntaxe que pour la base locale : le framework les traduit en appels OData.

#### Code — validation et enrichissement depuis S/4HANA

```javascript
const cds = require('@sap/cds');

module.exports = class PurReqTrackingService extends cds.ApplicationService {

  async init() {

    const { PurReqTrackings } = this.entities;

    // Connexion au service S/4HANA déclaré dans package.json
    const s4 = await cds.connect.to('API_PURCHASEREQ');

    // Validation : le poste de demande d'achat doit exister dans S/4HANA
    this.before(['CREATE', 'UPDATE'], PurReqTrackings, async (req) => {
      const { purchaseRequisition, purchaseRequisitionItem } = req.data;
      if (!purchaseRequisition || !purchaseRequisitionItem) return;

      const item = await s4.run(
        SELECT.one.from('A_PurchaseRequisitionItem').where({
          PurchaseRequisition:     purchaseRequisition,
          PurchaseRequisitionItem: purchaseRequisitionItem
        })
      );

      if (!item) {
        return req.error({
          code: 'PR_NOT_FOUND',
          message: `Le poste ${purchaseRequisition}/${purchaseRequisitionItem} n'existe pas dans S/4HANA.`,
          target: 'purchaseRequisition',
          status: 400
        });
      }

      // Enrichissement depuis le système source
      req.data.material = item.Material;
    });

    return super.init();
  }
};
```

> **📸 COPIE D'ÉCRAN N°17** — VS Code : logique d'intégration au système source
> *Remplacer cette ligne par :* `![Copie 17](images/CAP-BTP/capture-17.png)`

> **📸 COPIE D'ÉCRAN N°18** — Navigateur : création rejetée car le poste n'existe pas dans S/4HANA
> *Remplacer cette ligne par :* `![Copie 18](images/CAP-BTP/capture-18.png)`

---

### B4 — Simuler le système source en local

Le développement local ne dispose pas de la destination. Le framework permet de simuler le service distant à partir du modèle importé, ce qui autorise un développement complet sans connectivité.

```bash
cds mock API_PURCHASEREQ
```

> **📸 COPIE D'ÉCRAN N°19** — Terminal : service distant simulé en local
> *Remplacer cette ligne par :* `![Copie 19](images/CAP-BTP/capture-19.png)`

> **ℹ️ Limite de la simulation**
> Elle valide la structure des appels, pas leur comportement réel. Les écarts de contenu, de volumétrie et de performance n'apparaissent qu'au premier test contre le système réel. Prévoir ce test tôt dans le projet.

---

## Partie C — Persistance HANA Cloud

### C1 — Créer l'instance

1. Ouvrir le cockpit BTP et se placer dans l'espace Cloud Foundry cible.
2. Créer une instance SAP HANA Cloud.
3. Renseigner le mot de passe administrateur et le conserver dans le coffre du projet.
4. Autoriser les connexions depuis les applications de l'environnement.
5. Attendre le démarrage complet et vérifier l'état de l'instance.
6. Configurer l'arrêt automatique hors heures ouvrées sur les environnements hors production.

> **📸 COPIE D'ÉCRAN N°20** — Cockpit BTP : création de l'instance HANA Cloud
> *Remplacer cette ligne par :* `![Copie 20](images/CAP-BTP/capture-20.png)`

> **📸 COPIE D'ÉCRAN N°21** — Cockpit BTP : instance HANA Cloud en état opérationnel
> *Remplacer cette ligne par :* `![Copie 21](images/CAP-BTP/capture-21.png)`

---

### C2 — Configurer la persistance du projet

1. Ajouter la configuration HANA au projet par la commande dédiée.
2. Vérifier la déclaration de la base de données dans la configuration CAP.
3. Contrôler la génération des artefacts de déploiement de la base.
4. Vérifier le format de déploiement des tables.

```bash
cds add hana
```

> **📸 COPIE D'ÉCRAN N°22** — VS Code : configuration de la persistance HANA
> *Remplacer cette ligne par :* `![Copie 22](images/CAP-BTP/capture-22.png)`

---

### C3 — Tester en mode hybride

Le mode hybride exécute le code sur le poste de développement tout en utilisant les services de la plateforme. C'est la façon la plus efficace de mettre au point contre une base réelle sans redéployer à chaque modification.

1. Lier le projet local à l'instance de base de données déployée.
2. Déployer le modèle dans le conteneur.
3. Lancer l'exécution avec le profil hybride.
4. Contrôler la création des tables dans la base.
5. Vérifier le chargement des listes de codes.

> **📸 COPIE D'ÉCRAN N°23** — Terminal : liaison du projet local aux services de la plateforme
> *Remplacer cette ligne par :* `![Copie 23](images/CAP-BTP/capture-23.png)`

> **📸 COPIE D'ÉCRAN N°24** — Terminal : déploiement du modèle dans le conteneur HDI
> *Remplacer cette ligne par :* `![Copie 24](images/CAP-BTP/capture-24.png)`

> **📸 COPIE D'ÉCRAN N°25** — SAP HANA Database Explorer : tables créées dans le conteneur
> *Remplacer cette ligne par :* `![Copie 25](images/CAP-BTP/capture-25.png)`

---

## Partie D — Sécurité et autorisations

### D1 — Définir les rôles

Le modèle d'autorisation repose sur des portées, regroupées en modèles de rôle, eux-mêmes regroupés en collections de rôles affectées aux utilisateurs.

#### Code — `xs-security.json`

```json
{
  "xsappname": "purreq-cockpit",
  "tenant-mode": "dedicated",
  "description": "Suivi des demandes d'achat",
  "scopes": [
    {
      "name": "$XSAPPNAME.Viewer",
      "description": "Consultation des suivis"
    },
    {
      "name": "$XSAPPNAME.Manager",
      "description": "Création et modification des suivis"
    }
  ],
  "role-templates": [
    {
      "name": "Viewer",
      "description": "Consultation",
      "scope-references": [ "$XSAPPNAME.Viewer" ]
    },
    {
      "name": "Manager",
      "description": "Gestion",
      "scope-references": [ "$XSAPPNAME.Viewer", "$XSAPPNAME.Manager" ]
    }
  ],
  "role-collections": [
    {
      "name": "PurReqCockpit_Viewer",
      "description": "Consultation du suivi des demandes d'achat",
      "role-template-references": [ "$XSAPPNAME.Viewer" ]
    },
    {
      "name": "PurReqCockpit_Manager",
      "description": "Gestion du suivi des demandes d'achat",
      "role-template-references": [ "$XSAPPNAME.Manager" ]
    }
  ]
}
```

| Niveau | Rôle |
|---|---|
| `scopes` | Droits élémentaires référencés dans le code |
| `role-templates` | Regroupements de portées |
| `role-collections` | Ce qui est affecté aux utilisateurs |

---

### D2 — Restreindre l'accès au service

Les restrictions s'expriment par annotation, sans code. Elles sont évaluées par le framework à chaque requête, en fonction des portées portées par le jeton de l'utilisateur.

#### Code — `srv/auth.cds`

```cds
using { PurReqTrackingService } from './purreq-service';

annotate PurReqTrackingService with @(requires: 'authenticated-user');

annotate PurReqTrackingService.PurReqTrackings with @(restrict: [
  { grant: 'READ',                      to: 'Viewer'  },
  { grant: ['READ','CREATE','UPDATE','DELETE'], to: 'Manager' },
  { grant: 'markAsCompleted',           to: 'Manager' }
]);

annotate PurReqTrackingService.TrackingStatuses with @(restrict: [
  { grant: 'READ', to: 'authenticated-user' }
]);
```

> **📸 COPIE D'ÉCRAN N°26** — VS Code : annotations d'autorisation du service
> *Remplacer cette ligne par :* `![Copie 26](images/CAP-BTP/capture-26.png)`

---

### D3 — Affecter les collections de rôles

1. Après déploiement, ouvrir le cockpit BTP au niveau du sous-compte.
2. Vérifier la création des collections de rôles déclarées.
3. Affecter la collection à l'utilisateur de test.
4. Se déconnecter puis se reconnecter pour que le jeton soit renouvelé.
5. Vérifier le comportement attendu selon le rôle affecté.

> **📸 COPIE D'ÉCRAN N°27** — Cockpit BTP : collections de rôles créées
> *Remplacer cette ligne par :* `![Copie 27](images/CAP-BTP/capture-27.png)`

> **📸 COPIE D'ÉCRAN N°28** — Cockpit BTP : affectation de la collection à l'utilisateur
> *Remplacer cette ligne par :* `![Copie 28](images/CAP-BTP/capture-28.png)`

> **⚠️ Renouvellement du jeton**
> Une modification de rôle n'est pas prise en compte tant que le jeton en cours n'a pas expiré. Toujours se reconnecter avant de conclure qu'une autorisation ne fonctionne pas.

---

## Partie E — Interface utilisateur

### E1 — Générer l'application

1. Lancer le générateur d'application SAP Fiori depuis VS Code.
2. Choisir le modèle de liste avec page de détail, en version OData V4.
3. Sélectionner comme source de données le projet CAP local.
4. Sélectionner l'entité principale du service.
5. Renseigner le nom du module et le titre de l'application.
6. Ajouter la configuration launchpad : objet sémantique, action et titre de tuile.
7. Vérifier la génération dans le dossier des applications.

> **📸 COPIE D'ÉCRAN N°29** — VS Code : générateur, sélection du projet CAP comme source
> *Remplacer cette ligne par :* `![Copie 29](images/CAP-BTP/capture-29.png)`

> **📸 COPIE D'ÉCRAN N°30** — VS Code : application générée dans le dossier des interfaces
> *Remplacer cette ligne par :* `![Copie 30](images/CAP-BTP/capture-30.png)`

---

### E2 — Annoter l'interface

Comme en ABAP, l'interface est pilotée par annotations. Elles s'écrivent ici dans un fichier CDS du projet, en réutilisant les mêmes vocabulaires.

#### Code — `app/purreq/annotations.cds`

```cds
using PurReqTrackingService as service from '../../srv/purreq-service';

annotate service.PurReqTrackings with @(
  UI: {
    HeaderInfo: {
      TypeName:       'Suivi de DA',
      TypeNamePlural: 'Suivis de DA',
      Title:          { Value: purchaseRequisition },
      Description:    { Value: material }
    },
    SelectionFields: [ purchaseRequisition, status_code, targetDate ],
    LineItem: [
      { Value: purchaseRequisition,     Label: 'Demande d''achat' },
      { Value: purchaseRequisitionItem, Label: 'Poste' },
      { Value: status_code,             Label: 'Statut',
        Criticality: status.criticality },
      { Value: targetDate,              Label: 'Échéance' },
      { Value: amount,                  Label: 'Montant' },
      { $Type: 'UI.DataFieldForAction',
        Action: 'PurReqTrackingService.markAsCompleted',
        Label: 'Clôturer' }
    ],
    Facets: [
      { $Type: 'UI.ReferenceFacet',
        ID: 'General',
        Label: 'Informations générales',
        Target: '@UI.FieldGroup#General' }
    ],
    FieldGroup#General: {
      Data: [
        { Value: purchaseRequisition },
        { Value: purchaseRequisitionItem },
        { Value: status_code },
        { Value: targetDate },
        { Value: amount },
        { Value: comments }
      ]
    }
  }
);

annotate service.PurReqTrackings with {
  status @(
    Common.Text: status.name,
    Common.TextArrangement: #TextOnly,
    ValueList.entity: 'TrackingStatuses'
  );
}
```

> **📸 COPIE D'ÉCRAN N°31** — VS Code : annotations d'interface
> *Remplacer cette ligne par :* `![Copie 31](images/CAP-BTP/capture-31.png)`

> **📸 COPIE D'ÉCRAN N°32** — Navigateur : application exécutée en local avec les annotations
> *Remplacer cette ligne par :* `![Copie 32](images/CAP-BTP/capture-32.png)`

---

### E3 — Configurer l'approuter

L'approuter est le point d'entrée unique de l'application déployée : il assure l'authentification et l'acheminement des requêtes vers le service et vers le référentiel d'interfaces.

#### Code — `xs-app.json`

```json
{
  "welcomeFile": "/index.html",
  "authenticationMethod": "route",
  "routes": [
    {
      "source": "^/purreq/(.*)$",
      "target": "/purreq/$1",
      "destination": "srv-api",
      "authenticationType": "xsuaa",
      "csrfProtection": true
    },
    {
      "source": "^/(.*)$",
      "target": "/$1",
      "service": "html5-apps-repo-rt",
      "authenticationType": "xsuaa"
    }
  ]
}
```

> **📸 COPIE D'ÉCRAN N°33** — VS Code : configuration de l'approuter
> *Remplacer cette ligne par :* `![Copie 33](images/CAP-BTP/capture-33.png)`

---

## Partie F — Construction et déploiement

### F1 — Descripteur de déploiement

L'archive de déploiement regroupe le service, le déployeur de base de données et l'interface, ainsi que les instances de service dont ils dépendent.

#### Code — `mta.yaml`

```yaml
_schema-version: '3.1'
ID: purreq-cockpit
version: 1.0.0

modules:
  - name: purreq-cockpit-srv
    type: nodejs
    path: gen/srv
    parameters:
      buildpack: nodejs_buildpack
    requires:
      - name: purreq-cockpit-db
      - name: purreq-cockpit-auth
      - name: purreq-cockpit-destination
    provides:
      - name: srv-api
        properties:
          srv-url: ${default-url}

  - name: purreq-cockpit-db-deployer
    type: hdb
    path: gen/db
    parameters:
      buildpack: nodejs_buildpack
    requires:
      - name: purreq-cockpit-db

  - name: purreq-cockpit-app-deployer
    type: com.sap.application.content
    path: gen/app
    requires:
      - name: purreq-cockpit-html5-repo

resources:
  - name: purreq-cockpit-db
    type: com.sap.xs.hdi-container
    parameters:
      service: hana
      service-plan: hdi-shared

  - name: purreq-cockpit-auth
    type: org.cloudfoundry.managed-service
    parameters:
      service: xsuaa
      service-plan: application
      path: ./xs-security.json

  - name: purreq-cockpit-destination
    type: org.cloudfoundry.managed-service
    parameters:
      service: destination
      service-plan: lite
```

| Élément | Rôle |
|---|---|
| `modules` | Composants déployés : service, déployeur de base, interface |
| `resources` | Instances de service créées ou réutilisées |
| `requires` | Dépendances d'un module vers une ressource |
| `provides` | Valeurs exposées par un module aux autres |

---

### F2 — Construire et déployer

```bash
# Ajout des artefacts de déploiement
cds add hana,xsuaa,mta,approuter

# Construction de l'archive
mbt build

# Déploiement dans l'espace Cloud Foundry
cf deploy mta_archives/purreq-cockpit_1.0.0.mtar

# Contrôle
cf apps
cf services
cf logs purreq-cockpit-srv --recent
```

1. Ajouter les artefacts de déploiement au projet.
2. Construire l'archive et vérifier l'absence d'erreur.
3. Se connecter à l'espace Cloud Foundry cible.
4. Déployer l'archive.
5. Contrôler l'état des applications et des instances de service.
6. Consulter les journaux en cas d'échec.

> **📸 COPIE D'ÉCRAN N°34** — Terminal : construction de l'archive de déploiement
> *Remplacer cette ligne par :* `![Copie 34](images/CAP-BTP/capture-34.png)`

> **📸 COPIE D'ÉCRAN N°35** — Terminal : déploiement en cours
> *Remplacer cette ligne par :* `![Copie 35](images/CAP-BTP/capture-35.png)`

> **📸 COPIE D'ÉCRAN N°36** — Cockpit BTP : applications déployées et instances de service
> *Remplacer cette ligne par :* `![Copie 36](images/CAP-BTP/capture-36.png)`

> **📸 COPIE D'ÉCRAN N°37** — Navigateur : application déployée, accès par l'approuter
> *Remplacer cette ligne par :* `![Copie 37](images/CAP-BTP/capture-37.png)`

> **ℹ️ Première exécution**
> Le déploiement initial crée les instances de service et peut durer plusieurs minutes. Les suivants sont nettement plus rapides. En cas d'échec, les journaux du déployeur de base de données sont généralement les plus explicites.

---

## Partie G — Publication dans le launchpad

### G1 — Enregistrer l'application

1. Ouvrir le service de launchpad du sous-compte.
2. Vérifier la présence de l'application dans le fournisseur de contenu du référentiel HTML5.
3. Créer ou compléter le catalogue du projet et y ajouter l'application.
4. Créer le groupe ou l'espace destiné aux utilisateurs.
5. Créer le site, ou ajouter le contenu au site existant.
6. Affecter le rôle de site aux collections de rôles concernées.

> **📸 COPIE D'ÉCRAN N°38** — Service de launchpad : fournisseur de contenu et application disponible
> *Remplacer cette ligne par :* `![Copie 38](images/CAP-BTP/capture-38.png)`

> **📸 COPIE D'ÉCRAN N°39** — Service de launchpad : catalogue et groupe du projet
> *Remplacer cette ligne par :* `![Copie 39](images/CAP-BTP/capture-39.png)`

> **📸 COPIE D'ÉCRAN N°40** — Service de launchpad : affectation des rôles de site
> *Remplacer cette ligne par :* `![Copie 40](images/CAP-BTP/capture-40.png)`

---

### G2 — Vérifier l'accès utilisateur

1. Ouvrir l'adresse du site avec l'utilisateur de test.
2. Vérifier la présence de la tuile.
3. Ouvrir l'application et contrôler la remontée des données.
4. Tester la création, la validation et l'action de clôture.
5. Recommencer avec un utilisateur d'un autre rôle et vérifier les restrictions.

> **📸 COPIE D'ÉCRAN N°41** — Launchpad BTP : tuile de l'application visible
> *Remplacer cette ligne par :* `![Copie 41](images/CAP-BTP/capture-41.png)`

> **📸 COPIE D'ÉCRAN N°42** — Launchpad BTP : application ouverte sur données réelles
> *Remplacer cette ligne par :* `![Copie 42](images/CAP-BTP/capture-42.png)`

---

## Partie H — Recette et exploitation

### H1 — Fiche de recette

| N° | Point de contrôle | Résultat attendu | OK / KO |
|---|---|---|---|
| 1 | Projet initialisé et dépendances installées | Conforme | ☐ |
| 2 | Modèle de données et service compilés | Sans erreur | ☐ |
| 3 | Exécution locale opérationnelle | Conforme | ☐ |
| 4 | Détermination du statut initial | Conforme | ☐ |
| 5 | Validation de la date d'échéance | Message correct | ☐ |
| 6 | Action de clôture | Fonctionnelle | ☐ |
| 7 | Modèle du service source importé | Conforme | ☐ |
| 8 | Destination joignable depuis BTP | Test concluant | ☐ |
| 9 | Validation contre le système source | Conforme | ☐ |
| 10 | Instance HANA Cloud opérationnelle | Active | ☐ |
| 11 | Tables créées dans le conteneur | Présentes | ☐ |
| 12 | Listes de codes chargées | Présentes | ☐ |
| 13 | Collections de rôles créées | Présentes | ☐ |
| 14 | Restrictions effectives selon le rôle | Conformes | ☐ |
| 15 | Interface générée et annotée | Conforme | ☐ |
| 16 | Archive construite sans erreur | Conforme | ☐ |
| 17 | Déploiement réussi | Applications démarrées | ☐ |
| 18 | Accès par l'approuter | Authentification demandée | ☐ |
| 19 | Tuile visible dans le launchpad | Visible | ☐ |
| 20 | Comportement conforme pour un second rôle | Conforme | ☐ |

> **📸 COPIE D'ÉCRAN N°43** — Synthèse de recette : application CAP en fonctionnement
> *Remplacer cette ligne par :* `![Copie 43](images/CAP-BTP/capture-43.png)`

### H2 — Livraison vers les autres environnements

| Élément | Mode de propagation |
|---|---|
| Code source | Dépôt de sources, par branche ou étiquette |
| Archive de déploiement | Construite par environnement, ou promue depuis le dépôt |
| Instances de service | Créées par le déploiement dans chaque espace |
| Destinations | Créées manuellement ou par script dans chaque sous-compte |
| Collections de rôles | Créées par le déploiement, affectées manuellement |
| Contenu du launchpad | Configuré par site |

> **⚠️ Différence de fond avec l'ABAP**
> Il n'y a pas d'ordre de transport. La cohérence entre environnements repose sur le dépôt de sources et sur la reproductibilité de la construction. Cette différence doit être intégrée par l'équipe d'exploitation avant la mise en production.

### H3 — Exploitation

| Sujet | Point de vigilance |
|---|---|
| Coût de l'instance HANA Cloud | Arrêt automatique hors heures ouvrées en développement |
| Disponibilité du Cloud Connector | Son arrêt rend l'intégration inopérante |
| Journaux applicatifs | Consultation par la ligne de commande ou le service de journalisation |
| Versions du framework | Mises à jour régulières, à tester avant montée |
| Volumétrie | Surveiller la taille du conteneur HDI |
| Expiration des jetons | Prendre en compte dans les tests d'autorisation |

---

## Annexe A — Commandes utiles

| Commande | Usage |
|---|---|
| `cds init` | Créer un projet |
| `cds watch` | Exécuter en local avec rechargement automatique |
| `cds add hana,xsuaa,mta,approuter` | Ajouter les artefacts de déploiement |
| `cds import` | Importer le modèle d'un service distant |
| `cds mock` | Simuler un service distant en local |
| `cds bind` | Lier le projet local aux services de la plateforme |
| `cds watch --profile hybrid` | Exécuter en local contre les services distants |
| `cds deploy --to sqlite` | Déployer le modèle en base locale |
| `cds compile` | Compiler un modèle et contrôler sa validité |
| `mbt build` | Construire l'archive de déploiement |
| `cf deploy` | Déployer l'archive |
| `cf logs --recent` | Consulter les journaux d'une application |
| `cf services` | Lister les instances de service |

---

## Annexe B — Diagnostic des incidents fréquents

| Symptôme | Cause probable | Action corrective |
|---|---|---|
| Le modèle ne compile pas | Aspect ou entité non importé | Contrôler les instructions d'import du fichier |
| Le service démarre sans entité | Projection non déclarée dans le service | Compléter la définition du service |
| Les données de test ne sont pas chargées | Nom de fichier non conforme | Reprendre espace de noms et nom d'entité |
| Le service distant n'est pas joignable | Destination absente ou mal nommée | Contrôler le nom dans le cockpit et dans la configuration |
| Le Cloud Connector ne répond pas | Ressource non exposée | Autoriser le chemin du service dans le Cloud Connector |
| Le déploiement échoue sur la base | Modification incompatible du modèle | Consulter les journaux du déployeur |
| Erreur d'autorisation après affectation | Jeton non renouvelé | Se déconnecter puis se reconnecter |
| L'application est inaccessible | Approuter mal configuré | Contrôler les routes et le fichier de configuration |
| La tuile n'apparaît pas | Rôle de site non affecté | Affecter le rôle de site à la collection |
| L'instance HANA est arrêtée | Arrêt automatique déclenché | Redémarrer l'instance depuis le cockpit |

---

## Annexe C — Index des copies d'écran

| N° | Chapitre | Contenu attendu |
|---|---|---|
| 01 à 03 | Chapitre 2.2 | Souscriptions et Cloud Connector |
| 04 à 05 | Chapitre 2.3 | Outillage et connexion |
| 06 | Étape A1 | Projet initialisé |
| 07 à 09 | Étapes A2 à A4 | Modèle, service et logique |
| 10 | Étape A5 | Données de test |
| 11 à 13 | Étape A6 | Exécution locale |
| 14 à 19 | Partie B | Intégration au système source |
| 20 à 25 | Partie C | Instance HANA Cloud et persistance |
| 26 à 28 | Partie D | Rôles et autorisations |
| 29 à 33 | Partie E | Interface utilisateur |
| 34 à 37 | Partie F | Construction et déploiement |
| 38 à 42 | Partie G | Publication et accès utilisateur |
| 43 | Partie H | Synthèse de recette |

---

## Annexe D — Historique des versions

| Version | Date | Auteur | Nature des modifications |
|---|---|---|---|
| 1.0 | … | … | Création du document |
| | | | |
