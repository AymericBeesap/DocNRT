# Guide d'installation, de configuration et d'exploitation de gCTS

**SAP S/4HANA 2021 FPS02 — GitHub / GitLab — paysage DEV → QUAL**

| Métadonnée | Valeur |
|---|---|
| Version du document | 1.0 — 09/10/2026 |
| Produit SAP | SAP S/4HANA 2021 FPS02 (on-premise / ABAP Platform 2021) |
| Fonctionnalité | Git-enabled Change and Transport System (gCTS) |
| Systèmes documentés | **DEV** (développement) et **QUAL** (qualification) |
| Paysage réel | DEV → QUAL → PROD |
| Hors périmètre | Installation, configuration, import et captures d'écran de **PROD** |
| Fournisseur Git | Choisir **GitHub OU GitLab** pour un dépôt donné ; le workflow proposé est identique |
| Public cible | Équipes SAP Basis, ABAP, responsables de dépôt Git, responsables des mises en qualification |
| Statut | **Modèle à compléter et à valider sur le paysage client avant exécution** |

> **Convention de lecture** — Les valeurs `DEV`, `QUAL`, `GSD`, `ZSD_GCTS`, les hôtes et les URLs sont des **exemples**. Ne pas les recopier tels quels sans vérifier les SID, mandants, autorisations et règles de transport du client. Les boutons et libellés peuvent différer selon la langue de l'interface et les SAP Notes appliquées.
>
> **Important** — Ce guide est adapté à S/4HANA **2021 FPS02**. Avant installation, consulter la **SAP Note 2821718** (note centrale gCTS) et les corrections applicables au niveau exact de `SAP_BASIS` et du kernel. Le fait qu'une fonctionnalité figure dans une documentation SAP plus récente ne garantit pas sa présence sans correction dans le niveau 2021 FPS02.

<!-- CAPTURE 01 — INSÉRER ICI un schéma ou une capture de la documentation de paysage SAP montrant DEV, QUAL et PROD. Encadrer DEV et QUAL comme SEUL périmètre documenté ; ne pas détailler PROD. Exemple d'image : ![Paysage SAP](./captures/01-paysage-sap.png). -->

## 1. Objectif et principes

Ce document explique comment :

1. préparer les **deux systèmes SAP DEV et QUAL** et un dépôt **GitHub ou GitLab** ;
2. activer et contrôler l'application **gCTS** sur chacun ;
3. relier les développements ABAP de DEV à un dépôt distant Git ;
4. versionner les objets ABAP à l'aide des **demandes de transport**, commits et branches ;
5. **promouvoir une version revue** vers QUAL et y exécuter les contrôles ;
6. proposer une architecture de dépôts et une stratégie de branches simples à gouverner.

### 1.1 Ce que fait gCTS (et ce qu'il ne fait pas)

- Les développeurs continuent de travailler dans **ADT/Eclipse**, SAP GUI et les outils ABAP habituels, avec des **tâches et demandes de transport**.
- Sur **DEV**, la libération d'une demande de transport gCTS peut créer un commit dans le dépôt Git local du serveur ABAP, puis le pousser vers le dépôt distant, selon les paramètres retenus.
- Sur **QUAL**, un *pull* gCTS d'un commit/état Git déclenche un **import ABAP** et fournit des journaux de transport : il ne s'agit **pas** d'un simple `git pull` exécuté sur un poste développeur.
- **Git n'est pas un substitut à SAP CTS pour tous les objets**. Certains types d'objets, SAP Notes, ajustements, effets post-import et scénarios de Customizing exigent de consulter les restrictions ou de conserver un chemin CTS classique.
- Dans un **système DEV partagé**, les objets ABAP actifs ne sont pas isolés par utilisateur comme des branches de travail locales : **éviter de changer fréquemment de branche gCTS** lorsqu'il y a des développements en cours.

### 1.2 Architecture cible recommandée

```text
                       GitHub OU GitLab (dépôt privé unique)
                         s4-sd-extensions
                    ┌─────────────────────────┐
                    │ branche develop         │  ← commits produits par DEV
                    │ branche main            │  ← version validée pour QUAL
                    └───────────┬─────────────┘
                                │
                       Pull Request / Merge Request
                        develop ───────► main
                            ▲                │
                            │                ▼
                  push via gCTS         pull via gCTS
                            │                │
                   ┌────────┴───────┐ ┌──────┴──────────┐
                   │ SAP DEV        │ │ SAP QUAL        │
                   │ rôle gCTS :    │ │ rôle gCTS :     │
                   │ Development    │ │ Provided        │
                   │ branche active │ │ branche active  │
                   │ develop        │ │ main            │
                   └────────────────┘ └─────────────────┘

              PROD existe dans le paysage mais n'est pas traité ici.
```

**Décision d'architecture :** **un dépôt Git distant partagé par DEV et QUAL, et non un dépôt par système**. La séparation des environnements se fait par le **rôle gCTS** et par **la branche / le commit déployé**, pas en dupliquant le code dans deux dépôts indépendants.

<!-- CAPTURE 02 — INSÉRER ICI une vue GitHub/GitLab du dépôt privé et de ses branches develop/main. Ne pas afficher de jeton ou de secrets. Exemple : ![Dépôt et branches Git](./captures/02-repository-git.png). -->

## 2. Fiche d'environnement à compléter

| Paramètre | DEV | QUAL |
|---|---|---|
| SID réel / description | `[SID_DEV]` | `[SID_QUAL]` |
| Mandant utilisé | `[CLIENT_DEV]` | `[CLIENT_QUAL]` |
| URL Fiori Launchpad | `https://[hote-dev]/...` | `https://[hote-qual]/...` |
| Version SAP_BASIS / kernel | `[à relever]` | `[à relever]` |
| Répertoire de travail gCTS | `[chemin-dev]/gcts` | `[chemin-qual]/gcts` |
| Utilisateur technique d'initialisation | `[compte-dev]` | `[compte-qual]` |
| Rôle du dépôt gCTS | **Development** | **Provided** |
| Branche suivie | **develop** | **main** |
| Compte Git | écriture contrôlée | lecture seule si techniquement possible |
| Fréquence de mise à jour | au fil des transports validés | **manuelle après approbation** |

