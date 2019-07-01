This project is developed to create a highly-available K8S cluster, setup CI/CD, monitoring and logging for guest-book application.

Prerequisites: 
# Install Ansible >=2.8 

# Create the compute in GCP
$ ansible-playbook gcp-compute.yml 
# Deploy a Production Ready Kubernetes Cluster
# Install dependencies from ``requirements.txt``
$ sudo pip install -r requirements.txt

# Copy ``inventory/sample`` as ``inventory/mycluster``
$ cp -rfp inventory/sample inventory/mycluster

# Update Ansible inventory file with inventory builder
$ declare -a IPS=(10.10.1.3 10.10.1.4 10.10.1.5)
$ CONFIG_FILE=inventory/mycluster/hosts.yml python3 contrib/inventory_builder/inventory.py ${IPS[@]}

# Review and change parameters under ``inventory/mycluster/group_vars``
$ cat inventory/mycluster/group_vars/all/all.yml
$ cat inventory/mycluster/group_vars/k8s-cluster/k8s-cluster.yml

# Deploy Kubespray with Ansible Playbook - run the playbook as root
# The option `--become` is required, as for example writing SSL keys in /etc/,
# installing packages and interacting with various systemd daemons.
# Without --become the playbook will fail to run!
$ ansible-playbook -i inventory/mycluster/hosts.yml --become --become-user=root cluster.yml
$ ansible-playbook -i inventory/mycluster/inventory.ini --user=app --become --become-user=root cluster.yml

# Install Jenkins for CI/CD
Installing Jenkins - https://medium.com/containerum/configuring-ci-cd-on-kubernetes-with-jenkins-89eab7234270

