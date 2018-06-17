# Jenkins Guide for Docker EE - Kubernetes

## Deploy your Master

Fortunatley for Kubernetes, as a service account is mounted into every pod containing the relevant certificates to communicate with the API server. We do not need to Build a custom Docker Image, and can use the one straight from UpStream! Yay!

**Deploy a Namespace and a SA**

Firstly we will create a kubernetes namespace for Jenkins, and also create a Service account. 

```
$ cd jenkins-master/kubernetes
$ kubectl apply -f jenkins-ns.yaml
```

**Configure RBAC for your Service Account**

Due to a limitation in Docker EE 2.0, we have to manually configure permissions for the Jenkins SA rather than using Kubernetes RBAC yaml. This should be changing soon. 

Login to your UCP as an Administrator > User Management > Grants. Click New Grant in the top corner, and create a Full Control. Using the Jenkins Namespace as your Resource Set, Full Control as your Role, and the Jenkins-Servie-Account as your subject. 

![SA RBAC](/docs/images/sarbac.png?raw=true "SA RBAC")

**Persistent Storage (Optional)**

As part of my Jenkins deployment I plan to use an NFS volume to sit behind my Jenkins master. This will provide my master an element of persistence if it has to migrate around nodes in my cluster, or as I upgrade the pod over time.

In the example .yaml I have defined a 5Gi volume for use for Jenkins, and have created the relevant Physical Volume (PV) and Physical Volume Claim (PVC) within Kubernetes. You would need to adjust this yaml for your NFS server.

```
$ cd jenkins-master/kubernetes
$ kubectl apply -f jenkins-pv.yaml
```

**Deploy the Master**

Now deploy the Jenkins Master. For my instance, I will deploy this as a Kubernetes Deployment of 1 Replica, I will apply resource contraints and mount my NFS Volume. This .yaml made need to be customized for your environment. 

Also within this yaml, I will deploy a kubernetes service, exposing the master on ports 80 (For the web UI) and port 5000 (for slave communication back to the master). To access the UI I have not defined a NodePort or a Loadbalancer. Instead I plan to user a Kubernetes Ingress Controller, this has already been deployed on the cluster and is an Optional next step. 

```
$ cd jenkins-master/kubernetes
$ kubectl apply -f jenkins.yaml
```

## Configure Your Master

Browse to the Jenkins Master in Chrome, you will need to retrieve the default log in password from the Jenkins Master logs.

```
$ kubectl -n jenkins logs jenkins-master-7c898bd487-xhcbc
```

Configure a new password, and install Jenkins with the default plugins. 

**Configure a Cloud Provider**

Firstly we need to install the Kubernetes Plugin. 

Manage Jenkins > Manage Plugins > Available. Search for Kubernetes.

![Kube Plugin](/docs/images/kubeplugin.png?raw=true "Kube Plugin")

Now we can go into the System and Configure the plugin. 

Firstly we will configure the Kubernetes SA credentials within Jenkins. This give us a user to communicate with the Kubernetes API Server.

Credentials > System > Global Credentials > Add Credentials

![Add SA](/docs/images/addsa.png?raw=true "Add SA")

Next we can Add a new cloud into Jenkins.

Manage Jenkins > Configure System. Scroll down to Cloud. 

There isn't many fields we will need to configure here. Firstly name your cloud, for the URL I will use the Kubernetes API Server's internal IP which by default is 10.96.0.1. For Credentials I will select "Secret Text", which should refer to our Service Account Credential we configured previously. If you click test connection, it should all work :) 

Note I have also configured a jenkins URL field. This is required so that the slaves, use the internal kubernetes DNS service to route back to our master. The field "jenkins-master" refers to our kubernetes service name. 

![Kube Cloud](/docs/images/newkubecloud.png?raw=true "Kube Cloud")

At this point the Master is done. And is ready to configure some build pipelines. 

As the pipelines are consistent with Swarm, they can be found here: [Pipelines](docs/pipelines.md)