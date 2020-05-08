pipeline {

    agent {
        label "master"
    }

    environment {
        // GLobal Vars
        PIPELINES_NAMESPACE = "labs-ci-cd"
        HELM_CHART_NAME = "pet-battle-api"
        HELM_REPO = "http://nexus.nexus.svc.cluster.local:8081/repository/helm-charts/"
        JENKINS_TAG = "${JOB_NAME}.${BUILD_NUMBER}".replace("%2F", "-")
        JOB_NAME = "${JOB_NAME}".replace("/", "-")
        GIT_SSL_NO_VERIFY = true
        GIT_CREDENTIALS = credentials("${PIPELINES_NAMESPACE}-git-auth")
        NEXUS_CREDS = credentials("${PIPELINES_NAMESPACE}-nexus-password")
        ARGOCD_CREDS = credentials("${PIPELINES_NAMESPACE}-argocd-token")
        NEXUS_REPO_NAME = "labs-static"
    }

    // The options directive is for configuration that applies to the whole job.
    options {
        buildDiscarder(logRotator(numToKeepStr: '50', artifactNumToKeepStr: '1'))
        timeout(time: 15, unit: 'MINUTES')
        ansiColor('xterm')
        timestamps()
    }

    stages {
        stage("prepare environment for master deploy") {
            agent {
                node {
                    label "master"
                }
            }
            when {
                expression { GIT_BRANCH ==~ /(.*master)/ }
            }
            steps {
                script {
                    env.PROJECT_NAMESPACE = "labs-dev"
                    env.APP_NAME = "pet-battle-api"
                }
            }
        }
        stage("prepare environment for develop deploy") {
            agent {
                node {
                    label "master"
                }
            }
            when {
                expression { GIT_BRANCH ==~ /(.*develop)/ }
            }
            steps {
                script {
                    env.PROJECT_NAMESPACE = "labs-dev"
                    env.APP_NAME = "pet-battle-api-dev"
                }
            }
        }
        stage("prepare environment for test deploy") {
            agent {
                node {
                    label "master"
                }
            }
            when {
                expression { GIT_BRANCH ==~ /test\/jenkins/ }
            }
            steps {
                script {
                    env.PROJECT_NAMESPACE = "labs-dev"
                    env.APP_NAME = "pet-battle-api-test"
                }
            }
        }
        stage("prepare environment for PR deploy") {
            agent {
                node {
                    label "master"
                }
            }
            when {
                expression { GIT_BRANCH ==~ /PR*/ }
            }
            steps {
                script {
                    env.PROJECT_NAMESPACE = "labs-dev"
                    env.APP_NAME = "pet-battle-api-pr"
                }
            }
        }

        stage("Build (Compile App)") {
            agent {
                node {
                    label "jenkins-slave-mvn"
                }
            }
            steps {
                echo '### configure ###'
                script {
                    // repoint nexus
                    settings = readFile("/home/jenkins/.m2/settings.xml")
                    def newsettings = settings.replace("<id>maven-public</id>", "<id>nexus</id>");
                    newsettings = newsettings.replace("<url>http://nexus:8081/repository/maven-public/</url>", "<url>http://nexus-service:8081/repository/maven-public/</url>")
                    writeFile file: "/tmp/settings.xml", text: "${newsettings}"
                    // versions
                    env.PACKAGE = "${JENKINS_TAG}.tar.gz"
                    // we want jdk.11 - for now in :4.3 slave-mvn
                    env.JAVA_HOME = "/usr/lib/jvm/java-11-openjdk"
                }
                sh 'printenv'

                echo '### Running tests ###'
                // sh 'mvn test'

                echo '### Running build ###'
                sh '''
                    mvn package -DskipTests -s /tmp/settings.xml
                '''

                echo '### Packaging App for Nexus ###'
                sh '''                    
                    tar -zcvf ${PACKAGE} Dockerfile.jvm target/lib target/*-runner.jar
                    curl -vvv -u ${NEXUS_CREDS} --upload-file ${PACKAGE} http://${NEXUS_SERVICE_SERVICE_HOST}:${NEXUS_SERVICE_SERVICE_PORT}/repository/${NEXUS_REPO_NAME}/${APP_NAME}/${PACKAGE}
                '''
            }
            // Post can be used both on individual stages and for the entire build.
        }

        stage("Create OpenShift Build") {
            agent {
                node {
                    label "jenkins-slave-argocd"
                }
            }
            when {
                expression {
                    openshift.withCluster() {
                        openshift.withProject("${PIPELINES_NAMESPACE}") {
                            return !openshift.selector("bc", "${APP_NAME}").exists();
                        }
                    }
                }
            }
            steps {
                echo '### Create BuildConfig ###'
                sh '''
                    # oc patch bc ${APP_NAME} -p "{\\"spec\\":{\\"output\\":{\\"imageLabels\\":[{\\"name\\":\\"THINGY\\",\\"value\\":\\"MY_AWESOME_THINGY\\"},{\\"name\\":\\"OTHER_THINGY\\",\\"value\\":\\"MY_OTHER_AWESOME_THINGY\\"}]}}}"
                    oc new-build --binary --name=${APP_NAME} -l app=${APP_NAME} --strategy=docker --dry-run -o yaml > /tmp/bc.yaml
                    yq w -i /tmp/bc.yaml items[1].spec.strategy.dockerStrategy.dockerfilePath Dockerfile.jvm
                    oc apply -f /tmp/bc.yaml
                '''
            }
        }

        stage("Bake (OpenShift Build)") {
            agent {
                node {
                    label "jenkins-slave-argocd"
                }
            }
            steps {
                echo '### Get Binary from Nexus and shove it in a box ###'
                sh '''
                    curl -v -f -u ${NEXUS_CREDS} http://${NEXUS_SERVICE_SERVICE_HOST}:${NEXUS_SERVICE_SERVICE_PORT}/repository/${NEXUS_REPO_NAME}/${APP_NAME}/${PACKAGE} -o ${PACKAGE}
                    oc start-build ${APP_NAME} --from-archive=${PACKAGE} --follow
                    oc tag ${PIPELINES_NAMESPACE}/${APP_NAME}:latest ${PROJECT_NAMESPACE}/${APP_NAME}:${JENKINS_TAG}
                '''
            }
        }

        stage("Git Commit Chart") {
            agent {
                node {
                    label "jenkins-slave-argocd"
                }
            }
            steps {
                echo '### Commit new image tag to git ###'
                script {
                    env.SEM_VER = sh(returnStdout: true, script: "./update_version.sh chart/Chart.yaml patch").trim()
                }
                sh 'printenv'
                sh '''
                    yq w -i chart/Chart.yaml 'name' ${HELM_CHART_NAME}                    
                    yq w -i chart/Chart.yaml 'appVersion' ${JENKINS_TAG}
                    yq w -i chart/values.yaml 'image_repository' 'image-registry.openshift-image-registry.svc:5000'
                    yq w -i chart/values.yaml 'image_name' ${APP_NAME}
                    yq w -i chart/values.yaml 'image_namespace' ${PROJECT_NAMESPACE}

                    git checkout -b ${GIT_BRANCH} origin/${GIT_BRANCH}
                    git config --global user.email "jenkins@rht-labs.bot.com"
                    git config --global user.name "Jenkins"
                    git config --global push.default simple
                    git add chart/Chart.yaml chart/values.yaml
                    git commit -m "🚀 AUTOMATED COMMIT - Deployment new app version ${JENKINS_TAG} 🚀"
                    git remote set-url origin https://${GIT_CREDENTIALS_USR}:${GIT_CREDENTIALS_PSW}@github.com/eformat/pet-battle-api.git
                    git push origin ${GIT_BRANCH}
                '''
            }
        }

        stage("Upload Helm Chart") {
            agent {
                node {
                    label "jenkins-slave-helm"
                }
            }
            steps {
                echo '### Upload Helm Chart to Nexus ###'
                sh '''
                    git checkout ${GIT_BRANCH}
                    git pull
                    helm package chart/
                    curl -vvv -u ${NEXUS_CREDS} ${HELM_REPO} --upload-file ${HELM_CHART_NAME}-${SEM_VER}.tgz
                '''
            }
        }

        stage("ArgoCD Create App") {
            agent {
                node {
                    label "master"
                }
            }
            when {
                expression {
                    def retVal = sh(returnStatus: true, script: "oc -n \"${PIPELINES_NAMESPACE}\" get applications.argoproj.io \"${APP_NAME}\" -o name")
                    if (retVal == null || retVal == "") {
                        return 0;
                    }
                    return 1;
                }
            }
            steps {
                echo '### Create ArgoCD App ###'
                sh '''
                    cat <<EOF | oc apply -n ${PIPELINES_NAMESPACE} -f -
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  finalizers:
  - resources-finalizer.argocd.argoproj.io
  name: ${APP_NAME}
spec:
  destination:
    namespace: ${PROJECT_NAMESPACE}
    server: https://kubernetes.default.svc
  project: default
  source:
    helm:
      releaseName: ${APP_NAME}
    path: chart
    repoURL: ${HELM_REPO}
    targetRevision: ${SEM_VER}
    chart: ${HELM_CHART_NAME}
    automated:
      prune: true
      selfHeal: true
      validate: true
  ignoreDifferences:
  - group: apps.openshift.io
    jsonPointers:
    - /spec/template/spec/containers/0/image
    - /spec/triggers/0/imageChangeParams/lastTriggeredImage
    - /spec/triggers/1/imageChangeParams/lastTriggeredImage
    kind: DeploymentConfig
EOF
                '''
            }
        }

        stage("Deploy (ArgoCD)") {
            agent {
                node {
                    label "jenkins-slave-argocd"
                }
            }
            steps {
                script {
                    echo '### Ask ArgoCD to Sync the changes and roll it out ###'
                    def patch = $/argocd app patch "${APP_NAME}" --patch $'{\"spec\":{\"source\":{\"targetRevision\":\"${SEM_VER}\"}}}' --type merge --auth-token ${ARGOCD_CREDS_PSW} --server ${ARGOCD_SERVER_SERVICE_HOST}:${ARGOCD_SERVER_SERVICE_PORT_HTTP} --insecure/$
                    sh patch
                }
                sh '''
                    ARGOCD_INFO="--auth-token ${ARGOCD_CREDS_PSW} --server ${ARGOCD_SERVER_SERVICE_HOST}:${ARGOCD_SERVER_SERVICE_PORT_HTTP} --insecure"
                    argocd app sync ${APP_NAME} ${ARGOCD_INFO} --force --async --prune
                    argocd app wait ${APP_NAME} ${ARGOCD_INFO}
                '''
            }
        }
    }
}
