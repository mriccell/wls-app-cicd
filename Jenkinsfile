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
        IMAGE_TAG = "phx.ocir.io/weblogick8s/onprem-domain-image:"
    }

    stages {
        stage ('Environment') {
            steps {
                sh '''
                    mkdir -p  ${WLSIMG_BLDDIR} ${WLSIMG_CACHE_DIR}
                    export IMAGE_TAG="${IMAGE_TAG}$(date +%Y%m%d)"
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
                    export KUBECONFIG=/scratch/k8s-demo/mrcluster_kubeconfig
                    export OCI_CLI_PROFILE=MONICA
                    export OCI_CONFIG_FILE=/var/lib/jenkins/.oci/config
                    export PATH=/home/opc/bin:$PATH
                    whoami
                    which oci
                    oci
                    kubectl get nodes
                    curl -SLO  https://github.com/oracle/weblogic-image-tool/releases/download/release-1.6.0/imagetool.zip
                    jar xvf imagetool.zip
                    chmod +x imagetool/bin/*
                    rm -rf ${WLSIMG_CACHEDIR}
                    imagetool/bin/imagetool.sh cache addInstaller --type wdt --path /scratch/artifacts/imagetool/weblogic-deploy.zip --version 1.1.1
                    imagetool/bin/imagetool.sh cache addInstaller --type wls --path /scratch/artifacts/imagetool/fmw_12.2.1.4.0_wls_Disk1_1of1.zip --version 12.2.1.4.0
                    imagetool/bin/imagetool.sh cache addInstaller --type jdk --path /scratch/artifacts/imagetool/jdk-8u212-linux-x64.tar.gz --version 8u212
                    imagetool/bin/imagetool.sh @build_app_image_args
                '''
            }
        }
        stage ('Push Image') {
            steps {
                sh '''
                    docker push ${IMAGE_TAG}
                    docker rmi ${IMAGE_TAG} 
                '''
            }
        }
        stage('Roll Updates') {
            steps {
                sh '''
                    export KUBECONFIG=/scratch/k8s-demo/mrcluster_kubeconfig
                    export OCI_CLI_PROFILE=MONICA
                    export OCI_CONFIG_FILE=/var/lib/jenkins/.oci/config
                    ls ./domain.yaml
                    kubectl patch domain onprem-domain -n onprem-domain-ns --type='json' -p='[{"op": "replace", "path": "/spec/image", "value": ${IMAGE_TAG} }]' 
                    kubectl get po -n onprem-domain-ns
                '''
            }
     }
  }
}
