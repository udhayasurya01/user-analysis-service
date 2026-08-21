pipeline {
    agent any

    triggers {
        githubPush()
    }

    tools {
        maven 'maven-3.9.12'
        jdk 'jdk-25'
    }

    environment {
        IMAGE_NAME = 'vijayjeyam/prodmexaanalysis'
        IMAGE_TAG  = "${env.BUILD_NUMBER}"
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'dev',
                    url: 'https://github.com/udhayasurya01/user-analysis-service.git'
            }
        }

        stage('Show Commit Info') {
            steps {
                script {
                    env.GIT_AUTHOR = bat(script: '@git log -1 --pretty=format:%%an', returnStdout: true).trim()
                    env.GIT_DATE   = bat(script: '@git log -1 --pretty=format:%%ad', returnStdout: true).trim()
                    env.GIT_HASH   = bat(script: '@git log -1 --pretty=format:%%h', returnStdout: true).trim()

                    echo "================ COMMIT INFO ================"
                    echo "Commit  : ${env.GIT_HASH}"
                    echo "Author  : ${env.GIT_AUTHOR}"
                    echo "Date    : ${env.GIT_DATE}"
                    echo "=============================================="
                }
            }
        }

        stage('Test') {
            steps {
                bat script: 'mvn test', returnStatus: true
            }
        }

        stage('Check Test Pass % + Failure Reasons') {
            steps {
                script {
                    def testResults = findFiles(glob: 'target/surefire-reports/*.xml')
                    int totalTests = 0
                    int totalFailures = 0
                    def failedCases = []

                    testResults.each { file ->
                        def content = readFile(file.path)
                        def testsMatch = (content =~ /tests="(\d+)"/)
                        def failMatch  = (content =~ /failures="(\d+)"/)
                        def errMatch   = (content =~ /errors="(\d+)"/)

                        if (testsMatch) totalTests += testsMatch[0][1].toInteger()
                        if (failMatch)  totalFailures += failMatch[0][1].toInteger()
                        if (errMatch)   totalFailures += errMatch[0][1].toInteger()

                        def caseMatcher = (content =~ /<testcase[^>]*name="([^"]*)"[^>]*classname="([^"]*)"[^>]*>([\s\S]*?)<\/testcase>/)
                        caseMatcher.each { match ->
                            def caseName  = match[1]
                            def className = match[2]
                            def body      = match[3]
                            def failMsg   = (body =~ /<(?:failure|error)[^>]*message="([^"]*)"/)
                            if (failMsg) {
                                failedCases << "${className}#${caseName} -> ${failMsg[0][1]}"
                            }
                        }
                    }

                    echo "Total Tests: ${totalTests}, Failures: ${totalFailures}"

                    if (!failedCases.isEmpty()) {
                        echo "================ FAILED TEST CASES ================"
                        failedCases.each { echo it }
                        echo "===================================================="
                    }

                    if (totalTests == 0) {
                        echo "No tests found - skipping pass % check."
                    } else {
                        int passed = totalTests - totalFailures
                        double passPercent = (passed / totalTests) * 100
                        echo "Pass Percentage: ${passPercent}%"
                        env.PASS_PERCENT = "${passPercent}"

                        if (passPercent < 80) {
                            error("Test pass rate is ${passPercent}%, which is below the required 80%. Blocking build.")
                        } else {
                            echo "Test pass rate OK (${passPercent}%) - proceeding to build."
                        }
                    }
                }
            }
        }

        stage('Build') {
            steps {
                bat 'mvn clean package -DskipTests'
            }
        }

        stage('Docker Build') {
            steps {
                bat "docker build -t %IMAGE_NAME%:%IMAGE_TAG% -t %IMAGE_NAME%:latest ."
            }
        }

        stage('Save Docker Image') {
            steps {
                bat "docker save -o app-image.tar %IMAGE_NAME%:latest"
            }
        }

        stage('Transfer to Server') {
            steps {
                sshPublisher(publishers: [
                    sshPublisherDesc(
                        configName: 'office-server',
                        transfers: [
                            sshTransfer(
                                sourceFiles: 'app-image.tar, docker_env/prod.yml',
                                remoteDirectory: 'user-analysis-service'
                            )
                        ]
                    )
                ])
            }
        }

        stage('Deploy via Docker Compose') {
            steps {
                sshPublisher(publishers: [
                    sshPublisherDesc(
                        configName: 'office-server',
                        transfers: [
                            sshTransfer(
                                execCommand: '''
                                    cd /home/mani/user-analysis-service &&
                                    docker load -i app-image.tar &&
                                    docker network create devmexa || true &&
                                    docker compose -f prod.yml up -d
                                '''
                            )
                        ]
                    )
                ])
            }
        }
    }

    post {
        always {
            echo "Build #${env.BUILD_NUMBER} | Commit ${env.GIT_HASH} by ${env.GIT_AUTHOR} on ${env.GIT_DATE} | Result: ${currentBuild.currentResult}"
        }
        success {
            echo "Pipeline passed - test pass rate ${env.PASS_PERCENT}% - deployed to 122.165.70.116"
        }
        failure {
            echo "Pipeline failed - check test results / deployment logs above."
        }
    }
}