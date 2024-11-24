pipeline {
    agent any
    environment {
        KUBECONFIG = credentials('kubeconfig') // Use your kubeconfig credentials
    }
    stages {
        stage('Uninstall Old Helm Release') {
            steps {
                script {
                    sh '''                    
                    # Check if the release exists
                    if helm ls -n free5gc | grep -q free5gc-helm; then
                        echo "Uninstalling existing Helm release..."
                        helm uninstall free5gc-helm -n free5gc
                        

                        # Wait for all pods to terminate
                        echo "Waiting for pods to terminate..."
                        timeout=15 # Set timeout in seconds
                        while [ $timeout -gt 0 ]; do
                            pods=$(kubectl get pods -n free5gc --no-headers | wc -l)
                            if [ $pods -eq 0 ]; then
                                echo "All pods have been terminated."
                                break
                            else
                                echo "Pods still terminating. Waiting..."
                                sleep 2
                                timeout=$((timeout-2))
                            fi
                        done

                        echo "Delete old cert pv"
                        kubectl delete pv cert-pv
                        sleep 10
                    else
                        echo "No existing release found. Skipping uninstall."
                    fi    
                    '''
                }
            }
        }
        stage('Reinstall Helm Release') {
            steps {
                script {
                    sh '''
                    echo "Create cert persistent volumes"
                    kubectl apply -f /home/hieupt/free5gc-helm/cert-pv.yaml

                    if kubectl get pv | grep -q mongo-pv; then
                        echo "Mongo persistent volumes already exist"    
                    else
                        echo "Create mongo persistent volume"
                        kubectl apply -f /home/hieupt/free5gc/helm/mongo-pv.yaml
                    fi

                    echo "Installing Helm release..."
                    helm install free5gc-helm /home/hieupt/free5gc-helm/charts/free5gc/ -n free5gc
                    '''
                }
            }
        }

        stage('Wait and Kill dbpython Pod') {
            steps {
                script {
                    sh '''                    
                    echo "Waiting for 10 seconds before killing dbpython pod..."
                    sleep 10
                    
                    echo "Killing pod free5gc-helm-free5gc-dbpython..."
                    kubectl get pods -n free5gc --no-headers | grep '^free5gc-helm-free5gc-dbpython' | awk '{print $1}' | xargs kubectl delete pod -n free5gc
                    '''
                }
            }
        }
    }
    post {
        always {
            echo "Helm reinstall process completed."
        }
    }
}
