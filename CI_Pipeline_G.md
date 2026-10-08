# Guide pas à pas — Pipeline CI gCTS + GitHub + Jenkins

**Paysage :** SAP S/4HANA **2021 FPS02** on-premise — **DEV** et **TEST** — dépôt **GitHub** — **Jenkins / Project « Piper »**  
**Date du document :** 8 octobre 2026  
**Public :** équipes SAP Basis, développeurs ABAP, administrateurs Jenkins/DevOps  
**Objectif :** automatiser le déploiement des commits GitHub vers **TEST**, puis l'exécution des contrôles **ATC** et **ABAP Unit**.  
**État initial supposé :** gCTS fonctionne déjà sur DEV et TEST ; les deux systèmes communiquent avec le même dépôt GitHub et les transferts manuels fonctionnent.

> **Périmètre.** Ce guide **n'explique pas comment installer gCTS** : il ajoute une couche CI Jenkins sur la configuration existante. Les libellés Jenkins ci-dessous correspondent aux interfaces Jenkins courantes ; leur position peut varier selon la version et les plugins. Certains libellés SAP doivent être vérifiés dans l'interface effectivement livrée sur le système FPS02 (et ses SAP Notes).
>
> **Précaution :** faites le premier essai dans un dépôt/ensemble d'objets sans conséquence métier. Le pipeline importe du contenu **dans TEST** : ce n'est pas une simple compilation.

## 1. Architecture cible

```text
SAP S/4HANA 2021 FPS02                GitHub                        Jenkins                         SAP S/4HANA 2021 FPS02
        DEV                        dépôt distant                 CI pipeline                              TEST
         |                               |                            |                                    |
Modification / release TR                 |                            |                                    |
         +--- gCTS PUSH / commit ------->|                            |                                    |
                                         +--- push webhook ---------->|                                    |
                                         |                            +-- checkout du commit ------------|
                                         |                            +-- gctsDeploy ------------------->| gCTS pull + import
                                         |                            +-- gctsExecuteABAPQualityChecks ->| ATC + ABAP Unit
                                         |                            +-- publication des rapports        |
                                         |                            +-- (option) gctsRollback ------->| si échec
```

