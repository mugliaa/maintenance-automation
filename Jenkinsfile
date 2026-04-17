pipeline {
    agent any

    parameters {
        string(
            name: 'TARGET_REPO_URL',
            defaultValue: '',
            description: 'Full SSH clone URL of the target Bitbucket repo (e.g. ssh://git@bitbucket.aoins.com/mktg/mktg-tomcat-starter.git)'
        )
        string(
            name: 'TARGET_BRANCH',
            defaultValue: 'develop',
            description: 'The branch in the target repo to check out and use for the changes'
        )
        string(
            name: 'RECIPE_NAME',
            defaultValue: '',
            description: 'The recipe name to run (e.g. org.openrewrite.java.migrate.UpgradeToJava21 or com.aoins.rewrite.example.MyRecipe)'
        )
    }

    tools {
        jdk 'OpenJDK21'
    }

    environment {
        JAVA_HOME = "${tool 'OpenJDK21'}"
        PATH = "${env.JAVA_HOME}/bin:${env.PATH}"
        REWRITE_BRANCH = 'feature/openrewrite'
        TARGET_DIR = 'target-repo'
    }

    stages {
        stage('Checkout Target Repo') {
            steps {
                dir(env.TARGET_DIR) {
                    git(
                        url: params.TARGET_REPO_URL,
                        branch: params.TARGET_BRANCH,
                        credentialsId: 'prod-bitbucket-ssh'
                    )
                }
            }
        }

        stage('Run OpenRewrite') {
            steps {
                script {
                    def recipeFiles = findFiles(glob: 'recipes/**/*.yml').sort { it.path }
                    def sb = new StringBuilder()
                    recipeFiles.eachWithIndex { f, i ->
                        if (i > 0) sb.append('\n---\n')
                        sb.append(readFile(f.path))
                    }
                    writeFile file: "${env.WORKSPACE}/combined-rewrite.yml", text: sb.toString()
                }
                dir(env.TARGET_DIR) {
                    sh """
                        ./gradlew rewriteRun \
                            --init-script "${WORKSPACE}/init.gradle" \
                            -Drewrite.activeRecipe="${params.RECIPE_NAME}" \
                            -Drewrite.configFile="${WORKSPACE}/combined-rewrite.yml" \
                            --no-daemon
                    """
                }
            }
        }

        stage('Commit and Push') {
            steps {
                script {
                    dir(env.TARGET_DIR) {
                        def changes = sh(
                            script: 'git status --porcelain',
                            returnStdout: true
                        ).trim()

                        if (!changes) {
                            echo 'No changes produced by OpenRewrite. Skipping commit and pull request.'
                            return
                        }

                        sshagent(credentials: ['prod-bitbucket-ssh']) {
                            sh """
                                git config user.email "jenkins@aoins.com"
                                git config user.name  "Jenkins"
                                git checkout -B ${env.REWRITE_BRANCH}
                                git add -A
                                git commit -m "Apply OpenRewrite recipe: ${params.RECIPE_NAME}"
                                git push origin ${env.REWRITE_BRANCH} --force
                            """
                        }
                    }
                }
            }
        }
    }

    post {
        always {
            cleanWs()
        }
    }
}
