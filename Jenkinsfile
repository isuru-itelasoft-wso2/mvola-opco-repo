pipeline {

    agent any

    environment {
        APIM_ENV = "mvola"
        APICTL_HOME = "/opt/wso2/apictl"
        PATH = "/opt/wso2/apictl:${env.PATH}"
    }

    stages {

        /*
         * ============================================================
         * 1. CHECK APICTL
         * ============================================================
         */
        stage('Check APICTL') {
            steps {
                sh '''
                    echo "========================================"
                    echo "CHECKING APICTL"
                    echo "========================================"

                    echo "USER=$(whoami)"
                    echo "HOME=$HOME"
                    echo "APICTL_HOME=$APICTL_HOME"
                    echo "PATH=$PATH"

                    echo ""
                    echo "APICTL LOCATION:"
                    which apictl

                    echo ""
                    echo "APICTL VERSION:"
                    apictl version

                    echo "========================================"
                '''
            }
        }


        /*
         * ============================================================
         * 2. CHECKOUT SOURCE CODE
         * ============================================================
         */
        stage('Checkout') {
            steps {
                checkout scm

                sh '''
                    echo "========================================"
                    echo "CHECKOUT COMPLETED"
                    echo "========================================"

                    echo "Git branch:"
                    git branch --show-current || true

                    echo ""
                    echo "Latest commit:"
                    git log -1 --oneline

                    echo ""
                    echo "Repository contents:"
                    ls -la

                    echo "========================================"
                '''
            }
        }


        /*
         * ============================================================
         * 3. DETECT CHANGED APIs
         *
         * Example:
         *
         * TigoCustomerAPI/api.yaml
         *
         * becomes:
         *
         * TigoCustomerAPI
         *
         * Only those APIs will be validated and deployed.
         * ============================================================
         */
        stage('Detect Changed APIs') {
            steps {
                script {

                    echo "========================================"
                    echo "DETECTING CHANGED APIs"
                    echo "========================================"

                    /*
                     * Determine the previous commit.
                     *
                     * In a normal Jenkins checkout HEAD~1 is available.
                     */
                    def changedFiles = sh(
                        script: '''
                            git diff --name-only HEAD~1 HEAD
                        ''',
                        returnStdout: true
                    ).trim()

                    echo ""
                    echo "Changed files:"
                    echo "----------------------------------------"

                    if (changedFiles) {
                        echo changedFiles
                    } else {
                        echo "No changed files detected."
                    }

                    echo "----------------------------------------"


                    /*
                     * Extract API project names.
                     *
                     * Any file directly inside an API directory
                     * is considered part of that API.
                     *
                     * Example:
                     *
                     * TigoCustomerAPI/api.yaml
                     * TigoCustomerAPI/metadata.yaml
                     *
                     * => TigoCustomerAPI
                     */
                    def apiProjects = []

                    if (changedFiles) {

                        changedFiles.split("\\n").each { file ->

                            file = file.trim()

                            if (!file) {
                                return
                            }


                            /*
                             * Ignore repository-level files.
                             */
                            if (!file.contains("/")) {
                                echo "Ignoring repository-level file: ${file}"
                                return
                            }


                            /*
                             * Extract first directory.
                             */
                            def parts = file.split("/")

                            if (parts.length > 1) {

                                def apiProject = parts[0]

                                /*
                                 * Ignore common non-API directories.
                                 */
                                if (
                                    apiProject == ".git" ||
                                    apiProject == "governance" ||
                                    apiProject == "scripts" ||
                                    apiProject == "config"
                                ) {
                                    echo "Ignoring non-API directory: ${apiProject}"
                                    return
                                }


                                /*
                                 * Verify the directory actually exists.
                                 */
                                if (!fileExists(apiProject)) {
                                    echo "Ignoring unknown directory: ${apiProject}"
                                    return
                                }


                                /*
                                 * Avoid duplicates when multiple files
                                 * inside the same API changed.
                                 */
                                if (!apiProjects.contains(apiProject)) {
                                    apiProjects.add(apiProject)
                                }
                            }
                        }
                    }


                    /*
                     * Fail if no API was changed.
                     */
                    if (apiProjects.isEmpty()) {

                        echo ""
                        echo "========================================"
                        echo "NO API CHANGES DETECTED"
                        echo "========================================"
                        echo "No API will be deployed."
                        echo "========================================"

                        error(
                            "No API project was changed in this commit. " +
                            "Pipeline stopped."
                        )
                    }


                    /*
                     * Sort APIs for predictable execution order.
                     */
                    apiProjects.sort()


                    /*
                     * Store the list for subsequent stages.
                     */
                    env.API_PROJECTS = apiProjects.join(",")


                    echo ""
                    echo "========================================"
                    echo "APIs TO PROCESS"
                    echo "========================================"

                    apiProjects.each { apiProject ->
                        echo "  -> ${apiProject}"
                    }

                    echo ""
                    echo "Total APIs to process: ${apiProjects.size()}"
                    echo "========================================"
                }
            }
        }


        /*
         * ============================================================
         * 4. LOGIN TO TIGO APIM
         * ============================================================
         */
        stage('Login to APIM') {
            steps {

                withCredentials([
                    usernamePassword(
                        credentialsId: 'cicd-apictl',
                        usernameVariable: 'APIM_USER',
                        passwordVariable: 'APIM_PASS'
                    )
                ]) {

                    sh '''
                        echo "========================================"
                        echo "LOGIN TO APIM"
                        echo "========================================"

                        printf "%s\\n" "$APIM_PASS" | \
                        apictl login "${APIM_ENV}" \
                            -u "$APIM_USER" \
                            --password-stdin \
                            --insecure

                        echo ""
                        echo "Successfully logged in to APIM environment:"
                        echo "${APIM_ENV}"

                        echo "========================================"
                    '''
                }
            }
        }


        /*
         * ============================================================
         * 5. GOVERNANCE GATE
         *
         * Each changed API is validated independently.
         *
         * If ANY changed API contains ERROR violations,
         * the complete deployment is blocked.
         *
         * Example:
         *
         * Customer API  -> PASS
         * Payment API   -> FAIL
         * Account API   -> PASS
         *
         * Result:
         *
         * DEPLOYMENT BLOCKED
         *
         * ============================================================
         */
        stage('Governance Gate') {
            steps {
                script {

                    def apiProjects = env.API_PROJECTS.split(",")

                    def failedApis = []
                    def passedApis = []

                    echo ""
                    echo "========================================"
                    echo "GOVERNANCE GATE"
                    echo "========================================"

                    apiProjects.each { apiProject ->

                        echo ""
                        echo "----------------------------------------"
                        echo "GOVERNANCE CHECK: ${apiProject}"
                        echo "----------------------------------------"


                        /*
                         * Make sure the API project still exists.
                         */
                        if (!fileExists(apiProject)) {

                            echo "ERROR: API project not found: ${apiProject}"

                            failedApis.add(apiProject)

                            return
                        }


                        /*
                         * Run APIM dry-run validation.
                         *
                         * The command output is captured so that
                         * governance violations can be evaluated.
                         */
                        def governanceOutput = sh(
                            script: """
                                set +e

                                apictl import api \\
                                    --file "${apiProject}" \\
                                    --environment "${APIM_ENV}" \\
                                    --dry-run \\
                                    --insecure

                                exit 0
                            """,
                            returnStdout: true
                        ).trim()


                        echo ""
                        echo "========== GOVERNANCE RESULT =========="
                        echo governanceOutput
                        echo "========================================"


                        /*
                         * Count ERROR occurrences.
                         */
                        def errorCount = governanceOutput.count("ERROR")


                        echo ""
                        echo "Governance ERROR count for ${apiProject}: ${errorCount}"


                        if (errorCount > 0) {

                            echo ""
                            echo "❌ GOVERNANCE FAILED: ${apiProject}"

                            failedApis.add(apiProject)

                        } else {

                            echo ""
                            echo "✅ GOVERNANCE PASSED: ${apiProject}"

                            passedApis.add(apiProject)
                        }
                    }


                    /*
                     * Print complete governance summary.
                     */
                    echo ""
                    echo "========================================"
                    echo "GOVERNANCE SUMMARY"
                    echo "========================================"

                    echo ""
                    echo "PASSED APIs:"
                    if (passedApis.isEmpty()) {
                        echo "  None"
                    } else {
                        passedApis.each {
                            echo "  ✅ ${it}"
                        }
                    }

                    echo ""
                    echo "FAILED APIs:"
                    if (failedApis.isEmpty()) {
                        echo "  None"
                    } else {
                        failedApis.each {
                            echo "  ❌ ${it}"
                        }
                    }

                    echo ""
                    echo "========================================"


                    /*
                     * Block deployment if ANY API failed.
                     */
                    if (!failedApis.isEmpty()) {

                        error(
                            "GOVERNANCE GATE FAILED. " +
                            "The following API(s) failed governance validation: " +
                            failedApis.join(", ") +
                            ". NO APIs will be deployed."
                        )
                    }


                    echo ""
                    echo "========================================"
                    echo "ALL GOVERNANCE CHECKS PASSED"
                    echo "========================================"
                }
            }
        }


        /*
         * ============================================================
         * 6. DEPLOY CHANGED APIs
         *
         * IMPORTANT:
         *
         * Only APIs stored in API_PROJECTS are deployed.
         *
         * Unchanged APIs are NOT deployed.
         * ============================================================
         */
        stage('Deploy Changed APIs') {
            steps {
                script {

                    def apiProjects = env.API_PROJECTS.split(",")


                    echo ""
                    echo "========================================"
                    echo "DEPLOYING CHANGED APIs"
                    echo "========================================"


                    apiProjects.each { apiProject ->

                        echo ""
                        echo "----------------------------------------"
                        echo "DEPLOYING: ${apiProject}"
                        echo "----------------------------------------"


                        sh """
                            apictl import api \\
                                --file "${apiProject}" \\
                                --environment "${APIM_ENV}" \\
                                --update \\
                                --insecure
                        """


                        echo ""
                        echo "✅ Deployment completed: ${apiProject}"
                    }


                    echo ""
                    echo "========================================"
                    echo "ALL CHANGED APIs DEPLOYED"
                    echo "========================================"
                }
            }
        }
    }


    /*
     * ================================================================
     * POST BUILD
     * ================================================================
     */

    post {

        success {

            echo ""
            echo "========================================"
            echo "       API CI/CD SUCCESS"
            echo "========================================"

            echo ""
            echo "APIs processed:"

            script {
                if (env.API_PROJECTS) {
                    echo "${env.API_PROJECTS}"
                } else {
                    echo "None"
                }
            }

            echo ""
            echo "Governance: PASSED"
            echo "Deployment: COMPLETED"
            echo ""
            echo "Only changed APIs were deployed."

            echo "========================================"
        }

        failure {

            echo ""
            echo "========================================"
            echo "       API CI/CD FAILED"
            echo "========================================"

            echo ""
            echo "APIs detected:"

            script {
                if (env.API_PROJECTS) {
                    echo "${env.API_PROJECTS}"
                } else {
                    echo "No API list available."
                }
            }

            echo ""
            echo "Governance / Deployment: FAILED"
            echo "API deployment was blocked or failed."

            echo ""
            echo "========================================"
        }
    }
}

