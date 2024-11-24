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
                        helm uninstall free5gc-helm --namespace free5gc
                        
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
                    if kubectl get pv | grep cert-pv; then
                        echo "Cert persistent volumes already exist"
                    else
                        echo "Create cert persistent volumes"
                        kubectl apply -f /home/hieupt/free5gc-helm/cert-pv.yaml
                    
                    if kubectl get pv | grep mongo-pv; then
                        echo "Mongo persistent volumes already exist"    
                    else
                        echo "Create mongo persistent volume"
                        kubectl apply -f /home/hieupt/free5gc/helm/mongo-pv.yaml
                    
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
