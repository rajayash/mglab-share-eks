## k8s-servicemesh+tracing-istio-and-jaeger

#### GIVEN:
  - A developer desktop with docker & git installed (AWS Cloud9)
  - An EKS cluster created via eksctl from demo 04-create-advanced-cluster-eksctl-existing-vpc

#### WHEN:
  - I create a service mesh with istio
  - I enable tracing with jaeger

#### THEN:
  - Simulate traffic to the Service mesh

#### SO THAT:
  - I can see how CNCF service mesh and tracing works on EKS

#### [Return to Main Readme](https://github.com/bwer432/mglab-share-eks#demos)

---------------------------------------------------------------
---------------------------------------------------------------
### REQUIRES
- 00-setup-cloud9
- 03/create-cluster-eksctl-existing-vpc-advanced


---------------------------------------------------------------
---------------------------------------------------------------
### DEMO

#### 1: Install Istio & Jeager
- Reset your region & AWS account variables in case you launched a new terminal session:
```
cd ~/environment/mglab-share-eks/demos/07/k8s-servicemesh+tracing-istio-and-jaeger
export C9_REGION=$(curl --silent http://169.254.169.254/latest/dynamic/instance-identity/document |  grep region | awk -F '"' '{print$4}')
export C9_AWS_ACCT=$(curl -s http://169.254.169.254/latest/dynamic/instance-identity/document | grep accountId | awk -F '"' '{print$4}')
export AWS_ACCESS_KEY_ID=$(cat ~/.aws/credentials | grep aws_access_key_id | awk '{print$3}')
export AWS_SECRET_ACCESS_KEY=$(cat ~/.aws/credentials | grep aws_secret_access_key | awk '{print$3}')
clear
echo $C9_REGION
echo $C9_AWS_ACCT
```
- Update our kubeconfig to interact with the cluster created in 04-create-advanced-cluster-eksctl-existing-vpc.
```
eksctl utils write-kubeconfig --name cluster-eksctl --region $C9_REGION --authenticator-role-arn arn:aws:iam::${C9_AWS_ACCT}:role/cluster-eksctl-creator-role
kubectl config view --minify | grep 'cluster-name' -A 1
kubectl get ns
```
- Install istioctl:  istio version 1.14.1. 
```
export ISTIO_VERSION="1.14.1"
curl -L https://istio.io/downloadIstio | ISTIO_VERSION=${ISTIO_VERSION} | sh -
cd istio-1.*
sudo cp ./bin/istioctl /usr/local/bin/
sudo chmod +x /usr/local/bin/istioctl
```
- Install istio with istioctl:
```
istioctl install --set profile=demo --set hub=gcr.io/istio-release -y
kubectl -n istio-system get deploy
kubectl get svc -n istio-system
```
- Verify the Istio installation:
```
istioctl verify-install
```

#### 2: Deploy 'Book Info' Sample Application that has been instrumented with Zipkin
- Create Namespace & enable envoy inection:
- [Book Info app](https://istio.io/latest/docs/examples/bookinfo/)
```
kubectl create namespace istio-bookinfo
kubectl label namespace istio-bookinfo istio-injection=enabled
```
- Setup Docker login credentials for bookinfo images stored in docker hub (silly rate limiting):
- docker login username is case sensitive and doesn't work with your email address.
```
docker login
```
```
kubectl create secret generic regcred -n istio-bookinfo \
    --from-file=.dockerconfigjson=/home/ec2-user/.docker/config.json \
    --type=kubernetes.io/dockerconfigjson
kubectl get secret regcred -n istio-bookinfo --output="jsonpath={.data.\.dockerconfigjson}" | base64 --decode
```
- Deploy the bookinfo app:
```
kubectl apply -f ./samples/bookinfo/platform/kube/bookinfo.yaml -n istio-bookinfo
for i in $(kubectl get sa -n istio-bookinfo | grep -v NAME | awk '{print$1}');do
    kubectl patch sa $i -n istio-bookinfo -p '"imagePullSecrets": [{"name": "regcred" }]'
done
for i in $(kubectl get deploy -n istio-bookinfo | grep -v NAME | awk '{print$1}');do
    kubectl rollout restart deployments/$i -n istio-bookinfo
done
```
- Deploy the Istio Ingress gateway:
```
kubectl apply -f ./samples/bookinfo/networking/bookinfo-gateway.yaml -n istio-bookinfo
```
- Get the bookinfo URL:
```
export INGRESS_HOST=$(kubectl -n istio-system \
    get service istio-ingressgateway \
    -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
export INGRESS_PORT=$(kubectl -n istio-system \
    get service istio-ingressgateway \
    -o jsonpath='{.spec.ports[?(@.name=="http2")].port}')
echo GATEWAY_URL=$INGRESS_HOST:$INGRESS_PORT/productpage
```
#### 3 Review Kiali & Jaeger dashboards
-  kubectl apply -f samples/addons (x2)
```
kubectl apply -f samples/addons
kubectl apply -f samples/addons
```
-  Launch Kiali UI in a web browser:
```
kubectl -n istio-system patch svc kiali -p '{"spec": {"type": "LoadBalancer"}}'
sleep 3
echo "http://"$(kubectl -n istio-system get svc kiali --output=jsonpath={.status.loadBalancer.ingress[].hostname})":20001"
```
-  Launch Jaeger UI in a web browser:
```
kubectl -n istio-system patch svc tracing -p '{"spec": {"type": "LoadBalancer"}}'
sleep 3
echo "http://"$(kubectl -n istio-system get svc tracing --output=jsonpath={.status.loadBalancer.ingress[].hostname})
```
- Experiment with Changing some service mesh routing rules for Bookinfo:
  - Create 3 versioned 'DestinationRule's to K8s svc targets of the Review microservice ...
  ```
  kubectl apply -f ./samples/bookinfo/networking/destination-rule-reviews.yaml -n istio-bookinfo
  ```
  - Create a 'VirtualService' that sends a test user logged in as 'jason' to V2 of Reviews b& all other requests to V1 ...
  ```  
  kubectl apply -f ./samples/bookinfo/networking/virtual-service-reviews-test-v2.yaml -n istio-bookinfo
  ```
  - Update the 'VirtualService' to send 50% of requests to V3 of Reviews ...
  ```
  kubectl apply -f ./samples/bookinfo/networking/virtual-service-reviews-50-v3.yaml -n istio-bookinfo
  ```

---------------------------------------------------------------
---------------------------------------------------------------
### DEPENDENTS

---------------------------------------------------------------
---------------------------------------------------------------
### CLEANUP
- Do not cleanup if you plan to run any dependent demos
```
istioctl x uninstall --purge -y
kubectl delete ns istio-bookinfo
kubectl delete ns istio-system
```
