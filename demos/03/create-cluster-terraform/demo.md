## create-cluster-terraform

#### GIVEN:
  - A developer desktop with docker & git installed (AWS Cloud9)

#### WHEN:
  - I Install terraform on onto the Cloud9 instance
  - I create terraform templates

#### THEN:
  - I will get a VPC
  - I will get all required IAM Roles & security groups created via terraform
  - I will get an EKS cluster using the existing VPC created via terraform
  - I will get an Amazon Linux 2 Managed Nodegroup created via terraform
  - I will get a Fargate Profile created via terraform
  - I will get IRSA enabled on my EKS cluster
  - I will get the Cluster AutoScaler 'auto' installed on my EKS cluster

#### SO THAT:
  - I can test the Cluster Autoscaler
  - I can run my 2 x tier tv application on Fargate & EC2
  - I can use this cluster for all other 'CNCF' demos that require an EKS cluster

#### [Return to Main Readme](https://github.com/bwer432/mglab-share-eks#demos)

---------------------------------------------------------------
---------------------------------------------------------------
### REQUIRES
- 00-setup-cloud9

---------------------------------------------------------------
---------------------------------------------------------------
### DEMO


#### 0: Reset Cloud9 Instance environ from previous demo(s).
- Reset your region & AWS account variables in case you launched a new terminal session:
```
cd ~/environment/mglab-share-eks/demos/03/create-cluster-terraform/
export C9_REGION=$(curl --silent http://169.254.169.254/latest/dynamic/instance-identity/document |  grep region | awk -F '"' '{print$4}')
export C9_AWS_ACCT=$(curl -s http://169.254.169.254/latest/dynamic/instance-identity/document | grep accountId | awk -F '"' '{print$4}')
clear
echo $C9_REGION
echo $C9_AWS_ACCT
```

#### 1: Install the terraform cli onto the Cloud9 IDE instance.
- Verify that AWS Managed Temporary Credentials have been disabled.  If not set correctly, please re-visit [DEMO LINK: 00-setup-cloud9](demos/00-setup-cloud9/demo.md) and check steps.
    - If set correctly you will see the following in your response: `SAFE TO CREATE CLUSTER :)`, else the py script will either return _NOT SAFE_ or will fail to run.

