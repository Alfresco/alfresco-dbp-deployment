# Alfresco Digital Business Platform Deployment

## Prerequisites

### Kubernetes Cluster

See the Anaxes Shipyard documentation on [running a cluster](https://github.com/Alfresco/alfresco-anaxes-shipyard/blob/master/SECRETS.md).

Note the DBP resource requirements:
* Minikube: At least 12 gigs of memory, i.e.:
```bash
minikube start --memory 12000
```
* AWS: A VPC and cluster with 5 nodes. Each node should be a m4.xlarge EC2 instance.

### Helm Tiller

Initialize the Helm Tiller:
```bash
helm init
```

### K8s Cluster Namespace

As mentioned as part of the Anaxes Shipyard guidelines, you should deploy into a separate namespace in the cluster to avoid conflicts:
```bash
export DESIREDNAMESPACE=example
kubectl create namespace $DESIREDNAMESPACE
```

This environment variable will be used in the deployment steps.

### Docker Registry Pull Secrets

See the Anaxes Shipyard documentation on [secrets](https://github.com/Alfresco/alfresco-anaxes-shipyard/blob/master/SECRETS.md).

Be sure to use the same namespace as above.

## Deployment

### 1. EFS Storage (**NOTE! ONLY FOR AWS!**)

Create a EFS storage on AWS and make sure it is in the same VPC as your cluster. Make sure you open inbound traffic in the security group to allow NFS traffic. Save the name of the server ex: 
```bash
export NFSSERVER=fs-d660549f.efs.us-east-1.amazonaws.com
```

### 2. Deploy the infrastructure charts:
```bash
cd charts/incubator

#ON MINIKUBE
helm install alfresco-dbp-infrastructure --namespace $DESIREDNAMESPACE

#ON AWS
helm install alfresco-dbp-infrastructure --namespace $DESIREDNAMESPACE --set persistence.volumeEnv=aws --set persistence.nfs.server="$NFSSERVER"
```

### 3. Get the infrastructure release name from the previous command and set it as a variable:
```bash
export INFRARELEASE=enervated-deer
```

### 4. Wait for the infrastructure release to get deployed.  When checking status all your pods should be AVAILABLE 1/1):
```bash
helm status $INFRARELEASE
```

### 5. Get the nginx-ingress-controller port for the infrastructure (**NOTE! ONLY FOR MINIKUBE**):
```bash
export INFRAPORT=$(kubectl get service $INFRARELEASE-nginx-ingress-controller -o jsonpath={.spec.ports[0].nodePort})
```

### 6. Get Minikube or ELB IP and set it as a variable for future use:

```bash
#ON MINIKUBE
export ELBADDRESS=$(minikube ip)

#ON AWS
export ELBADDRESS=$(kubectl get services $INFRARELEASE-nginx-ingress-controller --namespace=$DESIREDNAMESPACE -o jsonpath={.status.loadBalancer.ingress[0].hostname})
```

### 7. Enable or disable the components you want up from the dbp deployment.
To do this open the alfresco-dbp/values.yaml file and set to true/false the components you want enabled.

Example:
alfresco-content-services:
  enabled: true

### 8. Deploy the DBP

```bash
#On MINIKUBE
helm install alfresco-dbp --set alfresco-activiti-cloud-gateway.keycloakURL="http://$ELBADDRESS:$INFRAPORT/auth/" --set alfresco-activiti-cloud-gateway.eurekaURL="http://$ELBADDRESS:$INFRAPORT/registry/" --set alfresco-activiti-cloud-gateway.rabbitmqReleaseName="$INFRARELEASE-rabbitmq" --namespace=$DESIREDNAMESPACE

#On AWS
helm install alfresco-dbp --set alfresco-activiti-cloud-gateway.keycloakURL="http://$ELBADDRESS/auth/" --set alfresco-activiti-cloud-gateway.eurekaURL="http://$ELBADDRESS/registry/" --set alfresco-activiti-cloud-gateway.rabbitmqReleaseName="$INFRARELEASE-rabbitmq" --namespace=$DESIREDNAMESPACE
```

### 9. Get the DBP release name from the previous command and set it as a variable:
```bash
export DBPRELEASE=littering-lizzard
```

### 10. Checkout the status of your DBP deployment:

```bash
helm status $INFRARELEASE
helm status $DBPRELEASE
```

### 11. Teardown:

```bash
helm delete $INFRARELEASE
helm delete $DBPRELEASE
kubectl delete namespace $DESIREDNAMESPACE
```
Depending on your cluster type you should be able to also delete it if you want.
For minikube you can just run 
```bash
minikube delete
```
For more information on running and tearing down k8s environemnts, follow this [guide](https://github.com/Alfresco/alfresco-anaxes-shipyard/blob/master/docs/running-a-cluster.md).