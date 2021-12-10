# kubernetes_project

* TASK: clone https://github.com/avielb/rmqp-example repo.   
    Create helm release from the docker-compose files.
    Create full ci cd
    Install RabbitMQ with expoter


CI - 
    Jenkins using git push notification from github.
    Each app has it's own Jenkins file that create and push docker image on kubernetes cluster using kaniko.
CD - 
    Using argo-CD to update changes for the helm chart.
    Using Flagger with istio to implement progressive delivery.

# install jenkins on kubernetes cluster
https://www.jenkins.io/doc/book/installing/kubernetes/
$ helm repo add jenkinsci https://charts.jenkins.io
$ helm repo update
$ helm search repo jenkinsci
$ k craete ns jenkins
$ helm show values jenkinsci/jenkins > ~/Desktop/jenkins.yaml

* check jenkins-values/values.yaml

edited -

    service type: NodePort
    under podLavels change the nodePort to 32323

    installPlugins:
        - kubernetes:1.30.11
        - workflow-aggregator:2.6
        - git:4.10.0
        - configuration-as-code:1.54
        - ansicolor:1.0.1
        - blueocean:1.25.2
        - ace-editor:1.1
        - workflow-support:3.8
        - junit:1.53
        - lockable-resources:2.12
        - pipeline-rest-api:2.19
        - workflow-cps-global-lib:2.21
        - pipeline-graph-analysis:1.12

    # Set to false to download the minimum required version of all dependencies.
    installLatestPlugins: true

    # Set to true to download latest dependencies of any plugin that is requested to have the latest version.
    installLatestSpecifiedPlugins: true

    set persistence to fasle - *just for the demo! not for production*

$ helm upgrade -i jenkins jenkinsci/jenkins --values ~/Desktop/jenkins.yaml -n jenkins

# Connect Jenkins to minikube
go to manage system-> Configure system -> cloud -> add a new cloud -> kubernetes -> kubernetes cloud details

in the kubernetes url paste the server url -> use the command kubectl config view

before setting the the credentials set .kube/config

change the data in the kubeconfig - minikube sets the file location and not the data it self
for example: switch the 

*certificate-authority: /home/guy/.minikube/ca.crt*

to

cat /home/guy/.minikube/ca.crt | base64 -w 0; echo

*certificate-authority-data: <base64 root ca>*

do the same to 
    client-certificate: /home/guy/.minikube/profiles/minikube/client.crt
    client-key: /home/guy/.minikube/profiles/minikube/client.key
dont forget to add 'data' at the end of the day

add a new credentials -> set credentials from file and add .kube/config

test connection to minikube

# Create secret for kaniko

	export REGISTRY_SERVER=https://index.docker.io/v1/

	# Replace `[...]` with the registry username
	export REGISTRY_USER=[...]
	
	# Replace `[...]` with the registry password
	export REGISTRY_PASS=[...]

	# Replace `[...]` with the registry email
	export REGISTRY_EMAIL=[...]

	kubectl create secret docker-registry regcred \
    	--docker-server=$REGISTRY_SERVER \
    	--docker-username=$REGISTRY_USER \
    	--docker-password=$REGISTRY_PASS \
    	--docker-email=$REGISTRY_EMAIL

# Pipline
Each app has it's own pipline.
The only deffrence was that we set the jenkins file to the app folder path.

pipline using scm -> my git repo -> path to jenkins file: ./producer-app/Jenkinsfile

# Create github pages for helm charts

# ArgoCD

$ kubectl create namespace argocd
$ kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
$ kubectl port-forward svc/argocd-server -n argocd 8080:443
$ kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

* connect to git repo using HTTPS