// ═══════════════════════════════════════════════════════════════════
// Base Pipeline (Jenkinsfile)
//
// Starting point for the workshop. Provisions a Kubernetes
// agent pod with three containers — build-tools (Maven + JDK17),
// oc-tools (OpenShift CLI), and bob (IBM Bob CLI) — sharing a
// workspace-volume emptyDir mounted at /workspace on all three,
// with HOME set to /workspace so Bob can read .bob/custom_modes.yaml
// from the checkout.
//
// This file has a single Checkout stage. Each lab adds one more
// stage beneath it; see labs/LAB<N>_*.md for instructions.
//
// Before running: if your OpenShift project is NOT named `jenkins`,
// update the `bob` container image URL below.
// ═══════════════════════════════════════════════════════════════════
pipeline {
    agent {
        kubernetes {
            yaml """
apiVersion: v1
kind: Pod
spec:
  serviceAccountName: jenkins
  containers:
  - name: build-tools
    image: maven:3.9-eclipse-temurin-17
    command: ['sleep', 'infinity']
    workingDir: /workspace
    volumeMounts:
    - name: workspace-volume
      mountPath: /workspace
    env:
    - name: HOME
      value: /workspace
  - name: oc-tools
    image: quay.io/openshift/origin-cli:latest
    command: ['sleep', 'infinity']
    workingDir: /workspace
    volumeMounts:
    - name: workspace-volume
      mountPath: /workspace
    env:
    - name: HOME
      value: /workspace
  - name: lint-tools
    image: image-registry.openshift-image-registry.svc:5000/jenkins/lint-tools:latest
    command: ['sleep', 'infinity']
    workingDir: /workspace
    volumeMounts:
    - name: workspace-volume
      mountPath: /workspace
    env:
    - name: HOME
      value: /workspace
  - name: bob
    image: image-registry.openshift-image-registry.svc:5000/jenkins/bob-cli:latest
    command: ['sleep', 'infinity']
    workingDir: /workspace
    volumeMounts:
    - name: workspace-volume
      mountPath: /workspace
    env:
    - name: BOBSHELL_API_KEY
      valueFrom:
        secretKeyRef:
          name: bob-cli-credentials
          key: BOBSHELL_API_KEY
    - name: BOB_ACCEPT_LICENSE
      value: "true"
    - name: HOME
      value: /workspace
  volumes:
  - name: workspace-volume
    emptyDir: {}
"""
            defaultContainer 'build-tools'
        }
    }

    stages {
        stage('Checkout') {
            steps {
                echo "=== Checking out repository ==="
                checkout scm

                echo "=== Verifying workspace contents ==="
                sh '''
                    echo "Current directory: $(pwd)"
                    echo "Workspace contents:"
                    ls -la
                '''
            }
        }

        // ── Lab 1: PR / Git Diff Review ──────────────────────────
        stage('PR Review') {
            options {
                timeout(time: 5, unit: 'MINUTES')
            }
            steps {
                script {
                    catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                        sh '''
                            git config --global --add safe.directory "$WORKSPACE"
                            git diff origin/main...HEAD > git-diff.txt || : > git-diff.txt
                        '''

                        def analysis = askBob(
                            "Read git-diff.txt and produce the senior-developer PR overview.",
                            'pipeline-git-diff-overview'
                        )

                        echo '════════════════════════════════════════════════════════'
                        echo '  Bob — PR Review'
                        echo '════════════════════════════════════════════════════════'
                        echo analysis
                        echo '════════════════════════════════════════════════════════'

                        writeFile file: 'bob-pr-review.md', text: analysis
                    }
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: 'bob-pr-review.md', allowEmptyArchive: true
                }
            }
        }

        // ── Lab 2: Unit Testing ──────────────────────────────────
        stage('Unit Tests') {
            options {
                timeout(time: 5, unit: 'MINUTES')
            }
            steps {
                script {
                    def testResult = 0
                    dir('order-service') {
                        testResult = sh(
                            script: 'mvn test',
                            returnStatus: true
                        )
                    }

                    junit testResults: 'order-service/target/surefire-reports/*.xml', allowEmptyResults: true

                    if (testResult != 0) {
                        currentBuild.result = 'UNSTABLE'
                        catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                            def prompt = "Analyze the test failure reports under order-service/target/surefire-reports/ alongside the relevant source files under order-service/src/ and diagnose the failures."

                            def analysis = askBob(prompt, 'pipeline-test-failure-analyzer')

                            echo '════════════════════════════════════════════════════════'
                            echo '  Bob — Unit Test Failure Analysis'
                            echo '════════════════════════════════════════════════════════'
                            echo analysis
                            echo '════════════════════════════════════════════════════════'

                            writeFile file: 'bob-test-analysis.md', text: analysis
                        }
                    }
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: 'bob-test-analysis.md', allowEmptyArchive: true
                }
            }
        }

        // ── Lab 3: Security Scanning ─────────────────────────────
        //    Add a scanner stage + Bob CVE/vuln analysis.
        //    See labs/lab3/SECURITY_ANALYSIS.md.

        // ── Lab 4: Linting ───────────────────────────────────────
        stage('Run Linters') {
            options {
                timeout(time: 10, unit: 'MINUTES')
            }
            steps {
                sh 'mkdir -p lint-results'

                // Checkstyle (Maven → build-tools, the default container)
                catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                    sh 'mvn -f order-service/pom.xml checkstyle:check'
                }
                sh 'cp order-service/target/checkstyle-result.xml lint-results/checkstyle.xml || true'

                container('lint-tools') {
                    catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                        sh 'hadolint --no-color order-service/Dockerfile > lint-results/hadolint.txt 2>&1'
                    }
                    catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                        sh 'checkov -d order-service/deploy-flawed/ --compact --quiet --no-color > lint-results/checkov.txt 2>&1'
                    }
                    catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                        sh 'kube-linter lint order-service/deploy-flawed/ > lint-results/kubelinter.txt 2>&1'
                    }
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: 'lint-results/*', allowEmptyArchive: true
                }
            }
        }

        stage('Bob Lint Analysis') {
            when {
                expression {
                    return ['checkstyle.xml', 'hadolint.txt', 'checkov.txt', 'kubelinter.txt']
                        .any { fileExists("lint-results/${it}") }
                }
            }
            options {
                timeout(time: 10, unit: 'MINUTES')
            }
            steps {
                script {
                    catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                        def lintPrompt = """
Read the following files directly (skip any that do not exist):
- lint-results/checkstyle.xml
- lint-results/hadolint.txt
- lint-results/checkov.txt
- lint-results/kubelinter.txt
- order-service/src/
- order-service/Dockerfile
- order-service/deploy-flawed/

Then:
1. Group all findings by severity.
2. De-duplicate overlapping findings reported by more than one tool.
3. Identify the highest-priority remediations.
4. Recommend concrete fixes for each.
"""
                        def report = askBob(lintPrompt, 'pipeline-lint-analyzer')
                        writeFile file: 'bob-lint-report.md', text: report

                        def commentPrompt = """
Read bob-lint-report.md and produce a concise pull request comment summarizing
the lint results: overall status, the top priority issues, and the key fixes.
"""
                        def prComment = askBob(commentPrompt, 'pipeline-lint-analyzer')
                        writeFile file: 'bob-lint-pr-comment.md', text: prComment

                        echo '════════════════════════════════════════════════════════'
                        echo '  Bob — Lint Analysis'
                        echo '════════════════════════════════════════════════════════'
                        echo prComment
                        echo '  Full report: bob-lint-report.md (archived)'
                        echo '  PR comment:  bob-lint-pr-comment.md (archived)'
                        echo '════════════════════════════════════════════════════════'
                    }
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: 'bob-lint-report.md,bob-lint-pr-comment.md', allowEmptyArchive: true
                }
            }
        }

    }

    post {
        always {
            echo "=== Pipeline Complete ==="
            echo "Result: ${currentBuild.result ?: 'SUCCESS'}"
        }
    }
}

// ── Helper: ask Bob, optionally with a specific custom mode ───────────────────
// Writes the prompt to a tempfile in the shared workspace and runs `bob` in
// the bob container.
// Returns the analysis as a string. Using a tempfile (instead of inlining the
// prompt on the command line) avoids shell-escaping issues when the prompt
// contains quotes, backticks, or newlines — common with diffs.
def askBob(String prompt, String mode = null) {
    container('bob') {
        def promptFile = ".bob-prompt-${System.currentTimeMillis()}.txt"
        writeFile file: promptFile, text: prompt

        def modeFlag = mode ? "--mode ${mode}" : ""
        def analysis = sh(
            script: """cat ${promptFile} | bob run ${modeFlag} --accept-license""",
            returnStdout: true
        ).trim()

        sh "rm -f ${promptFile}"
        return analysis
    }
}