# Mode opératoire — Extension d'une application SAP Fiori standard

## Manage Purchase Requisitions (F1048)

**Données · Logique métier · Interface**
**SAP S/4HANA (Private Cloud and On-Premise) 2021 — FPS02**

| Attribut | Valeur |
|---|---|
| Référence du document | MO-FIORI-F1048-EXT-2021FPS02-v1.0 |
| Document lié | [MO-FIORI-F1048-2021FPS02-v1.0 — activation de la tuile](MO%20Activation.md) |
| Application concernée | Manage Purchase Requisitions (F1048) |
| Version produit | SAP S/4HANA 2021 FPS02 — Private Cloud / On-Premise |
| Couches d'extension traitées | Modèle de données · Logique métier (BAdI) · Interface (adaptation project) |
| Outils | Eclipse ADT · SAP GUI · Visual Studio Code + SAP Fiori tools |
| Auteur | … |
| Vérifié par | … |
| Approuvé par | … |
| Date de rédaction | … |
| Statut | Version de travail |

> **Convention de ce document**
> Chaque emplacement de copie d'écran est signalé par un bloc `📸 COPIE D'ÉCRAN N°XX`.
> Déposez vos images dans `images/F1048-EXT/` puis remplacez la ligne indiquée par le lien Markdown correspondant.

---

## Sommaire

- [1. Objet et périmètre](#1-objet-et-périmètre)
- [2. Rappels techniques sur l'application de base](#2-rappels-techniques-sur-lapplication-de-base)
- [3. Stratégie d'extension](#3-stratégie-dextension)
- [4. Prérequis et outillage](#4-prérequis-et-outillage)
- [5. Synoptique de la démarche](#5-synoptique-de-la-démarche)
- [Partie A — Extension du modèle de données](#partie-a--extension-du-modèle-de-données)
  - [A1 — Créer l'élément de données](#a1--créer-lélément-de-données)
  - [A2 — Étendre l'include client CI_EBANDB](#a2--étendre-linclude-client-ci_ebandb)
  - [A3 — Autoriser la modification du champ par la logique métier](#a3--autoriser-la-modification-du-champ-par-la-logique-métier)
  - [A4 — Étendre les vues CDS de lecture](#a4--étendre-les-vues-cds-de-lecture)
  - [A5 — Exposer le champ dans le service OData](#a5--exposer-le-champ-dans-le-service-odata)
  - [A6 — Tester le service étendu](#a6--tester-le-service-étendu)
- [Partie B — Extension de la logique métier](#partie-b--extension-de-la-logique-métier)
  - [B1 — Identifier le point d'extension](#b1--identifier-le-point-dextension)
  - [B2 — Créer l'implémentation du BAdI](#b2--créer-limplémentation-du-badi)
  - [B3 — Implémenter les méthodes](#b3--implémenter-les-méthodes)
  - [B4 — Tester la logique métier](#b4--tester-la-logique-métier)
- [Partie C — Extension de l'interface utilisateur](#partie-c--extension-de-linterface-utilisateur)
  - [C1 — Créer le projet d'adaptation](#c1--créer-le-projet-dadaptation)
  - [C2 — Définir les modifications d'interface](#c2--définir-les-modifications-dinterface)
  - [C3 — Prévisualiser la variante](#c3--prévisualiser-la-variante)
  - [C4 — Déployer la variante d'application](#c4--déployer-la-variante-dapplication)
  - [C5 — Publier la tuile de la variante](#c5--publier-la-tuile-de-la-variante)
- [Partie D — Recette, transport et montée de version](#partie-d--recette-transport-et-montée-de-version)
- [Annexe A — Outils et transactions](#annexe-a--outils-et-transactions)
- [Annexe B — Diagnostic des incidents fréquents](#annexe-b--diagnostic-des-incidents-fréquents)
- [Annexe C — Index des copies d'écran](#annexe-c--index-des-copies-décran)
- [Annexe D — Historique des versions](#annexe-d--historique-des-versions)

---

## 1. Objet et périmètre

### 1.1 Objet

Ce mode opératoire décrit la démarche d'extension de l'application SAP Fiori standard « Manage Purchase Requisitions » (App ID **F1048**) sur SAP S/4HANA 2021 FPS02, selon les trois couches classiques d'une extension : le modèle de données, la logique métier et l'interface utilisateur.

Il prend pour prérequis que la tuile standard est déjà activée et fonctionnelle, conformément au mode opératoire d'activation.

### 1.2 Cas d'usage de référence

Pour rendre la procédure concrète, l'ensemble du document s'appuie sur un exemple fil rouge : l'ajout d'un champ personnalisé « Motif d'achat » au poste de demande d'achat, son alimentation et son contrôle par de la logique métier, puis son affichage dans l'application Fiori. Les noms d'objets sont donnés à titre d'exemple et doivent être adaptés à la convention de nommage du projet.

| Objet | Exemple utilisé | Convention projet |
|---|---|---|
| Élément de données | `ZZ_MOTIF_ACHAT` | … |
| Champ dans EBAN | `ZZMOTIF_ACHAT` | … |
| Implémentation BAdI | `ZME_PROCESS_REQ_CUST` | … |
| Classe d'implémentation | `ZCL_IM_ME_PROCESS_REQ_CUST` | … |
| Projet d'adaptation UI | `zf1048.appvar` | … |
| Repository SAPUI5 ABAP | `ZF1048_EXT` | … |
| Package de développement | `Z_MM_PUR_EXT` | … |

### 1.3 Périmètre couvert

- Choix de la stratégie d'extension et arbitrage entre extensibilité key user et extensibilité développeur.
- **Partie A** — extension du modèle de données : élément de données, include client `CI_EBANDB`, structure d'autorisation de modification, vues CDS, exposition OData.
- **Partie B** — extension de la logique métier : implémentation du BAdI de traitement des demandes d'achat, contrôles et messages.
- **Partie C** — extension de l'interface : projet d'adaptation SAPUI5, variante d'application, déploiement et publication de la tuile.
- **Partie D** — recette, transport et gestion des montées de version.

### 1.4 Hors périmètre

- Activation initiale de la tuile standard (voir le mode opératoire lié).
- Extension du workflow d'approbation des demandes d'achat.
- Développement d'une application Fiori entièrement nouvelle.
- Extension des API publiques et des scénarios d'intégration.
- Mise en place de la chaîne CI/CD et de la gestion de sources Git.

> **ℹ️ Principe directeur**
> Toute extension doit rester **sans modification du standard**. Les objets SAP ne sont jamais modifiés directement : on utilise les points d'extension prévus (include client, append, BAdI, variante d'application). Une modification du standard fait perdre le bénéfice des corrections SAP et complique chaque montée de version.

---

## 2. Rappels techniques sur l'application de base

| Attribut | Valeur |
|---|---|
| App ID | `F1048` |
| Type d'application | Transactionnelle — SAP Fiori (SAPUI5) freestyle |
| Application BSP (SAPUI5) | `MM_PR_PRCS1` |
| Nœud ICF | `/sap/bc/ui5_ui5/sap/mm_pr_prcs1` |
| ID composant SAPUI5 | `ui.ssuite.s2p.mm.pur.pr.prcss.s1` |
| Service OData principal | `MM_PUR_PR_PROCESS_SRV` |
| Rôle métier principal | `SAP_BR_PURCHASER` |
| Table applicative principale | `EBAN` — postes de demande d'achat |
| Include client de EBAN | `CI_EBANDB` |
| Composant support SAP | `MM-FIO-PUR-REQ` |

> **⚠️ Point structurant**
> F1048 est une application SAPUI5 **freestyle** et non une application Fiori Elements. L'extension de son interface passe donc obligatoirement par un projet d'adaptation (variante d'application) et non par de simples annotations d'interface utilisateur.

---

## 3. Stratégie d'extension

### 3.1 Les trois couches d'extension

| Couche | Objet de l'extension | Technique | Outil |
|---|---|---|---|
| Données | Ajouter une information non prévue par le standard | Include client, append, extension de vue CDS, exposition OData | SAP GUI / Eclipse ADT |
| Logique | Alimenter, contrôler, dériver la donnée | Implémentation de BAdI | Eclipse ADT / SAP GUI |
| Interface | Afficher, saisir, réorganiser | Projet d'adaptation SAPUI5 (variante d'application) | VS Code + SAP Fiori tools |

### 3.2 Extensibilité key user ou extensibilité développeur

Avant d'engager un développement, évaluer si le besoin est couvert par l'extensibilité in-app (applications « Champs personnalisés » et « Logique personnalisée »). Cette voie est plus rapide, sans transport de code et sans reprise à chaque montée de version, mais elle est limitée aux contextes métier et aux applications déclarés extensibles par SAP.

| Critère | Extensibilité key user | Extensibilité développeur |
|---|---|---|
| Compétence requise | Utilisateur clé fonctionnel | Développeur ABAP / SAPUI5 |
| Objets créés | Générés par le framework | Objets Z transportables |
| Périmètre | Contextes métier déclarés extensibles | Tout point d'extension ouvert |
| Publication dans les UI | Automatique pour les applications supportées | Manuelle, application par application |
| Application F1048 | À vérifier : application SAPUI5 freestyle, la publication automatique n'est pas garantie | **Voie retenue dans ce document** |
| Effort de montée de version | Faible | Moyen à élevé — retest requis |

> **📸 COPIE D'ÉCRAN N°01** — Application « Champs personnalisés » : recherche du contexte métier lié à la demande d'achat, pour arbitrage entre les deux voies
> *Remplacer cette ligne par :* `![Copie 01](images/F1048-EXT/capture-01.png)`

### 3.3 Décision retenue

| Question | Réponse à documenter |
|---|---|
| Voie retenue | ☐ Key user  ☐ Développeur  ☐ Mixte |
| Justification | … |
| Couches concernées | ☐ Données  ☐ Logique  ☐ Interface |
| Validé par | … |
| Date | … |

---

## 4. Prérequis et outillage

### 4.1 Prérequis fonctionnels et techniques

- La tuile standard F1048 est activée et testée (mode opératoire lié, fiche de recette validée).
- Le besoin métier est spécifié : champ, format, règles de gestion, comportement attendu dans l'interface.
- Le système de développement est ouvert aux modifications (transactions `SCC4` et `SE06`).
- Une clé de développeur est disponible pour l'utilisateur qui réalise les travaux.
- Les ordres de transport de workbench et de customizing sont créés.

### 4.2 Outillage par couche

| Couche | Outil principal | Compléments |
|---|---|---|
| Données | Eclipse + ABAP Development Tools | SAP GUI : `SE11`, `SE14`, `SEGW` |
| Logique métier | Eclipse + ABAP Development Tools | SAP GUI : `SE18`, `SE19`, `SE24` |
| Interface | Visual Studio Code + SAP Fiori tools | Alternative : SAP Business Application Studio |
| Publication | SAP GUI | `SICF`, `/IWFND/MAINT_SERVICE`, `/UI2/FLPD_CUST`, `PFCG` |

### 4.3 Installation du poste de développement

#### Eclipse et ABAP Development Tools

1. Installer une version d'Eclipse supportée par les ABAP Development Tools.
2. Ajouter le site de mise à jour des ABAP Development Tools et installer le plug-in.
3. Créer un projet ABAP pointant vers le système de développement et le mandant cible.
4. Vérifier l'ouverture d'un objet standard en affichage pour valider la connexion.

> **📸 COPIE D'ÉCRAN N°02** — Eclipse ADT : création du projet ABAP et connexion au système de développement
> *Remplacer cette ligne par :* `![Copie 02](images/F1048-EXT/capture-02.png)`

#### Visual Studio Code et SAP Fiori tools

1. Installer Node.js dans une version supportée par SAP Fiori tools.
2. Installer Visual Studio Code puis l'extension pack SAP Fiori tools depuis la marketplace.
3. Déclarer le système ABAP cible dans la vue des systèmes SAP de l'extension.
4. Vérifier la connexion : la liste des applications du système doit être récupérée.

> **📸 COPIE D'ÉCRAN N°03** — Visual Studio Code : extension pack SAP Fiori tools installée
> *Remplacer cette ligne par :* `![Copie 03](images/F1048-EXT/capture-03.png)`

> **📸 COPIE D'ÉCRAN N°04** — Visual Studio Code : déclaration et test du système ABAP cible
> *Remplacer cette ligne par :* `![Copie 04](images/F1048-EXT/capture-04.png)`

---

## 5. Synoptique de la démarche

| N° | Étape | Outil | Partie |
|---|---|---|---|
| A1 | Créer l'élément de données | `SE11` / ADT | Données |
| A2 | Étendre l'include client `CI_EBANDB` | `SE11` | Données |
| A3 | Autoriser la modification du champ | `SE11` | Données |
| A4 | Étendre les vues CDS de lecture | ADT | Données |
| A5 | Exposer le champ dans le service OData | `SEGW` / ADT | Données |
| A6 | Tester le service étendu | `/IWFND/GW_CLIENT` | Données |
| B1 | Identifier le point d'extension | `SE18` / `SE84` | Logique |
| B2 | Créer l'implémentation du BAdI | `SE19` / ADT | Logique |
| B3 | Implémenter les méthodes | ADT | Logique |
| B4 | Tester la logique métier | `ME51N` / `ME52N` | Logique |
| C1 | Créer le projet d'adaptation | VS Code | Interface |
| C2 | Définir les modifications d'interface | VS Code | Interface |
| C3 | Prévisualiser la variante | VS Code | Interface |
| C4 | Déployer la variante d'application | VS Code | Interface |
| C5 | Publier la tuile de la variante | `/UI2/FLPD_CUST`, `PFCG` | Interface |
| D | Recette, transport, montée de version | — | Transverse |

---

## Partie A — Extension du modèle de données

Objectif : ajouter un champ personnalisé au poste de demande d'achat, le rendre modifiable par la logique métier, puis l'exposer dans le service OData consommé par l'application Fiori.

### A1 — Créer l'élément de données

#### Mode opératoire

1. Lancer la transaction `SE11` ou créer l'objet depuis Eclipse ADT.
2. Créer si nécessaire un domaine (type et longueur du champ, table de valeurs éventuelle).
3. Créer l'élément de données en le rattachant au domaine.
4. Renseigner les libellés courts, moyens, longs et l'en-tête : ils alimenteront les libellés de colonne dans l'interface.
5. Activer l'objet et l'affecter au package de développement et à l'ordre de transport du projet.

> **📸 COPIE D'ÉCRAN N°05** — Transaction `SE11` : création du domaine (type de données, longueur)
> *Remplacer cette ligne par :* `![Copie 05](images/F1048-EXT/capture-05.png)`

> **📸 COPIE D'ÉCRAN N°06** — Transaction `SE11` : création de l'élément de données et saisie des libellés
> *Remplacer cette ligne par :* `![Copie 06](images/F1048-EXT/capture-06.png)`

> **ℹ️ Nommage**
> Respecter l'espace de noms client. Le préfixe `ZZ` (ou `ZZ1_` pour rester aligné sur les conventions d'extensibilité SAP) évite tout conflit avec un futur champ standard.

---

### A2 — Étendre l'include client CI_EBANDB

#### Objectif

`CI_EBANDB` est l'include client prévu par SAP dans la table `EBAN` (postes de demande d'achat). Il constitue le point d'extension officiel : l'ajout d'un champ dans cet include ne constitue pas une modification du standard.

#### Mode opératoire

1. Lancer la transaction `SE11` et ouvrir la structure `CI_EBANDB` en modification.
2. Ajouter une ligne avec le nom du champ et l'élément de données créé en A1.
3. Activer la structure.
4. Vérifier l'adaptation de la table `EBAN` à la base de données ; si la table est signalée incohérente, lancer la transaction `SE14` et exécuter « Adapter et activer ».
5. Contrôler la présence du champ dans la table `EBAN` via `SE11`.

```
SE11 → Structure : CI_EBANDB → Modifier → ajouter ZZMOTIF_ACHAT → Activer
```

> **📸 COPIE D'ÉCRAN N°07** — Transaction `SE11` : structure `CI_EBANDB` avec le champ ajouté
> *Remplacer cette ligne par :* `![Copie 07](images/F1048-EXT/capture-07.png)`

> **📸 COPIE D'ÉCRAN N°08** — Transaction `SE11` : table `EBAN`, vérification de la présence du champ
> *Remplacer cette ligne par :* `![Copie 08](images/F1048-EXT/capture-08.png)`

> **📸 COPIE D'ÉCRAN N°09** — Transaction `SE14` : adaptation et activation de la table `EBAN` (si nécessaire)
> *Remplacer cette ligne par :* `![Copie 09](images/F1048-EXT/capture-09.png)`

> **⚠️ Vigilance**
> L'adaptation de `EBAN` est une opération sur une table volumineuse. La planifier hors période d'activité et la faire valider par l'équipe Basis.

---

### A3 — Autoriser la modification du champ par la logique métier

#### Objectif

Un champ ajouté dans `CI_EBANDB` n'est pas automatiquement modifiable depuis le BAdI de traitement des demandes d'achat. Il doit être déclaré dans la structure des champs client autorisés, faute de quoi les valeurs affectées par le BAdI seront silencieusement ignorées à l'enregistrement.

#### Mode opératoire

1. Lancer la transaction `SE11` et ouvrir la structure `MEREQ_ITEM_S_CUST_ALLOWED`.
2. Vérifier la présence de l'include `CI_EBANDB` dans cette structure.
3. Si le champ n'est pas repris, créer un append sur la structure et y ajouter le champ.
4. Activer et transporter.

> **📸 COPIE D'ÉCRAN N°10** — Transaction `SE11` : structure `MEREQ_ITEM_S_CUST_ALLOWED` et déclaration du champ
> *Remplacer cette ligne par :* `![Copie 10](images/F1048-EXT/capture-10.png)`

> **⚠️ Symptôme typique**
> Si cette étape est omise, la valeur affectée dans le BAdI est bien visible en cours de traitement mais n'est pas enregistrée en base. C'est la cause la plus fréquente des incidents « le champ personnalisé ne se sauvegarde pas ».

---

### A4 — Étendre les vues CDS de lecture

#### Objectif

Rendre le champ disponible dans le modèle de lecture utilisé en amont du service OData, sans modifier les vues livrées par SAP.

#### Mode opératoire

1. Identifier la vue CDS qui alimente l'entité du service OData correspondant au poste de demande d'achat.
2. Vérifier que cette vue est extensible : la catégorie d'extension de vue doit l'autoriser.
3. Dans Eclipse ADT, créer un objet de type « Data Definition » contenant une extension de vue.
4. Ajouter le champ en le préfixant de l'alias de la table ou de la vue source.
5. Activer et transporter, puis contrôler le résultat par un aperçu de données.

```abap
@AbapCatalog.sqlViewAppendName: 'ZZIPRITEMEXT'
extend view I_PurchaseRequisitionItem with ZZ_I_PurReqItem_Ext
{
  eban.zzmotif_achat as ZZMotifAchat
}
```

> **📸 COPIE D'ÉCRAN N°11** — Eclipse ADT : création de l'extension de vue CDS
> *Remplacer cette ligne par :* `![Copie 11](images/F1048-EXT/capture-11.png)`

> **📸 COPIE D'ÉCRAN N°12** — Eclipse ADT : aperçu de données montrant le champ ajouté
> *Remplacer cette ligne par :* `![Copie 12](images/F1048-EXT/capture-12.png)`

> **ℹ️ À vérifier**
> Le nom exact de la vue à étendre et son alias source dépendent de la version. Les relever dans Eclipse à partir de la définition de la vue standard avant d'écrire l'extension : ne pas recopier l'exemple ci-dessus sans contrôle.

---

### A5 — Exposer le champ dans le service OData

#### A5.1 Déterminer le type de service

La technique d'exposition dépend de la nature du service. Cette détermination est un préalable obligatoire.

1. Lancer la transaction `/IWFND/MAINT_SERVICE` et sélectionner le service de l'application.
2. Ouvrir la vue « Service Implementation » et relever les classes du modèle et du fournisseur de données.
3. Si un projet est visible dans la transaction `SEGW`, le service est construit avec le Service Builder.
4. Si le modèle est généré à partir d'une vue CDS exposée, le service est basé sur le modèle de données.

> **📸 COPIE D'ÉCRAN N°13** — Transaction `/IWFND/MAINT_SERVICE` : vue « Service Implementation » du service
> *Remplacer cette ligne par :* `![Copie 13](images/F1048-EXT/capture-13.png)`

> **📸 COPIE D'ÉCRAN N°14** — Transaction `SEGW` : projet du service, arborescence du modèle de données
> *Remplacer cette ligne par :* `![Copie 14](images/F1048-EXT/capture-14.png)`

#### A5.2 Variante 1 — service construit avec le Service Builder

1. Ne jamais modifier le projet SAP standard : créer un projet d'extension dédié dans la transaction `SEGW`.
2. Redéfinir le service de base pour hériter de son modèle.
3. Ajouter la propriété correspondant au champ personnalisé sur le type d'entité concerné.
4. Redéfinir les méthodes utiles des classes d'extension du modèle et du fournisseur de données.
5. Alimenter la propriété dans la méthode de lecture et prendre en charge sa mise à jour dans la méthode de modification.
6. Générer le projet, puis enregistrer le service étendu dans `/IWFND/MAINT_SERVICE`.

> **📸 COPIE D'ÉCRAN N°15** — Transaction `SEGW` : projet d'extension, ajout de la propriété sur le type d'entité
> *Remplacer cette ligne par :* `![Copie 15](images/F1048-EXT/capture-15.png)`

> **📸 COPIE D'ÉCRAN N°16** — Transaction `SEGW` : redéfinition des méthodes dans les classes d'extension
> *Remplacer cette ligne par :* `![Copie 16](images/F1048-EXT/capture-16.png)`

> **📸 COPIE D'ÉCRAN N°17** — Transaction `SEGW` : génération du projet d'extension
> *Remplacer cette ligne par :* `![Copie 17](images/F1048-EXT/capture-17.png)`

#### A5.3 Variante 2 — service basé sur le modèle de données

1. Vérifier que l'extension de vue réalisée en A4 est bien remontée dans le modèle exposé.
2. Créer une extension de métadonnées pour porter les annotations d'affichage du champ.
3. Activer les objets et vider le cache des métadonnées du service.
4. Contrôler la présence de la propriété dans le document de métadonnées.

```abap
annotate view I_PurchaseRequisitionItem with
{
  @UI.lineItem: [ { position: 90, label: 'Motif d''achat' } ]
  ZZMotifAchat;
}
```

> **📸 COPIE D'ÉCRAN N°18** — Eclipse ADT : extension de métadonnées et annotations du champ
> *Remplacer cette ligne par :* `![Copie 18](images/F1048-EXT/capture-18.png)`

---

### A6 — Tester le service étendu

1. Lancer la transaction `/IWFND/GW_CLIENT`.
2. Appeler le document de métadonnées du service et rechercher la nouvelle propriété.
3. Exécuter une lecture sur une entité contenant une valeur renseignée et vérifier la restitution du champ.
4. En cas d'absence de la propriété, vider les caches de métadonnées puis relancer le test.

```
/sap/opu/odata/sap/MM_PUR_PR_PROCESS_SRV/$metadata
```

> **📸 COPIE D'ÉCRAN N°19** — Transaction `/IWFND/GW_CLIENT` : propriété personnalisée visible dans les métadonnées
> *Remplacer cette ligne par :* `![Copie 19](images/F1048-EXT/capture-19.png)`

> **📸 COPIE D'ÉCRAN N°20** — Transaction `/IWFND/GW_CLIENT` : lecture d'une entité avec la valeur du champ personnalisé
> *Remplacer cette ligne par :* `![Copie 20](images/F1048-EXT/capture-20.png)`

#### Critères de validation de la partie A

- Le champ existe dans la table `EBAN` et est déclaré modifiable.
- La vue CDS étendue restitue le champ.
- La propriété apparaît dans le document de métadonnées du service OData.

---

## Partie B — Extension de la logique métier

Objectif : alimenter, dériver et contrôler le champ personnalisé au fil du traitement de la demande d'achat, sans modifier le standard.

### B1 — Identifier le point d'extension

#### Mode opératoire

1. Lancer la transaction `SE18` et rechercher les BAdI du domaine des demandes d'achat.
2. Le point d'extension de référence pour le traitement des postes est le BAdI de traitement des demandes d'achat.
3. Consulter sa documentation et la liste de ses méthodes avant toute implémentation.
4. Recenser les autres points d'extension utiles selon le besoin (contrôle, enregistrement, restitution dans les listes).

| Point d'extension | Usage | Portée |
|---|---|---|
| `ME_PROCESS_REQ_CUST` | Traitement des postes de demande d'achat : dérivation, contrôle, enregistrement | Transactions et applications s'appuyant sur le moteur d'achat |
| `ME_REQ_POSTED` | Traitement après enregistrement de la demande d'achat | Post-traitement |
| `ME_CHANGE_OUTTAB_CUS` | Alimentation des colonnes personnalisées dans les listes d'achat | Éditions et listes |
| `MEREQ001` | Sous-écran client dans les transactions de demande d'achat | Interface SAP GUI uniquement |

> **📸 COPIE D'ÉCRAN N°21** — Transaction `SE18` : définition du BAdI de traitement des demandes d'achat et liste des méthodes
> *Remplacer cette ligne par :* `![Copie 21](images/F1048-EXT/capture-21.png)`

> **⚠️ Périmètre d'appel**
> Un BAdI n'est pas déclenché par tous les points d'entrée. Certains scénarios de création automatique de demandes d'achat ne l'appellent pas. Vérifier le comportement pour chacun des canaux réellement utilisés dans le projet.

---

### B2 — Créer l'implémentation du BAdI

1. Lancer la transaction `SE19` et choisir la création d'une implémentation classique.
2. Saisir le nom du BAdI à implémenter puis valider.
3. Nommer l'implémentation et la classe d'implémentation selon la convention du projet.
4. Renseigner un texte court explicite et affecter au package et à l'ordre de transport.
5. Activer l'implémentation et vérifier qu'elle est marquée active.

> **📸 COPIE D'ÉCRAN N°22** — Transaction `SE19` : création de l'implémentation du BAdI
> *Remplacer cette ligne par :* `![Copie 22](images/F1048-EXT/capture-22.png)`

> **📸 COPIE D'ÉCRAN N°23** — Transaction `SE19` : implémentation créée, classe générée et statut actif
> *Remplacer cette ligne par :* `![Copie 23](images/F1048-EXT/capture-23.png)`

---

### B3 — Implémenter les méthodes

#### Répartition des traitements

| Méthode | Traitement attendu |
|---|---|
| `PROCESS_ITEM` | Dérivation et valorisation du champ au niveau du poste |
| `CHECK` | Contrôles de cohérence et émission des messages d'erreur ou d'avertissement |
| `POST` | Traitements complémentaires au moment de l'enregistrement |
| `PROCESS_HEADER` | Traitements au niveau de l'en-tête de la demande d'achat |
| `OPEN` / `CLOSE` | Initialisation et libération du contexte de traitement |

#### Règle d'écriture des valeurs

La modification d'un champ de poste suit toujours le même enchaînement : lecture des données courantes, modification de la structure de valeurs, positionnement du drapeau correspondant dans la structure de modification, puis écriture des deux structures.

```abap
METHOD if_ex_me_process_req_cust~process_item.
  DATA: ls_item  TYPE mereq_item,
        ls_itemx TYPE mereq_itemx.

  ls_item  = im_item->get_data( ).
  ls_itemx = im_item->get_datax( ).

  IF ls_item-zzmotif_achat IS INITIAL.
    ls_item-zzmotif_achat  = 'STANDARD'.
    ls_itemx-zzmotif_achat = 'X'.
    im_item->set_data( ls_item ).
    im_item->set_datax( ls_itemx ).
  ENDIF.
