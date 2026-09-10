# mca-devops
Devops solution

1. INTRODUCTION
This project is about deploying a 3-components applicaiton :

- front : a Node application to interect with the backend insert and list users
- back : Java/Maven app with REST API
- database : Postgres database

My target architecture is :

<img src="images/image.jpg.png" alt="EKS Architecture" width="800">

I choosed to deploy the solution on AWS using EKS (Elastic Kubernetes Service)
For docker image registry, i use : https://hub.docker.com.

The CI/CD workflow is like this  :

<img src="images/ci-cd.jpg" alt="CI CD diagram" width="800">
2. PREREQUISITES
Jenkins must have these tools installed :

docker
aws
kubectl
eksctl
ansible
node
npm
java
mvn

Ansible requires the Kubernetes collection and Python dependencies:

ansible-galaxy collection install kubernetes.core

pip install kubernetes boto3 botocore

AWS credentials must also be available to Jenkins.

Docker Hub credentials should be stored in Jenkins Credentials rather


3. BUILD steps

I build all the components locally before depolying. The Front and the back as well.

Front : npm install; ng build
Back : mvn clean package

To build and push the docker images :
mkeita/mca-devops-frontend : docker build -t mketia/mca-devops-frontend .
mkeita/mca-devops-backend : docker build -t mkeita/mca-devops-backend .

docker push mkeita/mca-devops-frontend:1.0.0
docker push mkeita/mca-devops-backend:1.0.0


4. DEPLOYMENT steps
4.1. EKS
I choosed to deploy a EKS cluster with 2 nodes on eu-west-3 (Paris) that should be enough.
Added a namespace and EBS CSI driver.

5. ANSIBLE
I decided to deploy on EKS as well.
For the inventory  Kubernetes communicates with nodes so need to fill the inventory.
For the versionning, i increment with the number of build : 1.0.1, 1.0.2 ...

mkeiita/mca-devops-frontend:1.0.2 ...

6. JENKINS
for the Jenkins pipeline, i first store the docker huhb credentials  on Jenkins Credentials, the AWS AK SK ...
and all the env vars.

7. REPOSITORY
 For deployments purposes, i separate Jenkins and ANsible.
I choose to have the same manifests files both in Ansible folder and on k8s folder for the 2 types of deployment.
With Ansible or just Jenkins on my cluster mca-devops-cluster.

8. DECISIONS and ISSUES
I decided to deploy everything on AWS.
I also added some firewall rules to let internet connect to the frontend port  on Security Groups.
I decided to use one ALB.
I decided to use PVC to persist the data from Postgres pods.

I faced somes issues : 
when deploying the Frontend on Dockerfile  when copying /app/dist --> /usr/share/...html : i had to find the real path 
i faced also issue when creating the PVC. I had to add OCI driver on the kube-system node.
ANd many other small isuues for installing tools.

9. SCREENSHOTS
9.1. Frontend with data
<img src="images/screenshot.png" alt="FrontEnd" width="800">
9.2. Docker hub
<img src="images/screenshot2.png" alt="Docker hub" width="800">
9.3. Frontend on AWS
<img src="images/screenshot3.png" alt="Frontend on AWS" width="800">
9.4. CLuster AWS
<img src="images/screenshot4.png" alt="EKS Cluster" width="800">
