분기하는 ingress를 통한 진입점 통일

ingress.yaml
~~~
apiVersion: "extensions/v1beta1"
kind: "Ingress"
metadata: 
  name: "shopping-ingress"
  annotations: 
    kubernetes.io/ingress.class: "nginx"
spec: 
  rules: 
    - 
      http: 
        paths: 
          - 
            path: /orders
            pathType: Prefix
            backend: 
              serviceName: order
              servicePort: 8080
          - 
            path: /deliveries
            pathType: Prefix
            backend: 
              serviceName: delivery
              servicePort: 8080
          - 
            path: /products
            pathType: Prefix
            backend: 
              serviceName: product
              servicePort: 8080
~~~

~~~
kubectl create -f ingress.yaml

#ingress check
# address 안채워짐. 
$ kubectl get ingress shopping-ingress -w

~~~
아무리 기다려도 ADDRESS 부분에 값이 채워지지 않음을 알 수 있다. 원인은 내게 gateway provider 가 없기 때문이다. 
Ingress 는 Kubernetes 의 스펙일 뿐, 이를 실질적으로 지원하는 ingress controller 가 필요하기 때문이다. 
무료로 nginx 인그레스 프로바이더를 사용할 수 있다.


- Ingress Provider 설치하기

Helm repo 설정
~~~
helm repo add stable https://charts.helm.sh/stable
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
kubectl create namespace ingress-basic
~~~
nginx controller 설치
~~~
helm install nginx-ingress ingress-nginx/ingress-nginx --namespace=ingress-basic
~~~


설치 확인
~~~
kubectl get all --namespace=ingress-basic

# show address 
$ kubectl get ingress

# 위 ip 참고해서 http로 접속
~~~