**Dépôt distant (exemple)** :

- GitHub : `https://github.com/<organisation>/s4-sd-extensions`
- GitLab SaaS : `https://gitlab.com/<groupe>/s4-sd-extensions`
- GitLab privé : `https://gitlab.<domaine>/<groupe>/s4-sd-extensions`
- Branche par défaut du dépôt : `main` ; branche d'intégration ABAP : `develop`.

## 3. Prérequis et décisions avant installation

### 3.1 Logiciels et réseau

**Dans chacun des systèmes DEV et QUAL :**

- Confirmer **S/4HANA 2021 FPS02**, la version de `SAP_BASIS` et le niveau de kernel.
- Vérifier les prérequis/notes de **gCTS** dans la [SAP Note 2821718](https://me.sap.com/notes/2821718) ; contrôler les [types d'objets supportés](https://me.sap.com/notes/2888887).
- Installer une **JRE compatible** avec la version gCTS : la documentation **2021 FPS02** exige **au moins Java 8 (1.8)** ; choisir une version LTS maintenue/validée par Basis et l'éditeur (p. ex. SapMachine compatible), plutôt que de déduire la version Java de celle de HANA. Vérifier `java -version`.
- Vérifier la présence de **`abap2vcs.jar`**, livré avec le SAP Kernel, dans le répertoire d'exécutables du système SAP.
- Créer/valider un répertoire nommé **`gcts`**, accessible en lecture/écriture à l'utilisateur OS `<sid>adm`. S'il existe plusieurs instances ABAP, utiliser un **stockage partagé propre à chaque système** ; ne pas partager le même répertoire de travail entre DEV et QUAL.
- Autoriser les sorties réseau vers le serveur Git : **HTTPS/443** si URLs HTTPS ; SSH/22 seulement si ce protocole est explicitement utilisé. Vérifier DNS, pare-feu, proxy et confiance de la chaîne TLS (notamment GitLab self-managed).
- Créer un dépôt Git **non vide**, initialisé a minima avec un `README.md`.
- Disposer d'un moyen de sauvegarde/reprise côté SAP et Git et d'une fenêtre de tests maîtrisée.

Exemples de vérification par l'équipe Basis **sur l'hôte ABAP** (commandes et chemins à adapter) :

```bash
# Exemples indicatifs : ne pas lancer avec des chemins non validés.
java -version
ls -l /usr/sap/<SID>/SYS/exe/run/abap2vcs.jar
ls -ld /chemin/valide/gcts
```

<!-- CAPTURE 03 — INSÉRER ICI AL11 (répertoire gcts) ou une preuve Basis des chemins Java et abap2vcs.jar dans DEV, sans information sensible. Exemple : ![Prérequis OS DEV](./captures/03-prerequis-os-dev.png). -->

### 3.2 Application Fiori gCTS et autorisations

Pour **DEV et QUAL**, réaliser l'activation de l'application Fiori gCTS selon la procédure SAP de la version 2021 :

1. **`/IWFND/MAINT_SERVICE`** : vérifier/activer le service OData **`SCTS_GCTS_SRV`** (scénario Fiori embarqué : alias `LOCAL`).
2. **`SICF`** : vérifier/activer **`/sap/bc/ui5_ui5/sap/bc_cts_git`** et **`/sap/bc/cts_abapvcs`**.
3. Mettre la tuile **« Git-enabled CTS »** à disposition via le catalogue/rôle Fiori pertinent.
4. Affecter les rôles d'administration selon le principe du moindre privilège. Rôles gCTS de référence :
   - `SAP_BC_GCTS_ADMIN` : modèle d'administration globale, réservé à l'installation/tests privilégiés ;
   - `SAP_BC_GCTS_SYSTEM_ADMIN` : administration des paramètres système ;
   - `SAP_BC_GCTS_REPOSITORY_ADMIN` : administration des dépôts/branches ;
   - `SAP_BC_GCTS_REPO_DEVELOPER` : opérations de développement, **avec permissions de collaboration sur les dépôts** si nécessaires.
5. Vérifier les autorisations de développement/transports classiques, les droits des jobs et les éventuels objets d'autorisation `S_GCTS_SYS`.

> **Sécurité** — Les rôles livrés par SAP sont des **modèles**. Créer si besoin des rôles PFCG dérivés et limités. L'utilisateur qui initialise gCTS et planifie le job observateur doit disposer des droits Git nécessaires aux opérations déclenchées par ce job.

<!-- CAPTURE 04 — INSÉRER ICI la tuile « Git-enabled CTS » visible dans le FLP de DEV. Exemple : ![Tuile gCTS DEV](./captures/04-tuile-gcts-dev.png). -->

<!-- CAPTURE 05 — INSÉRER ICI la tuile « Git-enabled CTS » visible dans le FLP de QUAL. Exemple : ![Tuile gCTS QUAL](./captures/05-tuile-gcts-qual.png). -->

<!-- CAPTURE 06 — INSÉRER ICI les services OData et ICF actifs dans /IWFND/MAINT_SERVICE et SICF (DEV puis QUAL), avec les hôtes masqués. Exemple : ![Services gCTS SAP](./captures/06-services-gcts.png). -->

### 3.3 Identité Git et politique d'accès

Choisir **un fournisseur Git pour le dépôt**. Les deux variantes sont décrites ci-dessous ; il n'est pas nécessaire de mettre en place les deux.

| Élément | GitHub | GitLab |
|---|---|---|
| Dépôt | privé, dans une organisation | projet privé, dans un groupe |
| Authentification recommandée | token compatible avec **gCTS 2021** | token compatible avec **gCTS 2021** |
| DEV | autoriser l'écriture sur `develop` | autoriser l'écriture sur `develop` |
| QUAL | limiter à la lecture du dépôt si possible | limiter à la lecture du dépôt si possible |
| Promotion | **Pull Request** `develop` → `main` | **Merge Request** `develop` → `main` |
| Protection | branch protection/ruleset sur `main` | branch rule/protection sur `main` |

Créer des identités et secrets **distincts** pour DEV et QUAL si la politique de sécurité le permet. Pour les tokens de l'ancien gCTS, vérifier à la fois les **permissions Git** et les **appels API** nécessaires à l'identification. À titre indicatif, les docs SAP ont historiquement demandé les portées `repo` et `user` sur les PAT classiques GitHub. GitLab peut nécessiter la portée `api` pour les appels d'API gCTS en plus des droits d'accès au dépôt : appliquer les portées minimales **après test fonctionnel** et selon les politiques locales.

Ne stocker **aucun token** dans le Markdown, le code ABAP, le dépôt Git ni les captures d'écran. Prévoir une **date de rotation** des secrets et la révocation des identités obsolètes.

<!-- CAPTURE 07 — INSÉRER ICI les paramètres du dépôt privé dans GitHub OU GitLab et ses membres/rôles (masquer adresses personnelles, jetons, clés). Exemple : ![Paramètres Git](./captures/07-droits-git.png). -->

## 4. Installer et activer gCTS sur DEV

### 4.1 Lancer l'assistant système

1. Ouvrir le **Fiori Launchpad DEV** avec un utilisateur autorisé.
2. Cliquer sur la tuile **Git-enabled CTS**.
3. Dans la vue **System**, lancer **Enable gCTS**.
4. Compléter l'assistant selon le tableau ci-dessous ; enregistrer les étapes successives.

| Étape de l'assistant | Paramètre gCTS | Valeur attendue / exemple |
|---|---|---|
| gCTS Directory | `VCS_PATH` | chemin **DEV** se terminant par `/gcts` ; par exemple `/usr/sap/<SID_DEV>/D00/gcts` si adapté au paysage |
| Java Runtime | `JAVA_RUNTIME` | chemin complet vers `java` ou valeur résoluble par le système |
| Git Client | `A2G_RUNTIME` | chemin vers `abap2vcs.jar`, p. ex. `$(SYSTEM/DIR_CT_RUN)/abap2vcs.jar` |
| Git-Enabled CTS | Initialisation TMS/job | **Initialize System**, selon les étapes de l'assistant |
| Summary and Check | **Health Check** | pas d'erreur bloquante ; justifier toute alerte |

5. Dans **`SM37`**, vérifier le job **`SCTS_ABAP_VCS_IMPORT_OBSERVER`** (création, utilisateur d'exécution, fréquence/exécution et éventuelles erreurs).
6. Dans l'onglet **Configurations** de la vue System, consigner `VCS_PATH`, `JAVA_RUNTIME` et `A2G_RUNTIME`.

> En cas de plusieurs serveurs applicatifs, valider le chemin partagé avant d'ajouter le premier dépôt. Ne pas modifier `VCS_PATH` après démarrage des dépôts sans procédure de migration du contenu disque.

<!-- CAPTURE 08 — INSÉRER ICI « Enable gCTS » en DEV avec les champs VCS_PATH/JAVA_RUNTIME/A2G_RUNTIME, sans données sensibles. Exemple : ![Activation gCTS DEV](./captures/08-activation-gcts-dev.png). -->

<!-- CAPTURE 09 — INSÉRER ICI « Health Check » DEV et le job SM37 SCTS_ABAP_VCS_IMPORT_OBSERVER. Exemple : ![Contrôles gCTS DEV](./captures/09-healthcheck-dev.png). -->

### 4.2 Préparer le vSID et la couche de transport DEV

**Contexte :** pour les développements Workbench classiques suivis par gCTS **sans registry**, le lien entre un package ABAP et un dépôt passe notamment par une **couche de transport gCTS** et un **SID virtuel non-ABAP (vSID)**. Le vSID n'est **ni le SID réel de DEV ni celui de QUAL**.

Exemple de convention pour un dépôt métier SD :

| Concept | Valeur d'exemple | Sens |
|---|---|---|
| Dépôt distant | `s4-sd-extensions` | projet Git |
| vSID gCTS | `GSD` | SID virtuel, 3 caractères, unique dans le domaine de transport |
| Couche de transport | `ZGSD` | format `Z` + vSID |
| Package principal | `ZSD_GCTS` | package ABAP géré par gCTS |
| Sous-package | `ZSD_GCTS_CORE` | même dépôt, vérifier sa couche et sa prise en charge |

**Cas A — DEV est contrôleur du domaine TMS :** l'assistant de création du dépôt gCTS peut initialiser le vSID et la couche correspondante. Vérifier le résultat dans **STMS**.

**Cas B — DEV n'est pas contrôleur du domaine TMS (cas courant en paysage existant) :** demander à Basis de préparer dans **STMS** le système virtuel **non-ABAP** de type VCS, le **vSID**, la couche `ZGSD` et la route de consolidation du système de développement vers ce vSID, selon la procédure SAP « Prepare ABAP Systems for Repositories ». Les paramètres techniques du système virtuel peuvent inclure `NON_ABAP_SYSTEM=VCS`, `JAVAPATH` et `SAPVCSPATH`.

**Important :** ne pas supprimer ni modifier les routes CTS existantes vers QUAL/PROD. Délimiter les packages qui relèveront de gCTS et ceux qui resteront sous **CTS classique**. L'usage d'une **gCTS Registry**, utile notamment pour le Customizing, nécessite un cadrage et une intégration supplémentaires : **hors périmètre du pilote Workbench de ce guide**.

<!-- CAPTURE 10 — INSÉRER ICI STMS en DEV : vSID non-ABAP GSD, NON_ABAP_SYSTEM=VCS et couche ZGSD/route de consolidation. Exemple : ![STMS et vSID](./captures/10-stms-vsid-dev.png). -->

### 4.3 Associer les packages ABAP

1. Créer ou identifier le package applicatif `ZSD_GCTS` et ses éventuels sous-packages.
2. Associer la **couche de transport `ZGSD`** aux packages devant relever de gCTS (opération SE21 / SE80 / ADT selon le processus de développement).
3. Vérifier le comportement de chaque sous-package et éviter les objets locaux `$TMP`.
4. Affecter les objets de test au package et consigner les dépendances hors dépôt.
5. Préparer une demande de transport **Workbench** gCTS avec un objet ABAP de test.

> **Règle de gouvernance :** un package ne doit pas être pris simultanément dans deux stratégies de livraison concurrentes sans conception explicite. La couche de transport permet de matérialiser le périmètre gCTS et d'éviter les envois involontaires via l'ancienne route CTS.

<!-- CAPTURE 11 — INSÉRER ICI l'écran SAP des attributs du package ZSD_GCTS avec la couche ZGSD et éventuellement sa liste d'objets ABAP. Exemple : ![Package ABAP et couche](./captures/11-package-transport-layer.png). -->

## 5. Préparer GitHub ou GitLab et la stratégie de branches

### 5.1 Créer le dépôt distant

1. Créer un **dépôt privé** `s4-sd-extensions` sur le fournisseur retenu.
2. Initialiser le dépôt avec **`README.md`** : gCTS nécessite un dépôt distant non vide.
3. Conserver **`main`** comme branche par défaut, créer **`develop`** à partir de `main`.
4. Mettre en place les droits décrits au §3.3.
5. Protéger `main` : pas de push direct, pas de *force-push* ou suppression, **PR/MR requise**, au moins **une revue** et, si disponibles, contrôles automatisés obligatoires.
6. Autoriser **DEV gCTS** à pousser vers `develop` sans demander une PR/MR à chaque commit automatique de transport. Éviter les règles de validation de commit incompatibles avec les messages générés par gCTS.
7. Préférer le **merge commit** ou une avancée qui préserve les commits gCTS ; **ne pas utiliser automatiquement squash/rebase** sur des commits déjà suivis/importés par gCTS sans validation de la stratégie d'historique.
8. Ne pas modifier manuellement les fichiers ABAP ou métadonnées générés par gCTS depuis le navigateur Git.

**Point de vigilance GitLab :** la branche `main` doit explicitement interdire **Allowed to push and merge = No one** si l'objectif est d'empêcher les pushes directs ; vérifier la cohérence de la politique d'approbation avec l'édition/abonnement GitLab disponible.

<!-- CAPTURE 12 — INSÉRER ICI la page Branches GitHub OU GitLab montrant develop et main. Exemple : ![Branches Git](./captures/12-branches-develop-main.png). -->

<!-- CAPTURE 13 — INSÉRER ICI la protection de main (PR/MR + approbation + absence de force-push). Exemple : ![Protection main](./captures/13-protection-main.png). -->

### 5.2 Architecture des dépôts recommandée

**Recommandation : un dépôt par périmètre fonctionnel/cohérent de livraison**, regroupant des packages ABAP qui doivent normalement être promus et éventuellement restaurés ensemble.

```text
organisation-ou-groupe-sap/
├── s4-sd-extensions      # pilote : packages ZSD_GCTS*
├── s4-mm-extensions      # ultérieur, si cycle de livraison indépendant
├── s4-fi-extensions      # ultérieur, si cycle de livraison indépendant
└── s4-gcts-runbook       # facultatif : documentation d'exploitation, hors gCTS ABAP
```

**Pourquoi ?**

- **Un seul dépôt pour le pilote** réduit les risques et simplifie l'exploitation DEV/QUAL.
- Plusieurs dépôts deviennent utiles lorsqu'il existe des **équipes, calendriers de promotion et frontières de dépendances** réellement différents.
- **Ne pas créer** `s4-sd-dev` et `s4-sd-qual` : cela casserait la traçabilité naturelle d'un même commit entre DEV et QUAL.
- **Éviter un gigantesque monorepo ABAP** regroupant tous les domaines du SI et imposant des déploiements couplés.
- Si le **Customizing** entre plus tard dans le périmètre, prévoir de préférence un **dépôt séparé**, une étude du mandant, des types d'objets et de l'utilisation de **gCTS Registry**. Ne pas l'intégrer silencieusement au pilote Workbench.

**Dans chaque dépôt métier :** une **couche gCTS/vSID dédiée** lorsque le mapping repose sur la couche de transport, des packages identifiés et un propriétaire métier/technique. Les éléments de documentation (processus, capture et gouvernance) peuvent vivre dans le dépôt de runbook séparé pour éviter tout effet lors des opérations de déploiement ABAP.

### 5.3 Stratégie de branches

| Branche | Usage | Qui écrit ? | Système consommateur |
|---|---|---|---|
| `develop` | Intégration continue des transports Workbench terminés sur DEV | gCTS DEV + administrateurs explicitement autorisés | **DEV** (rôle Development) |
| `main` | Révision promue, autorisée pour QUAL | fusion **approuvée** depuis `develop` | **QUAL** (rôle Provided) |
| `feature/*` | **Optionnel**, uniquement si un besoin de développement réellement isolé existe | sous contrôle de l'équipe | **Pas de changement de branche opportuniste sur le DEV partagé** |
| `hotfix/*` | Exception avec procédure dédiée si besoin futur | équipe de maintenance | **Non traité ici** |

**Choix délibéré :** **ne pas créer de branche `qual` supplémentaire** pour ce pilote. `main` représente la dernière version autorisée pour la qualification et QUAL suit `main` (ou un **SHA précis** de `main`). La branche n'est **pas un environnement SAP** : une branche est un état de code, tandis que les imports dépendent aussi du contexte applicatif et des données SAP.

**Règles obligatoires :**

1. `develop` est la **branche active gCTS de DEV** pendant les périodes de développement.
2. `main` est **protégée** et reçoit le contenu de `develop` par **PR/MR approuvée**.
3. **QUAL n'effectue jamais de push** vers Git. Le responsable déploie **manuellement** une version revue, de préférence identifiée par son **SHA**.
4. La PR/MR n'est **pas** le déploiement : le **pull gCTS en QUAL** est une étape distincte, suivie de contrôles SAP.
5. Un seul état actif existe pour les objets d'un même dépôt dans le DEV partagé : **ne pas utiliser les branches feature comme des espaces de travail indépendants par développeur dans ce système**.
6. Préserver l'historique des commits utilisés comme référence de livraison ; ne pas réécrire les branches déployées (pas de `push --force`, `reset` distant ou suppression des commits gCTS).

<!-- CAPTURE 14 — INSÉRER ICI un graphe des commits Git avec des commits sur develop, une PR/MR approuvée et le commit de main déployé en QUAL. Exemple : ![Graphe de commits](./captures/14-graphe-commits.png). -->

## 6. Déclarer le dépôt dans gCTS sur DEV

### 6.1 Saisir les identifiants Git

1. Ouvrir **gCTS DEV**, vue **System**.
2. Ouvrir **Manage Credentials / Credential Store**.
3. Créer une entrée pour le serveur Git (type **GitHub** ou **GitLab**) en utilisant le mode **Token**, avec le jeton autorisé pour DEV.
4. Pour `github.com` et `gitlab.com`, la documentation 2021 (sous réserve des corrections requises) utilise l'URL du serveur, par exemple `https://github.com` ou `https://gitlab.com`. Pour GitLab **self-managed**, vérifier le type d'endpoint, son URL et les éventuelles SAP Notes si la validation échoue.
5. Enregistrer et vérifier l'authentification. En cas d'erreur, contrôler le **scope du jeton, la date d'expiration, le type d'endpoint et la connectivité**, avant d'étendre des droits.

> Les identifiants **sont généralement liés au mandant et à l'utilisateur ABAP** qui les enregistre. Identifier les comptes utilisés par les tâches de fond afin qu'une connexion manuelle réussie ne masque pas un défaut d'accès lors de la libération des transports.

<!-- CAPTURE 15 — INSÉRER ICI les identifiants du serveur Git dans l'écran gCTS DEV, avec jeton entièrement masqué. Exemple : ![Identifiants gCTS DEV](./captures/15-credentials-dev.png). -->

### 6.2 Créer et cloner le dépôt DEV

1. Vue **System → Repositories → Create**.
2. Paramétrer le dépôt :

| Champ | Valeur d'exemple DEV |
|---|---|
| URL | `https://github.com/<organisation>/s4-sd-extensions` **ou** URL équivalente GitLab (sans `.git` si le formulaire l'attend) |
| Description | `Extensions S/4 SD — gCTS` |
| vSID | `GSD` |
| Role | **Development** |
| Type | **GitHub** ou **GitLab** |
| Visibility | selon règles d'accès dans **gCTS** ; ne pas confondre avec la visibilité du dépôt Git distant |

3. Enregistrer ; vérifier les paramètres liés à l'URL (`CLIENT_VCS_URI`, `CLIENT_VCS_CONNTYPE`, selon les champs proposés).
4. Lancer **Clone Repository**. Le dépôt sera initialement récupéré sur la branche par défaut distante (`main` dans ce guide).
5. Dans l'onglet **Branches**, passer sur **`develop`** et confirmer qu'elle est la **branche active** avant le premier transport gCTS destiné au pilote. Toute opération de changement de branche peut importer des objets ; ici elle intervient **avant** les premiers développements gérés par le dépôt.
6. Vérifier le statut **READY**, l'URL, la branche et le commit courant. Refaire un **Health Check** si nécessaire.
7. Contrôler les paramètres `VCS_AUTOMATIC_PUSH` et `VCS_AUTOMATIC_PULL` et **documenter leur valeur réellement utilisée**. Pour ce pilote : laisser le comportement standard de création/push de commit après libération uniquement si le flux `develop` fonctionne de bout en bout. Ne pas activer des synchronisations non maîtrisées.

> Ne pas changer de branche pendant qu'une demande de transport ouverte verrouille des objets du dépôt. L'onglet **Activities** et les journaux permettent de diagnostiquer clone, push et import.

<!-- CAPTURE 16 — INSÉRER ICI formulaire « Create Repository » en DEV, rôle Development, type GitHub/GitLab et vSID. Exemple : ![Repository DEV](./captures/16-repository-dev.png). -->

<!-- CAPTURE 17 — INSÉRER ICI dépôt gCTS DEV en statut READY, branche active develop et commit courant. Exemple : ![gCTS branche develop DEV](./captures/17-dev-branche-develop.png). -->

## 7. Installer et configurer gCTS sur QUAL

### 7.1 Activer gCTS dans QUAL

Appliquer **également dans QUAL** les sections **3.1**, **3.2** et **4.1**, avec les **chemins, identités techniques, autorisations et répertoires propres à QUAL** :

1. Application Fiori et services actifs.
2. **Enable gCTS** et contrôle de `VCS_PATH`, `JAVA_RUNTIME`, `A2G_RUNTIME`.
3. **Health Check** sans erreur bloquante.
4. Job **`SCTS_ABAP_VCS_IMPORT_OBSERVER`** vérifié dans `SM37`.
5. Définition des identifiants Git de **QUAL**, idéalement en **lecture seule**, selon les possibilités de la plateforme et la validation du token.

**Attention :** ne pas reproduire dans QUAL la configuration source de transport Workbench destinée à pousser du code. Le rôle du dépôt dans QUAL est **Provided**.

<!-- CAPTURE 18 — INSÉRER ICI l'assistant gCTS et le Health Check dans QUAL, plus la preuve du job SM37. Exemple : ![Activation gCTS QUAL](./captures/18-activation-qual.png). -->

### 7.2 Créer et cloner le dépôt cible

1. Dans **gCTS QUAL → System → Repositories → Create**.
2. Utiliser **la même URL distante** que sur DEV, mais les paramètres suivants :

| Champ | Valeur d'exemple QUAL |
|---|---|
| URL | **exactement le même dépôt Git** que DEV |
| vSID | **`GSD` recommandé**, identique à DEV pour une traçabilité simple (ce n’est pas nécessairement une obligation technique pour un système cible) |
| Role | **Provided** |
| Type | **GitHub** ou **GitLab**, comme DEV |
| Branche à suivre | **main** |

3. Enregistrer et **Clone Repository**. Cette opération **peut importer des objets** si `main` contient déjà du code ABAP. Pour un pilote, il est recommandé d'initialiser QUAL alors que `main` ne contient encore que le `README`/la structure technique, ou de prévoir une fenêtre d'import contrôlée.
4. Vérifier que **`main`** est bien la branche active, et que le dépôt passe au statut **READY**.
5. Laisser le **déploiement de QUAL manuel** : pas de déclenchement automatique de pull/import non approuvé. Contrôler la valeur du paramètre `VCS_AUTOMATIC_PULL` selon la configuration réelle et les mécanismes de synchronisation souhaités ; ne pas assimiler ce paramètre à un réglage universel « désactiver tout import ».
6. Contrôler les droits : le rôle **Provided** ne permet pas le push gCTS vers Git.

<!-- CAPTURE 19 — INSÉRER ICI « Create Repository » en QUAL avec rôle Provided et même URL qu'en DEV. Exemple : ![Repository QUAL](./captures/19-repository-qual.png). -->

<!-- CAPTURE 20 — INSÉRER ICI dépôt gCTS QUAL en statut READY, branche main et commit initial. Exemple : ![gCTS main QUAL](./captures/20-qual-branche-main.png). -->

## 8. Mode opératoire : développer en DEV et promouvoir en QUAL

### 8.1 Scénario de bout en bout

**Exemple de changement** : création d'une classe utilitaire `ZCL_SD_GCTS_DEMO` dans le package `ZSD_GCTS`.

#### Étape A — Développement dans DEV

1. Vérifier que le dépôt gCTS DEV est **READY** avec **branche active `develop`**.
2. Dans **ADT** ou **SE24**, créer/modifier un objet **dans un package gCTS**.
3. Affecter l'objet à une tâche et à une **demande de transport Workbench** pertinente (par exemple `[SID_DEV]K9xxxxx`).
4. Activer l'objet ; lancer les tests ABAP disponibles.
5. Libérer la tâche puis la demande de transport lorsque le contenu est prêt, en tenant compte du processus gCTS local.
6. Dans gCTS DEV, vérifier sur **Commits**, **Objects**, **Activities** et/ou **Logs** le commit et le push effectués vers **`develop`**.
7. Dans GitHub/GitLab, comparer les derniers commits, les objets représentés et le SHA.

> Pour une granularité de commits liée aux **tâches** plutôt qu'à la seule libération des **demandes**, SAP propose des mécanismes d'intégration BAdI : ils ne sont **pas activés implicitement** par cette documentation. Les tester séparément avant de les intégrer au runbook.

<!-- CAPTURE 21 — INSÉRER ICI l'objet ABAP dans ADT/SE24 avec son package et son transport de DEV. Exemple : ![Objet ABAP DEV](./captures/21-objet-abap-dev.png). -->

<!-- CAPTURE 22 — INSÉRER ICI la demande de transport DEV dans SE09/SE10 avant/après libération (masquer les données sensibles). Exemple : ![Transport DEV](./captures/22-transport-dev.png). -->

<!-- CAPTURE 23 — INSÉRER ICI l'onglet Commits/Activities de gCTS DEV prouvant le push vers develop et montrant un SHA. Exemple : ![Push gCTS DEV](./captures/23-gcts-push-dev.png). -->

#### Étape B — Revue et promotion dans Git

1. Vérifier que le commit gCTS attendu est bien dans **`develop`**.
2. Créer une **Pull Request GitHub** ou **Merge Request GitLab** : **source `develop` → cible `main`**.
3. Relire les objets modifiés, les dépendances, les risques d'activation/import et les résultats des contrôles qualité.
4. Faire approuver la PR/MR par au moins **une personne différente de l'auteur**, si la politique interne le demande.
5. Fusionner **sans réécrire les SHAs des commits gCTS** ; noter le **SHA de tête de `main`** résultant. Le commit de fusion peut avoir un SHA distinct des commits ABAP originaux.
6. Renseigner le registre des promotions : date, responsable, PR/MR, commit attendu et test envisagé.

> **Ne pas modifier les sources ABAP exportées par gCTS dans l'éditeur Git pour « réparer » un conflit.** Suspendre la fusion, analyser la divergence, corriger de préférence dans DEV via le processus SAP, puis générer un nouveau commit gCTS. Si des conflits de fusion subsistent, utiliser un processus validé spécifique à gCTS.

<!-- CAPTURE 24 — INSÉRER ICI une Pull Request GitHub OU une Merge Request GitLab avec source develop, cible main et approbation visible. Exemple : ![PR MR vers main](./captures/24-pr-mr-develop-main.png). -->

<!-- CAPTURE 25 — INSÉRER ICI la page du commit ou historique main indiquant le SHA final, sans secrets. Exemple : ![Commit main](./captures/25-commit-main.png). -->

#### Étape C — Import contrôlé dans QUAL

1. Obtenir l'accord de passage en **QUAL** ; vérifier que l'import ne perturbera pas une campagne de tests en cours.
2. Ouvrir gCTS **QUAL**, sélectionner le même dépôt en rôle **Provided**.
3. Vérifier **branche `main`**, commit actuellement installé et **SHA attendu** de l'étape B.
4. Dans l'onglet **Commits**, récupérer/actualiser l'historique distant si nécessaire et déclencher l'action gCTS permettant d'**importer le dernier commit de `main`** (par exemple **Update to Latest Commit**) **ou le commit approuvé explicitement** si le scénario et l'interface du niveau installé le permettent.
5. Confirmer l'opération d'import ; suivre **Activities / Logs / Import History** et le résultat des transports générés par gCTS.
6. Vérifier le statut de l'activité, les éventuels return codes, les erreurs d'activation/dépendance et l'état des objets. Un statut Git « au dernier commit » **ne suffit pas** si l'import SAP a échoué.
7. Réaliser les **tests techniques et fonctionnels** en QUAL ; renseigner le résultat et le **SHA réellement installé**.

> Un *pull* gCTS en QUAL génère une demande de transport technique d'import représentant les écarts entre commits ; **on n'importe pas la demande DEV via STMS comme dans le CTS classique**. Les transports et logs SAP restent cependant disponibles pour diagnostiquer l'exécution.

<!-- CAPTURE 26 — INSÉRER ICI l'onglet Commits de gCTS QUAL avant l'import : main, commit courant, commit cible. Exemple : ![gCTS QUAL avant pull](./captures/26-qual-avant-pull.png). -->

<!-- CAPTURE 27 — INSÉRER ICI l'opération gCTS Update to Latest Commit / pull sur QUAL et sa confirmation. Exemple : ![Pull QUAL](./captures/27-pull-gcts-qual.png). -->

<!-- CAPTURE 28 — INSÉRER ICI Activities, Logs et/ou Import History dans QUAL montrant un import réussi et le SHA installé. Exemple : ![Logs import QUAL](./captures/28-logs-import-qual.png). -->

<!-- CAPTURE 29 — INSÉRER ICI l'objet ABAP importé/activé dans QUAL, plus le résultat d'un test ABAP Unit/ATC ou d'un test métier. Exemple : ![Validation QUAL](./captures/29-validation-abap-qual.png). -->

### 8.2 Exemple de registre de promotion

| Champ | Valeur d'exemple |
|---|---|
| Application / dépôt | `s4-sd-extensions` |
| Numéro de demande SAP source | `[SID_DEV]K9xxxxx` |
| Branche source | `develop` |
| PR/MR | `#123` |
| Branche cible | `main` |
| SHA validé | `[SHA_COMMIT_MAIN]` |
| Date/heure d'import QUAL | `[AAAA-MM-JJ HH:MM TZ]` |
| Résultat gCTS | `[Succès / Échec et référence log]` |
| Résultat de tests QUAL | `[OK / KO et preuve]` |
| Décision | `[Accepté / Retour en correction]` |

## 9. Contrôles d'exploitation, incidents et retour arrière

### 9.1 Checklist de mise en service

#### DEV

- [ ] SAP Notes et restrictions du niveau 2021 FPS02 revues par Basis.
- [ ] Java, `abap2vcs.jar`, répertoire `gcts` et sortie réseau validés.
- [ ] Services Fiori/OData/ICF actifs ; tuile visible.
- [ ] Utilisateurs/permissions SAP et Git vérifiés.
- [ ] `VCS_PATH`, `JAVA_RUNTIME`, `A2G_RUNTIME` et job observateur validés.
- [ ] vSID, couche gCTS et packages ABAP documentés.
- [ ] Dépôt **Development** prêt, branche `develop` active.
- [ ] Transport ABAP de test visible comme commit/push sur Git.

#### QUAL

- [ ] Fiori, Java, client Git et Health Check validés indépendamment de DEV.
- [ ] Dépôt distant identique à DEV, rôle **Provided**, branche `main` active.
- [ ] Accès Git en lecture et import manuel testés.
- [ ] Import du SHA approuvé tracé dans **Activities / Import History**.
- [ ] Objets activés et tests QUAL satisfaisants.
- [ ] Documentation des écarts et procédures de retour arrière approuvées.

### 9.2 Diagnostic rapide

| Symptôme | Vérification prioritaire |
|---|---|
| Tuile gCTS absente | rôles/cataloques Fiori, service OData `SCTS_GCTS_SRV`, services SICF |
| **Health Check** en erreur | paramètres gCTS, chemin Java, `abap2vcs.jar`, permissions `<sid>adm`, job `SM37` |
| Clone échoue | dépôt non vide, URL correcte, droits Git, TLS/proxy/ports, gestion des endpoints |
| Erreur d'authentification GitLab/GitHub | expiration et scopes du token, type Git, URL de l'endpoint, SAP Notes d'authentification |
| Aucun commit après libération du transport | couche de transport et package gCTS, branche active, statut READY, job/logs, paramètres push |
| Refus de push sur `develop` | protection Git, compte d'exécution, autorisations Git, conflits éventuels |
| `main` sans le changement attendu | PR/MR non fusionnée, mauvaise branche, commit non poussé ou SHA erroné |
| QUAL ne voit pas le dernier commit | actualisation Git distante, branche active, authentification/connexion |
| Import QUAL en erreur | gCTS Activities/Logs, Import History, types d'objets et dépendances, activation/locks |
| Écart entre code et SHA affiché | examiner le résultat effectif de l'import SAP ; ne pas conclure au succès d'après Git seul |

### 9.3 Retour arrière et correction

**Ne pas promettre qu'un `git revert` ou un retour sur un ancien commit restaure intégralement le système SAP.** Un import gCTS peut rencontrer des dépendances, changements de dictionnaire, données ou opérations non réversibles. La stratégie de remédiation doit donc être :

1. **Geler** toute nouvelle promotion vers QUAL.
2. Identifier dernier **SHA sain**, SHA importé, erreurs d'import et impacts métier.
3. Préférer une **correction compensatoire** dans DEV, nouveau transport/commit gCTS sur `develop`, nouvelle PR/MR vers `main`, puis nouvel import QUAL.
4. Si retour à un commit précédent envisagé, **le faire valider/tester par Basis et ABAP**, y compris données, DDIC, dépendances et objets supprimés.
5. Conserver les journaux, les SHAs et la décision d'incident ; **ne jamais réécrire l'historique de `main`** pour dissimuler un import en échec.

## 10. RACI indicatif

| Activité | Basis | Développement ABAP | Responsable Git | Responsable QUAL |
|---|---|---|---|---|
| Versions/Notes, Java, kernel, répertoires, réseau | **R** | C | C | I |
| Services Fiori, PFCG, Health Check, jobs gCTS | **R** | C | I | I |
| vSID, couche gCTS et architecture des transports | **R** | **C** | I | I |
| Création/protection des dépôts et gestion des secrets | C | C | **R** | I |
| Packages, demandes de transport, tests DEV | C | **R** | I | I |
| PR/MR et revue technique | I | **R** | **C** | I |
| Autorisation du commit à qualifier | I | C | C | **R** |
| Pull gCTS, suivi d'import, validation QUAL | **R** (technique) | C | I | **R** (fonctionnel) |

*R = responsable opérationnel ; C = consulté ; I = informé. Adapter à l'organisation réelle.*

## 11. Limites du périmètre et évolutions possibles

La chaîne réelle est **DEV → QUAL → PROD**, mais **aucune configuration ou procédure PROD n'est décrite dans ce document**. Les illustrations et preuves demandées concernent **DEV, QUAL, Git et SAP** uniquement. Si le dispositif est étendu, faire une **conception séparée de la promotion vers PROD**, sans recopier mécaniquement les réglages de QUAL.

Évolutions possibles à instruire **après validation du pilote** : automatisation de vérifications ABAP Unit/ATC via SAP Project Piper ou autre orchestrateur ; dépôt dédié au Customizing avec gCTS Registry ; règles de déploiement par SHA/tag ; surveillance des jobs/jetons ; stratégie de maintenance/hotfix selon les contraintes du paysage.

## 12. Références officielles

**SAP (priorité aux instructions correspondant à ABAP Platform/S/4HANA 2021 FPS02) :**

1. [SAP Help — Prérequis gCTS, version 2021 FPS02](https://help.sap.com/docs/ABAP_PLATFORM_2021/4a368c163b08418890a406d413933ba7/275739524c7348f0a0ab54fe1fe05954.html?locale=de-DE&version=202110.002) (page versionnée 2021 FPS02 ; disponible en allemand).
2. [SAP Help — App Implementation: gCTS, ABAP Platform 2021](https://help.sap.com/viewer/b5670aaaa2364a29935f40b16499972d/202110.LATEST/en-US/89e7c923ac7c46859ec87280bbf594d7.html) (OData/ICF/Fiori).
3. [SAP Help — Enable gCTS in an ABAP System](https://help.sap.com/docs/ABAP_PLATFORM_2021/4a368c163b08418890a406d413933ba7/e7a002e42d8544ae975c53bd112eb2f8.html) (assistant système et job observateur ; sélectionner **2021 FPS02** sur le portail).
4. [SAP Help — Create Git Repositories on an ABAP System](https://help.sap.com/docs/ABAP_PLATFORM_2021/4a368c163b08418890a406d413933ba7/4ca031d6cbee4569873e9b85a4aeb4a0.html) (rôles **Development/Provided** ; sélectionner **2021 FPS02**).
5. [SAP Help — Prepare ABAP Systems for Repositories](https://help.sap.com/docs/ABAP_PLATFORM_NEW/4a368c163b08418890a406d413933ba7/a938f8cddde0409c87deb7f9a0d90184.html) (vSID, TMS, couche ; sélectionner la version correspondante).
6. [SAP Help — CTS Entities Used in gCTS](https://help.sap.com/docs/ABAP_PLATFORM_NEW/4a368c163b08418890a406d413933ba7/ffd38f5ef3a74d93ba6123a5899e5658.html) (relations transports / Git, différences avec CTS classique).
7. [SAP Note 2821718 — Central Note gCTS](https://me.sap.com/notes/2821718) (**à consulter obligatoirement**).
8. [SAP Note 2888887 — Restrictions des types d'objets gCTS](https://me.sap.com/notes/2888887) ; [SAP Note 3046346 — Registry](https://me.sap.com/notes/3046346) si Customizing ultérieur.
9. [SAP — Pipeline gCTS / ABAP avec Project Piper](https://github.com/SAP/jenkins-library/blob/master/documentation/docs/scenarios/gCTS_Scenario.md) (piste d'automatisation ultérieure).

**Git :**

10. [GitHub — Protection des branches et revues](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/about-protected-branches).
11. [GitLab — Branches protégées](https://docs.gitlab.com/user/project/repository/branches/protected/) ; [GitLab — Personal access tokens](https://docs.gitlab.com/user/profile/personal_access_tokens/).

**Note sur les sources :** certaines URLs `help.sap.com` ouvrent la **dernière version** par défaut. Pour toute étape susceptible d'avoir évolué, **sélectionner explicitement « 2021 FPS02 »** dans le portail avant d'exécuter une action. Les portées des tokens, les mécanismes d'authentification et les politiques GitHub/GitLab évoluent : utiliser les documentations de sécurité de l'instance réellement déployée.

---

## Annexe A — Inventaire des captures à produire

| N° | Sujet | Système / outil | Emplacement prévu |
|---|---|---|---|
| 01 | Paysage réel et périmètre | SAP / architecture | Introduction |
| 02 | Dépôt et branches | GitHub ou GitLab | §1.2 |
| 03 | Prérequis OS | SAP DEV / Basis | §3.1 |
| 04 | Tuile gCTS DEV | Fiori DEV | §3.2 |
| 05 | Tuile gCTS QUAL | Fiori QUAL | §3.2 |
| 06 | Services OData/ICF | SAP DEV / QUAL | §3.2 |
| 07 | Accès au dépôt distant | GitHub ou GitLab | §3.3 |
| 08 | Assistant gCTS DEV | gCTS DEV | §4.1 |
| 09 | Health Check / job | SAP DEV | §4.1 |
| 10 | vSID et couche STMS | SAP DEV / TMS | §4.2 |
| 11 | Package et couche | SAP DEV | §4.3 |
| 12 | Branches develop/main | GitHub ou GitLab | §5.1 |
| 13 | Protection main | GitHub ou GitLab | §5.1 |
| 14 | Graphe des commits | GitHub ou GitLab | §5.3 |
| 15 | Credentials masqués | gCTS DEV | §6.1 |
| 16 | Dépôt rôle Development | gCTS DEV | §6.2 |
| 17 | Branche develop active | gCTS DEV | §6.2 |
| 18 | Health Check / job | gCTS QUAL / SM37 | §7.1 |
| 19 | Dépôt rôle Provided | gCTS QUAL | §7.2 |
| 20 | Branche main active | gCTS QUAL | §7.2 |
| 21 | Objet ABAP de test | SAP DEV | §8.1 A |
| 22 | Transport de test | SAP DEV | §8.1 A |
| 23 | Push et commit | gCTS DEV | §8.1 A |
| 24 | PR/MR validée | GitHub ou GitLab | §8.1 B |
| 25 | SHA de main | GitHub ou GitLab | §8.1 B |
| 26 | État avant import | gCTS QUAL | §8.1 C |
| 27 | Action de pull | gCTS QUAL | §8.1 C |
| 28 | Logs / import history | gCTS QUAL | §8.1 C |
| 29 | Objet et tests après import | SAP QUAL | §8.1 C |

Les emplacements sont des **commentaires HTML `<!-- CAPTURE ... -->`** placés dans le corps du guide. Ils ne s'affichent pas dans le rendu Markdown. Pour ajouter une image, insérer à leur place une syntaxe telle que :

```markdown
![Tuile gCTS du système DEV](./captures/04-tuile-gcts-dev.png)
```

En partage externe, **masquer systématiquement** : secrets, tokens, informations de connexion sensibles, données de clients, chemins internes non nécessaires et identifiants personnels.
