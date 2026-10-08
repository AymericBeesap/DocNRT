# Pipeline CI gCTS ↔ GitHub ↔ Jenkins — guide pas à pas

**Contexte** : SAP S/4HANA 2021 FPS02, deux systèmes ABAP (DEV et TEST), gCTS déjà configuré et fonctionnel avec un dépôt GitHub.
**Objectif** : à chaque nouveau commit poussé par gCTS depuis DEV vers GitHub, Jenkins déploie automatiquement ce commit sur TEST, exécute ABAP Unit + ATC, publie les résultats et fait un rollback si les contrôles échouent.

**Outillage** : bibliothèque SAP Project "Piper" (`piper-lib-os`), steps `gctsDeploy`, `gctsExecuteABAPQualityChecks`, `gctsRollback`.

---

## 0. Vue d'ensemble du flux

```
[DEV] Libération de l'OT
   └─► gCTS push → commit sur GitHub (branche main)
          └─► Webhook GitHub → Jenkins (job Multibranch)
                 ├─ 1. gctsDeploy                 → pull du commit sur TEST
                 ├─ 2. gctsExecuteABAPQualityChecks → ABAP Unit + ATC sur TEST
                 ├─ 3. recordIssues               → résultats dans Warnings NG
                 └─ 4. gctsRollback (si échec)     → TEST revient au dernier commit OK
```

