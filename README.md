# k8s_webserver

######## Have used Kubernetes to acheive highly available WebServer which can host my "Hello World" container.

We Require 3 things to have a highly scable and available Web Page:
1. Docker Image which has NGINX to host my application
2. Kubernetes Cluster: Have used kops to build production scale Kuberneted Cluster
3. Kubernetes Files to Deploy and Access the Application


## 1. Docker Image 
Clone the git repositor, build Docker Image by executing the docker build script with a version number

``` ./docker_build $VERSION_NUMBER ```

Above command will execute and upload the image to my docker repository 
Docker Repository Location: ```arokiagetz/k8s-web-server:$VERSION_NUMBER```

This image can be pulled using the command: ```docker pull arokiagetz/k8s-web-server:v1```

Now the image is ready which can host my web-page which can be pulled into my Kubernetes Cluster.

## 2. Kubernetes Cluster
Assuming this Kubernetes cluster is to be run in AWS Cloud platform, we need to build one for if it dont have Kubernetes Cluster available to deploy.

I have used kops which can build Production Scale Kubernetes Cluster.

Pre-Requisites:
AWS command line login credentials
S3 Bucket: I named it as 'k8s-web-server'
kubernetes cluster name ending with k8s.local: I name it as 'k8s-web-server.k8s.local'
kops installed in your machine
kubectl installed in your machine

Steps to setup Kubernetes Cluster using kops:
If you need more details, please follow official [kops documentation](https://github.com/kubernetes/kops)
Run the below steps in your local terminal to set ENVIRONMENT_VARIABLE


```
# Set the cluster name as env_variable NAME
export NAME=k8s-web-server.k8s.local

# Create S3 bucke with name k8s-web-server
aws s3api create-bucket --bucket k8s-web-server1 --region eu-west-2 --create-bucket-configuration LocationConstraint=eu-west-2

# Set the Cluser State as env_variable KOPS_STATE_STORE which has the S3 bucket name
export KOPS_STATE_STORE=s3://k8s-web-server
```


1. First prepare the k8s(kubernetes) cluster, below command will initialize the cluster to run the nodes on 3 Availability Zones and 1 Master Node in the region eu-west

```kops create cluster --zones eu-west-2a,eu-west-2b,eu-west-2c  --master-zones eu-west-2a $NAME ```

2. Create Secrey Key to access the nodes(if necessary):
```
# Create a public-private key which is only for this cluster
ssh-keygen -b 2048 -t rsa -f /Users/<user-dir>/.ssh/<local-directory>/id_rsa ```

#Add the public to the kops cluster
kops create secret --name ${NAME} sshpublickey admin -i ~/.ssh/<local-directory>/id_rsa.pub 
```

3. Edit the k8s cluster according to the needs
```

kops get ig --name $NAME


Output of above command:
NAME			ROLE	MACHINETYPE	MIN	MAX	ZONES
master-eu-west-2a	Master	t2.micro	1	1	eu-west-2a
nodes			Node	t2.micro	2	2	eu-west-2a,eu-west-2b,eu-west-2c

#Above Cluster will be created by kops, use below command to edit the cluser

kops use kops edit ig --name $NAME

```

4. Create the cluser in AWS:

 # Note: Below command would create the cluster and would cose money based on the usage

```
kops update cluster --name $NAME --yes
```

5. Validate the cluster created

```
kops validate cluster --name $NAME

```
And wait until see the below output

```
Using cluster from kubectl context: k8s-web-server.k8s.local

Validating cluster k8s-web-server.k8s.local

INSTANCE GROUPS
NAME			ROLE	MACHINETYPE	MIN	MAX	SUBNETS
master-eu-west-2a	Master	t2.micro	1	1	eu-west-2a
nodes			Node	t2.micro	2	2	eu-west-2a,eu-west-2b,eu-west-2c

NODE STATUS
NAME						ROLE	READY
ip-172-20-40-130.eu-west-2.compute.internal	master	True
ip-172-20-62-175.eu-west-2.compute.internal	node	True
ip-172-20-96-97.eu-west-2.compute.internal	node	True

Your cluster k8s-web-server.k8s.local is ready
```
## Your cluster k8s-web-server.k8s.local is ready 

## 3. Kubernetes Deployment

Kubernetes related files are already available in this repository under the directory names "kubernetes"

Run the below command to deploy our [image](https://hub.docker.com/r/arokiagetz/k8s-web-server) into K8s Cluster.

```
#To deploy the pod run the below command
#In the yaml file, replicas given are 2, it can be increased to have more available pods
#Pod with image arokiagetz/k8s-web-server:v1 is used to deploy
#Please note that minReadySeconds is given as 10 secs so Pod would be ready in atleast 10 secs
kubectl apply -f web-server-deployment.yaml

#To deploy the service. Since the deployment is in Cloud Platform, service is exposed Load Balancer.
#All the Pods with matching labels "app: web-server" will be accessed by the below service
kubectl apply -f web-server-services-cloud.yaml

###########
Below steps if you run the k8s cluster locally

#Exposing the nodePort 30080, you should access the http://cluster-ip:30080 to access the service.
kubectl apply -f web-server-services-local.yaml
```
## YOU ARE ALMOST THERE

Get the Load Balancer Link to access your Web-Page:

```kubectl get services

OUTPUT:
NAME                 TYPE           CLUSTER-IP      EXTERNAL-IP                                                              PORT(S)        AGE
kubernetes           ClusterIP      100.64.0.1      <none>                                                                   443/TCP        1h
web-server-service   LoadBalancer   100.64.69.160   ab365290f702511e9a3670624a8928ee-202673565.eu-west-2.elb.amazonaws.com   80:32273/TCP   57m
```
###### External IP(ab365290f702511e9a3670624a8928ee-202673565.eu-west-2.elb.amazonaws.com)is the LB Link to access your application.
Please allow some for the LB to connect to the instances and Pod to be available.
 

Finally, we have our web-page available in AWS Kubernetes Cluster which will be always available and scalable according to business requirement. 

Below is the screenshot when accessed from AWS LB.


![LB](https://github.com/arokiagetz/k8s_webserver/blob/master/img2.png)


Below is the screenshot when accessed from local k8s ip on nodePort 30080.


![Local](https://github.com/arokiagetz/k8s_webserver/blob/master/img1.png)

## Do not forget to destroy your k8s cluster, not to get charged by AWS. Run the below command to stop and delete your cluster

```
#Below command will stop and delete the cluster
kops delete cluster --name $NAME --yes

#Wait until you see the below message 
Deleted cluster: "k8s-web-server.k8s.local"
```