- check the version of pip and install boto3
```
sudo su
curl -O https://bootstrap.pypa.io/pip/3.6/get-pip.py
python3 get-pip.py --user
export PATH=~/.local/bin:$PATH
pip install boto3
exit
```
```
export AWS_DEFAULT_REGION=$C9_REGION
python3 ./pre-reqs/check-c9-autocreds.py --region $C9_REGION --c9envname c9-eks-demo-XXXXX ** replace the Cloud9 environment name
```
- Install the 'terraform' CLI onto Cloud9
  - [DOC LINK: Installing Terraform](https://learn.hashicorp.com/tutorials/terraform/install-cli)
```
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://rpm.releases.hashicorp.com/AmazonLinux/hashicorp.repo
sudo yum -y install terraform
```

#### 2: Test the EKS tf templates by running a 'plan'
- Edit demos/03/create-cluster-terraform/artifacts/terraform/variables.tf, change the default region
- Init the required terraform modules to create the EKS cluster:
```
cd artifacts/terraform/
terraform init
```
- Run a terraform 'plan' to evaluate the expected output:
```
terraform plan
```

#### 3: Create the EKS Cluster
- Create the Cluster, wait for terraform to complete
```
terraform apply -auto-approve
```
- See the resources created by terraform:
```
terraform state list
```

#### 4: Generate a kubeconfig & Access the EKS Cluster with kubectl.
- Install kubectl & review your kubeconfig:  https://docs.aws.amazon.com/eks/latest/userguide/install-kubectl.html
```
curl -O https://s3.us-west-2.amazonaws.com/amazon-eks/1.23.16/2023-01-30/bin/linux/amd64/kubectl
curl -O https://s3.us-west-2.amazonaws.com/amazon-eks/1.23.16/2023-01-30/bin/linux/amd64/kubectl.sha256
echo "$(<kubectl.sha256) kubectl" | sha256sum --check
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```
- Confirm your current IAM user configured for use with the CLI:
```
export AWS_ACCESS_KEY_ID=$(cat ~/.aws/credentials | grep aws_access_key_id | awk '{print$3}')
export AWS_SECRET_ACCESS_KEY=$(cat ~/.aws/credentials | grep aws_secret_access_key | awk '{print$3}')
aws sts get-caller-identity
```
- Use aws cli to manually create/update a kubeconfig context to the cluster:
```
aws eks update-kubeconfig --name cluster-terraform --region $C9_REGION
kubectl config use-context arn:aws:eks:$C9_REGION:$C9_AWS_ACCT:cluster/cluster-terraform
kubectl config view --minify
```
- Confirm you now have access to run kubectl commands as well as eksctl commands:
```
kubectl get all -A
kubectl get nodes
```

#### 7a: Test the Cluster Autoscaler.

- Confirm the Cluster Autoscaler is functional, use _ctrl-c_ to exit:
```
kubectl logs deployment.app/cluster-autoscaler-aws-cluster-autoscaler -f -n kube-system
```
#### 7b: Deploy the Amazon EFS CSI Driver to your Amazon EKS cluster and verify that it works.
- https://docs.aws.amazon.com/eks/latest/userguide/eksctl.html (to install eksctl)
- https://helm.sh/docs/intro/install/ (to install helm 3.8.2)
```
wget https://get.helm.sh/helm-v3.8.2-linux-amd64.tar.gz
tar -zxvf helm-v3.8.2-linux-amd64.tar.gz
sudo mv linux-amd64/helm /usr/local/bin  
```
- https://docs.aws.amazon.com/eks/latest/userguide/efs-csi.html
- Add "AmazonEKS_EFS_CSI_Driver_Policy" to cluster-terraform-nodeRole 

#### 8: Deploy Wordpress app front end on Fargate & Mysql backend on managed Nodegroup.
- Deploy Wordpress front & back end workloads:
```
cat ../k8s/k8s-all-in-one-fargate.yaml  | sed "s/<REGION>/$C9_REGION/" | kubectl apply -f -
watch kubectl get pods -o wide -n wordpress-fargate
```
- Get all K8s nodes, you should see some additional `fargate` nodes:
```
kubectl get nodes -o wide
```
- Get the URL for our new app and test access in your browser:
```
echo "http://"$(kubectl get svc wordpress -n wordpress-fargate \
--output jsonpath='{.status.loadBalancer.ingress[0].hostname}')
```

---------------------------------------------------------------
---------------------------------------------------------------
### DEPENDENTS

- 04-CNCF*
- 05-CNCF*
- 06-CNCF*
- 07-CNCF*
- 08-CNCF*


---------------------------------------------------------------
---------------------------------------------------------------
### CLEANUP
- Do not cleanup if you plan to run any dependent demos
```
export C9_REGION=$(curl --silent http://169.254.169.254/latest/dynamic/instance-identity/document |  grep region | awk -F '"' '{print$4}')
export C9_AWS_ACCT=$(curl -s http://169.254.169.254/latest/dynamic/instance-identity/document | grep accountId | awk -F '"' '{print$4}')
export AWS_ACCESS_KEY_ID=$(cat ~/.aws/credentials | grep aws_access_key_id | awk '{print$3}')
export AWS_SECRET_ACCESS_KEY=$(cat ~/.aws/credentials | grep aws_secret_access_key | awk '{print$3}')
kubectl config use-context arn:aws:eks:$C9_REGION:$C9_AWS_ACCT:cluster/cluster-terraform
kubectl delete namespace wordpress-fargate --force
cd ~/environment/mglab-share-eks/demos/03/create-cluster-terraform/artifacts/terraform
terraform destroy -auto-approve
```
