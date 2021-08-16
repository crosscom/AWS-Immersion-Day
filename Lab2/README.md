# Lab2: EKS for CNF engineer (Application creation)

### Please keep in mind..
* You only have to use 'us-east-1' Region not the one closest to you.

## 1. Create your own Docker image

## 2. Upload it to ECR

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
