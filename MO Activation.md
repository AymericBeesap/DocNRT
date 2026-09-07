# Mode opératoire — Activation d'une tuile standard SAP Fiori

## Manage Purchase Requisitions (F1048)

**SAP S/4HANA (Private Cloud and On-Premise) 2021 — FPS02**

| Attribut | Valeur |
|---|---|
| Référence du document | MO-FIORI-F1048-2021FPS02-v1.0 |
| Application concernée | Manage Purchase Requisitions (F1048) |
| Version produit | SAP S/4HANA 2021 FPS02 — Private Cloud / On-Premise |
| Type de déploiement | Embedded (front-end intégré au back-end) — à adapter si hub séparé |
| Auteur | … |
| Vérifié par | … |
| Approuvé par | … |
| Date de rédaction | … |
| Statut | Version de travail |

> **Convention de ce document**
> Chaque emplacement de copie d'écran est signalé par un bloc `📸 COPIE D'ÉCRAN N°XX`.
> Déposez vos images dans `images/F1048/` puis remplacez la ligne indiquée par le lien Markdown correspondant.

---

## Sommaire

- [1. Objet et périmètre](#1-objet-et-périmètre)
- [2. Fiche technique de l'application](#2-fiche-technique-de-lapplication)
- [3. Prérequis](#3-prérequis)
- [4. Synoptique de la procédure](#4-synoptique-de-la-procédure)
- [5. Procédure d'activation](#5-procédure-dactivation)
  - [Étape 1 — Vérifier la version du système et les composants logiciels](#étape-1--vérifier-la-version-du-système-et-les-composants-logiciels)
  - [Étape 2 — Relever les informations d'implémentation dans la Fiori Apps Library](#étape-2--relever-les-informations-dimplémentation-dans-la-fiori-apps-library)
  - [Étape 3 — Vérifier la configuration SAP Gateway et l'alias système](#étape-3--vérifier-la-configuration-sap-gateway-et-lalias-système)
  - [Étape 4 — Activer les services ICF](#étape-4--activer-les-services-icf)
  - [Étape 5 — Enregistrer et activer le service OData](#étape-5--enregistrer-et-activer-le-service-odata)
  - [Étape 6 — Tester le service OData](#étape-6--tester-le-service-odata)
  - [Étape 7 — Vérifier le contenu Launchpad (catalogues et groupes)](#étape-7--vérifier-le-contenu-launchpad-catalogues-et-groupes)
  - [Étape 8 — Créer le rôle PFCG et générer le profil d'autorisations](#étape-8--créer-le-rôle-pfcg-et-générer-le-profil-dautorisations)
  - [Étape 9 — Affecter le rôle à l'utilisateur](#étape-9--affecter-le-rôle-à-lutilisateur)
  - [Étape 10 — Configurer les Spaces & Pages](#étape-10--configurer-les-spaces--pages)
  - [Étape 11 — Invalider les caches](#étape-11--invalider-les-caches)
  - [Étape 12 — Vérifier et recetter dans le Launchpad](#étape-12--vérifier-et-recetter-dans-le-launchpad)
- [6. Fiche de recette](#6-fiche-de-recette)
- [7. Transport vers les systèmes cibles](#7-transport-vers-les-systèmes-cibles)
- [Annexe A — Transactions utiles](#annexe-a--transactions-utiles)
- [Annexe B — Diagnostic des incidents fréquents](#annexe-b--diagnostic-des-incidents-fréquents)
- [Annexe C — Index des copies d'écran](#annexe-c--index-des-copies-décran)
- [Annexe D — Historique des versions](#annexe-d--historique-des-versions)

---

## 1. Objet et périmètre

### 1.1 Objet

Ce mode opératoire décrit la procédure complète d'activation de la tuile SAP Fiori standard « Manage Purchase Requisitions » (App ID **F1048**) sur un système SAP S/4HANA 2021 FPS02 (Private Cloud ou On-Premise), afin de la rendre accessible aux utilisateurs finaux depuis le SAP Fiori Launchpad.

### 1.2 Périmètre couvert

- Vérification de la version du système et des composants logiciels installés.
- Relevé des informations d'implémentation dans la SAP Fiori Apps Library.
- Vérification de la configuration Gateway et de l'alias système.
- Activation des services ICF (application SAPUI5 et services d'infrastructure).
- Enregistrement et activation du service OData, puis test via le SAP Gateway Client.
- Identification du catalogue technique, du catalogue métier et du groupe / de la page.
- Création du rôle PFCG, génération du profil d'autorisations et affectation à l'utilisateur.
- Configuration des Spaces & Pages (mode d'affichage recommandé à partir de S/4HANA 2021).
- Invalidation des caches et vérification finale dans le Launchpad.
- Recette technique et fonctionnelle, transport vers les systèmes cibles.

### 1.3 Hors périmètre

- Paramétrage fonctionnel MM (types de document de DA, stratégies de libération, groupes d'acheteurs).
- Configuration du workflow d'approbation des demandes d'achat.
- Installation ou montée de version du serveur front-end (FES) et des composants UI.
- Développements spécifiques et extensions de l'application standard.
- Mise en œuvre du SSO, du reverse proxy (SAP Web Dispatcher) et de la configuration HTTPS.

> **ℹ️ Architecture**
> Ce document est rédigé pour un déploiement **embedded**, le cas le plus fréquent en S/4HANA 2021 On-Premise / Private Cloud. En cas de serveur front-end (FES) séparé, les étapes ICF, OData et rôles PFCG se répartissent entre les deux systèmes ; les paragraphes concernés le précisent.

---

## 2. Fiche technique de l'application

Les informations ci-dessous proviennent de la SAP Fiori Apps Reference Library pour la version cible. Elles doivent impérativement être confirmées sur la fiche de l'application correspondant à votre release (S/4HANA 2021), les objets techniques pouvant évoluer d'une version à l'autre.

**Fiche de référence :** [SAP Fiori Apps Library — F1048](https://pr.alm.me.sap.com/launchpad#FALApp-display&/detail/F1048-S23OP/TwoColumnsMidExpanded)

| Attribut | Valeur | Statut |
|---|---|---|
| Nom de l'application | Manage Purchase Requisitions | Confirmé |
| App ID (Fiori Apps Library) | `F1048` | Confirmé |
| Type d'application | Transactionnelle — SAP Fiori (SAPUI5) | Confirmé |
| Version produit | SAP S/4HANA 2021 FPS02 | Confirmé |
| Domaine fonctionnel | Sourcing & Procurement — Purchase Requisition | Confirmé |
| Composant support SAP | `MM-FIO-PUR-REQ` | Confirmé |
| Base de données | SAP HANA (exclusif) | Confirmé |
| Application BSP (SAPUI5) | `MM_PR_PRCS1` | Confirmé |
| Nœud ICF de l'application | `/sap/bc/ui5_ui5/sap/mm_pr_prcs1` | Confirmé |
| ID composant SAPUI5 | `ui.ssuite.s2p.mm.pur.pr.prcss.s1` | Confirmé |
| Service OData principal | `MM_PUR_PR_PROCESS_SRV` | Confirmé |
| Rôle métier principal | `SAP_BR_PURCHASER` — Purchaser / Acheteur | Confirmé |
| Objet sémantique / action | à relever dans la Fiori Apps Library (onglet Configuration) | **À compléter** |
| Catalogue technique | à relever dans la Fiori Apps Library | **À compléter** |
| Catalogue métier (business catalog) | à relever dans la Fiori Apps Library (famille `SAP_MM_BC_PR_*`) | **À compléter** |
| Groupe métier / page | à relever dans la Fiori Apps Library | **À compléter** |
| Composants logiciels requis | à relever dans l'onglet « Implementation Information » | **À compléter** |

> **⚠️ Important**
> Les lignes « À compléter » sont les seuls éléments à relever manuellement. Elles conditionnent l'**étape 7** (contenu Launchpad) et l'**étape 8** (rôle PFCG) : ne démarrez pas ces étapes sans les avoir renseignées.

> **📸 COPIE D'ÉCRAN N°01** — SAP Fiori Apps Library : fiche de l'application F1048, onglet « App Details » (nom, type, rôle métier)
> *Remplacer cette ligne par :* `![Copie 01](images/F1048/capture-01.png)`

> **📸 COPIE D'ÉCRAN N°02** — SAP Fiori Apps Library : onglet « Implementation Information », section « Required Software Components »
> *Remplacer cette ligne par :* `![Copie 02](images/F1048/capture-02.png)`

> **📸 COPIE D'ÉCRAN N°03** — SAP Fiori Apps Library : onglet « Implementation Information », sections « OData Service » et « SAPUI5 Application »
> *Remplacer cette ligne par :* `![Copie 03](images/F1048/capture-03.png)`

> **📸 COPIE D'ÉCRAN N°04** — SAP Fiori Apps Library : onglet « Configuration », catalogue technique, catalogue métier et groupe
> *Remplacer cette ligne par :* `![Copie 04](images/F1048/capture-04.png)`

---

## 3. Prérequis

### 3.1 Prérequis techniques

| Élément | Exigence |
|---|---|
| Système | SAP S/4HANA 2021 FPS02, base de données SAP HANA |
| Composants UI | Composants front-end SAP Fiori installés et au niveau de SP requis par l'application |
| SAP Gateway | Activé (services d'infrastructure IWFND / IWBEP opérationnels) |
| Launchpad | SAP Fiori Launchpad accessible (`/sap/bc/ui2/flp`) |
| Navigateur | Navigateur supporté par la matrice SAP (Product Availability Matrix) |
| Accès système | Accès SAP GUI + accès HTTP/HTTPS au serveur ICM |

### 3.2 Autorisations nécessaires pour réaliser la procédure

| Domaine | Objets / rôles requis |
|---|---|
| Administration ICF | `S_ICF_ADMIN` — maintenance des services ICF (transaction SICF) |
| Administration Gateway | `S_SERVICE`, `/IWFND/RT_ADMIN` — activation des services OData |
| Administration des rôles | `S_USER_AGR`, `S_USER_TCD`, `S_USER_VAL`, `S_USER_AUT` (transaction PFCG) |
| Administration des utilisateurs | `S_USER_GRP` (transaction SU01) |
| Contenu Launchpad | Accès au Launchpad Designer (`/UI2/FLPD_CUST`, `/UI2/FLPD_CONF`) |
| Transport | `S_TRANSPRT` — création et libération des ordres de transport |

### 3.3 Organisation

- Réaliser l'ensemble de la procédure sur le système de développement, puis transporter.
- Créer au préalable un ordre de transport de customizing et un ordre de type workbench.
- Disposer d'un utilisateur de test représentatif du profil acheteur.
- Disposer d'au moins une demande d'achat en base pour permettre la recette fonctionnelle.

---

## 4. Synoptique de la procédure

| N° | Étape | Transaction / Outil | Système |
|---|---|---|---|
| 1 | Vérifier la version du système et les composants | Système > Statut | Back-end |
| 2 | Relever les informations d'implémentation | Fiori Apps Library | — |
| 3 | Vérifier la configuration Gateway et l'alias système | `/IWFND/MAINT_SERVICE`, `SM59` | Front-end |
| 4 | Activer les services ICF | `SICF` | Front-end |
| 5 | Activer le service OData | `/IWFND/MAINT_SERVICE` | Front-end |
| 6 | Tester le service OData | `/IWFND/GW_CLIENT` | Front-end |
| 7 | Vérifier le contenu Launchpad (catalogues / groupes) | `/UI2/FLPD_CUST` | Front-end |
| 8 | Créer le rôle PFCG et générer le profil | `PFCG` | Front-end + Back-end |
| 9 | Affecter le rôle à l'utilisateur | `SU01` / `PFCG` | Front-end + Back-end |
| 10 | Configurer les Spaces & Pages | `/UI2/FLP_SYS_CONF`, apps de gestion | Front-end |
| 11 | Invalider les caches | `/UI2/INVALIDATE_CACHES` | Front-end |
| 12 | Vérifier et recetter dans le Launchpad | Navigateur | — |

---

## 5. Procédure d'activation

### Étape 1 — Vérifier la version du système et les composants logiciels

#### Objectif

Confirmer que le système cible est bien en SAP S/4HANA 2021 FPS02 et relever le niveau des composants logiciels back-end et front-end, afin de les comparer aux prérequis de l'application.

#### Mode opératoire

1. Se connecter au système SAP via SAP GUI, sur le mandant cible.
2. Dans la barre de menu, choisir **Système > Statut…**
3. Relever le release SAP, le niveau de patch du noyau et le nom du système.
4. Cliquer sur l'icône loupe « Détails du composant » dans la zone « Données système SAP ».
5. Relever les niveaux des composants logiciels et les reporter dans le tableau ci-dessous.

> **📸 COPIE D'ÉCRAN N°05** — SAP GUI : écran Système > Statut (identification du système et du release)
> *Remplacer cette ligne par :* `![Copie 05](images/F1048/capture-05.png)`

> **📸 COPIE D'ÉCRAN N°06** — SAP GUI : liste des composants logiciels installés (Détails du composant)
> *Remplacer cette ligne par :* `![Copie 06](images/F1048/capture-06.png)`

#### Tableau de relevé

| Composant | Valeur attendue (indicative) | Valeur constatée |
|---|---|---|
| `S4CORE` | 106 — SP02 (FPS02) | … |
| `SAP_BASIS` | 756 | … |
| `SAP_ABA` | 756 | … |
| `SAP_UI` | 756 | … |
| `SAP_GWFND` | 756 | … |
| Composants UI applicatifs (`UIS4HOP1`, …) | cf. Fiori Apps Library | … |

> **ℹ️ Contrôle**
> `S4CORE 106` correspond à SAP S/4HANA 2021 ; le niveau SP02 correspond au Feature Pack Stack 02. Si un écart est constaté, ne pas poursuivre et remonter le point à l'équipe Basis.

#### Critère de validation

- Le release et les composants relevés sont conformes aux prérequis de l'application.

---

### Étape 2 — Relever les informations d'implémentation dans la Fiori Apps Library

#### Objectif

Compléter la fiche technique du chapitre 2 avec les objets techniques exacts délivrés pour la version S/4HANA 2021 : services OData, application SAPUI5, catalogues, groupe et rôle métier.

#### Mode opératoire

1. Ouvrir la SAP Fiori Apps Reference Library et rechercher l'application `F1048`.
2. Sélectionner la ligne de version correspondant à SAP S/4HANA 2021.
3. Ouvrir l'onglet « Implementation Information » et relever : composants logiciels requis, service OData, application SAPUI5 (nom BSP et ID composant).
4. Ouvrir la section « Configuration » et relever : catalogue technique, catalogue métier, groupe métier et rôle métier de référence.
5. Reporter l'ensemble de ces valeurs dans le tableau du chapitre 2 et marquer les lignes comme confirmées.

> **📸 COPIE D'ÉCRAN N°07** — Fiori Apps Library : sélection de la version produit S/4HANA 2021 pour l'application F1048
> *Remplacer cette ligne par :* `![Copie 07](images/F1048/capture-07.png)`

> **📸 COPIE D'ÉCRAN N°08** — Fiori Apps Library : récapitulatif des objets techniques relevés (services, catalogues, rôle)
> *Remplacer cette ligne par :* `![Copie 08](images/F1048/capture-08.png)`

#### Critère de validation

- Toutes les lignes « À compléter » du chapitre 2 sont renseignées.

---

### Étape 3 — Vérifier la configuration SAP Gateway et l'alias système

#### Objectif

S'assurer que l'infrastructure SAP Gateway est active et qu'un alias système est correctement défini pour router les appels OData vers le système back-end.

#### Mode opératoire

1. Lancer la transaction `/IWFND/MAINT_SERVICE`.
2. Choisir le bouton **System Alias** (ou lancer la vue de maintenance des alias système).
3. Vérifier l'existence de l'alias utilisé pour les services applicatifs.

```
Déploiement embedded : alias LOCAL, destination RFC = NONE, indicateur « Local GW » coché.
```

4. En déploiement hub, vérifier la destination RFC associée via la transaction `SM59` (test de connexion et test d'autorisation).

> **📸 COPIE D'ÉCRAN N°09** — Transaction `/IWFND/MAINT_SERVICE` : écran de maintenance des alias système
> *Remplacer cette ligne par :* `![Copie 09](images/F1048/capture-09.png)`

> **📸 COPIE D'ÉCRAN N°10** — Transaction `SM59` : test de connexion de la destination RFC vers le back-end (déploiement hub uniquement)
> *Remplacer cette ligne par :* `![Copie 10](images/F1048/capture-10.png)`

#### Critère de validation

- L'alias système est défini et le test de connexion RFC est concluant (le cas échéant).

---

### Étape 4 — Activer les services ICF

#### Objectif

Activer le nœud ICF de l'application SAPUI5 ainsi que les services d'infrastructure indispensables au fonctionnement du Launchpad.

#### 4.1 Activer le service de l'application

1. Lancer la transaction `SICF`.
2. Dans le champ « Nom de service hiérarchique », saisir la valeur ci-dessous puis exécuter (F8).

```
mm_pr_prcs1
```

3. Dérouler l'arborescence jusqu'au nœud : `default_host > sap > bc > ui5_ui5 > sap > mm_pr_prcs1`.
4. Vérifier la couleur du nœud : un nœud inactif apparaît en grisé.
5. Clic droit sur le nœud > **Activer le service**, puis confirmer (bouton « Oui » ou « Oui » avec sous-arborescence).
6. Vérifier que le nœud n'est plus grisé.

> **📸 COPIE D'ÉCRAN N°11** — Transaction `SICF` : écran de sélection, filtre sur le service `mm_pr_prcs1`
> *Remplacer cette ligne par :* `![Copie 11](images/F1048/capture-11.png)`

> **📸 COPIE D'ÉCRAN N°12** — Transaction `SICF` : nœud `/sap/bc/ui5_ui5/sap/mm_pr_prcs1` avant activation (état inactif)
> *Remplacer cette ligne par :* `![Copie 12](images/F1048/capture-12.png)`

> **📸 COPIE D'ÉCRAN N°13** — Transaction `SICF` : menu contextuel « Activer le service » et fenêtre de confirmation
> *Remplacer cette ligne par :* `![Copie 13](images/F1048/capture-13.png)`

> **📸 COPIE D'ÉCRAN N°14** — Transaction `SICF` : nœud actif après activation
> *Remplacer cette ligne par :* `![Copie 14](images/F1048/capture-14.png)`

#### 4.2 Vérifier les services d'infrastructure

Vérifier que les nœuds ICF suivants sont actifs. Ils sont communs à toutes les applications Fiori et sont normalement déjà activés :

| Nœud ICF | Rôle |
|---|---|
| `/sap/bc/ui2/flp` | Point d'entrée du SAP Fiori Launchpad |
| `/sap/bc/ui5_ui5/ui2/ushell` | Shell du Launchpad |
| `/sap/bc/ui2/start_up` | Service de démarrage du Launchpad |
| `/sap/public/bc/ui2` | Ressources publiques UI2 |
| `/sap/public/bc/ui5_ui5` | Ressources publiques SAPUI5 |
| `/sap/bc/bsp/sap/sysinfo` | Informations système |
| `/sap/opu/odata` | Point d'entrée des services OData |

> **📸 COPIE D'ÉCRAN N°15** — Transaction `SICF` : arborescence des services d'infrastructure du Launchpad (état actif)
> *Remplacer cette ligne par :* `![Copie 15](images/F1048/capture-15.png)`

#### 4.3 Contrôle direct par navigateur

Tester l'accès direct à la ressource de l'application depuis un navigateur :

```
https://<hôte>:<port>/sap/bc/ui5_ui5/sap/mm_pr_prcs1/index.html
```

> **📸 COPIE D'ÉCRAN N°16** — Navigateur : réponse du serveur à l'appel direct de l'application SAPUI5
> *Remplacer cette ligne par :* `![Copie 16](images/F1048/capture-16.png)`

#### Critère de validation

- Le nœud `/sap/bc/ui5_ui5/sap/mm_pr_prcs1` est actif.
- Aucune erreur HTTP 403 ou 404 n'est retournée lors de l'appel direct.

---

### Étape 5 — Enregistrer et activer le service OData

#### Objectif

Enregistrer le service OData de l'application sur le serveur front-end et l'associer à l'alias système afin de permettre la lecture et la mise à jour des demandes d'achat.

#### Mode opératoire

1. Lancer la transaction `/IWFND/MAINT_SERVICE`.
2. Choisir le bouton **Add Service** (Ajouter un service).
3. Renseigner l'alias système (par exemple `LOCAL`) puis, dans le filtre « Technical Service Name », saisir la valeur ci-dessous et exécuter.

```
MM_PUR_PR_PROCESS_SRV
```

4. Sélectionner le service dans la liste des services back-end disponibles.
5. Choisir **Add Selected Services** ; renseigner le package (ou choisir « Objet local » en environnement de test) et l'ordre de transport.
6. Valider : le message de confirmation indique la création du modèle et du service.
7. Revenir à l'écran principal, filtrer sur le nom du service et vérifier que le statut ICF est actif (voyant vert).
8. Le cas échéant, activer le nœud ICF du service via le bouton **ICF Node > Activate**.

> **📸 COPIE D'ÉCRAN N°17** — Transaction `/IWFND/MAINT_SERVICE` : écran principal, bouton « Add Service »
> *Remplacer cette ligne par :* `![Copie 17](images/F1048/capture-17.png)`

> **📸 COPIE D'ÉCRAN N°18** — Transaction `/IWFND/MAINT_SERVICE` : recherche du service `MM_PUR_PR_PROCESS_SRV` sur l'alias système
> *Remplacer cette ligne par :* `![Copie 18](images/F1048/capture-18.png)`

> **📸 COPIE D'ÉCRAN N°19** — Transaction `/IWFND/MAINT_SERVICE` : écran d'ajout du service (package et ordre de transport)
> *Remplacer cette ligne par :* `![Copie 19](images/F1048/capture-19.png)`

> **📸 COPIE D'ÉCRAN N°20** — Transaction `/IWFND/MAINT_SERVICE` : message de confirmation de l'enregistrement du service
> *Remplacer cette ligne par :* `![Copie 20](images/F1048/capture-20.png)`

> **📸 COPIE D'ÉCRAN N°21** — Transaction `/IWFND/MAINT_SERVICE` : service enregistré, statut ICF actif et alias système affecté
> *Remplacer cette ligne par :* `![Copie 21](images/F1048/capture-21.png)`

> **ℹ️ Remarque**
> Certains écrans de l'application peuvent solliciter des services OData complémentaires (aides à la saisie, cartes de contexte, pièces jointes). La liste exhaustive figure dans la Fiori Apps Library ; répéter cette étape pour chacun d'eux.

#### Critère de validation

- Le service apparaît dans la liste des services enregistrés.
- Le statut ICF est vert et l'alias système est renseigné.

---

### Étape 6 — Tester le service OData

#### Objectif

Valider techniquement le service avant toute configuration de rôle, afin d'isoler les erreurs d'autorisation des erreurs de service.

#### Mode opératoire

1. Lancer la transaction `/IWFND/GW_CLIENT`.
2. Saisir l'URI de test du document de métadonnées, puis exécuter.

```
/sap/opu/odata/sap/MM_PUR_PR_PROCESS_SRV/$metadata
```

3. Vérifier le code retour **HTTP 200** et l'affichage du document XML de métadonnées.
4. Exécuter éventuellement une requête de lecture sur une entité principale pour valider la remontée des données.
5. En cas d'erreur, consulter les journaux : `/IWFND/ERROR_LOG` côté front-end et `/IWBEP/ERROR_LOG` côté back-end.

> **📸 COPIE D'ÉCRAN N°22** — Transaction `/IWFND/GW_CLIENT` : saisie de l'URI `$metadata`
> *Remplacer cette ligne par :* `![Copie 22](images/F1048/capture-22.png)`

> **📸 COPIE D'ÉCRAN N°23** — Transaction `/IWFND/GW_CLIENT` : réponse HTTP 200 et document de métadonnées
> *Remplacer cette ligne par :* `![Copie 23](images/F1048/capture-23.png)`

> **📸 COPIE D'ÉCRAN N°24** — Transaction `/IWFND/ERROR_LOG` : consultation du journal en cas d'erreur
> *Remplacer cette ligne par :* `![Copie 24](images/F1048/capture-24.png)`

#### Critère de validation

- Le service répond en HTTP 200 et retourne un document de métadonnées valide.

---

### Étape 7 — Vérifier le contenu Launchpad (catalogues et groupes)

#### Objectif

Contrôler que la tuile et le target mapping de l'application sont bien présents dans le catalogue technique livré par SAP, et identifier le catalogue métier à affecter au rôle.

#### Mode opératoire

1. Lancer la transaction `/UI2/FLPD_CUST` (périmètre client) ou `/UI2/FLPD_CONF` (périmètre configuration).
2. Rechercher le catalogue technique relevé à l'étape 2 et vérifier la présence de la tuile de l'application.
3. Ouvrir l'onglet **Target Mappings** et vérifier l'existence du mapping (objet sémantique et action) pointant vers l'application SAPUI5.
4. Rechercher le catalogue métier relevé à l'étape 2 et vérifier qu'il référence bien la tuile du catalogue technique.
5. Le cas échéant, identifier le groupe métier standard contenant la tuile.

> **📸 COPIE D'ÉCRAN N°25** — Launchpad Designer : catalogue technique, onglet « Tiles » avec la tuile de l'application
> *Remplacer cette ligne par :* `![Copie 25](images/F1048/capture-25.png)`

> **📸 COPIE D'ÉCRAN N°26** — Launchpad Designer : onglet « Target Mappings » (objet sémantique, action, application SAPUI5)
> *Remplacer cette ligne par :* `![Copie 26](images/F1048/capture-26.png)`

> **📸 COPIE D'ÉCRAN N°27** — Launchpad Designer : catalogue métier contenant la tuile référencée
> *Remplacer cette ligne par :* `![Copie 27](images/F1048/capture-27.png)`

> **📸 COPIE D'ÉCRAN N°28** — Launchpad Designer : groupe métier standard contenant la tuile
> *Remplacer cette ligne par :* `![Copie 28](images/F1048/capture-28.png)`

> **ℹ️ Bonne pratique**
> Ne jamais modifier les catalogues standard livrés par SAP. Si une adaptation est nécessaire (titre, sous-titre, tuile dynamique), créer un catalogue `Z` par référence au catalogue technique standard.

#### Critère de validation

- La tuile et le target mapping sont présents et correctement paramétrés.
- Le catalogue métier à affecter au rôle est identifié.

---

### Étape 8 — Créer le rôle PFCG et générer le profil d'autorisations

#### Objectif

Créer un rôle applicatif projet contenant le catalogue métier et le groupe de l'application, et générer les autorisations nécessaires à son exécution.

#### 8.1 Créer le rôle à partir du rôle métier standard

1. Lancer la transaction `PFCG`.
2. Saisir le rôle métier standard `SAP_BR_PURCHASER` puis choisir **Copier le rôle**.
3. Nommer le rôle cible selon la convention du projet, par exemple `Z_BR_PURCHASER_F1048`.
4. Saisir une description explicite du rôle et enregistrer.

> **📸 COPIE D'ÉCRAN N°29** — Transaction `PFCG` : écran initial, copie du rôle `SAP_BR_PURCHASER`
> *Remplacer cette ligne par :* `![Copie 29](images/F1048/capture-29.png)`

> **📸 COPIE D'ÉCRAN N°30** — Transaction `PFCG` : saisie du nom et de la description du rôle Z
> *Remplacer cette ligne par :* `![Copie 30](images/F1048/capture-30.png)`

> **ℹ️ Alternative**
> Pour un périmètre strictement limité à l'application F1048, créer un rôle vierge et n'y ajouter que le catalogue métier concerné, plutôt que de copier l'intégralité du rôle métier standard.

#### 8.2 Ajouter le catalogue et le groupe dans le menu du rôle

1. Ouvrir le rôle en modification puis l'onglet **Menu**.
2. Cliquer sur la flèche à côté du bouton **Ajouter** et choisir **Catalogue SAP Fiori Launchpad**.
3. Sélectionner le catalogue métier identifié à l'étape 7 et valider.
4. Répéter l'opération en choisissant **Groupe SAP Fiori Launchpad** pour ajouter le groupe.
5. Enregistrer le rôle.

> **📸 COPIE D'ÉCRAN N°31** — Transaction `PFCG` : onglet « Menu », option « Catalogue SAP Fiori Launchpad »
> *Remplacer cette ligne par :* `![Copie 31](images/F1048/capture-31.png)`

> **📸 COPIE D'ÉCRAN N°32** — Transaction `PFCG` : sélection du catalogue métier de l'application
> *Remplacer cette ligne par :* `![Copie 32](images/F1048/capture-32.png)`

> **📸 COPIE D'ÉCRAN N°33** — Transaction `PFCG` : menu du rôle après ajout du catalogue et du groupe
> *Remplacer cette ligne par :* `![Copie 33](images/F1048/capture-33.png)`

#### 8.3 Générer le profil d'autorisations

1. Ouvrir l'onglet **Autorisations** puis **Modifier les données d'autorisation**.
2. Traiter les zones organisationnelles : organisation d'achat, groupe d'acheteurs, division.
3. Vérifier la présence des objets d'autorisation back-end listés ci-dessous et renseigner leurs valeurs.
4. Compléter l'objet `S_SERVICE`, qui contrôle le droit d'exécution du service OData.
5. Générer le profil (icône de génération), puis enregistrer le rôle.

| Objet | Description | Système |
|---|---|---|
| `M_BANF_BSA` | Demande d'achat : type de document | Back-end |
| `M_BANF_EKG` | Demande d'achat : groupe d'acheteurs | Back-end |
| `M_BANF_EKO` | Demande d'achat : organisation d'achat | Back-end |
| `M_BANF_WRK` | Demande d'achat : division | Back-end |
| `M_BANF_FRG` | Demande d'achat : code de libération | Back-end |
| `S_SERVICE` | Contrôle de démarrage des services externes | Front-end |
| `S_RFC` | Contrôle des appels RFC (déploiement hub) | Back-end |

> **📸 COPIE D'ÉCRAN N°34** — Transaction `PFCG` : onglet « Autorisations », arborescence des objets d'autorisation
> *Remplacer cette ligne par :* `![Copie 34](images/F1048/capture-34.png)`

> **📸 COPIE D'ÉCRAN N°35** — Transaction `PFCG` : saisie des valeurs des niveaux organisationnels (organisation d'achat, groupe d'acheteurs, division)
> *Remplacer cette ligne par :* `![Copie 35](images/F1048/capture-35.png)`

> **📸 COPIE D'ÉCRAN N°36** — Transaction `PFCG` : objets d'autorisation `M_BANF_*` renseignés
> *Remplacer cette ligne par :* `![Copie 36](images/F1048/capture-36.png)`

> **📸 COPIE D'ÉCRAN N°37** — Transaction `PFCG` : profil généré, voyant vert sur l'onglet « Autorisations »
> *Remplacer cette ligne par :* `![Copie 37](images/F1048/capture-37.png)`

#### Critère de validation

- Le rôle contient le catalogue et le groupe de l'application.
- Le profil est généré et l'onglet « Autorisations » affiche un voyant vert.

---

### Étape 9 — Affecter le rôle à l'utilisateur

#### Objectif

Rendre l'application accessible à l'utilisateur de test, puis aux utilisateurs finaux.

#### Mode opératoire

1. Dans la transaction `PFCG`, ouvrir l'onglet **Utilisateur**.
2. Saisir l'identifiant de l'utilisateur de test.
3. Cliquer sur **Comparaison utilisateur** puis choisir **Comparaison complète**.
4. Vérifier que le voyant de l'onglet « Utilisateur » passe au vert.
5. Contrôler l'affectation via la transaction `SU01`, onglet **Rôles**.
6. En déploiement hub, affecter également le rôle back-end correspondant sur le système ERP.

> **📸 COPIE D'ÉCRAN N°38** — Transaction `PFCG` : onglet « Utilisateur », affectation de l'utilisateur de test
> *Remplacer cette ligne par :* `![Copie 38](images/F1048/capture-38.png)`

> **📸 COPIE D'ÉCRAN N°39** — Transaction `PFCG` : résultat de la comparaison utilisateur (voyant vert)
> *Remplacer cette ligne par :* `![Copie 39](images/F1048/capture-39.png)`

> **📸 COPIE D'ÉCRAN N°40** — Transaction `SU01` : onglet « Rôles » de l'utilisateur de test
> *Remplacer cette ligne par :* `![Copie 40](images/F1048/capture-40.png)`

#### Critère de validation

- Le rôle est affecté et la comparaison utilisateur est à jour.

---

### Étape 10 — Configurer les Spaces & Pages

#### Objectif

À partir de SAP S/4HANA 2021, le mode d'affichage recommandé du Launchpad repose sur les espaces (spaces) et les pages, en remplacement des groupes. Cette étape n'est requise que si ce mode est activé sur votre système.

#### Mode opératoire

1. Vérifier l'état du paramètre d'activation des espaces dans la configuration du Launchpad (transaction `/UI2/FLP_SYS_CONF` ou `/UI2/FLP_CUS_CONF`).
2. Si le mode espaces est actif, créer ou identifier la page contenant la tuile de l'application via l'application de gestion des pages du Launchpad.
3. Affecter la page à l'espace concerné via l'application de gestion des espaces.
4. Affecter l'espace au rôle PFCG créé à l'étape 8 (onglet **Menu > Espace SAP Fiori Launchpad**).
5. Enregistrer et régénérer le profil du rôle.

> **📸 COPIE D'ÉCRAN N°41** — Configuration du Launchpad : paramètre d'activation du mode Spaces & Pages
> *Remplacer cette ligne par :* `![Copie 41](images/F1048/capture-41.png)`

> **📸 COPIE D'ÉCRAN N°42** — Application de gestion des pages : page contenant la tuile de l'application
> *Remplacer cette ligne par :* `![Copie 42](images/F1048/capture-42.png)`

> **📸 COPIE D'ÉCRAN N°43** — Application de gestion des espaces : affectation de la page à l'espace
> *Remplacer cette ligne par :* `![Copie 43](images/F1048/capture-43.png)`

> **📸 COPIE D'ÉCRAN N°44** — Transaction `PFCG` : affectation de l'espace au rôle
> *Remplacer cette ligne par :* `![Copie 44](images/F1048/capture-44.png)`

> **ℹ️ Si le mode espaces n'est pas activé**
> Le Launchpad continue de fonctionner en mode groupes ; l'étape 10 est alors sans objet et le groupe ajouté à l'étape 8.2 suffit.

---

### Étape 11 — Invalider les caches

#### Objectif

Forcer la prise en compte immédiate des modifications de contenu et d'autorisations dans le Launchpad.

#### Mode opératoire

1. Lancer la transaction `/UI2/INVALIDATE_CACHES` et confirmer l'invalidation des caches du mandant.
2. Lancer la transaction `/UI2/INVALIDATE_GLOBAL_CACHES` pour les caches globaux.
3. Côté poste client, vider le cache du navigateur puis recharger la page en forçant l'actualisation.
4. Demander à l'utilisateur de test de se déconnecter et de se reconnecter au Launchpad.

| Transaction | Portée |
|---|---|
| `/UI2/INVALIDATE_CACHES` | Caches du mandant courant |
| `/UI2/INVALIDATE_GLOBAL_CACHES` | Caches globaux (tous mandants) |
| `/UI2/DELETE_CACHE` | Suppression du cache UI2 |
| `/UI2/CHIP_SYNC` | Synchronisation du catalogue des CHIPs |

> **📸 COPIE D'ÉCRAN N°45** — Transaction `/UI2/INVALIDATE_CACHES` : écran d'exécution et message de confirmation
> *Remplacer cette ligne par :* `![Copie 45](images/F1048/capture-45.png)`

---

### Étape 12 — Vérifier et recetter dans le Launchpad

#### Objectif

Confirmer l'affichage de la tuile et le bon fonctionnement de l'application pour l'utilisateur cible.

#### Mode opératoire

1. Ouvrir le SAP Fiori Launchpad avec l'utilisateur de test.

```
https://<hôte>:<port>/sap/bc/ui2/flp
```

2. Vérifier la présence de la tuile « Manage Purchase Requisitions » dans le groupe ou la page attendue.
3. Si la tuile n'est pas visible, contrôler sa présence via **Personnaliser / Ajouter des applications**.
4. Cliquer sur la tuile et vérifier l'ouverture de l'application sans message d'erreur.
5. Lancer une recherche de demandes d'achat et vérifier la remontée des données.
6. Ouvrir le détail d'une demande d'achat et vérifier l'affichage des postes.
7. Tester l'affectation d'une source d'approvisionnement sur un poste.
8. Vérifier les navigations sortantes vers les applications liées.

> **📸 COPIE D'ÉCRAN N°46** — Launchpad : page d'accueil avec la tuile « Manage Purchase Requisitions » visible
> *Remplacer cette ligne par :* `![Copie 46](images/F1048/capture-46.png)`

> **📸 COPIE D'ÉCRAN N°47** — Application F1048 : écran de liste des demandes d'achat après lancement
> *Remplacer cette ligne par :* `![Copie 47](images/F1048/capture-47.png)`

> **📸 COPIE D'ÉCRAN N°48** — Application F1048 : détail d'une demande d'achat et de ses postes
> *Remplacer cette ligne par :* `![Copie 48](images/F1048/capture-48.png)`

> **📸 COPIE D'ÉCRAN N°49** — Application F1048 : affectation d'une source d'approvisionnement sur un poste
> *Remplacer cette ligne par :* `![Copie 49](images/F1048/capture-49.png)`

#### Critère de validation

- La tuile est visible et l'application s'ouvre sans erreur.
- Les données remontent conformément aux autorisations de l'utilisateur.

---

## 6. Fiche de recette

| N° | Point de contrôle | Résultat attendu | OK / KO — Commentaire |
|---|---|---|---|
| 1 | Version du système et composants logiciels | Conformes | ☐ |
| 2 | Informations d'implémentation relevées | Fiche complétée | ☐ |
| 3 | Alias système Gateway configuré | Test concluant | ☐ |
| 4 | Nœud ICF de l'application actif | Actif | ☐ |
| 5 | Services ICF d'infrastructure actifs | Actifs | ☐ |
| 6 | Service OData enregistré et actif | Statut vert | ☐ |
| 7 | Test `$metadata` du service OData | HTTP 200 | ☐ |
| 8 | Tuile et target mapping présents | Présents | ☐ |
| 9 | Rôle PFCG créé et profil généré | Voyant vert | ☐ |
| 10 | Rôle affecté à l'utilisateur de test | Comparaison à jour | ☐ |
| 11 | Espace / page configuré (si applicable) | Configuré | ☐ |
| 12 | Caches invalidés | Exécuté | ☐ |
| 13 | Tuile visible dans le Launchpad | Visible | ☐ |
| 14 | Ouverture de l'application | Sans erreur | ☐ |
| 15 | Remontée et affichage des données | Conforme | ☐ |
| 16 | Affectation d'une source d'approvisionnement | Fonctionnelle | ☐ |

> **📸 COPIE D'ÉCRAN N°50** — Synthèse de recette : capture globale du Launchpad avec la tuile en production
> *Remplacer cette ligne par :* `![Copie 50](images/F1048/capture-50.png)`

---

## 7. Transport vers les systèmes cibles

| Objet | Mode de propagation |
|---|---|
| Activation des services ICF | Non transportable — à réaliser manuellement dans chaque système |
| Enregistrement du service OData | Transportable si un package et un ordre ont été renseignés ; l'activation du nœud ICF reste manuelle |
| Alias système Gateway | Spécifique à chaque système — à créer manuellement |
| Catalogues et groupes Launchpad | Ordre de customizing (périmètre client) ou workbench (périmètre configuration) |
| Rôle PFCG | Ordre de customizing ; ne pas transporter les affectations utilisateurs |
| Espaces et pages | Ordre de customizing |

> **⚠️ Après import**
> Exécuter systématiquement l'invalidation des caches dans le système cible, puis rejouer la fiche de recette du chapitre 6 avec un utilisateur représentatif.

---

## Annexe A — Transactions utiles

| Transaction | Usage |
|---|---|
| `SICF` | Maintenance et activation des services ICF |
| `/IWFND/MAINT_SERVICE` | Enregistrement et activation des services OData |
| `/IWFND/GW_CLIENT` | Test des services OData |
| `/IWFND/ERROR_LOG` | Journal des erreurs Gateway côté front-end |
| `/IWBEP/ERROR_LOG` | Journal des erreurs Gateway côté back-end |
| `/UI2/FLPD_CUST` | Launchpad Designer — périmètre client |
| `/UI2/FLPD_CONF` | Launchpad Designer — périmètre configuration |
| `/UI2/FLP` | Lancement du Launchpad depuis SAP GUI |
| `/UI2/INVALIDATE_CACHES` | Invalidation des caches du mandant |
| `/UI2/INVALIDATE_GLOBAL_CACHES` | Invalidation des caches globaux |
| `/UI2/FLP_SYS_CONF` | Paramétrage système du Launchpad |
| `PFCG` | Maintenance des rôles et des autorisations |
| `SU01` | Maintenance des utilisateurs |
| `SU53` | Analyse du dernier contrôle d'autorisation en échec |
| `STAUTHTRACE` | Trace des contrôles d'autorisation |
| `ST22` | Analyse des dumps ABAP |
| `SM59` | Maintenance des destinations RFC |
| `SMICM` | Monitoring de l'ICM (ports HTTP/HTTPS) |

---

## Annexe B — Diagnostic des incidents fréquents

| Symptôme | Cause probable | Action corrective |
|---|---|---|
| La tuile n'apparaît pas dans le Launchpad | Catalogue ou groupe non affecté au rôle, profil non généré, cache non invalidé | Contrôler l'onglet Menu du rôle (étape 8.2), régénérer le profil, rejouer l'étape 11 |
| Erreur HTTP 404 au clic sur la tuile | Nœud ICF de l'application inactif ou target mapping erroné | Rejouer l'étape 4, puis vérifier le target mapping (étape 7) |
| Erreur HTTP 403 — Forbidden | Autorisation manquante sur l'objet `S_SERVICE` | Analyser via `SU53` ou `STAUTHTRACE`, compléter le rôle et régénérer le profil |
| Erreur « Service not found » sur l'appel OData | Service non enregistré ou alias système absent | Rejouer les étapes 3 et 5 |
| Erreur HTTP 500 à l'ouverture de l'application | Exception côté back-end | Analyser `ST22`, `/IWFND/ERROR_LOG` et `/IWBEP/ERROR_LOG` ; rechercher la note SAP correspondante |
| L'application s'ouvre mais aucune donnée n'est affichée | Autorisations `M_BANF_*` insuffisantes ou absence de données correspondant aux filtres | Compléter les valeurs organisationnelles du rôle, élargir les critères de sélection |
| Écran blanc au lancement | Cache navigateur ou cache UI2 obsolète | Vider le cache navigateur, rejouer l'étape 11 |
| Message « Reference lost » dans le Launchpad Designer | Catalogue technique non répliqué ou incomplet | Vérifier la disponibilité du catalogue technique et la réplication du contenu |
| Navigation vers une application liée impossible | Target mapping absent dans le catalogue affecté | Ajouter le catalogue de l'application cible au rôle |

---

## Annexe C — Index des copies d'écran

| N° | Chapitre / Étape | Contenu attendu |
|---|---|---|
| 01 à 04 | Chapitre 2 — Fiche technique | Fiches de la SAP Fiori Apps Library |
| 05 à 06 | Étape 1 | Statut système et composants logiciels |
| 07 à 08 | Étape 2 | Informations d'implémentation et de configuration |
| 09 à 10 | Étape 3 | Alias système Gateway et destination RFC |
| 11 à 16 | Étape 4 | Activation des services ICF et contrôle navigateur |
| 17 à 21 | Étape 5 | Enregistrement et activation du service OData |
| 22 à 24 | Étape 6 | Test du service via le Gateway Client |
| 25 à 28 | Étape 7 | Catalogues, target mapping et groupe |
| 29 à 37 | Étape 8 | Création du rôle, menu et autorisations |
| 38 à 40 | Étape 9 | Affectation du rôle à l'utilisateur |
| 41 à 44 | Étape 10 | Configuration des espaces et des pages |
| 45 | Étape 11 | Invalidation des caches |
| 46 à 49 | Étape 12 | Vérification et recette dans le Launchpad |
| 50 | Chapitre 6 | Synthèse de recette |

> La numérotation ci-dessus suppose que toutes les copies d'écran proposées sont insérées. Ajustez-la si vous en supprimez ou en ajoutez.

---

## Annexe D — Historique des versions

| Version | Date | Auteur | Nature des modifications |
|---|---|---|---|
| 1.0 | … | … | Création du document |
| | | | |
