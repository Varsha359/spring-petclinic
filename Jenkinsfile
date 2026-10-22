pipeline {
    agent any

    options {
        timeout(time: 30, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '15', artifactNumToKeepStr: '5'))
        timestamps()
        ansiColor('xterm')
    }

    triggers {
        // Continuous Integration: Poll SCM every 2 minutes for changes pushed to this branch
        pollSCM('H/2 * * * *')
    }

    environment {
        // SDLC & Toolchain Environment
        JAVA_HOME        = '/Library/Java/JavaVirtualMachines/jdk-24.jdk/Contents/Home'
        PATH             = "/opt/homebrew/bin:${JAVA_HOME}/bin:${env.PATH}"
        PYTHONUNBUFFERED = '1'
        MPLCONFIGDIR     = '/tmp/matplotlib-jenkins'
        GEMINI_MODEL     = 'gemini-2.5-flash'
        
        // STS Engine Directory
        STS_PROJECT_DIR  = '/Users/varshagajula/projects/smart-tests-sel'
        MAVEN_FILTER_FILE= "${WORKSPACE}/target/sts_selected_filter.txt"
    }

    stages {
        // =========================================================================
        // SDLC PHASE 1: SCM CHECKOUT & TOOLCHAIN VALIDATION
        // =========================================================================
        stage('1. SCM Checkout & Environment') {
            steps {
                echo '================================================================='
                echo ' [SDLC Phase 1] SCM Checkout & Toolchain Verification'
                echo " Building Branch : ${env.BRANCH_NAME ?: env.GIT_BRANCH ?: 'main'}"
                echo " Commit Hash     : ${env.GIT_COMMIT ?: 'HEAD'}"
                echo '================================================================='
                sh '''
                    echo "--> Java Runtime Environment:"
                    java -version
                    echo "--> Apache Maven Build System:"
                    mvn -version
                    echo "--> Git Commit Summary:"
                    git log -n 1 --oneline
                '''
            }
        }

        // =========================================================================
        // SDLC PHASE 2: STATIC CODE ANALYSIS & CODE QUALITY GATE (SAST)
        // =========================================================================
        stage('2. Static Code Analysis & Quality') {
            steps {
                echo '================================================================='
                echo ' [SDLC Phase 2] Static Code Analysis & Code Quality Checks'
                echo '================================================================='
                sh '''
                    echo "--> Checking Java formatting standards (Spring JavaFormat)..."
                    mvn spring-javaformat:validate

                    echo "--> Executing Checkstyle static rule analysis..."
                    mvn checkstyle:check@nohttp-checkstyle-validation
                '''
            }
        }

        // =========================================================================
        // SDLC PHASE 3: COMPILATION & BUILD INTEGRITY
        // =========================================================================
        stage('3. Build & Compilation') {
            steps {
                echo '================================================================='
                echo ' [SDLC Phase 3] Clean Build & Source Code Compilation'
                echo '================================================================='
                sh 'mvn clean compile'
            }
        }

        // =========================================================================
        // SDLC PHASE 4: SECURITY & SUPPLY CHAIN AUDIT (SBOM)
        // =========================================================================
        stage('4. Security & SBOM Generation') {
            steps {
                echo '================================================================='
                echo ' [SDLC Phase 4] Software Bill of Materials (SBOM) Generation'
                echo '================================================================='
                sh 'mvn cyclonedx:makeAggregateBom'
            }
        }

        // =========================================================================
        // SDLC PHASE 5: SMART TEST SELECTION (STS) - AST & AI ANALYSIS
        // =========================================================================
        stage('5. Smart Test Selection (AST + AI)') {
            steps {
                echo '================================================================='
                echo ' [SDLC Phase 5] STS Branch Lineage & Dynamic Test Selection'
                echo '================================================================='
                script {
                    def branchName = env.BRANCH_NAME ?: env.GIT_BRANCH ?: 'main'
                    def isFeatureBranch = branchName.contains('feature')
                    echo "--> Current Branch: '${branchName}' (Feature Branch: ${isFeatureBranch})"

                    if (isFeatureBranch && fileExists("${STS_PROJECT_DIR}/.venv/bin/python3")) {
                        echo "--> Feature branch detected! Invoking Smart Test Selection (STS)..."
                        sh """
                            mkdir -p "${WORKSPACE}/target"
                            mkdir -p /tmp/matplotlib-jenkins
                            cd "${STS_PROJECT_DIR}"

                            echo "--> Step 5.1: Detecting parent branch lineage..."
                            .venv/bin/python3 -m sts.cli branch --repo "${WORKSPACE}" || true

                            echo "--> Step 5.2: Analyzing code diff hunks against parent branch..."
                            .venv/bin/python3 -m sts.cli diff --repo "${WORKSPACE}" || true

                            echo "--> Step 5.3: Executing AST parsing + LLM behavioral analysis..."
                            .venv/bin/python3 -m sts.cli select \
                                --repo "${WORKSPACE}" \
                                --output-file "${MAVEN_FILTER_FILE}" || true
                        """
                    } else {
                        echo "--> Non-feature branch or STS offline: standard regression suite will run"
                    }
                }
            }
        }

        // =========================================================================
        // SDLC PHASE 6: AUTOMATED TEST EXECUTION & COVERAGE ANALYSIS
        // =========================================================================
        stage('6. Test Execution & Coverage (CI Gate)') {
            steps {
                echo '================================================================='
                echo ' [SDLC Phase 6] Automated Test Execution & Code Coverage'
                echo '================================================================='
                script {
                    def hasFilter = fileExists("${MAVEN_FILTER_FILE}")
                    def filterArg = hasFilter ? readFile(file: "${MAVEN_FILTER_FILE}").trim() : ""

                    if (filterArg && filterArg.length() > 0) {
                        echo "--> [STS ACTIVE] Running selected affected test suite: -Dtest=${filterArg}"
                        sh "mvn test -Dtest=\"${filterArg}\""
                    } else {
                        echo "--> [STANDARD CI] Running core regression test suite..."
                        sh 'mvn test -Dtest=OwnerTests,VetTests,PetValidatorTests'
                    }

                    echo "--> Generating JaCoCo code coverage report..."
                    sh 'mvn jacoco:report'
                }
            }
        }

        // =========================================================================
        // SDLC PHASE 7: STS BENCHMARKING & METRICS EVALUATION
        // =========================================================================
        stage('7. STS Metrics & Evaluation') {
            steps {
                echo '================================================================='
                echo ' [SDLC Phase 7] Recording STS Performance Metrics & Benchmarks'
                echo '================================================================='
                script {
                    if (fileExists("${STS_PROJECT_DIR}/.venv/bin/python3")) {
                        sh """
                            cd "${STS_PROJECT_DIR}"
                            .venv/bin/python3 -m sts.cli evaluate --repo "${WORKSPACE}" || true
                        """
                    } else {
                        echo "--> STS engine directory not found; skipping benchmark recording."
                    }
                }
            }
        }

        // =========================================================================
        // SDLC PHASE 8: PACKAGING & ARTIFACT ARCHIVAL
        // =========================================================================
        stage('8. Packaging & Artifact Integrity') {
            steps {
                echo '================================================================='
                echo ' [SDLC Phase 8] Packaging Production Executable Artifact'
                echo '================================================================='
                sh '''
                    echo "--> Packaging Spring Boot executable JAR..."
                    mvn package -DskipTests

                    echo "--> Generating SHA-256 artifact checksum..."
                    shasum -a 256 target/spring-petclinic-*.jar > target/checksums.sha256
                '''
            }
        }

        // =========================================================================
        // SDLC PHASE 9: CONTINUOUS DELIVERY (CD) / STAGING VERIFICATION GATE
        // =========================================================================
        stage('9. CD Deployment & Staging Gate') {
            steps {
                echo '================================================================='
                echo ' [SDLC Phase 9] Continuous Delivery - Staging Verification Gate'
                echo '================================================================='
                sh '''
                    echo "--> Verifying packaged JAR structure..."
                    jar tf target/spring-petclinic-*.jar | grep -i "PetClinicApplication.class"
                    echo "--> Artifact verification: SUCCESS (Ready for Deployment)"
                '''
            }
        }
    }

    // =========================================================================
    // PIPELINE POST ACTIONS & OBSERVABILITY
    // =========================================================================
    post {
        always {
            echo '================================================================='
            echo ' Pipeline Run Completed. Collecting Observability Reports...'
            echo '================================================================='
            junit testResults: 'target/surefire-reports/*.xml', allowEmptyResults: true
            archiveArtifacts artifacts: 'target/*.jar, target/checksums.sha256', allowEmptyArchive: true
        }
        success {
            echo "SUCCESS: SDLC & CI/CD Pipeline passed for ${env.BRANCH_NAME ?: env.GIT_BRANCH ?: 'current branch'}!"
        }
        failure {
            echo "FAILURE: SDLC & CI/CD Pipeline failed for ${env.BRANCH_NAME ?: env.GIT_BRANCH ?: 'current branch'}."
        }
    }
}