ENDMETHOD.
```

> **📸 COPIE D'ÉCRAN N°24** — Eclipse ADT : code de la méthode de traitement du poste
> *Remplacer cette ligne par :* `![Copie 24](images/F1048-EXT/capture-24.png)`

#### Contrôles et messages

1. Implémenter la méthode de contrôle pour valider les règles de gestion.
2. Émettre les messages par les utilitaires de messages du domaine achat, afin qu'ils remontent correctement dans l'interface Fiori.
3. Distinguer les messages bloquants des messages d'avertissement : un message bloquant empêche l'enregistrement.
4. Créer une classe de messages dédiée au projet plutôt que de réutiliser une classe standard.

> **📸 COPIE D'ÉCRAN N°25** — Eclipse ADT : code de la méthode de contrôle et émission des messages
> *Remplacer cette ligne par :* `![Copie 25](images/F1048-EXT/capture-25.png)`

> **📸 COPIE D'ÉCRAN N°26** — Transaction `SE91` : classe de messages du projet
> *Remplacer cette ligne par :* `![Copie 26](images/F1048-EXT/capture-26.png)`

---

### B4 — Tester la logique métier

1. Poser un point d'arrêt dans la méthode implémentée.
2. Créer une demande d'achat via la transaction `ME51N` et vérifier le passage dans le point d'arrêt.
3. Enregistrer, puis contrôler la valeur en base dans la table `EBAN` via `SE16N`.
4. Modifier la demande d'achat via `ME52N` et vérifier le comportement des contrôles.
5. Rejouer le scénario depuis l'application Fiori pour valider le comportement sur le canal cible.
6. En cas d'anomalie, analyser les dumps via `ST22` et les erreurs de service via les journaux Gateway.

> **📸 COPIE D'ÉCRAN N°27** — Transaction `ME51N` : création d'une demande d'achat de test
> *Remplacer cette ligne par :* `![Copie 27](images/F1048-EXT/capture-27.png)`

> **📸 COPIE D'ÉCRAN N°28** — Débogueur ABAP : arrêt dans la méthode du BAdI
> *Remplacer cette ligne par :* `![Copie 28](images/F1048-EXT/capture-28.png)`

> **📸 COPIE D'ÉCRAN N°29** — Transaction `SE16N` : table `EBAN`, valeur du champ personnalisé enregistrée
> *Remplacer cette ligne par :* `![Copie 29](images/F1048-EXT/capture-29.png)`

> **📸 COPIE D'ÉCRAN N°30** — Transaction `ME52N` : message de contrôle émis par le BAdI
> *Remplacer cette ligne par :* `![Copie 30](images/F1048-EXT/capture-30.png)`

#### Critères de validation de la partie B

- L'implémentation est active et appelée sur tous les canaux du périmètre.
- La valeur dérivée est correctement enregistrée en base.
- Les contrôles produisent les messages attendus, sans effet de bord sur le processus standard.

---

## Partie C — Extension de l'interface utilisateur

Objectif : afficher le champ personnalisé dans l'application Fiori et adapter l'interface, au moyen d'un projet d'adaptation SAPUI5. Le projet produit une variante d'application qui référence l'application standard et ne contient que les écarts : les corrections et évolutions livrées par SAP sur l'application de base restent donc reprises automatiquement.

### C1 — Créer le projet d'adaptation

1. Ouvrir Visual Studio Code et lancer le générateur de projet d'adaptation depuis la palette de commandes.
2. Sélectionner le système ABAP cible déclaré au chapitre 4 ; le type de projet affiché doit être **onPremise**.
3. Sélectionner l'application de base dans la liste : Manage Purchase Requisitions.
4. Renseigner le nom du projet et le titre de l'application variante.
5. Renseigner la configuration de déploiement : nom du repository SAPUI5 ABAP, package de développement, ordre de transport.
6. Terminer la création et vérifier la génération de l'arborescence du projet.

> **📸 COPIE D'ÉCRAN N°31** — Visual Studio Code : générateur de projet d'adaptation, sélection du système et de l'application
> *Remplacer cette ligne par :* `![Copie 31](images/F1048-EXT/capture-31.png)`

> **📸 COPIE D'ÉCRAN N°32** — Visual Studio Code : attributs du projet (nom, titre de la variante)
> *Remplacer cette ligne par :* `![Copie 32](images/F1048-EXT/capture-32.png)`

> **📸 COPIE D'ÉCRAN N°33** — Visual Studio Code : configuration de déploiement (repository, package, ordre de transport)
> *Remplacer cette ligne par :* `![Copie 33](images/F1048-EXT/capture-33.png)`

> **📸 COPIE D'ÉCRAN N°34** — Visual Studio Code : arborescence du projet d'adaptation généré
> *Remplacer cette ligne par :* `![Copie 34](images/F1048-EXT/capture-34.png)`

---

### C2 — Définir les modifications d'interface

Trois familles de modifications couvrent la grande majorité des besoins :

| Type de modification | Usage |
|---|---|
| Changement de propriété | Renommer un libellé, masquer un élément, modifier un comportement d'affichage |
| Ajout de fragment XML | Insérer une colonne, un champ ou un bloc dans un écran existant |
| Extension de contrôleur | Ajouter une logique côté client, une valeur par défaut, un contrôle de saisie |

#### Ajouter la colonne du champ personnalisé

1. Ouvrir l'éditeur de variante et charger l'aperçu de l'application de base.
2. Sélectionner le contrôle sur lequel greffer la modification.
3. Choisir l'ajout d'un fragment XML et nommer le fragment.
4. Déclarer dans le fragment la colonne et la liaison vers la propriété OData exposée en partie A.
5. Enregistrer : la modification est stockée dans le projet, l'application standard reste inchangée.

```xml
<core:FragmentDefinition xmlns:core="sap.ui.core" xmlns="sap.m">
  <Column>
    <Text text="{i18n>motifAchat}" />
  </Column>
