pipeline {
    agent any

    options {
        timeout(time: 30, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    triggers {
        // Poll SCM every 2 minutes for changes pushed to this branch
        pollSCM('H/2 * * * *')
    }

    environment {
        JAVA_HOME = '/Library/Java/JavaVirtualMachines/jdk-24.jdk/Contents/Home'
        PATH = "/opt/homebrew/bin:${JAVA_HOME}/bin:${env.PATH}"
        STS_PROJECT_DIR = '/Users/varshagajula/projects/smart-tests-sel'
    }

    stages {
        stage('Checkout & Info') {
            steps {
                echo "=========================================================="
                echo " Building Branch : ${env.BRANCH_NAME ?: env.GIT_BRANCH}"
                echo " Commit          : ${env.GIT_COMMIT}"
                echo "=========================================================="
                sh '''
                    java -version
                    mvn -version
                    git log -n 1 --oneline
                '''
            }
        }

        stage('Compile') {
            steps {
                echo "--> Compiling Spring PetClinic with Maven..."
                sh 'mvn clean compile'
            }
        }

        stage('Test Selection & Execution') {
            steps {
                script {
                    def branchName = env.BRANCH_NAME ?: env.GIT_BRANCH ?: ''
                    def isFeatureBranch = branchName.contains('feature')
                    def stsFilterFile = "${WORKSPACE}/target/sts_selected_filter.txt"

                    echo "Current branch: ${branchName} (Feature branch: ${isFeatureBranch})"

                    if (isFeatureBranch && fileExists("${STS_PROJECT_DIR}/.venv/bin/python3")) {
                        echo "--> Feature branch detected! Invoking Smart Test Selection (STS)..."
                        sh """
                            mkdir -p target
                            cd "${STS_PROJECT_DIR}"
                            .venv/bin/python3 -m sts.cli select \
                                --repo "${WORKSPACE}" \
                                --output-file "${stsFilterFile}" || true
                        """

                        if (fileExists(stsFilterFile)) {
                            def filterArg = readFile(stsFilterFile).trim()
                            if (filterArg && filterArg.length() > 0) {
                                echo "--> STS Filter determined: -Dtest=${filterArg}"
                                sh "mvn test -Dtest=\"${filterArg}\""
                            } else {
                                echo "--> No affected tests detected by STS; executing smoke suite..."
                                sh 'mvn test -Dtest=OwnerTests'
                            }
                        } else {
                            echo "--> STS filter file not generated; running core unit tests..."
                            sh 'mvn test -Dtest=OwnerTests,VetTests,PetValidatorTests'
                        }
                    } else {
                        echo "--> Non-feature branch or STS unavailable; running standard unit tests..."
                        sh 'mvn test -Dtest=OwnerTests,VetTests,PetValidatorTests'
                    }
                }
            }
        }

        stage('Package') {
            steps {
                echo "--> Packaging Spring PetClinic executable JAR..."
                sh 'mvn package -DskipTests'
            }
        }
    }

    post {
        always {
            junit testResults: 'target/surefire-reports/*.xml', allowEmptyResults: true
        }
        success {
            echo "Pipeline succeeded for ${env.BRANCH_NAME ?: env.GIT_BRANCH}!"
        }
        failure {
            echo "Pipeline failed for ${env.BRANCH_NAME ?: env.GIT_BRANCH}."
        }
    }
}