Les trois étapes SAP **Project « Piper »** pertinentes sont : [`gctsDeploy`](https://www.project-piper.io/steps/gctsDeploy/), [`gctsExecuteABAPQualityChecks`](https://www.project-piper.io/steps/gctsExecuteABAPQualityChecks/) et [`gctsRollback`](https://www.project-piper.io/steps/gctsRollback/). SAP donne un [scénario Jenkins de référence](https://www.project-piper.io/scenarios/gCTS_Scenario/), utilisable à partir de S/4HANA 2020 sous réserve des correctifs requis pour les tests.

### 1.1 Décisions à fixer avant de commencer

| Élément | Exemple fictif | À relever dans votre paysage |
|---|---|---|
| URL SAP DEV | `https://s4dev.exemple.local:44300` | Pas utilisée directement par le pipeline d'import |
| URL SAP TEST | `https://s4test.exemple.local:44300` | **Hôte cible** de l'API gCTS |
| Mandant SAP TEST | `100` | Mandant autorisé pour les tests ; **éviter `000` pour ABAP Unit** |
| GitHub URL du dépôt gCTS | `https://github.com/mon-org/mon-repo` | Reprendre **l'URL distante affichée par gCTS** |
| **ID du dépôt gCTS dans TEST** | `Z_MY_GCTS_REPO` | **ID local**, différent éventuellement du nom GitHub |
| Branche à déployer | `main` | Exemple : `main`, `dev`, `qa` ; utiliser la branche déjà prévue pour TEST |
| Utilisateur SAP technique TEST | `JENKINS_GCTS` | Compte avec autorisations d'exécution gCTS + ATC/AUnit |
| Jenkins URL | `https://jenkins.exemple.local/` | URL accessible pour GitHub webhook si déclenchement automatique |
| Identifiant credential ABAP | `sap-gcts-test` | Créé dans Jenkins, type *Username with password* |
| Identifiant credential GitHub SCM | `github-gcts-read` | Créé dans Jenkins, token HTTPS de lecture ou clé SSH |

**Convention de branches :** l'exemple utilise `main` et un job Jenkins **Pipeline** mono-branche. Si TEST suit `qa`, remplacez `main` par `qa` **à la fois** dans Jenkins et dans le `Jenkinsfile`. N'introduisez pas une stratégie de promotion de branches sans l'avoir définie avec les équipes DEV/TEST.

**Important sur les rôles :** dans l'application gCTS, le rôle du dépôt DEV doit normalement être **Development**, et celui de TEST **Provided** ([documentation SAP 2021](https://help.sap.com/docs/ABAP_PLATFORM_2021/4a368c163b08418890a406d413933ba7/4ca031d6cbee4569873e9b85a4aeb4a0.html)). Dans le paramètre API Piper `gctsDeploy.role`, les valeurs sont différentes : `SOURCE` / `TARGET`. **Pour votre dépôt TEST existant, le `Jenkinsfile` fourni ne renseigne pas `role` et ne cherche pas à modifier sa configuration.** Vérifiez impérativement l'ID du dépôt : `gctsDeploy` peut tenter de créer un dépôt s'il n'existe pas.

## 2. Vérifier les prérequis côté TEST (sans reconfigurer gCTS)

### Étape 2.1 — Vérifier l'état du dépôt sur TEST

1. Ouvrir le **SAP Fiori Launchpad** du système **TEST**.
2. Ouvrir l'application **Git-Enabled Change and Transport System** (souvent abrégée **gCTS**).
3. Dans **Repositories** / vue **Repository**, sélectionner le dépôt utilisé.
4. Noter l'**ID du dépôt**, son **Remote Repository URL**, son **Active Branch**, et le **commit actif** dans l'onglet **Commits**.
5. Vérifier que le rôle affiché est **Provided** et que le dépôt est cloné / opérationnel.
6. **Ne pas** relancer une création ni un clone du dépôt s'il fonctionne déjà.
7. Vérifier qu'aucun job automatique d'import gCTS n'entre en concurrence avec Jenkins sur ce même dépôt : le pipeline doit être l'orchestrateur des mises à jour TEST.

**Résultat attendu :** l'ID et le commit actif sur TEST sont connus et le dépôt peut être mis à jour manuellement.

### Étape 2.2 — Vérifier l'interface REST gCTS dans TEST

1. Dans SAP GUI sur **TEST**, exécuter la transaction **`SICF`**.
2. Dans le champ **Service Name**, entrer `cts_abapvcs`, puis **Execute (F8)**.
3. Vérifier que le service **`/sap/bc/cts_abapvcs`** est **activé**.
4. Si nécessaire, faire confirmer son activation par Basis. **Ne pas modifier l'authentification SICF sans analyse de sécurité.**
5. Récupérer l'hôte et le port HTTPS du serveur SAP depuis l'administration ICM (`SMICM`) / sa configuration d'accès ; ne pas confondre port Fiori, Web Dispatcher et port interne.
6. Depuis **l'agent Jenkins qui exécutera le job**, tester la résolution DNS, la connexion TLS et le routage vers l'URL cible.

Vérification réseau indicative depuis l'agent Linux :

```bash
curl -sS -o /dev/null -w 'HTTP %{http_code}\n' \
  'https://s4test.exemple.local:44300/sap/bc/cts_abapvcs'
```

Une réponse **401 / 403 / 405** peut indiquer que le serveur et le service sont joignables mais que l'appel de test n'est pas autorisé ou pas pris en charge en GET ; un **404** justifie une vérification du chemin/routage et un échec TLS justifie une vérification des certificats. Ce test **ne prouve pas** les autorisations gCTS.

Documentation SAP : [Implémentation de l'app gCTS — services ICF et OData](https://help.sap.com/docs/r/b5670aaaa2364a29935f40b16499972d/202110.LATEST/en-US/89e7c923ac7c46859ec87280bbf594d7.html).

### Étape 2.3 — Contrôler les autorisations et l'accès Git du compte Jenkins

1. Avec l'équipe Basis, créer ou réutiliser sur **TEST** un utilisateur technique dédié, exemple `JENKINS_GCTS` (`SU01`). Choisir un **type d'utilisateur** compatible avec la méthode d'authentification API et le mode de maintenance des credentials gCTS de votre système.
2. Lui accorder les droits nécessaires pour **lire/mettre à jour le dépôt gCTS existant, importer les objets, exécuter ATC et ABAP Unit** et lire les résultats. Utiliser les objets/roles gCTS et la politique de moindre privilège ; **ne pas attribuer `SAP_ALL` automatiquement**. Vérifier les refus via `SU53` ou `STAUTHTRACE` avec Basis.
3. Si le dépôt GitHub est **privé**, vérifier que **l'utilisateur ABAP réellement utilisé par la pipeline** a accès à ce dépôt via l'une des méthodes gCTS déjà en place :
   - **User-specific authentication** : dans l'app **gCTS > System > Manage Credentials** (selon l'UI FPS02, commande de gestion des paramètres/utilisateur), enregistrer un token GitHub approprié **pour le compte ABAP qui exécutera la pipeline**. Pour GitHub public, le point de terminaison d'API classique est `https://api.github.com/` ; type **GitHub**, authentification **Token**.
   - **Repository-specific authentication** : si votre exploitation utilise déjà cette option, vérifier son fonctionnement sans la remplacer. Elle peut être préférable si le compte technique SAP de type *System* ne peut pas ouvrir l'interface Fiori pour enregistrer des credentials utilisateur.
4. Tester un **Pull / Update to Latest** manuel avec le compte cible seulement dans un créneau sûr, et uniquement si cela n'introduit pas de changement non validé en TEST.

**Attention : trois authentifications distinctes** interviennent :

| Liaison | Identité/secret | Où configurer |
|---|---|---|
| Jenkins → GitHub (lecture du `Jenkinsfile`) | token de lecture GitHub ou clé SSH | **Jenkins Credentials** et configuration SCM du job |
| Jenkins → SAP TEST (appel REST gCTS) | utilisateur/mot de passe ABAP | **Jenkins Credentials**, `abapCredentialsId` |
| SAP TEST gCTS → GitHub (pull distant) | token GitHub utilisable par gCTS | **gCTS Credential Store** ou authentification du dépôt existant |

Un token GitHub enregistré dans Jenkins **n'est pas automatiquement transmis** au magasin des credentials gCTS de TEST.

Références : [authentification gCTS](https://help.sap.com/docs/latest/4a368c163b08418890a406d413933ba7/9e7368baf25641af8b65996ac97f5db0.html), [dépannage SAP S/4HANA 2021 — credentials / GitHub](https://help.sap.com/doc/fd883a5fdb2b4fc5824dcb2402e4f09b/2021/en-US/91f3b089d4de47d183b796b0e34b5685_1.pdf).

### Étape 2.4 — Préparer les contrôles qualité sur TEST

1. Dans SAP GUI sur **TEST**, ouvrir la transaction **`ATC`**.
2. Avec l'équipe ABAP, vérifier que les contrôles **ATC** sont disponibles et qu'une variante existe (par défaut `DEFAULT` si appropriée, ou une variante dédiée à votre projet).
3. S'assurer que les objets du dépôt disposent, lorsque pertinent, de **classes/méthodes de tests ABAP Unit**.
4. Vérifier avec Basis la mise en place de la **SAP Note `3159798`** pour l'utilisation de l'étape `gctsExecuteABAPQualityChecks` sur les versions concernées. **Ne pas considérer les checks utilisables uniquement parce que gCTS fonctionne.**
5. Utiliser un **mandant applicatif de TEST** et non `000` pour l'exécution d'ABAP Unit ; ne pas exécuter ces tests sur le mandant de production.

Référence : [Piper — gctsExecuteABAPQualityChecks, prérequis](https://www.project-piper.io/steps/gctsExecuteABAPQualityChecks/).

## 3. Préparer Jenkins et Project « Piper »

### Étape 3.1 — Vérifier les plugins Jenkins

1. Ouvrir Jenkins avec un administrateur.
2. Menu **Manage Jenkins > Plugins** (ancien UI : **Manage Plugins**).
3. Sous **Installed plugins**, vérifier la présence des plugins :
   - **Pipeline** (workflow / exécution de `Jenkinsfile`) ;
   - **Git** (checkout du dépôt GitHub) ;
   - **GitHub** (déclencheur *GitHub hook trigger for GITScm polling*) ;
   - **Credentials** et ses dépendances usuelles ;
   - **Warnings Next Generation** (publication des résultats ATC / AUnit).
4. Si un plugin manque : ouvrir **Available plugins**, le rechercher, cocher et choisir **Install** ; appliquer la procédure de redémarrage approuvée par votre équipe Jenkins.
5. Vérifier que les agents exécutant le job disposent d'un accès GitHub, d'un accès HTTPS vers SAP TEST et de Git (ou d'une méthode de checkout supportée). Les étapes Piper doivent pouvoir exécuter leurs dépendances sur l'agent.

**Ne pas confondre** le plugin tiers **ABAP Continuous Integration (`abapCi`)** avec les étapes **gCTS de Project « Piper »** : le premier **n'est pas requis** pour cette architecture.

### Étape 3.2 — Déclarer la Shared Library SAP Project « Piper »

1. Menu **Manage Jenkins > System** (ancien UI : **Manage Jenkins > Configure System**).
2. Rechercher la section **Global Trusted Pipeline Libraries** / **Global Pipeline Libraries**.
3. Choisir **Add**.
4. Renseigner :

   | Champ Jenkins | Valeur |
   |---|---|
   | **Name** | `piper-lib-os` |
   | **Default version** | Une branche ou un **tag validé par l'équipe Jenkins** (par exemple `master` pour un POC, mais **épingler un tag validé** en exploitation) |
   | **Load implicitly** | Désactivé si le `Jenkinsfile` appelle `@Library('piper-lib-os')` |
   | **Retrieval method** | `Modern SCM` |
   | **Source Code Management** | `Git` |
   | **Project Repository / Repository URL** | `https://github.com/SAP/jenkins-library.git` |
   | **Credentials** | Aucune pour ce dépôt public, sauf proxy/politique d'entreprise |

5. Cliquer **Save**.
6. Vérifier dans le premier build Jenkins que la bibliothèque est bien résolue et que les fonctions `gctsDeploy`, `gctsExecuteABAPQualityChecks` et `gctsRollback` sont connues.

Références : [Piper — Custom Jenkins](https://www.project-piper.io/infrastructure/customjenkins/), [Jenkins — Shared Libraries](https://www.jenkins.io/doc/book/pipeline/shared-libraries/).

### Étape 3.3 — Ajouter les credentials dans Jenkins

1. Menu **Manage Jenkins > Credentials**.
2. Sous **Stores scoped to Jenkins**, ouvrir **System > Global credentials (unrestricted)** (ou utiliser un domaine/espace limité au job si votre gouvernance le prévoit).
3. Cliquer **Add Credentials**.
4. Créer le secret **SAP TEST** :

   | Champ | Valeur |
   |---|---|
   | **Kind** | `Username with password` |
   | **Scope** | `Global` (ou portée du dossier/job si disponible) |
   | **Username** | `JENKINS_GCTS` |
   | **Password** | Mot de passe de cet utilisateur ABAP |
   | **ID** | `sap-gcts-test` |
   | **Description** | `Accès API gCTS SAP TEST` |

5. Cliquer **Create / OK**.
6. Si GitHub est privé et que Jenkins utilise **HTTPS**, créer le secret de lecture :

   | Champ | Valeur |
   |---|---|
   | **Kind** | `Username with password` (pour checkout Git HTTPS) |
   | **Username** | Compte GitHub autorisé |
   | **Password** | **Personal Access Token** GitHub avec droits de lecture nécessaires |
   | **ID** | `github-gcts-read` |
   | **Description** | `Lecture dépôt gCTS et Jenkinsfile` |

   Si Jenkins utilise Git via **SSH**, employer plutôt **SSH Username with private key** et sélectionner cette entrée dans la configuration SCM.
7. **Ne jamais** stocker le mot de passe SAP, la clé SSH ou le token GitHub en clair dans `Jenkinsfile`, `.pipeline/config.yml` ni dans des commandes imprimées dans la console.

Référence : [Jenkins — Using credentials](https://www.jenkins.io/doc/book/using/using-credentials/).

## 4. Créer le `Jenkinsfile` dans le dépôt GitHub

### Étape 4.1 — Ajouter un fichier à la racine

1. Dans GitHub, ouvrir **Repository > Code** et choisir la branche retenue (`main` dans l'exemple).
2. Cliquer **Add file > Create new file** (ou créer le fichier localement et ouvrir une pull request).
3. Dans **Name your file…**, entrer exactement **`Jenkinsfile`** (**sans extension**).
4. Coller le script ci-dessous, **remplacer toutes les valeurs fictives**.
5. Préférer une **pull request** et sa validation plutôt qu'un commit direct sur une branche protégée.

### Étape 4.2 — Exemple de pipeline prêt à adapter

Ce pipeline mono-branche effectue les actions suivantes : checkout **du commit qui déclenche le job**, déploiement déterministe de **ce commit** dans TEST, ATC + AUnit sur les objets de la dernière activité du dépôt local, publication XML, puis échec du job si les tests échouent. Le **rollback qualité est désactivé par défaut**, car il doit d'abord être validé dans votre contexte GitHub/gCTS (voir § 7).

```groovy
@Library('piper-lib-os') _

pipeline {
    agent any

    options {
        disableConcurrentBuilds()
        skipDefaultCheckout(true)
    }

    environment {
        // === A ADAPTER A VOTRE PAYSAGE ===
        SAP_TEST_URL          = 'https://s4test.exemple.local:44300'
        SAP_TEST_CLIENT       = '100'
        GCTS_REPOSITORY_ID    = 'Z_MY_GCTS_REPO'
        GCTS_REMOTE_URL       = 'https://github.com/mon-org/mon-repo'
        GCTS_BRANCH           = 'main'
        ABAP_CREDENTIALS_ID   = 'sap-gcts-test'

        // Mettre true APRES avoir teste le rollback (voir section 7).
        ENABLE_QUALITY_ROLLBACK = 'false'
        QUALITY_FAILED          = 'false'
    }

    stages {
        stage('01 - Checkout GitHub') {
            steps {
                script {
                    def checkoutInfo = checkout scm
                    env.TARGET_COMMIT = checkoutInfo.GIT_COMMIT
                    if (!env.TARGET_COMMIT?.trim()) {
                        error('GIT_COMMIT introuvable apres checkout SCM')
                    }
                    echo "Commit a deployer : ${env.TARGET_COMMIT}"
                }
            }
        }

        stage('02 - gCTS Deploy vers TEST') {
            steps {
                script {
                    gctsDeploy(
                        script: this,
                        host: env.SAP_TEST_URL,
                        client: env.SAP_TEST_CLIENT,
                        abapCredentialsId: env.ABAP_CREDENTIALS_ID,
                        repository: env.GCTS_REPOSITORY_ID,
                        remoteRepositoryURL: env.GCTS_REMOTE_URL,
                        branch: env.GCTS_BRANCH,
                        commit: env.TARGET_COMMIT,
                        rollback: true
                    )
                }
            }
        }

        stage('03 - ATC et ABAP Unit') {
            steps {
                script {
                    try {
                        gctsExecuteABAPQualityChecks(
                            script: this,
                            host: env.SAP_TEST_URL,
                            client: env.SAP_TEST_CLIENT,
                            abapCredentialsId: env.ABAP_CREDENTIALS_ID,
                            repository: env.GCTS_REPOSITORY_ID,
                            scope: 'localChangedObjects',
                            commit: env.TARGET_COMMIT,
                            workspace: env.WORKSPACE,
                            atcVariant: 'DEFAULT',
                            atcCheck: true,
                            aUnitTest: true
                        )
                    } catch (Exception ex) {
                        env.QUALITY_FAILED = 'true'
                        echo "ATC / AUnit en echec : ${ex.getMessage()}"
                    }
                }
            }
        }

        stage('04 - Rapports ATC et AUnit') {
            steps {
                recordIssues(
                    enabledForFailure: true,
                    aggregatingResults: true,
                    failOnError: false,
                    tools: [
                        checkStyle(pattern: 'ATCResults.xml', reportEncoding: 'UTF8'),
                        checkStyle(pattern: 'AUnitResults.xml', reportEncoding: 'UTF8')
                    ]
                )
            }
        }

        stage('05 - Rollback qualite (optionnel)') {
            when {
                expression {
                    env.QUALITY_FAILED == 'true' &&
                    env.ENABLE_QUALITY_ROLLBACK == 'true'
                }
            }
            steps {
                script {
                    gctsRollback(
                        script: this,
                        host: env.SAP_TEST_URL,
                        client: env.SAP_TEST_CLIENT,
                        abapCredentialsId: env.ABAP_CREDENTIALS_ID,
                        repository: env.GCTS_REPOSITORY_ID
                    )
                }
            }
        }

        stage('06 - Statut final') {
            steps {
                script {
                    if (env.QUALITY_FAILED == 'true') {
                        error('Pipeline KO : erreurs dans ATC et/ou ABAP Unit')
                    }
                    echo 'Pipeline OK : deployment et checks qualite termines'
                }
            }
        }
    }

    post {
        always {
            archiveArtifacts(
                artifacts: 'ATCResults.xml,AUnitResults.xml',
                allowEmptyArchive: true
            )
        }
    }
}
```

**À savoir sur cet exemple :**

- `checkout scm` utilise le dépôt **déclaré dans la configuration Jenkins**, pas un deuxième dépôt codé en dur. `checkoutInfo.GIT_COMMIT` correspond au commit réellement extrait ; il est ensuite passé à `gctsDeploy.commit` pour éviter de déployer silencieusement un commit plus récent.
- `scope: 'localChangedObjects'` se base sur la **dernière activité du dépôt local gCTS dans TEST**. Cela suppose que les mises à jour de ce dépôt TEST sont orchestrées de manière séquentielle par ce pipeline. Pour tester **tout le dépôt** à chaque exécution, choisir `scope: 'repository'` (plus coûteux).
- `atcVariant: 'DEFAULT'` doit être adaptée à votre variante ATC. Les paramètres `atcCheck` / `aUnitTest` activent explicitement les deux contrôles.
- `rollback: true` dans **`gctsDeploy`** traite certains échecs de déploiement/pull. **Cela ne remplace pas** le stage `gctsRollback` utilisé après un échec des contrôles qualité.
- Le fichier `ATCResults.xml` et `AUnitResults.xml` est attendu au format *Checkstyle* par **Warnings Next Generation** ; les noms correspondent aux valeurs par défaut documentées de l'étape Piper.
- Le pipeline est **un exemple à valider dans votre Jenkins**. Ne l'utilisez pas sur un périmètre sensible sans avoir vérifié la compatibilité de la version de la bibliothèque et les autorisations du compte technique.

Détails des paramètres : [gctsDeploy](https://www.project-piper.io/steps/gctsDeploy/) • [gctsExecuteABAPQualityChecks](https://www.project-piper.io/steps/gctsExecuteABAPQualityChecks/) • [gctsRollback](https://www.project-piper.io/steps/gctsRollback/).

### Étape 4.3 — Ajouter éventuellement `.pipeline/config.yml`

**Pas nécessaire pour commencer** : l'exemple ci-dessus centralise les paramètres dans `environment`. Project « Piper » permet aussi de les externaliser dans `.pipeline/config.yml` pour différencier les paysages (DEV/TEST). La configuration YAML est une **alternative**, pas un second moyen de définir simultanément des valeurs contradictoires.

Exemple minimal indicatif :

```yaml
# .pipeline/config.yml — exemple d'organisation, non utilise
# par le Jenkinsfile de la section 4.2
steps:
  gctsDeploy:
    host: 'https://s4test.exemple.local:44300'
    client: '100'
    repository: 'Z_MY_GCTS_REPO'
    remoteRepositoryURL: 'https://github.com/mon-org/mon-repo'
    branch: 'main'
    abapCredentialsId: 'sap-gcts-test'
```

Si vous adoptez cette approche, adapter les appels Piper dans le `Jenkinsfile` et tester leur chargement des paramètres selon la version de la bibliothèque retenue. Référence : [Piper — configuration](https://www.project-piper.io/configuration/).

## 5. Créer le job Pipeline dans Jenkins

1. Jenkins, menu **New Item**.
2. Champ **Enter an item name** : `CI-gCTS-S4-TEST`.
3. Type **Pipeline**, puis **OK**.
4. Section **General** : renseigner la **Description** : `Déploiement GitHub via gCTS + ATC/AUnit sur SAP TEST`.
5. Section **Build Triggers** : **ne pas activer le webhook avant le premier essai manuel**. Ce sera fait à l'étape 6.
6. Section **Pipeline** :

   | Champ / option | Valeur à renseigner |
   |---|---|
   | **Definition** | `Pipeline script from SCM` |
   | **SCM** | `Git` |
   | **Repositories > Repository URL** | `https://github.com/mon-org/mon-repo.git` (URL de clone Jenkins, ou URL SSH) |
   | **Repositories > Credentials** | `github-gcts-read` (si dépôt privé) |
   | **Branches to build > Branch Specifier (blank for 'any')** | `*/main` (adapter) |
   | **Script Path** | `Jenkinsfile` |
   | **Lightweight checkout** | Laisser activé si compatible ; sinon désactiver en cas de souci SCM |

7. Cliquer **Save**.
8. Ouvrir le job, cliquer **Build Now**.
9. Ouvrir **Build History > #N > Console Output** et vérifier le déroulement des six étapes et le SHA affiché.
10. Dans SAP TEST **gCTS > dépôt > Commits / Activities / Logs**, vérifier le commit réellement actif et les messages d'import. En cas d'erreur d'import, compléter par les journaux SAP de transport correspondants.

**Résultat attendu :** un build manuel vérifie l'accès GitHub, la Shared Library, la connexion API TEST et les contrôles qualité. Ne poursuivez pas avec un webhook avant d'obtenir un build propre et une concordance du SHA GitHub/SAP.

## 6. Activer le déclenchement automatique par GitHub Webhook

### Étape 6.1 — Activer le trigger du job Jenkins

1. Jenkins : ouvrir le job **`CI-gCTS-S4-TEST`** puis **Configure**.
2. Sous **Build Triggers**, cocher **GitHub hook trigger for GITScm polling**.
3. Cliquer **Save**.

**Important :** cette option est fournie par le plugin **GitHub**. Le webhook signale un évènement, puis Jenkins vérifie si une modification de la branche configurée doit lancer le build. Ce n'est pas la même chose qu'un job `Multibranch Pipeline` (qui dispose d'une configuration de source et de déclenchement spécifique).

### Étape 6.2 — Créer le webhook dans GitHub

1. Dans le dépôt GitHub : **Settings > Webhooks > Add webhook**.
2. Renseigner les champs :

   | Champ GitHub | Valeur |
   |---|---|
   | **Payload URL** | `https://jenkins.exemple.local/github-webhook/` |
   | **Content type** | `application/json` |
   | **Secret** | Secret fort **uniquement si sa validation est configurée côté Jenkins** ; faire correspondre exactement les valeurs selon le plugin et la politique sécurité |
   | **Which events would you like to trigger this webhook?** | `Just the push event` |
   | **Active** | Coché |

3. Cliquer **Add webhook**.
4. GitHub : ouvrir le webhook nouvellement créé > **Recent Deliveries** et vérifier le ping initial / la livraison.
5. Faire un **push contrôlé** sur la branche `main` ; vérifier la livraison `push` et l'apparition d'un nouveau build dans Jenkins.
6. Vérifier l'URL du webhook : le slash final de **`/github-webhook/`** est important. Si Jenkins est publié sous un *context path*, conserver ce chemin : `https://ci.exemple.local/jenkins/github-webhook/`.

**Réseau et sécurité :** GitHub doit pouvoir joindre l'URL publique ou accessible par son infrastructure. Si votre Jenkins est **strictement interne**, l'appel GitHub direct ne fonctionnera pas : demander un reverse proxy sécurisé, une passerelle compatible ou une stratégie de déclenchement alternatif approuvée (par exemple polling SCM temporaire). **Ne pas exposer directement Jenkins sans contrôle d'accès ni durcissement.**

Références : [GitHub — Créer un webhook](https://docs.github.com/fr/webhooks/using-webhooks/creating-webhooks), [Jenkins — plugin GitHub / Manual Mode](https://plugins.jenkins.io/github), [GitHub — vérifier les signatures webhook](https://docs.github.com/en/webhooks/using-webhooks/validating-webhook-deliveries).

## 7. Activer et fiabiliser le rollback (optionnel, après tests)

Il faut distinguer :

1. **Rollback d'échec d'import** : déjà demandé au paramètre `rollback: true` de `gctsDeploy` ; il vise un retour à un état exploitable si le déploiement ne réussit pas.
2. **Rollback d'échec des contrôles ATC/AUnit** : le stage Jenkins **05** appelle `gctsRollback` uniquement si `QUALITY_FAILED == 'true'` **et** `ENABLE_QUALITY_ROLLBACK == 'true'`.

**Comportement à connaître :** selon la [documentation actuelle de `gctsRollback`](https://www.project-piper.io/steps/gctsRollback/), si le paramètre **`commit`** n'est pas fourni et que le dépôt est sur **github.com**, la fonction peut chercher le **dernier commit ayant un statut GitHub `success`** ; dans d'autres cas elle revient au commit actif précédent dans le dépôt local. **Ce choix doit être vérifié dans votre installation avant activation**, notamment si Jenkins ne publie pas de statuts de commit GitHub.

### Procédure de mise en service du rollback qualité

1. Relever le **SHA stable actuellement actif dans TEST** (gCTS > dépôt > **Commits**).
2. Préparer un changement ABAP *non critique* provoquant volontairement un échec ATC/AUnit.
3. Dans un environnement de test contrôlé, tester le retour en arrière vers le **SHA stable connu**. Lorsqu'un SHA exact est nécessaire, l'étape accepte un paramètre **`commit: '<SHA_CIBLE>'`** ; **ne pas le confondre** avec le SHA du nouveau commit défaillant.
4. Vérifier les conséquences éventuelles des changements de structures DDIC, dépendances, personnalisations et données : un rollback de code **n'est pas une restauration intégrale de base de données**.
5. Lorsque le comportement `gctsRollback` sans SHA est validé, passer dans le `Jenkinsfile` : `ENABLE_QUALITY_ROLLBACK = 'true'` et ouvrir une pull request.
6. Rejouer les cas **succès**, **échec des tests** et **échec du déploiement** et confirmer que le statut Jenkins reste **FAILURE** après un échec même si le rollback a réussi.

**Recommandation :** pour des mises en production ou des paysages complexes, définir une procédure formelle de promotion et de rollback par SHA, approuvée par Basis/ABAP. Le cas présenté ici concerne **uniquement TEST**.

## 8. Recette de validation — cas à exécuter

| ID | Test | Résultat attendu |
|---|---|---|
| T01 | `Build Now` sur un commit connu | Pipeline `SUCCESS`, SHA attendu déployé en TEST |
| T02 | Push sur `main` avec changement ABAP valide | GitHub Webhook OK, Jenkins démarre, import TEST, ATC/AUnit OK |
| T03 | Commit contenant erreur ATC/AUnit contrôlée | Rapport visible, job `FAILURE` ; rollback seulement si activé |
| T04 | Modifier uniquement le `Jenkinsfile` | Job déclenché ; aucune donnée ABAP non voulue ne doit être importée |
| T05 | Identifiants SAP incorrects dans un environnement non critique | Échec d'authentification explicite ; aucun import réussi |
| T06 | Webhook indisponible | La livraison GitHub signale l'échec ; lancement manuel toujours possible |
| T07 | Déploiement du même SHA deux fois | Comportement réentrant vérifié ; pas d'import concurrent inattendu |
| T08 | Test du rollback avec SHA stable (si retenu) | Retour à l'état vérifié + job en échec après tests KO |

### Critères d'acceptation

- [ ] DEV pousse les changements ABAP dans GitHub comme aujourd'hui, sans adaptation de la configuration gCTS.
- [ ] Le job Jenkins lit le `Jenkinsfile` depuis GitHub.
- [ ] Les credentials Jenkins (GitHub SCM et SAP TEST) sont séparés et protégés.
- [ ] Le job déploie **le SHA attendu** sur TEST, sur le **dépôt gCTS existant**.
- [ ] Les rapports ATC / ABAP Unit sont publiés et consultables dans Jenkins.
- [ ] Un échec de qualité donne un job **FAILED**.
- [ ] Le webhook déclenche automatiquement le job après push sur la branche retenue.
- [ ] Aucune exécution parallèle d'import n'est autorisée pour le même dépôt TEST.
- [ ] Le rollback n'est activé qu'après recette explicite.

## 9. Diagnostic rapide

| Symptôme | Où regarder | Vérifications / résolution |
|---|---|---|
| `No such DSL method 'gctsDeploy'` | `Console Output` Jenkins | Bibliothèque `piper-lib-os` absente, mal nommée, non chargée ou version incompatible ; cf. § 3.2 |
| `Authentication failed`, HTTP 401/403 vers SAP | Jenkins logs, SAP `SU53` / `STAUTHTRACE` | Secret `sap-gcts-test`, client SAP, permissions du compte technique, SSO/SAML éventuellement incompatible avec authentification API |
| URL SAP injoignable, SSL handshake | Agent Jenkins, ICM, proxy | DNS/firewall, CA du certificat SAP, port HTTPS. **Ne pas contourner SSL en production** (`skipSSLVerification: false` par défaut) |
| HTTP 404 sur gCTS REST | `SICF`, proxy | Service `/sap/bc/cts_abapvcs` actif, bonne cible Web Dispatcher / routage |
| Dépôt gCTS inaccessible depuis TEST | App gCTS **System > Manage Credentials**, **Activities / Log** | Vérifier les credentials **du compte SAP exécutant la pipeline**, URL GitHub, proxy et certificats |
| Le commit TEST n'est pas le SHA Jenkins | Jenkins `Checkout`/`Deploy`, gCTS **Commits** | Vérifier `branch`, `TARGET_COMMIT`, auto-pull concurrent, dépôt/branche cible |
| Contrôles qualité échouent à démarrer | Jenkins, `ATC`, Basis | Vérifier **SAP Note 3159798**, mandant ≠ `000`, ATC actif, autorisations |
| Rapports XML absents | Workspace Jenkins, plugin Warnings | Vérifier que l'étape qualité a été exécutée, noms `ATCResults.xml` / `AUnitResults.xml`, droits d'écriture agent |
| Webhook GitHub OK mais aucun build | GitHub **Recent Deliveries**, Jenkins **Configure** | URL `/github-webhook/`, plugin GitHub, trigger coché, dépôt/branche SCM corrects, GitHub peut joindre Jenkins |
| `gctsRollback` ne revient pas au SHA attendu | Logs Jenkins, historique GitHub, gCTS commits | Vérifier statut GitHub `success`, historique gCTS, et tester le rollback avec **SHA explicite** avant d'automatiser |
| Import ABAP en erreur | App gCTS **Activities / Logs**, logs CTS / Basis | Analyser dépendances, activation objets, conflits, transports et erreurs d'import |

## 10. Ordre de déploiement recommandé

1. **Jour 1 :** inventaire des valeurs réelles (URL TEST, mandant, ID dépôt gCTS, branche) ; contrôle des droits et de `3159798`.
2. **Jour 1–2 :** Jenkins + plugins + Shared Library + credentials ; premier `Build Now` sur commit maîtrisé.
3. **Jour 2 :** activation GitHub webhook ; vérification d'un push réel ; revue des rapports.
4. **Après validation :** éventuellement passage du rollback qualité de `false` à `true`, après test de retour à un SHA stable.
5. **Exploitation :** maintenir la version Piper, les plugins Jenkins, la rotation des secrets, les logs et les autorisations des branches.

---

## 11. Liens officiels et ressources de référence

| Sujet | URL |
|---|---|
| **Scénario officiel Project Piper pour gCTS** | https://www.project-piper.io/scenarios/gCTS_Scenario/ |
| Étape `gctsDeploy` / paramètres | https://www.project-piper.io/steps/gctsDeploy/ |
| Étape `gctsExecuteABAPQualityChecks` / SAP Note 3159798 | https://www.project-piper.io/steps/gctsExecuteABAPQualityChecks/ |
| Étape `gctsRollback` | https://www.project-piper.io/steps/gctsRollback/ |
| SAP Help — Intégration gCTS dans les pipelines CI | https://help.sap.com/docs/sap_s4hana_on-premise/4a368c163b08418890a406d413933ba7/96b68f645556471083aaae0b2d574625.html |
| SAP Help — Création des dépôts gCTS, édition ABAP Platform 2021 | https://help.sap.com/docs/ABAP_PLATFORM_2021/4a368c163b08418890a406d413933ba7/4ca031d6cbee4569873e9b85a4aeb4a0.html |
| SAP Help — Implémentation de l'application Fiori gCTS (S/4HANA 2021) | https://help.sap.com/docs/r/b5670aaaa2364a29935f40b16499972d/202110.LATEST/en-US/89e7c923ac7c46859ec87280bbf594d7.html |
| SAP Help — Dépannage gCTS SAP S/4HANA 2021 | https://help.sap.com/doc/fd883a5fdb2b4fc5824dcb2402e4f09b/2021/en-US/91f3b089d4de47d183b796b0e34b5685_1.pdf |
| SAP Note centrale gCTS **2821718** (connexion SAP requise) | https://me.sap.com/notes/2821718 |
| SAP Note qualité **3159798** (connexion SAP requise) | https://me.sap.com/notes/3159798 |
| Jenkins — Bibliothèques partagées | https://www.jenkins.io/doc/book/pipeline/shared-libraries/ |
| Jenkins — Gestion des credentials | https://www.jenkins.io/doc/book/using/using-credentials/ |
| Jenkins — Plugin GitHub / déclencheurs webhook | https://plugins.jenkins.io/github |
| Jenkins — Plugin Warnings Next Generation | https://plugins.jenkins.io/warnings-ng/ |
| GitHub — Création de webhooks (FR) | https://docs.github.com/fr/webhooks/using-webhooks/creating-webhooks |
| GitHub — Validation de la signature d'un webhook | https://docs.github.com/en/webhooks/using-webhooks/validating-webhook-deliveries |

**Points restant à confirmer avec votre équipe avant mise en œuvre :** nom/ID exact du dépôt dans TEST, branche d'alimentation TEST, mandant cible, compte technique, version de Jenkins et des plugins, règle de rollback et statut de la SAP Note `3159798`. Les exemples `exemple.local` et `mon-org/mon-repo` sont fictifs.
