# AWS-Immersion-Day

## 1. Please download CFN template for this session (nokia-immersion-infra.yaml)

* This creates VPC, public/private subnets, subnet route tables, IGW, NAT-GW, Security Groups, EKS Cluster and Bastion Instance

## 2. Log in to AWS Event Engine 

* https://dashboard.eventengine.run/dashboard.

* Please put Event Hash (will be given at the session). 

* Please use OTP authentication with your company email.

  ![Screen Shot 2021-05-25 at 10.48.10 AM](/Users/jungy/Desktop/Screen Shot 2021-05-25 at 10.48.10 AM.png)

* Click "Set Team Name" (Team is equivalent to the account)

* 1. Please put TEAM name to be your name.  

* Click "SSH key" 

* 1. Download **ee-default-keypair** to your PC (just in case, copy key material to notepad as well).

* Click "AWS Console"

* 1. Copy credentials (export AWS_DEFAULT_REGION=..) 
  2. Click "Open AWS Console".

## 3. Create Environment with CloudFormation

* Type "CloudFormation" at search service section and go to CloudFormation.
* Create Stack -> upload a template file -> Choose file (select downloaded "nokia-immersion-infra.yaml").



## 4. Login to Bastion Host 

* (General)

  * We can use EC2 Instance Connect to login to EC2 instance.
  * EC2->Instances->"connect" (right top corner of screen). 
  * click "connect"

* (MAC user) Log in from your laptop

  * Let's use key pair we downloaded to access to the instance.

  ````
  chmod 600 ee-default-keypair.pem
  ssh-add ee-default-keypair.pem
  ssh -A ec2-user@54.208.182.244
  ````

  * ​	Copy AWS credentials (AWS_DEFAULT_REGION, AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY)

  ````
  export AWS_DEFAULT_REGION=us-east-1
  export AWS_ACCESS_KEY_ID=ASIA..
  export AWS_SECRET_ACCESS_KEY=4wyDA..
  export AWS_SESSION_TOKEN=IQo...
  ````

  * ​	Try whether AWS confidential is already configured well

    ````
    aws sts get-caller-identity
    {
      "Account": "400166681888", 
      "UserId": "AROAV2K6K7UQPEU2EMBY3:MasterKey", 
      "Arn": "arn:aws:sts::400166681888:assumed-role/TeamRole/MasterKey"
    }
    ````

* (Window user) Log in from your laptop
  * https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/putty.html

## 5. Make bastion to be for Kubectl client

* Download kubectl. 

  ````
  curl -o kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.19.6/2021-01-05/bin/linux/amd64/kubectl
  `curl -o kubectl.sha256 https://amazon-eks.s3.us-west-2.amazonaws.com/1.19.6/2021-01-05/bin/linux/``amd64``/kubectl.sha256`
  `openssl sha1 -sha256 kubectl`
  `chmod +x ./kubectl`
  `mkdir -p $HOME/bin && cp ./kubectl $HOME/bin/kubectl && export PATH=$PATH:$HOME/bin`
  `echo 'export PATH=$PATH:$HOME/bin' >> ~/.bashrc`
  `kubectl version —short —client`
  ````

* Check your name of EKS cluster (from CloudFormation output or EKS console (service search -> EKS))

* Config kubeconfig with EKS CLI

  ````
  aws eks update-kubeconfig --name=eks-my-first-stack
  ````

* Verify kubectl command

  ````
  kubectl get svc
  NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
  kubernetes   ClusterIP   172.20.0.1   <none>        443/TCP   31m
  ````

* Verify it from AWS CLI

  ````
  aws eks describe-cluster --name eks-my-first-stack
  ````

  

