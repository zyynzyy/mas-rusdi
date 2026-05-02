pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    environment {
        APP_DIR = '/var/www/dora-site'
        DORA_LOG = '/var/lib/jenkins/dora-metrics/deployments.csv'
        DORA_WINDOW_DAYS = '30'
        SEMGREP_BIN = '/var/lib/jenkins/semgrep-venv/bin/semgrep'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Capture Source Info') {
            steps {
                script {
                    env.GIT_COMMIT_SHORT = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
                    env.GIT_COMMIT_EPOCH = sh(script: "git log -1 --format=%ct", returnStdout: true).trim()

                    // waktu pipeline mulai (buat stopwatch LT)
                    env.PIPELINE_START_EPOCH = (System.currentTimeMillis() / 1000).toString()

                    echo "Commit: ${env.GIT_COMMIT_SHORT}"
                    echo "Commit epoch: ${env.GIT_COMMIT_EPOCH}"
                    echo "Pipeline start: ${env.PIPELINE_START_EPOCH}"
                }
            }
        }

        stage('Prepare Workspace') {
            steps {
                sh 'rm -rf build semgrep-report.json dora-metrics.json semgrep-status.txt'
            }
        }

        stage('Semgrep Analysis') {
            steps {
                script {
                    sh 'test -x "$SEMGREP_BIN"'

                    def semgrepExit = sh(
                        returnStatus: true,
                        script: '''
                            set +e
                            "$SEMGREP_BIN" scan --config=auto --error --json-output=semgrep-report.json .
                            exit_code=$?

                            if [ $exit_code -eq 0 ]; then
                                echo "OK" > semgrep-status.txt
                            elif [ $exit_code -eq 1 ]; then
                                echo "ISSUES" > semgrep-status.txt
                            else
                                echo "FAILED" > semgrep-status.txt
                            fi

                            exit $exit_code
                        '''
                    )

                    if (semgrepExit == 0 || semgrepExit == 1) {
                        env.SEMGREP_STATUS = readFile('semgrep-status.txt').trim()
                    } else {
                        error "Semgrep failed with exit code ${semgrepExit}"
                    }

                    archiveArtifacts artifacts: 'semgrep-report.json,semgrep-status.txt', fingerprint: true
                    echo "Semgrep status: ${env.SEMGREP_STATUS}"
                }
            }
        }

        stage('Build') {
            steps {
                echo "Build static web"
                sh '''
                    set -e
                    rm -rf build
                    mkdir -p build

                    for f in index.html templates.html templatemo-quantix-style.css templatemo-quantix-script.js; do
                        if [ -f "$f" ]; then cp "$f" build/; fi
                    done

                    for d in assets images img css js fonts vendor; do
                        if [ -d "$d" ]; then cp -r "$d" build/; fi
                    done
                '''
            }
        }

        stage('Deploy to Nginx') {
            steps {
                sh '''
                    mkdir -p "$APP_DIR"
                    rm -rf "$APP_DIR"/*
                    cp -r build/* "$APP_DIR"/
                '''
            }
        }

        stage('DORA Metrics') {
            steps {
                script {
                    def result = sh(
                        returnStdout: true,
                        script: """
                            set -e

                            DEPLOY_EPOCH=\$(date +%s)

                            # =========================
                            # 1. DORA Lead Time (VALID)
                            # =========================
                            LT_DORA=\$((DEPLOY_EPOCH - ${env.GIT_COMMIT_EPOCH}))

                            # =========================
                            # 2. Pipeline Lead Time (STOPWATCH)
                            # =========================
                            LT_PIPELINE=\$((DEPLOY_EPOCH - ${env.PIPELINE_START_EPOCH}))

                            LT_DORA_MIN=\$(awk -v s="\$LT_DORA" 'BEGIN { printf "%.2f", s/60 }')
                            LT_PIPE_MIN=\$(awk -v s="\$LT_PIPELINE" 'BEGIN { printf "%.2f", s/60 }')

                            WINDOW_START=\$(date -d "${env.DORA_WINDOW_DAYS} days ago" +%s)

                            mkdir -p "\$(dirname "${env.DORA_LOG}")"

                            if [ ! -f "${env.DORA_LOG}" ]; then
                                echo "build,commit,commit_epoch,deploy_epoch,lt_dora,lt_pipeline,status,semgrep" > "${env.DORA_LOG}"
                            fi

                            if [ "${env.SEMGREP_STATUS}" = "OK" ]; then
                                DEPLOY_STATUS="SUCCESS"
                            else
                                DEPLOY_STATUS="SUCCESS_WITH_ISSUES"
                            fi

                            printf '%s,%s,%s,%s,%s,%s,%s,%s\\n' \\
                                "${env.BUILD_NUMBER}" \\
                                "${env.GIT_COMMIT_SHORT}" \\
                                "${env.GIT_COMMIT_EPOCH}" \\
                                "\$DEPLOY_EPOCH" \\
                                "\$LT_DORA" \\
                                "\$LT_PIPELINE" \\
                                "\$DEPLOY_STATUS" \\
                                "${env.SEMGREP_STATUS}" >> "${env.DORA_LOG}"

                            DEPLOY_COUNT=\$(awk -F',' -v ws="\$WINDOW_START" '
                                NR > 1 && \$4 >= ws && \$7 ~ /^SUCCESS/ { c++ }
                                END { print c+0 }
                            ' "${env.DORA_LOG}")

                            DF_PER_DAY=\$(awk -v c="\$DEPLOY_COUNT" -v d="${env.DORA_WINDOW_DAYS}" 'BEGIN { printf "%.4f", c/d }')

                            echo "\$LT_DORA|\$LT_PIPELINE|\$LT_DORA_MIN|\$LT_PIPE_MIN|\$DEPLOY_COUNT|\$DF_PER_DAY"
                        """
                    ).trim()

                    def p = result.split("\\|")

                    env.LT_DORA_SEC = p[0]
                    env.LT_PIPE_SEC = p[1]
                    env.LT_DORA_MIN = p[2]
                    env.LT_PIPE_MIN = p[3]
                    env.DF_COUNT    = p[4]
                    env.DF_RATE     = p[5]

                    writeFile file: 'dora-metrics.json', text: groovy.json.JsonOutput.prettyPrint(
                        groovy.json.JsonOutput.toJson([
                            commit                : env.GIT_COMMIT_SHORT,
                            leadTimeDoraSeconds   : env.LT_DORA_SEC,
                            leadTimePipelineSec   : env.LT_PIPE_SEC,
                            deployCount30Days     : env.DF_COUNT,
                            deployFrequencyPerDay : env.DF_RATE,
                            semgrep               : env.SEMGREP_STATUS
                        ])
                    )

                    archiveArtifacts artifacts: 'dora-metrics.json', fingerprint: true

                    currentBuild.description = "LT=${env.LT_DORA_MIN}m | DF=${env.DF_COUNT} | SG=${env.SEMGREP_STATUS}"
                }
            }
        }
    }

    post {
        success {
            echo "=============================="
            echo "PIPELINE SUCCESS"
            echo "=============================="

            echo "DORA (VALID): ${env.LT_DORA_SEC} detik (${env.LT_DORA_MIN} menit)"
            echo "PIPELINE (STOPWATCH): ${env.LT_PIPE_SEC} detik (${env.LT_PIPE_MIN} menit)"
            echo "DF: ${env.DF_COUNT} deploy / ${env.DORA_WINDOW_DAYS} hari (${env.DF_RATE}/hari)"
            echo "Semgrep: ${env.SEMGREP_STATUS}"
        }

        failure {
            echo "PIPELINE FAILED"
        }
    }
}
