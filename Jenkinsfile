pipeline {

    agent none

    environment {
        AWS_CREDENTIAL_ID  = 'aws-jenkins'
        AWS_DEFAULT_REGION = 'us-east-1'
        AWS_ECR            = credentials('aws-ecr-private-registry')
        AWS_S3_PREFIX      = 's3://oss-jenkins/artifact/presto'
    }

    options {
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '500'))
        timeout(time: 3, unit: 'HOURS')
    }

    parameters {
        booleanParam(name: 'PUBLISH_ARTIFACTS_ON_CURRENT_BRANCH',
                     defaultValue: false,
                     description: 'whether to publish tar and docker image even if current branch is not master'
        )
    }

    stages {
        stage('Maven Build') {
            agent {
                kubernetes {
                    defaultContainer 'maven'
                    yamlFile 'jenkins/agent-maven.yaml'
                }
            }

            stages {
                stage('Setup') {
                    steps {
                        sh 'apt update && apt install -y awscli git tree'
                        sh 'git config --global --add safe.directory ${WORKSPACE}'
                    }
                }

                stage('PR Update') {
                    when { changeRequest() }
                    steps {
                        echo 'reset one commit for a PR branch, because Jenkins does a auto merge from the base branch, which creates a new commit SHA'
                        sh '''
                            git log -n 3
                            git reset HEAD~1
                            git log -n 3
                        '''
                    }
                }

                stage('Maven') {
                    steps {
                        sh 'unset MAVEN_CONFIG && ./mvnw versions:set -DremoveSnapshot'

                        script {
                            env.PRESTO_VERSION = sh(
                                script: 'unset MAVEN_CONFIG && ./mvnw org.apache.maven.plugins:maven-help-plugin:3.2.0:evaluate -Dexpression=project.version -q -DforceStdout',
                                returnStdout: true).trim()
                            env.PRESTO_PKG = "presto-server-${PRESTO_VERSION}.tar.gz"
                            env.PRESTO_CLI_JAR = "presto-cli-${PRESTO_VERSION}-executable.jar"
                            env.PRESTO_BUILD_VERSION = env.PRESTO_VERSION + '-' +
                                                    sh(script: 'TZ=UTC date +%Y%m%dT%H%M%S', returnStdout: true).trim() + '-' +
                                                    sh(script: 'git rev-parse --short=7 HEAD', returnStdout: true).trim()
                        }
                        sh 'printenv | sort'

                        echo "build prestodb source code with build version ${PRESTO_BUILD_VERSION}"
                        sh '''
                            exit 0
                            unset MAVEN_CONFIG && ./mvnw install -DskipTests -B -T C1 -P ci -pl '!presto-docs' --fail-at-end
                            tree /root/.m2/repository/com/facebook/presto/
                        '''

                        echo 'Publish Maven tarball'
                        withCredentials([[
                                $class:            'AmazonWebServicesCredentialsBinding',
                                credentialsId:     "${AWS_CREDENTIAL_ID}",
                                accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                                secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']]) {
                            sh '''
                                exit 0
                                aws s3 cp presto-server/target/${PRESTO_PKG}  ${AWS_S3_PREFIX}/${PRESTO_BUILD_VERSION}/ --no-progress
                                aws s3 cp presto-cli/target/${PRESTO_CLI_JAR} ${AWS_S3_PREFIX}/${PRESTO_BUILD_VERSION}/ --no-progress
                            '''
                        }
                    }
                    post {
                        always {
                            archiveArtifacts artifacts: '**/target/dependency-check-report.html',
                                            allowEmptyArchive: true
                        }
                    }
                }
            }
        }

        stage('Docker Build') {
            agent {
                kubernetes {
                    defaultContainer 'dind'
                    yamlFile 'jenkins/agent-dind.yaml'
                }
            }

            stages {
                stage('Setup') {
                    steps {
                        sh 'apk update && apk add aws-cli bash git'
                        sh 'git config --global --add safe.directory ${WORKSPACE}'
                        script {
                            env.DOCKER_IMAGE = env.AWS_ECR + "/oss-presto/presto:${PRESTO_BUILD_VERSION}"
                            env.DOCKER_NATIVE_IMAGE = env.AWS_ECR + "/oss-presto/presto-native:${PRESTO_BUILD_VERSION}"
                        }
                    }
                }
                stage('Docker') {
                    steps {
                        echo 'build docker image'
                        withCredentials([[
                                $class:            'AmazonWebServicesCredentialsBinding',
                                credentialsId:     "${AWS_CREDENTIAL_ID}",
                                accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                                secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']]) {
                            sh '''#!/bin/bash -ex
                                exit 0
                                cd docker/
                                aws s3 cp ${AWS_S3_PREFIX}/${PRESTO_BUILD_VERSION}/${PRESTO_PKG}     . --no-progress
                                aws s3 cp ${AWS_S3_PREFIX}/${PRESTO_BUILD_VERSION}/${PRESTO_CLI_JAR} . --no-progress

                                echo "Building ${DOCKER_IMAGE}"
                                docker buildx build --load --platform "linux/amd64" -t "${DOCKER_IMAGE}-amd64" \
                                    --build-arg "PRESTO_VERSION=${PRESTO_VERSION}" .
                                docker image ls
                            '''
                        }
                    }
                }

                stage('Docker Native Build') {
                    steps {
                        echo "Building ${DOCKER_NATIVE_IMAGE}"
                        withCredentials([[
                                $class:            'AmazonWebServicesCredentialsBinding',
                                credentialsId:     "${AWS_CREDENTIAL_ID}",
                                accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                                secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']]) {
                            sh '''#!/bin/bash -ex
                                git config --global --add safe.directory ${WORKSPACE}/presto-native-execution/velox
                                git submodule sync --recursive
                                git submodule update --init --recursive
                                cd presto-native-execution
                                docker buildx build --load --platform "linux/amd64" -t "${DOCKER_NATIVE_IMAGE}-amd64" \
                                    --build-arg "PRESTO_VERSION=${PRESTO_VERSION}" .
                            '''
                        }
                    }
                }

                stage('Publish Docker') {
                    when {
                        anyOf {
                            expression { params.PUBLISH_ARTIFACTS_ON_CURRENT_BRANCH }
                            branch "master"
                        }
                        beforeAgent true
                    }

                    steps {
                        echo 'Publish docker image'
                        withCredentials([[
                                $class:            'AmazonWebServicesCredentialsBinding',
                                credentialsId:     "${AWS_CREDENTIAL_ID}",
                                accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                                secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']]) {
                            sh '''
                                aws s3 ls ${AWS_S3_PREFIX}/${PRESTO_BUILD_VERSION}/
                                docker image ls
                                aws ecr get-login-password | docker login --username AWS --password-stdin ${AWS_ECR}
                                docker push "${DOCKER_IMAGE}-amd64"
                            '''
                        }
                    }
                }
            }
        }
    }
}
