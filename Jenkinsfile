def version = ""
def versionPattern = ~/v\d*\.\d*.\d*/
def user = env.BUILD_USER_ID

node(){
    parameterValue = params.remove_release_branch ?: false
}

pipeline{
    agent any
    options {
        skipDefaultCheckout(true)
    }

    tools {
      jdk 'jdk-17'
      maven 'maven'
    }

    stages{
        stage("Log params"){
            script{
                echo """
                ***************
                    ${parameterValue}
                ***************
                """
            }
        }

        stage("Prepare Parameters"){
            steps{
                script{
                    properties([
                        parameters([
                            string(defaultValue: '', name: 'new_version'),
                            string(defaultValue: 'git@github.com:Koziolek/maven-release-example.git', name: 'repository_url'),
                            string(defaultValue: 'development', name: 'dev_branch'),
                            string(defaultValue: 'master', name: 'master_branch'),
                            string(defaultValue: 'release', name: 'release_branch'),
                            string(defaultValue: 'jenkins-priv', name: 'git_credentials'),
                            booleanParam(defaultValue: parameterValue, name: 'remove_release_branch')
                        ])
                    ])
                }
            }
        }
        stage("Prepare Workspace"){
            steps{
                cleanWs() // clean workspace
            }
        }
        stage("Compile and test"){
            steps{
                git branch: params.dev_branch, url: params.repository_url, credentialsId: params.git_credentials
                sh 'git config user.name koziolek'
                sh 'git config user.email bjkuczynski@gmail.com'
                sshagent(credentials: ["jenkins-priv", "nexus"]) {
                    withMaven(mavenSettingsConfig: "3df3ff2d-fe3b-4539-8ed8-84f09f411b0f" ) {
                        sh """
                            mvn -V -U verify
                        """
                    }
                }
            }
        }
        stage("Calculate version"){
            parallel{
                stage("Use default version"){
                    when{
                        expression{
                            params.new_version == ''
                        }
                    }
                    steps{
                        sshagent(credentials: ["jenkins-priv", "nexus"]) {
                            withMaven(mavenSettingsConfig: "3df3ff2d-fe3b-4539-8ed8-84f09f411b0f" ) {
                                sh "mvn -B -q release:prepare -DskipTests"
                                script{
                                    def releaseProperties = readProperties  file: './release.properties'
                                    version = releaseProperties["scm.tag"]
                                    echo version
                                }
                                sh "mvn -B -q release:rollback -DskipTests"
                            }
                        }
                    }
                }
                stage("Use version from params"){
                    when{
                        expression{
                            params.new_version != ''
                        }
                    }
                    steps{
                        script{
                            version = params.new_version
                        }
                    }
                }
            }
        }
        stage("Validate version"){
            when{
                expression{
                    ! (version ==~ versionPattern)
                }
            }
            steps{
                error("""
                    Version format does not match!
                    Expected format is /${versionPattern}/, but given value is '${version}'
                """)
            }
        }
        stage("Pre-release summary"){
            steps{
                script{
                    echo """
                        ***************************************
                                Performing release!
                        ***************************************
                        Project: ${repository_url}
                        Version: ${version}

                        Started by: ${user}
                    """
                }
            }
        }
        stage("Release"){
            steps{
                sshagent(credentials: ["jenkins-priv", "nexus"]) {
                    withMaven(mavenSettingsConfig: "3df3ff2d-fe3b-4539-8ed8-84f09f411b0f" ) {
                        sh """
                            git checkout -b ${params.release_branch}/${version}
                            mvn -B -V release:prepare release:perform -DskipTests
                            git checkout ${params.dev_branch}
                            git merge --no-ff ${params.release_branch}/${version}
                            git push --set-upstream origin ${params.dev_branch}
                            git checkout ${params.release_branch}/${version}
                            git reset --hard HEAD~1
                            git push --force origin ${params.release_branch}/${version}
                            git checkout ${params.master_branch}
                            git merge --no-ff ${params.release_branch}/${version}
                            git push --set-upstream origin ${params.master_branch}
                        """
                    }
                }
            }
        }
        stage("cleanup"){
            when{
                expression{
                    params.remove_release_branch
                }
            }
            steps{
                sh "git branch -d ${params.release_branch}/${version}"
            }
        }
    }
    post {
        failure { // TODO: this cleanup need parametrization
            sshagent(credentials: ["jenkins-priv"]) {
                withMaven(mavenSettingsConfig: "3df3ff2d-fe3b-4539-8ed8-84f09f411b0f" ) {
                    sh """
                        mvn -B -V release:rollback
                        bash -c  '[[ ! -z  `git branch -a | grep ${params.release_branch}/${version}` ]] && git push --delete origin ${params.release_branch}/${version} '
                    """
                }
            }
        }
    }
}
