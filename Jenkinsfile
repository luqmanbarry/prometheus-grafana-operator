pipeline {
    agent {
        node {
            label 'centos-large'
        }
    }

    parameters {
        string(name: 'gitBranch', defaultValue: 'NonProd-ROSAChanges-Task', description: 'The Git branch to checkout from.', trim: true)
        string(name: 'dataSourceServiceAccount', defaultValue: 'grafana-thanos', description: 'Service Account name used by Grafana to access the metrics Server.', trim: true)
        booleanParam(name: 'baseGrafanaInstalled', defaultValue: true, description: 'Whether base Grafana Installation is available or not. e.g.: GrafanaDatasource, Grafana')
        string(name: 'ocTokenCredentialsId', defaultValue: 'abc-grafana-pipeline-sa', description: 'Token used by the OpenShift CLI to authenticate.', trim: true)
        string(name: 'targetNamespace', defaultValue: 'abc-grafana', description: 'The target namespace to deploy the monitoring resources.', trim: true)
        string(name: 'chartName', defaultValue: 'grafana-resources', description: 'The chart name to be installed.', trim: true)
        string(name: 'promQLTargetNamespaceWhitelist', defaultValue: 'abc-.*|ede-.*', description: 'PromQL Namespace whitelist. e.g.: abc-*|acb-.*', trim: true)
        string(name: 'promQLTargetNamespaceBlacklist', defaultValue: '.*-bld', description: 'PromQL Namespace blacklist. e.g.: .*-bld', trim: true)
        string(name: 'promQLTargetPodWhitelist', defaultValue: 'web.*|.*service.*', description: 'PromQL Pod whitelist. e.g.: web.*|.*service.*', trim: true)
        string(name: 'promQLTargetPodBlacklist', defaultValue: '.*-build', description: 'PromQL Pod blacklist. e.g.: .*-build', trim: true)
        choice(name: 'defaultUserRole', choices: ['Editor', 'Viewer', 'Admin'], description: 'Default role of authenticated users. For first time install, the role must be set to Admin.')
        string(name: 'valuesFile', defaultValue: 'charts/grafana-resources/values.yaml', description: 'The vavlues file to use. Specify values in a YAML file or a URL (can specify multiple)', trim: true)
    }

    options {
        timeout(time: 15, unit: 'MINUTES') 
        timestamps()
        buildDiscarder(logRotator(numToKeepStr: '10', daysToKeepStr: '30'))
        disableConcurrentBuilds()
        disableResume()
    }

    stages {
        stage("Verify Tools") {
            steps {
                sh ' echo "  Verying required tools are installed  " '
                sh '  echo "Verify HELM is installed." '
                sh '''
                    if ! helm version | grep 'v3'
                    then
                        echo "Helm v3.x could not be found."
                        exit -1
                    fi
                '''
                sh ' echo "  Verify OpenShift cli is installed.    " '
                sh '''
                    if ! oc version | grep 'Client Version: 4'
                    then
                        echo "oc cli v4.x could not be found."
                        exit -1
                    fi
                '''
            }
        }
        stage("Install Chart") {
            environment { 
                OCP_LOGIN                = credentials("${params.ocTokenCredentialsId}")
                NAMESPACE                = "${params.targetNamespace}"
                DS_SA_NAME               = "${params.dataSourceServiceAccount}"
                CHART_NAME               = "${params.chartName}"
                VALUES_FILE              = "${params.valuesFile}"
                NAMESPACE_WHITELIST      = "${params.promQLTargetNamespaceWhitelist}"
                NAMESPACE_BLACKLIST      = "${params.promQLTargetNamespaceBlacklist}"
                POD_WHITELIST            = "${params.promQLTargetPodWhitelist}"
                POD_BLACKLIST            = "${params.promQLTargetPodBlacklist}"
                BASE_GRAFANA_INSTALLED   = "${params.baseGrafanaInstalled}"
                DEFAULT_USER_ROLE        = "${params.defaultUserRole}"
                INSTALL_USER_ROLE        = "Admin"
            }
            steps {
                timeout(time: 10, unit: 'MINUTES') {
                    sh ' echo "    Installing the helm chart...        " '
                    sh ' oc login "${OCP_LOGIN_USR}" --token "${OCP_LOGIN_PSW}" '
                    sh ' oc project ${NAMESPACE} '
                    retry(3) {

                        script {
                            if ( env.BASE_GRAFANA_INSTALLED == 'true' ) {

                                 sh '''
                                    echo "Delete Grafana instance if in 'reconciling' phase."
                                    if [ "$(oc get grafana -o jsonpath={.items[*].status.phase} | awk 'NR==1 {print $1}')" == "reconciling" ]; then
                                        oc delete deployment/grafana-deployment
                                    fi

                                    sleep 15

                                    echo "Delete Grafananotificationchannel instance if in 'reconciling' phase."
                                    if [ "$(oc get grafananotificationchannel -o jsonpath={.items[*].status.phase} | awk 'NR==1 {print $1}')" == "reconciling" ]; then
                                        oc delete $(oc get grafananotificationchannel -o name)
                                    fi

                                    sleep 10
                                    
                                    helm upgrade --install ${CHART_NAME} ./charts/grafana-resources \
                                        --set baseGrafanaInstalled=${BASE_GRAFANA_INSTALLED} \
                                        --set grafanaInstance.serverRootUrl="$(oc get route grafana-route -o jsonpath={.spec.host} -n ${NAMESPACE})" \
                                        --set selectors.prometheusQueries.namespace.whitelist=${NAMESPACE_WHITELIST} \
                                        --set selectors.prometheusQueries.namespace.blacklist=${NAMESPACE_BLACKLIST} \
                                        --set selectors.prometheusQueries.pod.whitelist=${POD_WHITELIST} \
                                        --set selectors.prometheusQueries.pod.blacklist=${POD_BLACKLIST} \
                                        -f ${VALUES_FILE} \
                                        -n ${NAMESPACE}

                                    sleep 30

                                    echo "Rolling out grafana-operator Deployment"
                                    oc rollout restart deployment/grafana-operator-controller-manager

                                    sleep 80

                                    echo "Rolling out grafana-deployment Deployment"
                                    oc rollout restart deployment/grafana-deployment

                                    sleep 30
                                '''

                            } else {

                                sh '''
                                    echo "Delete Grafana instance if in 'reconciling' phase."
                                    if [ "$(oc get grafana -o jsonpath={.items[*].status.phase} | awk 'NR==1 {print $1}')" == "reconciling" ]; then
                                        oc delete deployment/grafana-deployment
                                    fi

                                    sleep 15

                                    echo "Delete Grafananotificationchannel instance if in 'reconciling' phase."
                                    if [ "$(oc get grafananotificationchannel -o jsonpath={.items[*].status.phase} | awk 'NR==1 {print $1}')" == "reconciling" ]; then
                                        oc delete $(oc get grafananotificationchannel -o name)
                                    fi

                                    sleep 10

                                    helm upgrade --install ${CHART_NAME} ./charts/grafana-resources \
                                        --set baseGrafanaInstalled=${BASE_GRAFANA_INSTALLED} \
                                        --set grafanaDataSource.auth.bearerToken="$(oc sa get-token ${DS_SA_NAME} -n ${NAMESPACE})" \
                                        --set grafanaInstance.serverRootUrl="$(oc get route grafana-route -o jsonpath={.spec.host} -n ${NAMESPACE})" \
                                        --set grafanaInstance.defaultRole=${INSTALL_USER_ROLE} \
                                        --set selectors.prometheusQueries.namespace.whitelist=${NAMESPACE_WHITELIST} \
                                        --set selectors.prometheusQueries.namespace.blacklist=${NAMESPACE_BLACKLIST} \
                                        --set selectors.prometheusQueries.pod.whitelist=${POD_WHITELIST} \
                                        --set selectors.prometheusQueries.pod.blacklist=${POD_BLACKLIST} \
                                        -f ${VALUES_FILE} \
                                        -n ${NAMESPACE}

                                    sleep 45

                                    echo " Setting default authentication user role "
                            
                                    helm upgrade --install ${CHART_NAME} ./charts/grafana-resources \
                                        --set baseGrafanaInstalled=${BASE_GRAFANA_INSTALLED} \
                                        --set grafanaDataSource.auth.bearerToken="$(oc sa get-token ${DS_SA_NAME} -n ${NAMESPACE})" \
                                        --set grafanaInstance.serverRootUrl="$(oc get route grafana-route -o jsonpath={.spec.host} -n ${NAMESPACE})" \
                                        --set grafanaInstance.defaultRole=${DEFAULT_USER_ROLE} \
                                        --set selectors.prometheusQueries.namespace.whitelist=${NAMESPACE_WHITELIST} \
                                        --set selectors.prometheusQueries.namespace.blacklist=${NAMESPACE_BLACKLIST} \
                                        --set selectors.prometheusQueries.pod.whitelist=${POD_WHITELIST} \
                                        --set selectors.prometheusQueries.pod.blacklist=${POD_BLACKLIST} \
                                        -f ${VALUES_FILE} \
                                        -n ${NAMESPACE}
                                        
                                    sleep 30

                                    echo "Rolling out grafana-operator Deployment"
                                    oc rollout restart deployment/grafana-operator-controller-manager

                                    sleep 80

                                    echo "Rolling out grafana-deployment Deployment"
                                    oc rollout restart deployment/grafana-deployment

                                    sleep 30
                                '''
                            }
                        }

                    }
                }
            }
        }
        stage("Verify Installation") {
            environment { 
                CHART_NAME   = "${params.chartName}"
                NAMESPACE    =  "${params.targetNamespace}"
            }
            steps {
                retry(3) {
                    sh 'echo "Post installation validation..."'
                    sleep 60
                    sh '''
                        if ! helm history ${CHART_NAME}
                        then
                            echo "Chart installation failed."
                            exit -1
                        fi
                        if [ "$(oc get po -n ${NAMESPACE} | grep Running | wc -l)" -gt "1" ];
                        then
                            echo "$$$$> Grafana resources installation successful."
                        else
                            echo "&&&&> Required Grafana Pods are down. Troubleshoot using the CLI and Web Console"
                            exit -1
                        fi
                    '''
                    sh 'echo "====> BEGIN: Listing Installed resources "'
                    sh 'oc get subscription,grafana,grafanadatasource,grafanadashboard,grafananotificationchannel,all'
                    sh 'echo "====> END: Listing Installed resources "'
                }
            }
        }
    }
    post {
        always {
            echo "Terminating OCP user session..."
            sh 'oc logout &> /dev/null'
            deleteDir()
        }
    }
}