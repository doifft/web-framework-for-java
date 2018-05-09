pipeline {
    agent any

    tools { 
        maven 'Maven 3.5'
    }

    environment { 
        DEPLOY_TARGET    = '../../../webapps/wf.war'
        EMAIL_RECIPIENTS = 'zb@bndy.net'
        HAS_DEPLOYMENT   = 'false'
    }

    stages {
        stage('Prepare') {
            steps {
                updateGithubStatus("PENDING", "Begin to build");
                //sh 'printenv'
                sh 'java -version'
                sh 'mvn -v'
            }
        }
        stage('Build') {
            steps {
                echo 'Building...'
                sh 'mvn -B -DskipTests clean package' 
            }
        }
        stage('Test') {
            steps {
                echo 'Skip tests'
            }
        }
        stage('Deploy') {
            when {
                // following expression is just available for multi-branch project
                branch 'master'
            }
            steps {
                echo 'Deploying....'
                sh 'rm -f $DEPLOY_TARGET'
                sh 'mv ./target/wf-*.war $DEPLOY_TARGET'
                script {
                    HAS_DEPLOYMENT = 'true'
                }
            }
        }
    }
    
    post {
        success {
            updateGithubStatus("SUCCESS", "Build complete");
            sendEmail("SUCCESS");
        }
        unstable {
            updateGithubStatus("UNSTABLE", "Build complete");
            sendEmail("UNSTABLE");
        }
        failure {
            updateGithubStatus("FAILURE", "Build complete");
            sendEmail("FAILURE");
        }
    }
}

// get change log to be send over the mail
@NonCPS
def getChangeString() {
    MAX_MSG_LEN = 100
    def changeString = ""

    echo "Gathering SCM changes"
    def changeLogSets = currentBuild.changeSets
    for (int i = 0; i < changeLogSets.size(); i++) {
        def entries = changeLogSets[i].items
        for (int j = 0; j < entries.length; j++) {
            def entry = entries[j]
            truncated_msg = entry.msg.take(MAX_MSG_LEN)
            changeString += "<li>${truncated_msg} [${entry.author}]</li>"
        }
    }

    if (!changeString) {
        changeString = "<li>No new changes</li>"
    }
    return changeString
}

def sendEmail(status) {
    def subject = "[" + status + "] ${currentBuild.fullDisplayName}"
    if (HAS_DEPLOYMENT == 'true') {
        subject += " - deployed"
    }

    // Default Email Notification Plugin: email(...), but below supports html
        emailext(
        to: "$EMAIL_RECIPIENTS",
        subject: "${subject}",
        body: "Changes:<ul>" + getChangeString() + "</ul><br />Check console output at: ${BUILD_URL}console" + "<br />",
        recipientProviders: [[$class: 'DevelopersRecipientProvider']]
    )
}

// set GitHub status
def getRepoURL() {
    sh "git config --get remote.origin.url > .git/remote-url"
    return readFile(".git/remote-url").trim()
}

def getCommitSha() {
    sh "git rev-parse HEAD > .git/current-commit"
    return readFile(".git/current-commit").trim()
}

def updateGithubStatus(status, message) {
    // status: pending, success, failure or error
    repoUrl = getRepoURL()
    commitSha = getCommitSha()

    step ([
        $class: 'GitHubCommitStatusSetter',
        reposSource: [$class: "ManuallyEnteredRepositorySource", url: repoUrl],
        commitShaSource: [$class: "ManuallyEnteredShaSource", sha: commitSha],
        errorHandlers: [[$class: 'ShallowAnyErrorHandler']],
        statusResultSource: [
            $class: 'ConditionalStatusResultSource',
            results: [
                [$class: 'AnyBuildResult', state: status, message: message]
            ]
        ]
    ])
}
