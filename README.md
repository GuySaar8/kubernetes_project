# kubernetes_project

* TASK: clone https://github.com/avielb/rmqp-example repo.   
    Create helm release from the docker-compose files.
    Create full ci cd
    Install RabbitMQ with expoter

* CI - 
    Jenkins using git push notification from github.
    Each app has it's own Jenkins file that create and push docker image on kubernetes cluster using kaniko.
* CD - 
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
the jenkins files looks for builder.yaml and pass kaniko the vlues for the build and push.  

# Github Pages 
    create a new orpahn branch by the name - gh-pages
    $ git checkout --orphan gh-pages

    * we can create index.html as "home page" for out chart.

# CI for github pages helm charts Repo

this CI will run on github action and will when new PR is sumitted.

Kind Action - A GitHub Action for Kubernetes IN Docker - local clusters for testing Kubernetes using kubernetes-sigs/kind.

Chart-testing Action - A GitHub Action for installing the helm/chart-testing CLI tool. Will run lint and deployment testing for our helm chart. 

Chart-releaser Action - A GitHub action to turn a GitHub project into a self-hosted Helm chart repo, using helm/chart-releaser CLI tool. 

workflows: 
1. lint test -
    on PR check out the repo
    Run testing with lint
    createing kind cluster and running the chart and test if the installation was successful.

    fast failing - if lint testing was failed it wont try and deploy our chart in the cluster.

2. release -
    we check it our and configure the config user and email becuase we will push code back in to the github pages branch.
    installs helm
    add dependent chart repo if neccery
    and does the helm chart release.

    the release uses a github token by the name CR_TOKEN

* set token:
    go to Settings -> Developer Settings -> Personal Access Tokens -> Generate Token -> Copy Token 
    go to your repository settings -> Secrets -> New Repository Secret -> name the secret CR_TOKEN and paste the Token Value.

# ArgoCD

$ kubectl create namespace argocd
$ kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
$ kubectl port-forward svc/argocd-server -n argocd 8080:443
$ kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

* connect to git repo using ssh
set github token with ssh-public
    - settings -> ssh and new gpg keys -> new ssh key
set argo with ssh-private
    - settings -> repositories -> connect repo using ssh

run argo apllication:
    $ kubectl apply -f app-of-apps.yaml
the application will is set up to check the manifests in the overlays.
we have 3 overlays:
    1. rabbitmq with metrics enabled
    2. producer chart - installed from our helm repo\github pages
    3. consumer chart - installed from our helm repo\github pages

# install istio
    helm repo add istio https://istio-release.storage.googleapis.com/charts
    helm repo update
    kubectl create namespace istio-system
    helm install istio-base istio/base -n istio-system
    helm install istiod istio/istiod -n istio-system --wait
    kubectl create namespace istio-ingress
    kubectl label namespace istio-ingress istio-injection=enabled
    helm install istio-ingress istio/gateway -n istio-ingress --wait

# install flagger with metrics
helm repo add flagger https://flagger.app
kubectl apply -f https://raw.githubusercontent.com/fluxcd/flagger/main/artifacts/flagger/crd.yaml
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.12/samples/addons/prometheus.yaml
helm upgrade -i flagger flagger/flagger --namespace=istio-system --set crd.create=false --set meshProvider=istio --set metricsServer=http://prometheus.istio-system:9090
kubectl create ns test
kubectl label namespace test istio-injection=enabled
kubectl apply -k https://github.com/fluxcd/flagger//kustomize/podinfo?ref=main
kubectl apply -k https://github.com/fluxcd/flagger//kustomize/tester?ref=main

'''template-metric.yaml
apiVersion: flagger.app/v1beta1
kind: MetricTemplate
metadata:
  name: error-rate
  namespace: istio-system
spec:
  provider:
    address: http://prometheus.istio-system.svc.cluster.local:9090
    type: prometheus
  query: |
    100 -
    (sum(rate(istio_requests_total{destination_service="podinfo-canary.test.svc.cluster.local",  response_code=~"5.*"}[30s]))
    /
    sum(rate(istio_requests_total{destination_service="podinfo-canary.test.svc.cluster.local"}[30s]))
    * 100
    )

'''

'''hpa.yaml
apiVersion: autoscaling/v2beta2
kind: HorizontalPodAutoscaler
metadata:
  name: podinfo
  namespace: test
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: podinfo
  minReplicas: 2
  maxReplicas: 4
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          # scale up if usage is above
          # 99% of the requested CPU (100m)
          averageUtilization: 99apiVersion: flagger.app/v1beta1
'''

'''canary.yaml
apiVersion: autoscaling/v2beta2
kind: HorizontalPodAutoscaler
metadata:
  name: podinfo
  namespace: test
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: podinfo
  minReplicas: 2
  maxReplicas: 4
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          # scale up if usage is above
          # 99% of the requested CPU (100m)
          averageUtilization: 99apiVersion: flagger.app/v1beta1
STEP 9: Deploy the Canary for podinfo application
kind: Canary
metadata:
  name: podinfo
  namespace: test
spec:
  # deployment reference
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: podinfo
  service:
    # service port number
    port: 9898
    # Istio traffic policy (optional)
    trafficPolicy:
      tls:
        # use ISTIO_MUTUAL when mTLS is enabled
        mode: DISABLE
  analysis:
    # schedule interval (default 60s)
    interval: 10s
    # max number of failed metric checks before rollback
    threshold: 5
    # max traffic percentage routed to canary
    # percentage (0-100)
    maxWeight: 100
    # canary increment step
    # percentage (0-100)
    stepWeight: 5
    metrics:
      - name: "500 percentage"
        templateRef:
          name: error-rate
          namespace: istio-system
        thresholdRange:
          min: 99
        interval: 15s
'''
