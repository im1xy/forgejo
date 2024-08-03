def nextTag
def tagPattern="^0\\.0\\.\\d+"

pipeline {
    agent none
    
    options {
        skipDefaultCheckout true
    }

    environment {
        AGENT='forgejo_agent'
        
        GO_ARTIFACT="go.tar.gz"
        NODE_ARTIFACT="node.tar.gz"

        GIT_REPO="https://github.com/im1xy/forgejo.git"
        GIT_TOKEN="github-token"
        GIT_BRANCH="main"

        ARTIFACTORY_REPO="im1xy/forgejo"
        ARTIFACTORY_TOKEN="github-token"
        ARTIFACTORY_BRANCH="main"
    }
    
    stages {
        stage('Node and Go') {
            parallel {
                stage('Node') {
                    agent {label env.AGENT}
                    tools {nodejs '20.0.0'}
                    stages {
                        stage('Checkout') {
                            steps {
                                git(
                                    url: env.GIT_REPO,
                                    branch: env.GIT_BRANCH,
                                    credentialsId: env.GIT_TOKEN
                                )
                            }
                        }
                        stage('Install Node Dependencies') {
                            steps {
                                sh 'npm install'
                            }
                        }
                        stage('Lint Frontend') {
                            steps {
                                sh 'npx eslint --color --max-warnings=0 --ext js,vue web_src/js tools *.config.js tests/e2e'
                                sh 'npx stylelint --color --max-warnings=0 web_src/css web_src/js/components/*.vue'
                            }
                        }
                        stage('Test Frontend') {
                            steps {
                                sh 'npx vitest'
                            }
                        }
                        stage('Build Node') {
                            steps {
                                sh 'npx webpack'
                            }
                        }
                        stage('Create Node Artifact') {
                            steps {
                                tar(
                                    file: env.NODE_ARTIFACT,
                                    archive: true,
                                    compress: true,
                                    glob: 'options/**, public/**, templates/**',
                                    overwrite: true
                                )
                            }
                        }
                    }
                }
                stage('Go') {
                    agent {label env.AGENT}
                    tools {go '1.22.5'}
                    stages {
                        stage('Checkout') {
                            steps {
                                git(
                                    url: env.GIT_REPO,
                                    branch: env.GIT_BRANCH,
                                    credentialsId: env.GIT_TOKEN
                                )
                            }
                        }
                        stage('Install Go Dependencies') {
                            steps {
                                sh 'go mod vendor'
                            }
                        }
                        stage('Lint Backend') {
                            steps {
                                sh 'go run github.com/golangci/golangci-lint/cmd/golangci-lint@v1.56.1 run'
                            }
                        }
                        stage('Test Backend') {
                            steps {
                                sh 'go test -tags="sqlite sqlite_unlock_notify"'
                            }
                        }
                        stage('Build GO') {
                            steps {
                                sh 'go build -o forgejo'
                            }
                        }
                        stage('Create GO Artifact') {
                            steps {
                                tar(
                                    file: env.GO_ARTIFACT,
                                    archive: true,
                                    compress: true,
                                    glob: 'forgejo',
                                    overwrite: true
                                )
                            }
                        }
                    }
                }
            }
        }
        stage('Sonar Scan') {
            agent {label env.AGENT}
            environment {
                SCANNER_HOME = tool 'sonar-scanner';    
            }
            steps {
                withSonarQubeEnv(installationName: 'sonarqube') {
                    sh '''
                    ${SCANNER_HOME}/bin/sonar-scanner \
                    -Dsonar.projectKey=forgejo \
                    -Dsonar.sources=. \
                    '''
                }
            }
        }
        
        
        stage('Approval') {
            steps {
                input(
                    message: "Proceed with release?",
                    ok: 'Submit'
                )
            }
        }
        stage('Get Next Release Tag') {
            agent any
            steps {
                script {
                    def releases = listGitHubReleases(
                        credentialId: env.ARTIFACTORY_TOKEN,
                        repository: env.ARTIFACTORY_REPO,
                        sortBy: 'SymantecVersion',
                        sortAscending: false,
                        tagNamePattern: "${tagPattern}"
                    )
                    if (!releases) {
                        throw new Exception('Releases should not be null.')
                    }
                    def release = releases[0]
                    nextTag = release.nextSymantecRevision()
                }
            }
        }
        stage('Create Release on Github') {
            agent any
            steps {
                writeFile(
                    file: 'description.md', 
                    text: 'No description'
                )
                createGitHubRelease(
                    credentialId: env.ARTIFACTORY_TOKEN,
                    repository: env.ARTIFACTORY_REPO,
                    tag: "${nextTag}",
                    commitish: env.ARTIFACTORY_BRANCH,
                    bodyFile: 'description.md'
                )
            }
        }
        stage('Push Artifacts to Github Release') {
            agent any
            steps {
                script {
                    def archive_path = "${env.JENKINS_HOME}/jobs/${env.JOB_NAME}/builds/${env.BUILD_NUMBER}/archive"
                    uploadGithubReleaseAsset(
                        credentialId: env.ARTIFACTORY_TOKEN,
                        repository: env.ARTIFACTORY_REPO,
                        tagName: "${nextTag}", 
                        uploadAssets: [
                            [filePath: "${archive_path}/${env.GO_ARTIFACT}"],
                            [filePath: "${archive_path}/${env.NODE_ARTIFACT}"]
                        ]
                    )
                }
            }
        }
        stage('Clean Workspace') {
            agent {label env.AGENT}
            steps {
                cleanWs()
            }
        }
        stage('Trigger CD') {
            steps {
                build job: 'Forgejo_CD',
                    parameters: [string(name: 'tag', value: "${nextTag}")]
            }
        }
    }
}