# Create a development namespace
a) Create a file called dev-namespace.yml and copy the below content in this file.
  {
  "apiVersion": "v1",
  "kind": "Namespace",
  "metadata": {
    "name": "development",
    "labels": {
      "name": "development"
    }
  }
}
b) Then run "kubectl create -f dev-namespace.yml"
c) To check whether the namespace is created or not, run below command.
  "kubectl get namespaces" should contain the development namespace
  NAME              STATUS   AGE
  default           Active   5d20h
  development       Active   6s
  kube-node-lease   Active   5d20h
  kube-public       Active   5d20h
  kube-system       Active   5d20h

 # Deploy Guest-Book App
 a) go to the location ~/guest-book/examples/guestbook
 b) Run the shell script "sh k8s-guest-book.sh", which will deploy the guest-book application in the K8S cluster

 # Install and configure Helm in Kubernetes
 $ wget https://get.helm.sh/helm-v2.14.1-linux-amd64.tar.gz && tar -xzvf helm-v2.14.1-linux-amd64.tar.gz && sudo mv linux-  amd64/helm /usr/local/bin && sudo -i -u jenkins && mkdir .kube ; $ touch .kube/config
  Run the command "helm init --history-max 200" to initialize the helm local repository and tiller(server-side component) 
 
 # Use Helm to deploy the application to Kubernetes cluster from CI server.
 a) Installing the Chart
      #Add the repository to your local environment:
      $ helm repo add my-repo https://ibm.github.io/helm101/

      To install the chart with the default release name:
      $ helm install jenkins-ci-cd-with-helm/charts/guestbook

      To install the chart with your preference of release name, for example, my-release:
      $ helm install my-repo/guestbook --name my-release

      Configuration
      The following tables lists the configurable parameters of the chart and their default values.

      Parameter           Description           Default
      image.repository    Image repository      guestbook
      image.tag           Image tag             v1
      image.pullPolicy    Image pull policy     Always
      service.type        Service type          LoadBalancer
      service.port        Service port          3000
      redis.slaveEnabled  Redis slave enabled   true
      redis.port          Redis port            6379

      Specify each parameter using the --set [key=value] argument to helm install.
      $ helm install my-repo/guestbook --set service.type=NodePort

 # Create a monitoring namespace in the cluster
  a) Create a file called dev-namespace.yml and copy the below content in this file.
  {
  "apiVersion": "v1",
  "kind": "Namespace",
  "metadata": {
    "name": "monitoring",
    "labels": {
      "name": "monitoring"
      }
    }
  }
  b) Then run "kubectl create -f dev-namespace.yml"
  c) To check whether the namespace is created or not, run below command.
    "kubectl get namespaces" should contain the development namespace
      NAME              STATUS   AGE
      default           Active   5d20h
      development       Active   6s
      kube-node-lease   Active   5d20h
      kube-public       Active   5d20h
      kube-system       Active   5d20h
      monitoring        Active   30s

  # Setup Prometheus (in monitoring namespace) for gathering host/container metrics along with health check status of the   application
    a) Monitoring Kubernetes workloads with Prometheus and Thanos
       What to Monitor?
          Monitoring Kubernetes should take into account all three layers mentioned above.

          Underlying VMs: to make sure the underlying virtual machines are healthy, the following metrics should be collected —

          Number of nodes
          Resource utilization per node (CPU, Memory, Disk, Network bandwidth)
          Node status (ready, not ready, etc.)
          Number of pods running per node
          Kubernetes Infrastructure: to make sure the Kubernetes infrastructure is healthy, the following metrics should be collected —

          Pods health — instances ready, status, restarts, age
          Deployments status — desired, current, up-to-date, available, age
          StatefulSets status
          CronJobs execution stats
          Pod resource utilization (CPU and Memory)
          Health checks
          Kubernetes Events
          API Server requests
          Etcd stats
          Mounted volumes stats
          User Applications: each application should expose its own metrics, based on its core functionality, however, there are metrics which are common to most of the applications, such as:

          HTTP requests (Total number, Latency, Response Code, etc.)
          Number of outgoing connections (e.g. database connections)
          Number of threads
          Collecting the metrics mentioned above, will allow you to build meaningful Alerts and Dashboards, we’ll cover that briefly below.

          Thanos
          Thanos is an open-source project, built as a set of components that can be composed into a highly available metric system with unlimited storage capacity, which can be added seamlessly on top of existing Prometheus deployments.

          Thanos leverages the Prometheus storage format to cost-efficiently store historical metric data in any object storage while retaining fast query latency. Additionally, it provides a global query view across all Prometheus installations.

          Thanos main components are:

          Sidecar: connects to Prometheus and exposes it for real time queries by the Query Gateway and/or upload its data to cloud storage for longer term usage
          Query Gateway: implements Prometheus’ API to aggregate data from the underlying components (such as Sidecar or Store Gateway)
          Store Gateway: exposes the content of a cloud storage
          Compactor: compacts and down-samples data stored in cloud storage
          Receiver: receives data from Prometheus’ remote-write WAL, exposes it and/or upload it to cloud storage
          Ruler: evaluates recording and alerting rules against data in Thanos for exposition and/or upload
          In this article, we will focus on the first three components.
        Download the helm chart from github - 
          $ git clone https://github.com/helm/charts.git
    b) Deploying Thanos -
        We’ll start with deploying Thanos Sidecar into our Kubernetes clusters, the same clusters we use for running our workloads and Prometheus and Grafana deployments.
        While there are many ways to install Prometheus, I prefer using Prometheus-Operator which gives you easy monitoring definitions for Kubernetes services and deployment and management of Prometheus instances.

        The easiest way to install Prometheus-Operator is using its Helm chart which has built-in support for high-availability, Thanos Sidecar injection and lots of pre-configured Alerts for monitoring the cluster VMs, Kubernetes infrastructure and your applications.

        Before we deploy the Thanos Sidecar, we need to have a Kubernetes Secret with details on how to connect to the cloud storage, for this demo, I will be using Microsoft Azure.

        Create a blob storage account—

        az storage account create --name <storage_name> --resource-group <resource_group> --location <location> --sku Standard_LRS --encryption blob
        Then, create a folder (aka container) for the metrics —

        az storage container create --account-name <storage_name> --name thanos
        Grab the storage keys —

        az storage account keys list -g <resource_group> -n <storage_name>
        Create a file for the storage settings (thanos-storage-config.yaml) —


        Create a Kubernetes Secret —

        kubectl -n monitoring create secret generic thanos-objstore-config --from-file=thanos.yaml=thanos-storage-config.yaml
        Create a values file (prometheus-operator-values.yaml) to override the default Prometheus-Operator settings —


        then deploy:

        helm install --namespace monitoring --name prometheus-operator stable/prometheus-operator -f prometheus-operator-values.yaml
        Now you should have a highly-available Prometheus running in your cluster, along with a Thanos Sidecar that uploads your metrics to Azure Blob Storage with infinite retention.

        To allow Thanos Store Gateway access to those Thanos Sidecars, we will need to expose them via an Ingress, I’m using Nginx Ingress Controller, but you can use any other Ingress Controller that supports gRPC (Envoy is probably the best option).

        For secured communication between the Thanos Store Gateway and the Thanos Sidecar we will use mutual-TLS, meaning the client will authenticate the server and vise-versa.

        Assuming you have .pfx file you can extract its Private Key, Public Key and CA using openssl —

        # public key
        openssl pkcs12 -in cert.pfx -nocerts -nodes | sed -ne '/-BEGIN PRIVATE KEY-/,/-END PRIVATE KEY-/p' > cert.key
        # private key
        openssl pkcs12 -in cert.pfx -clcerts -nokeys | sed -ne '/-BEGIN CERTIFICATE-/,/-END CERTIFICATE-/p' > cert.cer
        # certificate authority (CA)
        openssl pkcs12 -in cert.pfx -cacerts -nokeys -chain | sed -ne '/-BEGIN CERTIFICATE-/,/-END CERTIFICATE-/p' > cacerts.cer
        Create a two Kubernetes Secrets out of this —

        # a secret to be used for TLS termination
        kubectl create secret tls -n monitoring thanos-ingress-secret --key ./cert.key --cert ./cert.cer
        # a secret to be used for client authenticating using the same CA
        kubectl create secret generic -n monitoring thanos-ca-secret --from-file=ca.crt=./cacerts.cer
        Make sure you have a domain that resolves to your Kubernetes cluster and create two sub-domains to be used for routing to each Thaos SideCar:

        thanos-0.your.domain
        thanos-1.your.domain
        Now we can create the Ingress rules (replace the host value)—


        Now we have a secure way to access the Thanos Sidecars from outside of the clusters!

        Thanos Cluster

        In the Thanos diagram above, you can see I chose to deploy Thanos in a separate cluster, that’s because I wanted a dedicated cluster that can be easily re-created if needed, and give engineers access to it, without them needing to access real production clusters.

        To deploy Thanos components I chose to use this Helm chart (not official yet, but stay tuned, and wait for my PR to be merged).

        Create a thanos-values.yaml file to override the default chart settings —


        Since Thanos Store Gateway needs access to read from the blob storage, we re-create the storage secret in this cluster as well —

        kubectl -n thanos create secret generic thanos-objstore-config --from-file=thanos.yaml=thanos-storage-config.yaml
        To deploy this chart, we will use the same certificates we created earlier and inject them during as values —

        helm install --name thanos --namespace thanos ./thanos -f thanos-values.yaml --set-file query.tlsClient.cert=cert.cer --set-file query.tlsClient.key=cert.key --set-file query.tlsClient.ca=cacerts.cer --set-file store.tlsServer.cert=cert.cer --set-file store.tlsServer.key=cert.key --set-file store.tlsServer.ca=cacerts.cer
        This will install both Thanos Query Gateway and Thanos Storage Gateway, configuring them to use a secure channel.

        Validation

        To validate everything is working properly you can port-forward into the Thanos Query Gateway HTTP service using —

        kubectl -n thanos port-forward svc/thanos-query-http 8080:10902
        Then open your browser at http://localhost:8080 and you should see Thanos UI! —


        Note that I added three default dashboards to it, you can add your own dashboards as well (easiest way is using ConfigMap)

        Then deploy —

        helm install --name grafana --namespace thanos stable/grafana -f grafana-values.yaml
        Then again, port-forward —

        kubectl -n thanos port-forward svc/grafana 8080:80
        And… viola! you have completed the deployment of a highly-available monitoring solution, based on Prometheus, with long-term storage and a centralized view across multiple clusters!
        
