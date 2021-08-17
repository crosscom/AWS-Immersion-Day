# Lab2: EKS for CNF engineer (Application creation)

### Please keep in mind..
* You only have to use 'us-east-1' Region not the one closest to you.

## 1. Create a Docker Image 
* Prepare docker environment (at your bastion host EC2 instance)
  ````
  sudo yum install docker -y
  sudo groupadd docker
  sudo usermod -aG docker ${USER}
  sudo service docker start
  ````
  (Logout/Login back)
  ````
  //sudo chown "$USER":"$USER" /home/"$USER"/.docker -R
  //sudo chmod g+rwx "$HOME/.docker" -R
  ````
* Create a directory for Dockerfile creation.
  ````
  mkdir dockerimage
  ````
* Create a Dockerfile file (file name to be DockerBuild with below contents)
  ````
  FROM ubuntu:focal
  MAINTAINER Young Jung <young.jung93@gmail.com>
  ENV DEBIAN_FRONTEND noninteractive
  RUN apt-get update && apt-get install -y net-tools iputils-ping iproute2 python python3 pip/
      pip install boto3 requests/
  WORKDIR /
  ````
* Build a Docker image.
  ```` 
  docker build .
  ````
* Verify Docker image created.
  ````
  docker images 
  ````
  
## 2. Upload Image to the ECR Repository
* Create an ECR repository in ECR Console. "ECR" -> "Repositories"->"Create repository".
      * put your own repo name such as, "my-ecr-image" (The name must start with a letter and can only contain lowercase letters, numbers, hyphens, underscores, and forward slashes). -> "Create repository".
* Run below commands at Bastion host (where you created a docker image). 
  ````
  aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 914125631499.dkr.ecr.us-east-1.amazonaws.com
  docker tag 6a01b2ad2952 914125631499.dkr.ecr.us-east-1.amazonaws.com/my-ecr-image:latest
  docker push 914125631499.dkr.ecr.us-east-1.amazonaws.com/my-ecr-image:latest
  ````
  *keep in mind that, you have to use your Account ID and Image ID of your docker (you can retrieve all information from above docker images command and from AWS ECR console)*

## 3. Create Mulus App using uploaded Docker image from the ECR
* Create a new directory at your bastion named, "k8s-environment", and `cd k8s-environment`.
* Install multus CNI.
  ````
  kubectl apply -f https://raw.githubusercontent.com/aws/amazon-vpc-cni-k8s/master/config/multus/v3.7.2-eksbuild.1/aws-k8s-multus.yaml
  ````
* Create below networkAttachementDefinition (multus-ipvlan.yaml) and apply it to the cluster.

  ````
  apiVersion: "k8s.cni.cncf.io/v1"
  kind: NetworkAttachmentDefinition
  metadata:
    name: ipvlan-conf
  spec:
    config: '{
        "cniVersion": "0.3.0",
        "type": "ipvlan",
        "master": "eth1",
        "mode": "l3",
        "ipam": {
          "type": "host-local",
          "subnet": "10.0.4.0/24",
          "rangeStart": "10.0.4.70",
          "rangeEnd": "10.0.4.80",
          "gateway": "10.0.4.1"
        }
      }'
  ````

  ````
  kubectl apply -f multus-ipvlan.yaml
  ````

* Deploy your docker using above network attachment. (create a file named, app-ipvlan.yaml)
  ````
  apiVersion: v1
  kind: Pod
  metadata:
    name: samplepod
    annotations:
            #k8s.v1.cni.cncf.io/networks: private100a-cni
      k8s.v1.cni.cncf.io/networks: ipvlan-conf
  spec:
    containers:
    - name: samplepod
      command: ["/bin/bash", "-c", "trap : TERM INT; sleep infinity & wait"]
      image: 914125631499.dkr.ecr.us-east-1.amazonaws.com/my-ecr-image:latest
  ````

  ````
  kubectl apply -f app-ipvlan.yaml
  kubectl describe pod samplepod
  ````
* Verify your Pod whether it has 2 interfaces (one for default K8s networking and the other for Multus interface (10.0.40.0/24)
  ````
  kubectl exec -it samplepod -- /bin/bash
  (in samplepod) ifconfig
  ````  
* 

## 4. Clean up environment
* Delete Node Group in EKS menu. 
* Go to CloudFormation and Delete ng1 stack. 
* After completion of above, delete the first infra stack. 
