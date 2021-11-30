# Lab2: EKS for CNF engineer (Application creation)

### Please keep in mind..
* You only have to use 'us-west-2' Region.

## 1. Create a Docker Image 
* Prepare docker environment (at your bastion host EC2 instance)
  ````
  sudo yum install docker -y
  sudo groupadd docker
  sudo usermod -aG docker ${USER}
  sudo service docker start
  ````
  Logout from Bastion and Login back (then you have to re-configure AWS Credential (copy "export AWS_..." again). 
  ````
  docker
  ````
* Create a directory for Dockerfile creation.
  ````
  mkdir dockerimage
  ````
* Create a Dockerfile file (file name to be DockerBuild with below contents)
  ````
  FROM ubuntu:focal
  MAINTAINER Young Jung
  ENV DEBIAN_FRONTEND noninteractive
  RUN apt-get update && apt-get install -y net-tools iputils-ping iproute2 python python3 pip vim
  RUN pip install boto3 requests
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
      * put your own repo name such as, "my-ecr-image" (The name must start with a letter and can only contain lowercase letters, numbers, hyphens, underscores, and forward slashes) -> "Create repository".
* Run below commands at Bastion host (where you created a docker image). 
  ````
  aws ecr get-login-password --region us-west-2 | docker login --username AWS --password-stdin 914125631499.dkr.ecr.us-west-2.amazonaws.com
  docker tag 6a01b2ad2952 914125631499.dkr.ecr.us-west-2.amazonaws.com/my-ecr-image:latest
  docker push 914125631499.dkr.ecr.us-west-2.amazonaws.com/my-ecr-image:latest
  ````
  *keep in mind that, you have to use your Account ID and Image ID of your docker (you can retrieve all information from above docker images command and from AWS ECR console)*

## 3. Create Mulus App using uploaded Docker image from the ECR
* Create a new directory at your bastion named, "k8s-environment", and `cd k8s-environment`.
* Install multus CNI.
  ````
  kubectl apply -f https://raw.githubusercontent.com/aws/amazon-vpc-cni-k8s/master/config/multus/v3.7.2-eksbuild.1/aws-k8s-multus.yaml
  ````

* Create below networkAttachmentDefinition (multus-ipvlan.yaml) and apply it to the cluster.

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
        "mode": "l2",
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
      image: 914125631499.dkr.ecr.us-west-2.amazonaws.com/my-ecr-image:latest
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
* Create Python script to update Pod IP to ENI IP (vi ipUpdate.py and paste below)
  ````
  import requests
  import boto3, json
  import sys
  from requests.packages.urllib3 import Retry

  ec2_client = boto3.client('ec2', region_name='us-west-2')

  def assign_ip():
      instance_id = get_instance_id()
      subnet_cidr = "10.0.4.0/24"

      response = ec2_client.describe_subnets(
          Filters=[
              {
                  'Name': 'cidr-block',
                  'Values': [
                      subnet_cidr,
                  ]
              },
          ]
      )

      for i in response['Subnets']:
          subnet_id = i['SubnetId']
          break

      response = ec2_client.describe_network_interfaces(
          Filters=[
              {
                  'Name': 'subnet-id',
                  'Values': [
                      subnet_id,
                  ]
              },
              {
                  'Name': 'attachment.instance-id',
                  'Values': [
                      instance_id,
                  ]
              }
          ]
      )

      for j in response['NetworkInterfaces']:
          network_interface_id = j['NetworkInterfaceId']
          break

      response = ec2_client.assign_private_ip_addresses(
          AllowReassignment=True,
          NetworkInterfaceId=network_interface_id,
          PrivateIpAddresses=[
              "10.0.4.70",
          ]
      )

  def get_instance_id():
      instance_identity_url = "http://169.254.169.254/latest/dynamic/instance-identity/document"
      session = requests.Session()
      retries = Retry(total=3, backoff_factor=0.3)
      metadata_adapter = requests.adapters.HTTPAdapter(max_retries=retries)
      session.mount("http://169.254.169.254/", metadata_adapter)
      try:
          r = requests.get(instance_identity_url, timeout=(2, 5))
      except (requests.exceptions.ConnectTimeout, requests.exceptions.ConnectionError) as err:
          print("Connection to AWS EC2 Metadata timed out: " + str(err.__class__.__name__))
          print("Is this an EC2 instance? Is the AWS metadata endpoint blocked? (http://169.254.169.254/)")
          sys.exit(1)
      response_json = r.json()
      instanceid = response_json.get("instanceId")
      return(instanceid)

  assign_ip()
  ````
  ````
  python3 ipUpdate.py
  ````
* Check EC2 Console, whether this multus ENI of the worker node has Pod IP address (10.0.4.70/24) as secondary IP address.  
* Try ping frome the Pod to subnet default GW (10.0.4.1), it should be successful if you followed well! Congratulation, you made it all course successful!
  

## 4. Clean up environment
* Go to CloudFormation and "Delete" worker-node stack you created. 
* After completion of above, also "Delete" the first infra stack. 

## What next? 
* I highly encourage you to walk through below blog post contents. You already have done the most of parts of setup process guided in the blog. You can easily build your own functional 4G EPC Core on EKS environment within 45 min after following steps similar to this course. https://aws.amazon.com/blogs/opensource/open-source-mobile-core-network-implementation-on-amazon-elastic-kubernetes-service/
* For the last of Day2 steps (ipUpdate.py), you can refer to https://github.com/aws-samples/eks-automated-ipmgmt-multus-pods to see how we can automate this process using sidecar or initController of the container. 


