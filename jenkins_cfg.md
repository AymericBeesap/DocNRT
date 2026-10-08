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
