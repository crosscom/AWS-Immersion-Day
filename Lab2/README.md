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
  sudo chown "$USER":"$USER" /home/"$USER"/.docker -R
  sudo chmod g+rwx "$HOME/.docker" -R
  ````
* Create a directory for Dockerbuild file.
  ````
  mkdir dockerimage
  ````
* Create a DockerBuild file (file name to be DockerBuild with below contents)
  ````
  FROM ubuntu:focal
  MAINTAINER Young Jung <young.jung93@gmail.com>
  ENV DEBIAN_FRONTEND noninteractive
  RUN apt-get update && apt-get install -y net-tools iputils-ping iproute2   
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
* Create an ECR repository in ECR Console. "Create Repository" -> 
* Run below commands at Bastion host (where you created a docker image). 
  ````
  aws ecr get-login-password --region us-west-2 | docker login --username AWS --password-stdin 340256550793.dkr.ecr.us-west-2.amazonaws.com
  docker tag d2dfc4b1406a 340256550793.dkr.ecr.us-west-2.amazonaws.com/my-repository:tag
  docker push 340256550793.dkr.ecr.us-west-2.amazonaws.com/my-repository:tag
  ````

## 3. Create Mulus App using uploaded Docker image from the ECR

* Let's go to Bastion host where we can run kubectl. 
* Install multus CNI.
  ````
  git clone https://github.com/intel/multus-cni.git 
  kubectl apply -f ~/multus-cni/images/multus-daemonset.yml
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

* Deploy dummy app using above network attachment. (app-ipvlan.yaml)
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
      image: nginx
  ````

  ````
  kubectl apply -f app-ipvlan.yaml
  kubectl describe pod samplepod
  kubectl exec -it samplepod -- /bin/bash
  root@samplepod:/# apt-get update && apt-get install -y net-tools iputils-ping iproute2
  ````

## 4. Clean up environment
* Delete Node Group in EKS menu. 
* Go to CloudFormation and Delete ng1 stack. 
* After completion of above, delete the first infra stack. 
