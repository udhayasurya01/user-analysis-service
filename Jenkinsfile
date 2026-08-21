pipeline {
    agent any

    triggers {
        githubPush()
    }

    tools {
        maven 'maven-3.9.12'
        jdk 'jdk-25'
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'dev',
                    url: 'https://github.com/udhayasurya01/user-analysis-service.git'
            }
        }

        stage('Build') {
            steps {
                bat 'mvn clean package -DskipTests'
            }
        }

        stage('Test') {
            steps {
                bat 'mvn test'
            }
        }

        stage('Check Test Pass %') {
            steps {
                script {
                    def testResults = findFiles(glob: 'target/surefire-reports/*.xml')
                    int totalTests = 0
                    int totalFailures = 0

                    testResults.each { file ->
                        def content = readFile(file.path)
                        def testsMatch = (content =~ /tests="(\d+)"/)
                        def failMatch = (content =~ /failures="(\d+)"/)
                        def errMatch = (content =~ /errors="(\d+)"/)

                        if (testsMatch) totalTests += testsMatch[0][1].toInteger()
                        if (failMatch) totalFailures += failMatch[0][1].toInteger()
                        if (errMatch) totalFailures += errMatch[0][1].toInteger()
                    }

                    echo "Total Tests: ${totalTests}, Failures: ${totalFailures}"

                    if (totalTests == 0) {
                        echo "No tests found — skipping pass % check."
                    } else {
                        int passed = totalTests - totalFailures
                        double passPercent = (passed / totalTests) * 100
                        echo "Pass Percentage: ${passPercent}%"

                        if (passPercent < 50) {
                            error("Test pass rate is ${passPercent}%, which is below the required 50%. Blocking build.")
                        } else {
                            echo "Test pass rate OK — proceeding."
                        }
                    }
                }
            }
        }
    }

    post {
        success {
            echo 'Pipeline passed — 50%+ tests OK!'
        }
        failure {
            echo 'Pipeline failed — check test results.'
        }
    }
}