Référence officielle du scénario : [Project Piper – gCTS Scenario](https://www.project-piper.io/scenarios/gCTS_Scenario/)

---

## 1. Valeurs à préparer (à adapter)

| Variable | Exemple | Où la trouver |
|---|---|---|
| Hôte HTTPS du système TEST | `https://s4tst.mondomaine.fr:44300` | SMICM → *Goto > Services* (port HTTPS) |
| Mandant TEST | `100` | Ne **pas** utiliser 000 ni le mandant productif |
| ID du dépôt gCTS (local) | `zmon_repo` | App gCTS → colonne *Repository* |
| vSID du dépôt sur TEST | `TST` | App gCTS → dépôt → onglet *Configuration* / *Overview* |
| URL GitHub du dépôt | `https://github.com/mon-org/mon-repo.git` | GitHub |
| Branche suivie | `main` | GitHub |
| Utilisateur technique SAP | `JENKINS_CI` | À créer (étape 2.2) |
| URL publique Jenkins | `https://jenkins.mondomaine.fr` | Admin Jenkins |

---

## 2. Côté SAP (système TEST, et vérifs sur DEV)

### 2.1 Services ICF (transaction `SICF`)

Vérifier que ces nœuds sont **actifs** sur TEST (clic droit > *Activate Service*) :

| Service | Rôle |
|---|---|
| `/sap/bc/cts_abapvcs` | API REST gCTS (utilisée par `gctsDeploy` / `gctsRollback`) |
| `/sap/bc/ui5_ui5/sap/bc_cts_git` | App Fiori *Git-enabled CTS* |
| `/sap/bc/adt` | Services ADT, sollicités pour ABAP Unit / ATC |

### 2.2 Utilisateur technique pour Jenkins (transactions `SU01` / `PFCG`)

1. `SU01` → créer `JENKINS_CI` → onglet *Logon Data* → *User Type* : **System** (ou *Service* selon votre politique).
2. Affecter les rôles :
   - `SAP_BC_GCTS_ADMIN` (ou une copie `Z…`) — autorisations gCTS
   - un rôle contenant le catalogue Fiori `SAP_BASIS_TCR_T` si l'utilisateur doit ouvrir l'app (utile pour l'étape 2.4)
   - autorisations développeur en **affichage** + exécution ABAP Unit / ATC (`S_DEVELOP` ACTVT 03, `S_ATC_ADM`/`S_Q_ADM` selon vos règles) — ajuster après un premier passage avec `SU53` / `STAUTHTRACE`.
3. Mot de passe : définir un mot de passe productif (pas initial).

### 2.3 Configuration du dépôt gCTS sur TEST

App **Git-enabled CTS** (Fiori Launchpad, ou URL directe `https://<host>:<port>/sap/bc/ui5_ui5/sap/bc_cts_git/index.html?sap-client=<mandant>`)

1. *Repositories* → ouvrir votre dépôt.
2. Onglet **Configuration** → vérifier / modifier :

| Paramètre | Valeur sur TEST | Pourquoi |
|---|---|---|
| `VCS_AUTOMATIC_PULL` (*Automatic Pull*) | **FALSE** | C'est Jenkins qui pilote le pull ; sinon conflit avec le pipeline |
| `VCS_AUTOMATIC_PUSH` | FALSE | TEST ne pousse rien |
| `CLIENT_VCS_LOGLVL` | `info` (passer à `debug` si besoin) | Diagnostic |

3. Vérifier que le **rôle** du dépôt sur TEST est *Provided* (TARGET) et sur DEV *Development* (SOURCE).

> Sur DEV, rien ne change : le push vers GitHub se fait comme aujourd'hui à la libération de l'OT.

### 2.4 Authentification GitHub pour l'utilisateur technique

Le pipeline s'exécute sous `JENKINS_CI` : cet utilisateur doit avoir **ses propres credentials GitHub** dans gCTS.

1. Se connecter à l'app gCTS **avec `JENKINS_CI`** sur TEST.
2. Icône utilisateur / *User-specific settings* → **Add Credentials** :
   - *API Endpoint* : `https://api.github.com`
   - *Type* : **Token**
   - *Token* : PAT GitHub (droits lecture sur le repo — voir 3.1)
3. Enregistrer.

Doc SAP : [Set User-Specific Authentication](https://help.sap.com/docs/ABAP_PLATFORM_NEW/4a368c163b08418890a406d413933ba7/3431ebd6fbf241778cd60587e7b5dc3e.html) (libellés susceptibles de varier légèrement selon le SP).

### 2.5 ATC (transaction `ATC`)

Sur TEST : *ATC Administration* → *Setup* → *Configure ATC* → vérifier que l'ATC est activé et qu'une **variante globale** existe (par défaut `DEFAULT`, sinon noter le nom pour le paramètre `atcVariant`).

### 2.6 Notes SAP à contrôler (`SNOTE`)

| Note | Objet |
|---|---|
| [2821718](https://launchpad.support.sap.com/#/notes/2821718) | Note centrale gCTS — appliquer les corrections listées pour 2021 |
| [3159798](https://launchpad.support.sap.com/#/notes/3159798) | Prérequis du step `gctsExecuteABAPQualityChecks` (vérifier si déjà incluse dans votre SP) |

---

## 3. Côté GitHub

### 3.1 Personal Access Token pour Jenkins

GitHub → photo de profil → **Settings** → **Developer settings** → **Personal access tokens** → *Fine-grained tokens* → **Generate new token**

- *Repository access* : **Only select repositories** → votre dépôt
- *Permissions* :
  - *Contents* : **Read-only**
  - *Commit statuses* : **Read and write** (utile au rollback sur le dernier commit « success »)
  - *Metadata* : Read-only (automatique)
  - *Webhooks* : Read and write (si vous laissez Jenkins gérer le webhook)

Copier le token (affiché une seule fois).

### 3.2 Webhook vers Jenkins

Dépôt GitHub → **Settings** → **Webhooks** → **Add webhook**

| Champ | Valeur |
|---|---|
| *Payload URL* | `https://jenkins.mondomaine.fr/github-webhook/` (slash final obligatoire) |
| *Content type* | `application/json` |
| *Secret* | optionnel (à reporter dans Jenkins si utilisé) |
| *Which events…* | **Just the push event** |
| *Active* | coché |

> Si Jenkins n'est pas joignable depuis Internet : pas de webhook, utilisez le scan périodique (étape 4.5).

### 3.3 Jenkinsfile dans le dépôt

Le `Jenkinsfile` (section 5) est commité **à la racine** du dépôt GitHub, à côté du dossier des objets ABAP. gCTS ne traite que les objets ABAP ; vérifiez sur un premier pull que le fichier n'est pas signalé en erreur (sinon, isoler les objets via le paramètre de sous-répertoire du dépôt).

---

## 4. Côté Jenkins

Prérequis : Jenkins LTS sur Linux, accès sortant à `github.com` (Piper télécharge son binaire depuis les releases GitHub), accès réseau au port HTTPS du système TEST. Recommandations Piper : [Custom Jenkins Setup](https://www.project-piper.io/infrastructure/customjenkins/).

### 4.1 Plugins

**Manage Jenkins** → **Plugins** → *Available plugins* — installer :

- Pipeline (workflow-aggregator)
- Git, **GitHub Branch Source**
- Credentials Binding
- Pipeline Utility Steps
- **Warnings Next Generation** (`warnings-ng`) — affichage ATC / ABAP Unit

### 4.2 Bibliothèque partagée Piper

**Manage Jenkins** → **System** (anciennement *Configure System*) → section **Global Pipeline Libraries** → **Add**

| Champ | Valeur |
|---|---|
| *Name* | `piper-lib-os` |
| *Default version* | un tag de release figé (ex. `v1.xxx.0`, voir [releases](https://github.com/SAP/jenkins-library/releases)) plutôt que `master` |
| *Load implicitly* | décoché |
| *Allow default version to be overridden* | coché |
| *Retrieval method* | **Modern SCM** |
| *Source Code Management* | **Git** |
| *Project Repository* | `https://github.com/SAP/jenkins-library.git` |

→ **Save**

### 4.3 Credentials

**Manage Jenkins** → **Credentials** → *System* → *Global credentials (unrestricted)* → **Add Credentials**

| Kind | ID | Contenu | Usage |
|---|---|---|---|
| Username with password | `gcts-abap-tst` | `JENKINS_CI` / mot de passe SAP | `abapCredentialsId` |
| Username with password | `github-pat` | login GitHub / **PAT** comme mot de passe | Checkout du repo par le job multibranch |
| Secret text | `github-pat-text` | **PAT** | `githubPersonalAccessTokenId` du rollback |

### 4.4 Création du job

**Dashboard** → **New Item** → nom `gcts-<repo>-tst` → **Multibranch Pipeline** → OK

- **Branch Sources** → *Add source* → **GitHub**
  - *Credentials* : `github-pat`
  - *Repository HTTPS URL* : `https://github.com/mon-org/mon-repo.git` → *Validate*
  - *Behaviours* : *Discover branches* ; ajouter *Filter by name (with wildcards)* → *Include* : `main`
- **Build Configuration** → *Mode* : **by Jenkinsfile** → *Script Path* : `Jenkinsfile`
- **Orphaned Item Strategy** : à votre convenance
- **Save** → Jenkins lance un *Scan Repository*.

> Multibranch est requis pour que la condition `when { branch 'main' }` du Jenkinsfile fonctionne.

### 4.5 Déclenchement

- **Avec webhook** (3.2) : rien d'autre à faire, le plugin GitHub Branch Source écoute `/github-webhook/`.
- **Sans webhook** : dans le job → *Configure* → **Scan Multibranch Pipeline Triggers** → cocher *Periodically if not otherwise run* → *Interval* : `5 minutes`.

---

## 5. Jenkinsfile

À adapter (bloc `environment`) puis commiter à la racine du dépôt GitHub.

```groovy
@Library('piper-lib-os') _

pipeline {
  agent any

  options {
    disableConcurrentBuilds()          // un seul déploiement à la fois sur TEST
    timestamps()
  }

  environment {
    ABAP_CREDS   = 'gcts-abap-tst'
    GITHUB_TOKEN = 'github-pat-text'
    HOST         = 'https://s4tst.mondomaine.fr:44300'
    CLIENT       = '100'
    REPO         = 'zmon_repo'
    REPO_URL     = 'https://github.com/mon-org/mon-repo.git'
    VSID         = 'TST'
  }

  stages {

    stage('gCTS Deploy sur TEST') {
      when { branch 'main' }
      steps {
        gctsDeploy(
          script: this,
          host: HOST,
          client: CLIENT,
          abapCredentialsId: ABAP_CREDS,
          repository: REPO,
          remoteRepositoryURL: REPO_URL,
          role: 'TARGET',               // dépôt "Provided" sur TEST
          vSID: VSID,
          commit: "${env.GIT_COMMIT}",  // déploie exactement le commit qui a déclenché le build
          rollback: true                // rollback auto si le pull/switch échoue
        )
      }
    }

    stage('ABAP Unit + ATC') {
      when { branch 'main' }
      steps {
        script {
          try {
            gctsExecuteABAPQualityChecks(
              script: this,
              host: HOST,
              client: CLIENT,
              abapCredentialsId: ABAP_CREDS,
              repository: REPO,
              scope: 'localChangedObjects',   // objets de la dernière activité du dépôt
              commit: "${env.GIT_COMMIT}",
              workspace: "${WORKSPACE}",
              aUnitTest: true,
              atcCheck: true,
              atcVariant: 'DEFAULT'
            )
          } catch (Exception ex) {
            currentBuild.result = 'FAILURE'
            unstable(message: "${STAGE_NAME} : contrôles en échec")
          }
        }
      }
    }

    stage('Publication des résultats') {
      when { branch 'main' }
      steps {
        recordIssues(
          enabledForFailure: true,
          aggregatingResults: true,
          tools: [
            checkStyle(pattern: 'ATCResults.xml',   reportEncoding: 'UTF8'),
            checkStyle(pattern: 'AUnitResults.xml', reportEncoding: 'UTF8')
          ]
        )
      }
    }

    stage('Rollback TEST') {
      when {
        allOf {
          branch 'main'
          expression { currentBuild.result == 'FAILURE' }
        }
      }
      steps {
        gctsRollback(
          script: this,
          host: HOST,
          client: CLIENT,
          abapCredentialsId: ABAP_CREDS,
          repository: REPO,
          githubPersonalAccessTokenId: GITHUB_TOKEN  // revient au dernier commit au statut "success"
        )
      }
    }
  }
}
```

Points d'attention :

- `role` et `vSID` ne servent que si le dépôt n'existe pas encore sur TEST ; ils doivent rester cohérents avec l'existant.
- `scope` alternatifs : `remoteChangedObjects` (delta entre commits, `commit` obligatoire), `repository` (tout le dépôt, plus long), `packages`.
- Si vous désactivez `atcCheck` ou `aUnitTest`, retirez le fichier correspondant de `recordIssues`.
- Certificat HTTPS SAP non reconnu par Jenkins : importer la CA dans le truststore de l'agent. `skipSSLVerification: true` uniquement en bac à sable.
- Paramétrage alternatif : déplacer les valeurs dans `.pipeline/config.yml` ([doc configuration Piper](https://www.project-piper.io/configuration/)).

---

## 6. Test de bout en bout

1. **DEV** : modifier une classe ayant des tests ABAP Unit, l'enregistrer dans un OT, **libérer** l'OT (`SE09`).
2. **GitHub** : vérifier le nouveau commit sur `main` ; *Settings > Webhooks > Recent Deliveries* → réponse **200**.
3. **Jenkins** : le job démarre sur `main` ; suivre la *Console Output*.
4. **TEST** : app gCTS → dépôt → onglet *Commits* : le commit actif = `GIT_COMMIT` du build ; onglet *Activities* : action *Pull* sans erreur.
5. **Résultats** : dans le build → *ATC/ABAP Unit* (Warnings NG) → navigation jusqu'à la ligne de code.
6. **Test d'échec** : introduire volontairement un test ABAP Unit en échec → libérer → vérifier que le stage *Rollback TEST* s'exécute et que TEST revient au commit précédent.

---

## 7. Dépannage rapide

| Symptôme | Piste |
|---|---|
| `401` / `403` sur `/sap/bc/cts_abapvcs` | Utilisateur bloqué, mot de passe initial, autorisations → `SU53` sur `JENKINS_CI` |
| `404` sur l'API | Service ICF inactif (2.1) ou mauvais port/mandant |
| Pull KO « authentication failed » côté SAP | Credentials GitHub manquants pour `JENKINS_CI` dans gCTS (2.4), PAT expiré |
| TEST change de commit sans Jenkins | `VCS_AUTOMATIC_PULL` resté à TRUE (2.3) |
| Pas de déclenchement | Livraison webhook en erreur ; Jenkins non joignable → passer en scan périodique (4.5) |
| Fichiers `ATCResults.xml` absents | Note 3159798, service `/sap/bc/adt`, ATC non configuré (2.5) |
| Erreurs d'import gCTS | App gCTS → *Activities* / *Logs* ; `CLIENT_VCS_LOGLVL = debug` ; relancer avec `scope: 'LASTACTION'` sur `gctsDeploy` |
| Problème Piper | [Issues GitHub SAP/jenkins-library](https://github.com/SAP/jenkins-library/issues) avec le label `gcts` |

---

## 8. Liens de référence

- Scénario Piper gCTS : https://www.project-piper.io/scenarios/gCTS_Scenario/
- Step `gctsDeploy` : https://www.project-piper.io/steps/gctsDeploy/
- Step `gctsExecuteABAPQualityChecks` : https://www.project-piper.io/steps/gctsExecuteABAPQualityChecks/
- Step `gctsRollback` : https://www.project-piper.io/steps/gctsRollback/
- Jenkins custom pour Piper : https://www.project-piper.io/infrastructure/customjenkins/
- gCTS – SAP Help : https://help.sap.com/docs/ABAP_PLATFORM_NEW/4a368c163b08418890a406d413933ba7/f319b168e87e42149e25e13c08d002b9.html
- Paramètres de configuration des dépôts gCTS : https://help.sap.com/docs/ABAP_PLATFORM_NEW/4a368c163b08418890a406d413933ba7/99e471efcbee4a0faec82f9dd15897e1.html
- Webhooks GitHub : https://docs.github.com/en/webhooks/using-webhooks/creating-webhooks
- Plugin Warnings NG : https://plugins.jenkins.io/warnings-ng/