# Create a dashboard using Grafana to help visualize the Node/Container/API Server etc. metrics from prometheus server. Optionally create a custom dashboard on Grafana

   Grafana:
      helm install --name my-release stable/grafana
      
    Once the Grafan dashboard is up and running, login to Grafana dashboard and configure Elasticsearch as Data Source and import the Kubernetes Cluster (Prometheus) into Grafana. So all the K8S metrics data which is stored in Prometheus will be displayed in Grafana now
# Setup log analysis using Elasticsearch, Fluentd (or Filebeat), Kibana.
    a) Create a headless service called "elasticsearch" that will define a DNS domain for the PODS.
    use Elasticsearch and Kibana for cluster logging, you should set the following environment variable as shown below when creating your cluster with kube-up.sh:
    $ KUBE_LOGGING_DESTINATION=elasticsearch
Now, when you create a cluster, a message will indicate that the Fluentd log collection daemons that run on each node will target Elasticsearch:
$ cluster/kube-up.sh

Project: kubernetes-satnam
Zone: us-central1-b
... calling kube-up
Project: kubernetes-satnam
Zone: us-central1-b
+++ Staging server tars to Google Storage: gs://kubernetes-staging-e6d0e81793/devel
+++ kubernetes-server-linux-amd64.tar.gz uploaded (sha1 = 6987c098277871b6d69623141276924ab687f89d)
+++ kubernetes-salt.tar.gz uploaded (sha1 = bdfc83ed6b60fa9e3bff9004b542cfc643464cd0)
Looking for already existing resources
Starting master and configuring firewalls
Created [https://www.googleapis.com/compute/v1/projects/kubernetes-satnam/zones/us-central1-b/disks/kubernetes-master-pd].
NAME                 ZONE          SIZE_GB TYPE   STATUS
kubernetes-master-pd us-central1-b 20      pd-ssd READY
Created [https://www.googleapis.com/compute/v1/projects/kubernetes-satnam/regions/us-central1/addresses/kubernetes-master-ip].
+++ Logging using Fluentd to elasticsearch

The per-node Fluentd pods, the Elasticsearch pods, and the Kibana pods should all be running in the kube-system namespace soon after the cluster comes to life.

kubectl get pods --namespace=kube-system
NAME                                           READY     STATUS    RESTARTS   AGE
elasticsearch-logging-v1-78nog                 1/1       Running   0          2h
elasticsearch-logging-v1-nj2nb                 1/1       Running   0          2h
fluentd-elasticsearch-kubernetes-node-5oq0     1/1       Running   0          2h
fluentd-elasticsearch-kubernetes-node-6896     1/1       Running   0          2h
fluentd-elasticsearch-kubernetes-node-l1ds     1/1       Running   0          2h
fluentd-elasticsearch-kubernetes-node-lz9j     1/1       Running   0          2h
kibana-logging-v1-bhpo8                        1/1       Running   0          2h
kube-dns-v3-7r1l9                              3/3       Running   0          2h
monitoring-heapster-v4-yl332                   1/1       Running   1          2h
monitoring-influx-grafana-v1-o79xf             2/2       Running   0          2h

The fluentd-elasticsearch pods gather logs from each node and send them to the elasticsearch-logging pods, which are part of a service named elasticsearch-logging. These Elasticsearch pods store the logs and expose them via a REST API. The kibana-logging pod provides a web UI for reading the logs stored in Elasticsearch, and is part of a service named kibana-logging.

The Elasticsearch and Kibana services are both in the kube-system namespace and are not directly exposed via a publicly reachable IP address. To reach them, follow the instructions for Accessing services running in a cluster.
We can now access the Kibana dashboard and run the queries to monitor the logs.


# Demonstrate Blue/Green and Canary deployment for the application (For example: Change the background color or font in the new version  etc.)    
a) For the first time we deployed the application the version of the app is gcr.io/google-samples/gb-frontend:v4
b) Once we change the content in the source code, run the below command
  $ helm upgrade myrelease repo/guestbook  --set=image.tag=v5
c) Now we can see the latest changes in the browser.

