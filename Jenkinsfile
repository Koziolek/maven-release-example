def version = ""
def versionPattern = ~/v\d*\.\d*.\d*/
def user = env.BUILD_USER_ID

pipeline{
    agent any
    options {
        skipDefaultCheckout(true)
    }

    tools {
      jdk 'jdk-17'
    }

    stages{
        stage("Prepare Parameters"){
            steps{
                script{
                    properties([
                        parameters([
                            string(defaultValue: '', name: 'new_version'),
                            string(defaultValue: 'git@github.com:Koziolek/maven-release-example.git', name: 'repository_url'),
                            string(defaultValue: 'development', name: 'dev_branch'),
                            string(defaultValue: 'master', name: 'master_branch'),
                            string(defaultValue: 'release/', name: 'release_branch')
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
                git branch: 'development', url: 'git@github.com:Koziolek/maven-release-example.git', credentialsId: "jenkins-priv"
                sh 'git config user.name koziolek'
                sh 'git config user.email bjkuczynski@gmail.com'
                sshagent(credentials: ["jenkins-priv", "nexus"]) {
                    withMaven(maven: "maven", mavenSettingsConfig: "3df3ff2d-fe3b-4539-8ed8-84f09f411b0f" ) {
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
                            withMaven(maven: "maven", mavenSettingsConfig: "3df3ff2d-fe3b-4539-8ed8-84f09f411b0f" ) {
                                sh "mvn -B -V release:prepare -DskipTests"
                                script{
                                    def releaseProperties = readProperties  file: './release.properties'
                                    version = releaseProperties["scm.tag"]
                                    echo version
                                }
                                sh "mvn -B -V release:rollback"
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
                error("Version format does not match")
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
                    withMaven(maven: "maven", mavenSettingsConfig: "3df3ff2d-fe3b-4539-8ed8-84f09f411b0f" ) {
                        sh """
                            git checkout -b release/${version}
                            mvn -B -V release:prepare release:perform -DskipTests
                            git checkout development
                            git merge --no-ff release/${version}
                            git push --set-upstream origin development
                            git checkout release/${version}
                            git reset --hard HEAD~1
                            git push --force origin release/${version}
                            git checkout development
                            git checkout master
                            git merge --no-ff release/${version}
                            git push --set-upstream origin master
                            git branch -d release/${version}
                        """
                    }
                }
            }
        }
    }
    post {
        failure {
            sshagent(credentials: ["jenkins-priv"]) {
                withMaven(maven: "maven", mavenSettingsConfig: "3df3ff2d-fe3b-4539-8ed8-84f09f411b0f" ) {
                    sh """
                        mvn -B -V release:rollback
                        bash -c  '[[ ! -z  `git branch -a | grep release/${version}` ]] && git push --delete origin release/${version} '
                    """
                }
            }
        }
    }
}