## k8s-sloop

#### GIVEN:
  - A developer desktop with docker & git installed (AWS Cloud9)
  - An EKS cluster created via eksctl from demo 03/create-cluster-eksctl-existing-vpc-advanced or 03/create-cluster-terraform

#### WHEN:
  - I deploy the open-source Sloop to my cluster 

#### THEN:
  - I will be able to visualize (and administrate) my EKS cluster using the open-source [Sloop Kubernetes History Visualization](https://github.com/salesforce/sloop)

#### SO THAT:
  - I can see how to use EKS with open-source observability tooling

#### [Return to Main Readme](https://github.com/bwer432/mglab-share-eks#demos)

---------------------------------------------------------------
---------------------------------------------------------------
### REQUIRES
- 00-setup-cloud9
- 03/create-cluster-eksctl-existing-vpc-advanced or 03/create-cluster-terraform
- Helm CLI installed

---------------------------------------------------------------
---------------------------------------------------------------
### DEMO

#### 0: Reset Cloud9 Instance environ from previous demo(s).
- Reset your region & AWS account variables in case you launched a new terminal session:
```
cd ~/environment/mglab-share-eks/demos/05/k8s-sloop
export C9_REGION=$(curl --silent http://169.254.169.254/latest/dynamic/instance-identity/document |  grep region | awk -F '"' '{print$4}')
export C9_AWS_ACCT=$(curl -s http://169.254.169.254/latest/dynamic/instance-identity/document | grep accountId | awk -F '"' '{print$4}')
export AWS_ACCESS_KEY_ID=$(cat ~/.aws/credentials | grep aws_access_key_id | awk '{print$3}')
export AWS_SECRET_ACCESS_KEY=$(cat ~/.aws/credentials | grep aws_secret_access_key | awk '{print$3}')
clear
echo $C9_REGION
echo $C9_AWS_ACCT
```

#### 1a: Clone the Sloop source code, which includes a helm chart.
- The source code for Sloop includes a Helm chart. We can install this by having a copy of the chart locally.
```
git clone https://github.com/salesforce/sloop
```
- Edit sloop/values.yaml and set the parameter storageclass
```
storageClass: sloop-sc
```
#### 1b: Create a storageclass.yaml in sloop/templates folder with the following content.
```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: sloop-sc
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer
```

#### 2a: Complete the following pre-requisite steps 

- [Creating the Amazon EBS CSI driver IAM role for service accounts](https://docs.aws.amazon.com/eks/latest/userguide/csi-iam-role.html)

- [Managing the Amazon EBS CSI driver as an Amazon EKS add-on](https://docs.aws.amazon.com/eks/latest/userguide/managing-ebs-csi.html)

#### 2b: Perform a Helm install
- This demo will create a service account that grants cluster-admin permissions to the Dashboard.  This is the highest levels of privilege, but will allow you to both visualize and manage the cluster.
```
cd sloop/helm
helm upgrade --install mysloop sloop
```

#### 3: Connect to Sloop
- Forward traffic from local port 8080 to the Sloop Service.  You may want to wait a minute or two to ensure the Sloop service is up and running
```
kubectl port-forward svc/sloop 8080:80
```
- Then, use "Preview Running Application" on Cloud9



#### 5: Observe cluster activity
- Perform some activities to generate changes in the cluster state - launching deployments, deleting deployments or services, etc.
- Try making some changes to objects
- Observe that Sloop is now recording history of all k8s objects, and provides a diff view where changes have been made to existing objects.
- Enter query parameters on the left, and click Submit Query button
- Click on any resource in the timeline and select a "left payload" and a "right payload" to observe the differences.


---------------------------------------------------------------
---------------------------------------------------------------
### DEPENDENTS

---------------------------------------------------------------
---------------------------------------------------------------
### CLEANUP
- Do not cleanup if you plan to run any dependent demos, otherwise uninstall the Helm release.
```
helm uninstall mysloop
```
