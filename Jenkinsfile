@Library('Shared') _

pipeline {
    agent any
    
    environment {
        // Update the main app image name to match the deployment file
        DOCKER_IMAGE_NAME = 'uma1827/easyshop-app'
        DOCKER_MIGRATION_IMAGE_NAME = 'uma1827/easyshop-migration'
        DOCKER_IMAGE_TAG = "${BUILD_NUMBER}"
        GITHUB_CREDENTIALS = credentials('github-credentials')
        GIT_BRANCH = "master"
        DOCKER_BUILDKIT = "1"
        DOCKER_CLI_EXPERIMENTAL = "enabled"
    }
    
    stages {
        stage('Cleanup Workspace') {
            steps {
                script {
                    clean_ws()
                }
            }
        }
        
        stage('Clone Repository') {
            steps {
                script {
                    clone("https://github.com/Umanagalla27/tws-e-commerce-app_hackathon.git", env.GIT_BRANCH)
                }
            }
        }
        
        stage('Build Docker Images') {
            parallel {
                stage('Build Main App Image') {
                    steps {
                        script {
                            docker_build(
                                imageName: env.DOCKER_IMAGE_NAME,
                                imageTag: env.DOCKER_IMAGE_TAG,
                                dockerfile: 'Dockerfile',
                                context: '.'
                            )
                        }
                    }
                }
                
                stage('Build Migration Image') {
                    steps {
                        script {
                            docker_build(
                                imageName: env.DOCKER_MIGRATION_IMAGE_NAME,
                                imageTag: env.DOCKER_IMAGE_TAG,
                                dockerfile: 'scripts/Dockerfile.migration',
                                context: '.'
                            )
                        }
                    }
                }
            }
        }
        
        stage('Run Unit Tests') {
            steps {
                script {
                    run_tests()
                }
            }
        }
        
        stage('Security Scan with Trivy') {
            steps {
                script {
                    // Create directory for results
                  
                    trivy_scan()
                    
                }
            }
        }
        
        stage('Push Docker Images') {
            parallel {
                stage('Push Main App Image') {
                    steps {
                        script {
                            docker_push(
                                imageName: env.DOCKER_IMAGE_NAME,
                                imageTag: env.DOCKER_IMAGE_TAG,
                                credentials: 'docker-hub-credentials'
                            )
                        }
                    }
                }
                
                stage('Push Migration Image') {
                    steps {
                        script {
                            docker_push(
                                imageName: env.DOCKER_MIGRATION_IMAGE_NAME,
                                imageTag: env.DOCKER_IMAGE_TAG,
                                credentials: 'docker-hub-credentials'
                            )
                        }
                    }
                }
            }
        }
        
        stage('Update Kubernetes Manifests') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'github-credentials', usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_PASSWORD')]) {
                        sh """
                          set -e
                          git config user.name 'Jenkins CI'
                          git config user.email 'nagallauma88@gmail.com'
                          sed -i "s|^\\s*image: .*$|image: ${DOCKER_IMAGE_NAME}:${DOCKER_IMAGE_TAG}|g" kubernetes/08-easyshop-deployment.yaml
                          if [ -f kubernetes/12-migration-job.yaml ]; then
                            sed -i "s|^\\s*image: .*$|image: ${DOCKER_MIGRATION_IMAGE_NAME}:${DOCKER_IMAGE_TAG}|g" kubernetes/12-migration-job.yaml
                          fi
                          if [ -f kubernetes/10-ingress.yaml ]; then
                            sed -i "s|^\\s*host: .*$|host: easyshop.letsdeployit.com|g" kubernetes/10-ingress.yaml
                          fi
                          git add kubernetes/01-namespace.yaml kubernetes/02-mongodb-pv.yaml kubernetes/03-mongodb-pvc.yaml kubernetes/04-configmap.yaml kubernetes/05-secrets.yaml kubernetes/06-mongodb-service.yaml kubernetes/07-mongodb-statefulset.yaml kubernetes/08-easyshop-deployment.yaml kubernetes/09-easyshop-service.yaml kubernetes/10-ingress.yaml kubernetes/11-hpa.yaml kubernetes/12-migration-job.yaml || true
                          git commit -m "Update image tags to ${DOCKER_IMAGE_TAG} [ci skip]" || true
                          git push "https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/Umanagalla27/tws-e-commerce-app_hackathon.git" HEAD:${GIT_BRANCH}
                        """
                    }
                }
            }
        }
    }
}