</core:FragmentDefinition>
```

> **📸 COPIE D'ÉCRAN N°35** — Visual Studio Code : éditeur de variante, sélection du contrôle cible
> *Remplacer cette ligne par :* `![Copie 35](images/F1048-EXT/capture-35.png)`

> **📸 COPIE D'ÉCRAN N°36** — Visual Studio Code : ajout d'un fragment XML sur le contrôle
> *Remplacer cette ligne par :* `![Copie 36](images/F1048-EXT/capture-36.png)`

> **📸 COPIE D'ÉCRAN N°37** — Visual Studio Code : contenu du fragment et liaison vers la propriété OData
> *Remplacer cette ligne par :* `![Copie 37](images/F1048-EXT/capture-37.png)`

> **📸 COPIE D'ÉCRAN N°38** — Visual Studio Code : liste des modifications enregistrées dans le projet
> *Remplacer cette ligne par :* `![Copie 38](images/F1048-EXT/capture-38.png)`

> **ℹ️ Bonne pratique**
> Limiter le nombre de modifications et documenter chacune d'elles. Plus la variante s'éloigne du standard, plus le coût de validation à chaque montée de version augmente.

---

### C3 — Prévisualiser la variante

1. Lancer la prévisualisation du projet depuis Visual Studio Code.
2. Vérifier l'affichage de la modification dans l'application.
3. Contrôler la remontée de la valeur du champ personnalisé sur des données réelles.
4. Corriger et relancer autant que nécessaire avant tout déploiement.

> **📸 COPIE D'ÉCRAN N°39** — Navigateur : prévisualisation de la variante avec la colonne ajoutée
> *Remplacer cette ligne par :* `![Copie 39](images/F1048-EXT/capture-39.png)`

---

### C4 — Déployer la variante d'application

1. Lancer la commande de déploiement du projet d'adaptation.
2. Confirmer le repository SAPUI5 ABAP, le package et l'ordre de transport.
3. Attendre la fin du déploiement et contrôler l'absence d'erreur.
4. Vérifier dans le système la création de l'application BSP correspondant à la variante.
5. Activer le nœud ICF de la nouvelle application via la transaction `SICF`.

> **📸 COPIE D'ÉCRAN N°40** — Visual Studio Code : journal de déploiement de la variante
> *Remplacer cette ligne par :* `![Copie 40](images/F1048-EXT/capture-40.png)`

> **📸 COPIE D'ÉCRAN N°41** — Transaction `SE80` : application BSP de la variante créée dans le système
> *Remplacer cette ligne par :* `![Copie 41](images/F1048-EXT/capture-41.png)`

> **📸 COPIE D'ÉCRAN N°42** — Transaction `SICF` : activation du nœud ICF de la variante
> *Remplacer cette ligne par :* `![Copie 42](images/F1048-EXT/capture-42.png)`

---

### C5 — Publier la tuile de la variante

#### Objectif

La variante d'application possède son propre identifiant. Elle nécessite donc sa propre tuile et son propre target mapping, distincts de ceux de l'application standard.

1. Lancer la transaction `/UI2/FLPD_CUST`.
2. Créer un catalogue Z dédié aux extensions du projet.
3. Créer une tuile pointant vers la variante et renseigner titre, sous-titre et icône.
4. Créer le target mapping associé en référençant l'identifiant de la variante d'application.
5. Ajouter le catalogue Z au rôle PFCG utilisé pour l'application, puis régénérer le profil.
6. Décider du sort de la tuile standard : la conserver, ou la retirer du rôle pour éviter la coexistence de deux tuiles proches.
7. Invalider les caches et vérifier l'affichage.

> **📸 COPIE D'ÉCRAN N°43** — Launchpad Designer : catalogue Z de projet et tuile de la variante
> *Remplacer cette ligne par :* `![Copie 43](images/F1048-EXT/capture-43.png)`

> **📸 COPIE D'ÉCRAN N°44** — Launchpad Designer : target mapping vers la variante d'application
> *Remplacer cette ligne par :* `![Copie 44](images/F1048-EXT/capture-44.png)`

> **📸 COPIE D'ÉCRAN N°45** — Transaction `PFCG` : ajout du catalogue Z au rôle
> *Remplacer cette ligne par :* `![Copie 45](images/F1048-EXT/capture-45.png)`

> **📸 COPIE D'ÉCRAN N°46** — Launchpad : tuile de la variante et application affichant le champ personnalisé
> *Remplacer cette ligne par :* `![Copie 46](images/F1048-EXT/capture-46.png)`

#### Critères de validation de la partie C

- La variante est déployée et son nœud ICF est actif.
- La tuile de la variante est visible pour l'utilisateur habilité.
- Le champ personnalisé est affiché avec la bonne valeur.

---

## Partie D — Recette, transport et montée de version

### D1 — Fiche de recette

| N° | Point de contrôle | Résultat attendu | OK / KO |
|---|---|---|---|
| 1 | Élément de données créé et activé | Actif | ☐ |
| 2 | Champ présent dans `EBAN` | Présent | ☐ |
| 3 | Champ déclaré modifiable | Déclaré | ☐ |
| 4 | Extension de vue CDS active | Active | ☐ |
| 5 | Propriété présente dans les métadonnées OData | Présente | ☐ |
| 6 | Lecture OData restituant la valeur | Conforme | ☐ |
| 7 | Implémentation du BAdI active | Active | ☐ |
| 8 | Dérivation de la valeur à la création | Conforme | ☐ |
| 9 | Valeur enregistrée en base | Conforme | ☐ |
| 10 | Contrôles et messages | Conformes | ☐ |
| 11 | Absence de régression sur le processus standard | Conforme | ☐ |
| 12 | Variante d'application déployée | Déployée | ☐ |
| 13 | Nœud ICF de la variante actif | Actif | ☐ |
| 14 | Tuile de la variante visible | Visible | ☐ |
| 15 | Champ affiché dans l'application | Affiché | ☐ |
| 16 | Comportement identique pour un second utilisateur habilité | Conforme | ☐ |

> **📸 COPIE D'ÉCRAN N°47** — Synthèse de recette : application variante affichant le champ personnalisé alimenté
> *Remplacer cette ligne par :* `![Copie 47](images/F1048-EXT/capture-47.png)`

### D2 — Transport

| Objet | Mode de propagation |
|---|---|
| Domaine, élément de données, include client | Ordre de workbench |
| Adaptation de la table `EBAN` | Automatique à l'import ; contrôler la conversion dans le système cible |
| Extension de vue CDS et extension de métadonnées | Ordre de workbench |
| Projet d'extension du service OData | Ordre de workbench ; enregistrement du service à refaire dans chaque système |
| Implémentation du BAdI et classe de messages | Ordre de workbench |
| Variante d'application SAPUI5 | Ordre de workbench ; activation du nœud ICF manuelle |
| Catalogue Z, tuile et target mapping | Ordre de customizing |
| Rôle PFCG | Ordre de customizing |

> **⚠️ Ordre d'import**
> Importer les objets de dictionnaire et de code avant le contenu Launchpad. Après import, activer les nœuds ICF, enregistrer le service étendu, invalider les caches, puis rejouer la fiche de recette.

### D3 — Impact des montées de version

| Objet d'extension | Robustesse à la montée de version | Action de contrôle |
|---|---|---|
| Include client `CI_EBANDB` | Élevée — point d'extension prévu par SAP | Vérifier l'activation de la table après import |
| Structure des champs modifiables | Élevée | Contrôler la persistance du champ |
| Extension de vue CDS | Moyenne — la vue étendue peut évoluer | Retester l'activation et l'aperçu de données |
| Extension du service OData | Moyenne à faible — la redéfinition suit le modèle standard | Régénérer et retester les métadonnées |
| Implémentation du BAdI | Élevée — interface stable | Retester les canaux d'appel |
| Variante d'application | Moyenne — les changements sont rejoués sur la nouvelle version | Retester chaque modification après montée de version |

> **ℹ️ Gouvernance**
> Tenir un registre des extensions listant, pour chaque objet créé, son point d'extension, son propriétaire fonctionnel et le scénario de test associé. Ce registre est la base du plan de retest lors des montées de version.

---

## Annexe A — Outils et transactions

| Outil / Transaction | Usage |
|---|---|
| `SE11` | Dictionnaire : domaines, éléments de données, structures, tables |
| `SE14` | Utilitaire de base de données : adaptation et activation des tables |
| `SE16N` | Affichage du contenu des tables |
| `SE18` | Définition des BAdI |
| `SE19` | Implémentation des BAdI |
| `SE24` | Constructeur de classes |
| `SE80` | Navigateur d'objets, applications BSP |
| `SE91` | Classes de messages |
| `SEGW` | Service Builder : projets de services OData |
| `/IWFND/MAINT_SERVICE` | Enregistrement et activation des services OData |
| `/IWFND/GW_CLIENT` | Test des services OData |
| `/IWFND/ERROR_LOG` | Journal des erreurs Gateway |
| `SICF` | Activation des services ICF |
| `/UI2/FLPD_CUST` | Launchpad Designer |
| `/UI2/INVALIDATE_CACHES` | Invalidation des caches du Launchpad |
| `PFCG` | Rôles et autorisations |
| `ME51N` / `ME52N` / `ME53N` | Création, modification et affichage des demandes d'achat |
| `ST22` | Analyse des dumps ABAP |
| Eclipse ADT | Développement CDS, classes, extensions |
| VS Code + SAP Fiori tools | Projet d'adaptation SAPUI5 |

---

## Annexe B — Diagnostic des incidents fréquents

| Symptôme | Cause probable | Action corrective |
|---|---|---|
| Le champ n'apparaît pas dans `EBAN` | Structure activée mais table non adaptée | Lancer `SE14` et exécuter l'adaptation de la table |
| La valeur affectée dans le BAdI n'est pas enregistrée | Champ absent de la structure des champs client autorisés | Réaliser l'étape A3, puis retester |
| Le BAdI n'est pas appelé | Canal de création non couvert par le point d'extension | Identifier le point d'extension propre au canal concerné |
| La propriété n'apparaît pas dans les métadonnées | Projet non régénéré ou cache de métadonnées obsolète | Régénérer le projet, vider le cache, réenregistrer le service |
| Le champ est vide dans l'application | Propriété non alimentée dans la méthode de lecture | Compléter l'implémentation du fournisseur de données |
| La prévisualisation de la variante échoue | Système non joignable ou version SAPUI5 incompatible | Vérifier la déclaration du système et la version cible |
| Le déploiement de la variante échoue | Package, ordre de transport ou autorisation manquants | Contrôler les paramètres de déploiement et les droits |
| La tuile de la variante n'apparaît pas | Catalogue Z non affecté au rôle ou cache non invalidé | Compléter le rôle, régénérer le profil, vider les caches |
| Deux tuiles très proches sont visibles | Coexistence de la tuile standard et de la variante | Retirer la tuile standard du rôle concerné |
| Régression sur le processus standard | Contrôle du BAdI trop restrictif | Restreindre les conditions du contrôle, ajouter un test de non-régression |

---

## Annexe C — Index des copies d'écran

| N° | Chapitre / Étape | Contenu attendu |
|---|---|---|
| 01 | Chapitre 3 | Arbitrage key user / développeur |
| 02 à 04 | Chapitre 4 | Postes de développement Eclipse et VS Code |
| 05 à 06 | Étape A1 | Domaine et élément de données |
| 07 à 09 | Étape A2 | Include client, table `EBAN`, adaptation |
| 10 | Étape A3 | Structure des champs modifiables |
| 11 à 12 | Étape A4 | Extension de vue CDS |
| 13 à 18 | Étape A5 | Type de service et exposition OData |
| 19 à 20 | Étape A6 | Test du service étendu |
| 21 | Étape B1 | Définition du BAdI |
| 22 à 23 | Étape B2 | Création de l'implémentation |
| 24 à 26 | Étape B3 | Code des méthodes et classe de messages |
| 27 à 30 | Étape B4 | Tests de la logique métier |
| 31 à 34 | Étape C1 | Création du projet d'adaptation |
| 35 à 38 | Étape C2 | Modifications d'interface |
| 39 | Étape C3 | Prévisualisation |
| 40 à 42 | Étape C4 | Déploiement de la variante |
| 43 à 46 | Étape C5 | Publication de la tuile |
| 47 | Partie D | Synthèse de recette |

---

## Annexe D — Historique des versions

| Version | Date | Auteur | Nature des modifications |
|---|---|---|---|
| 1.0 | … | … | Création du document |
| | | | |
