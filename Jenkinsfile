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

        stage('Capture Source Info') {
            steps {
                script {
                    env.GIT_COMMIT_SHORT = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
                    env.GIT_COMMIT_EPOCH = sh(script: "git log -1 --format=%ct", returnStdout: true).trim()

                    echo "Commit: ${env.GIT_COMMIT_SHORT}"
                    echo "Commit epoch: ${env.GIT_COMMIT_EPOCH}"
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

                    sh '''
                        set +e
                        "$SEMGREP_BIN" scan --config=auto --error --json-output=semgrep-report.json .
                        exit_code=$?

                        if [ $exit_code -eq 0 ]; then
                            echo OK > semgrep-status.txt
                        elif [ $exit_code -eq 1 ]; then
                            echo ISSUES > semgrep-status.txt
                        else
                            echo FAILED > semgrep-status.txt
                        fi

                        exit 0
                    '''

                    env.SEMGREP_STATUS = readFile('semgrep-status.txt').trim()

                    archiveArtifacts artifacts: 'semgrep-report.json,semgrep-status.txt', fingerprint: true
                    echo "Semgrep status: ${env.SEMGREP_STATUS}"
                }
            }
        }

        stage('Build') {
            steps {
                echo "Build static web (generic)"

                sh '''
                    set -e
                    rm -rf build
                    mkdir -p build

                    for item in *; do
                        if [ "$item" != "build" ]; then
                            cp -r "$item" build/
                        fi
                    done
                '''
            }
        }

        stage('Deploy to Nginx') {
            steps {
                echo "Deploy ke Nginx"
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
                    def semgrepStatus = env.SEMGREP_STATUS ?: 'FAILED'

                    def result = sh(
                        returnStdout: true,
                        script: """
                            set -e

                            SEMGREP_STATUS="${semgrepStatus}"

                            DEPLOY_EPOCH=\$(date +%s)
                            LT_SECONDS=\$((DEPLOY_EPOCH - ${env.GIT_COMMIT_EPOCH}))

                            LT_MIN=\$((LT_SECONDS / 60))
                            LT_SEC=\$((LT_SECONDS % 60))

                            WINDOW_START=\$(date -d "${env.DORA_WINDOW_DAYS} days ago" +%s)

                            mkdir -p "\$(dirname "${env.DORA_LOG}")"

                            if [ ! -f "${env.DORA_LOG}" ]; then
                                echo "build_number,commit,commit_epoch,deploy_epoch,lt_seconds,status,semgrep_status" > "${env.DORA_LOG}"
                            fi

                            if [ "\$SEMGREP_STATUS" = "OK" ]; then
                                DEPLOY_STATUS="SUCCESS"
                            else
                                DEPLOY_STATUS="SUCCESS_WITH_ISSUES"
                            fi

                            printf '%s,%s,%s,%s,%s,%s,%s\\n' \\
                                "${env.BUILD_NUMBER}" \\
                                "${env.GIT_COMMIT_SHORT}" \\
                                "${env.GIT_COMMIT_EPOCH}" \\
                                "\$DEPLOY_EPOCH" \\
                                "\$LT_SECONDS" \\
                                "\$DEPLOY_STATUS" \\
                                "\$SEMGREP_STATUS" >> "${env.DORA_LOG}"

                            DEPLOY_COUNT=\$(awk -F',' -v ws="\$WINDOW_START" '
                                NR > 1 && \$4 >= ws && \$6 ~ /^SUCCESS/ { c++ }
                                END { print c+0 }
                            ' "${env.DORA_LOG}")

                            DF_PER_DAY=\$(awk -v c="\$DEPLOY_COUNT" -v d="${env.DORA_WINDOW_DAYS}" 'BEGIN { printf "%.4f", c/d }')

                            echo "\$LT_SECONDS|\$LT_MIN|\$LT_SEC|\$DEPLOY_COUNT|\$DF_PER_DAY"
                        """
                    ).trim()

                    def parts = result.split(/\|/)
                    env.DORA_LT_SECONDS = parts[0]
                    env.DORA_LT_MIN = parts[1]
                    env.DORA_LT_SEC = parts[2]
                    env.DORA_DF_COUNT = parts[3]
                    env.DORA_DF_PER_DAY = parts[4]

                    writeFile file: 'dora-metrics.json', text: groovy.json.JsonOutput.prettyPrint(
                        groovy.json.JsonOutput.toJson([
                            buildNumber           : env.BUILD_NUMBER,
                            commit                : env.GIT_COMMIT_SHORT,
                            leadTimeSeconds       : env.DORA_LT_SECONDS,
                            leadTimeFormatted     : "${env.DORA_LT_MIN}m ${env.DORA_LT_SEC}s",
                            deployCountLast30Days : env.DORA_DF_COUNT,
                            deployFrequencyPerDay : env.DORA_DF_PER_DAY,
                            semgrepStatus         : semgrepStatus
                        ])
                    )

                    archiveArtifacts artifacts: 'dora-metrics.json', fingerprint: true

                    currentBuild.description = "LT=${env.DORA_LT_MIN}m${env.DORA_LT_SEC}s | DF30=${env.DORA_DF_COUNT} | Semgrep=${semgrepStatus}"

                    echo "Semgrep FINAL: ${semgrepStatus}"
                }
            }
        }
    }

    post {
        success {
            echo "=============================="
            echo "PIPELINE SUCCESS"
            echo "=============================="
            echo "Lead Time (LT): ${env.DORA_LT_MIN} menit ${env.DORA_LT_SEC} detik"
            echo "Deployment Frequency (DF): ${env.DORA_DF_COUNT} deploy / ${env.DORA_WINDOW_DAYS} hari"
            echo "DF Rate: ${env.DORA_DF_PER_DAY} deploy/hari"
            echo "Semgrep: ${env.SEMGREP_STATUS}"
        }

        failure {
            echo "PIPELINE FAILED"
        }

        always {
            echo "Build status: ${currentBuild.currentResult}"
        }
    }
}
