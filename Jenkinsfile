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
                        sleep 20
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
                    sleep 10
                    

                    echo "Waiting for mongo pod to run before killing dbpython pod..."
                    while true; do
                        mongodb_status=$(kubectl get pod mongodb-0 -n free5gc -o jsonpath='{.status.phase}' 2>/dev/null)
                        if [ "$mongodb_status" != "Running" ]; then
                            echo "Waiting for pod mongodb-0 to be Running. Current status: $mongodb_status"
                            sleep 10
                        else 
                            echo "Pod mongodb-0 is running"
                            break
                        fi
                    done
                    

                    echo "Killing pod free5gc-helm-free5gc-dbpython..."
                    kubectl get pods -n free5gc --no-headers | grep dbpython | awk '{print $1}' | xargs kubectl delete pod -n free5gc
                    sleep 10

                    echo "Waiting for other pods to run..."
                    retry=0
                    while true; do
                        sleep 10
                        podNumber=$( kubectl get pods -n free5gc --no-headers | grep -v Running | wc -l)
                        if [ $podNumber -eq 0 ]; then
                            echo "All pods start."
                            break
                        else
                            echo "Recreating pods..."
                            if [ $retry -le 5 ]; then
                                retry=$((retry + 1))
                                pods=$( kubectl get pods -n free5gc --no-headers | grep -v Running)
                                for pod in $pods; do
                                    kubectl delete pod -n free5gc $pod
                                done  
                            else 
                                echo "Max retries reached. Exiting."
                                exit 1                                  
                            fi
                        fi
                    done
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
