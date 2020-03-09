pipeline {
    options {
        disableConcurrentBuilds()
    }
    agent any
    tools {
        jdk 'jdk11'
    }

    environment {
        WLSIMG_BLDDIR = "${env.WORKSPACE}/resources/build"
        WLSIMG_CACHEDIR = "${env.WORKSPACE}/resources/cache"
        IMAGE_TAG = "phx.ocir.io/weblogick8s/onprem-domain-image:${sh(returnStdout: true, script: 'date +%Y%m%d')}"
    }

    stages {
        stage ('Environment') {
            steps {
                sh '''
                    mkdir -p  ${WLSIMG_BLDDIR} ${WLSIMG_CACHE_DIR}
                    echo "IMAGE_TAG = ${IMAGE_TAG}" 
                    echo "PATH = ${PATH}"
                    echo "M2_HOME = ${M2_HOME}"
                    echo "JAVA_HOME = ${JAVA_HOME}"
                '''
            }
        }
        stage ('Build Archive') {
            steps {
                sh '''
                    chmod +x build-archive.sh 
                    ./build-archive.sh
                '''
            }
        }
        stage ('Build Image') {
            steps {
                sh '''
                    curl -SLO  https://github.com/oracle/weblogic-image-tool/releases/download/release-1.8.1/imagetool.zip
                    unzip -o ./imagetool.zip
                    rm -rf ${WLSIMG_CACHEDIR}
                    export OLD_IMAGE = "$(docker images phx.ocir.io/weblogick8s/onprem-domain-image | tail -n +2 | awk '{print $1":"$2}')"
                    echo "IMAGE_TAG = ${IMAGE_TAG}" 
                    echo "OLD_IMAGE = ${OLD_IMAGE}" 
                    docker rmi phx.ocir.io/weblogick8s/onprem-domain-image:
                    imagetool/bin/imagetool.sh cache addInstaller --type wdt --path /scratch/artifacts/imagetool/weblogic-deploy.zip --version 1.1.1
                    imagetool/bin/imagetool.sh cache addInstaller --type wls --path /scratch/artifacts/imagetool/fmw_12.2.1.4.0_wls_Disk1_1of1.zip --version 12.2.1.4.0
                    imagetool/bin/imagetool.sh cache addInstaller --type jdk --path /scratch/artifacts/imagetool/jdk-8u212-linux-x64.tar.gz --version 8u212
                    echo "items in cache"
                    imagetool/bin/imagetool.sh cache listItems
                    imagetool/bin/imagetool.sh update --tag=${IMAGE_TAG} --fromImage=${OLD_IMAGE} --wdtOperation deploy --wdtArchive=./archive.zip --wdtModel=./App_DataSource.yaml --wdtDomainHome=/u01/oracle/user_projects/domains/onprem-domain --wdtVariables=./domain.properties --wdtVersion=1.1.1
                    docker rmi -f ${OLD_IMAGE}
                '''
            }
        }
        stage ('Push Image') {
            steps {
                sh '''
                    docker push ${IMAGE_TAG}
                '''
            }
        }
        stage('Roll Updates') {
            steps {
                sh '''
                    export KUBECONFIG=/scratch/k8s-demo/mrcluster_kubeconfig
                    export OCI_CLI_PROFILE=MONICA
                    export OCI_CONFIG_FILE=/var/lib/jenkins/.oci/config
                    export PATH=/var/lib/jenkins/bin:$PATH
                    kubectl get nodes
                    ls ./domain.yaml
                    kubectl patch domain onprem-domain -n onprem-domain-ns --type='json' -p='[{"op": "replace", "path": "/spec/image", "value": "${IMAGE_TAG}" }]' 
                    kubectl get po -n onprem-domain-ns
                '''
            }
       }
  }
  post {
    cleanup {
        deleteDir() /* clean up the workspace */
    }
  }
}
