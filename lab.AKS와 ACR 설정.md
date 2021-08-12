

- Cloud IDE - Azure Client Config 설정
~~~
$ az login 
~~~

- AKS (Azure Kubernetes Service) 생성
~~~
$ az aks create --resource-group (RESOURCE-GROUP-NAME) --name (Cluster-NAME) --node-count 2 --enable-addons monitoring --generate-ssh-keys
~~~
azure portal에 가면 새롭게 kubernetes 서비스가 생성된 것을 확인 할 수 있음 (Cluster-NAME)으로 생성

- K8s Client 에 Target Context 설정
~~~
$ az aks get-credentials --resource-group (RESOURCE-GROUP-NAME) --name (Cluster-NAME)
~~~
위 명령어 실행하면 azure 바라보게 변경됨
다음을 통해 새로 생성한 클러스터에 부착됐는지 확인
~~~
kubectl get po
-> res : empty
kubectl get node
~~~

- ACR (Azure Container Registry) 생성
~~~
$ az acr create --resource-group (RESOURCE-GROUP-NAME) --name (REGISTRY-NAME) --sku Basic
~~~

- Azure AKS에 ACR Attach 설정
~~~
$ az aks update -n (Cluster-NAME) -g (RESOURCE-GROUP-NAME) --attach-acr (REGISTRY-NAME)
~~~
- Azure ACR Login 설정
~~~
$ az acr login --name (REGISTRY-NAME) --expose-token
~~~



빌드와 푸시를 한번에 하기
Dockerfile이 있는 곳에서 실행해야 함 !!
~~~
az acr build --registry [acr-레지트스리명] --image [acr레지스트리명].azurecr.io/welcome:v1 .

mvn package -B
az acr build --registry user14 --image user14.azurecr.io/welcome:v1 .
~~~

~~~
kubectl create deploy order --image=user14.azurecr.io/order:v1
kubectl get po -w
~~~
