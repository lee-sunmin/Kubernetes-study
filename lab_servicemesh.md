- Download Istio
다운로드 후, 압축을 해제한다
~~~
$ curl -L https://istio.io/downloadIstio | sh -
~~~
Istio 패키지 폴더로 이동시킨다
istio-< istio_full_version >:
~~~
$ cd istio-< istio_full_version >
~~~

해당 디렉토리에는 다음의 내용을 포함하고 있다:

- 샘플애플리케이션: `samples/`
- `istioctl` 클라이언트 툴은
  `bin/` 디렉토리에 포함되어있다.
istioctl 클라이언트를 PATH에 잡아준다:
~~~
$ export PATH=$PWD/bin:$PATH
~~~

- Install Istio
기본적인 구성인 demo 를 기반으로 설치한다.
~~~
   $ istioctl install --set profile=demo -y
    ✔ Istio core installed
    ✔ Istiod installed
    ✔ Egress gateways installed
    ✔ Ingress gateways installed
    ✔ Installation complete
~~~
Envoy 사이드카를 생성하는 Pod 들에 자동적으로 주입하게 하기 위해 다음의 설정을 추가한다:
~~~
$ kubectl label namespace default istio-injection=enabled
    namespace/default labeled
~~~

Deploy the sample application bookinfo
Bookinfo 샘플 애플리케이션을 설치한다
~~~
$ kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml
    service/details created
    serviceaccount/bookinfo-details created
    deployment.apps/details-v1 created
    service/ratings created
    serviceaccount/bookinfo-ratings created
    deployment.apps/ratings-v1 created
    service/reviews created
    serviceaccount/bookinfo-reviews created
    deployment.apps/reviews-v1 created
    deployment.apps/reviews-v2 created
    deployment.apps/reviews-v3 created
    service/productpage created
    serviceaccount/bookinfo-productpage created
    deployment.apps/productpage-v1 created
~~~
애플리케이션이 시작되고 각 Pod들이 준비상태가 된다. Istio Sidecar들이 같이 배포되었을 것이다.
~~~
$ kubectl get services
    NAME          TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
    details       ClusterIP   10.0.0.212      <none>        9080/TCP   29s
    kubernetes    ClusterIP   10.0.0.1        <none>        443/TCP    25m
    productpage   ClusterIP   10.0.0.57       <none>        9080/TCP   28s
    ratings       ClusterIP   10.0.0.33       <none>        9080/TCP   29s
    reviews       ClusterIP   10.0.0.28       <none>        9080/TCP   29s
~~~
그리고 다음과 같이 확인한다:
~~~
    $ kubectl get pods
    NAME                              READY   STATUS    RESTARTS   AGE
    details-v1-558b8b4b76-2llld       2/2     Running   0          2m41s
    productpage-v1-6987489c74-lpkgl   2/2     Running   0          2m40s
    ratings-v1-7dc98c7588-vzftc       2/2     Running   0          2m41s
    reviews-v1-7f99cc4496-gdxfn       2/2     Running   0          2m41s
    reviews-v2-7d79d5bd5d-8zzqd       2/2     Running   0          2m41s
    reviews-v3-7dbcdcbc56-m8dph       2/2     Running   0          2m41s
~~~
모든 Pod 가 2/2로 표시될때까지 기다린다
모든 상태가 Running 이 될때까지 기다린다

모든것이 제대로 된 후에는 다음의 주소로 접속하여 웹 페이지 HTML 콘텐츠 내용을 확인할 수 있어야 한다.
~~~
$ kubectl exec "$(kubectl get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}')" -c ratings -- curl -sS productpage:9080/productpage | grep -o "<title>.*</title>"
    <title>Simple Bookstore App</title>
~~~
Open the application to outside traffic
애플리케이션이 잘 디플로이 되었지만 외부에서는 접근이 되지 않은 상태이다. 외부접속이 가능하게 하기위해서는 Istio의 Ingress Gateway를 설정해야 한다.

애플리케이션들을 Istio Gateway 에 묶기위한 설정들을 배포한다:
~~~
$ kubectl apply -f samples/bookinfo/networking/bookinfo-gateway.yaml
    gateway.networking.istio.io/bookinfo-gateway created
    virtualservice.networking.istio.io/bookinfo created
# 발생한 문제가 없는지 확인한다:
    $ istioctl analyze
    ✔ No validation issues found when analyzing namespace: default.
~~~
Determining the ingress IP and ports
다음의 환경 변수를 인그레스의 접속 주소를 얻어와 설정한다:INGRESS_HOST 와 INGRESS_PORT
~~~
$ kubectl get svc istio-ingressgateway -n istio-system
NAME                   TYPE           CLUSTER-IP       EXTERNAL-IP     PORT(S)                                      AGE
istio-ingressgateway   LoadBalancer   [ip]   [ip]  80:31380/TCP,443:31390/TCP,31400:31400/TCP   17h
~~~
어느정도 기다렸을때, EXTERNAL-IP 값이 설정되었다면 외부 로드밸런서 설정이 잘 된것이다. 만약 설정값이 기다려도 나타나지 않으면, 외부 ㄹ드밸런서가 없는 경우이다.

인그레스의 IP 와 Port 번호를 가져온다:
~~~
$ export INGRESS_HOST=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
$ export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].port}')
$ export SECURE_INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="https")].port}')
~~~
AWS와 같은 특정한 환경에서는 IP address 대신 host명을 넘겨주는 경우가 있다.
이런 경우는 호스트명을 가져오도록 하는 명령으로 대치하여 설정한다:
~~~
$ export INGRESS_HOST=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
~~~
그런다음 다음의 환경변수에 Gateway URL을 생성할 수 있다: GATEWAY_URL:
~~~
$ export GATEWAY_URL=$INGRESS_HOST:$INGRESS_PORT
~~~
환경변수에 IP address 와 port 가 잘 설정되었는지 확인한다:
~~~
$ echo "$GATEWAY_URL"
~~~
Verify external access
외부에서의 접속이 문제 없는지 확인한다.

다음의 명령을 입력하여 얻어진 url 로 브라우저를 접속하여 Bookinfo application이 잘 동작하는지 확인한다.
~~~
$ echo "http://$GATEWAY_URL/productpage"
~~~
Bookinfo product page 를 여러번 리프래스 해본다.

## 동적 트래픽 제어 테스트
Review 서비스의 종류별 유입을 동적으로 변경하여 Canary 배포등의 시나리오에 적용하는 예시.


~~~
# VirtualService 를 먼저 배포
kubectl apply -f samples/bookinfo/networking/virtual-service-reviews-80-20.yaml 
# DestinationRule 을 배포
kubectl apply -f samples/bookinfo/networking/destination-rule-reviews.yaml
~~~


80:20 의 확률로 v1과 v2가 선택됨을 확인할 수 있다.

virtual-service-reviews-80-20.yaml 의 내용을 아래와 같이 수정한다:
~~~
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
    - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v1
      weight: 0
    - destination:
        host: reviews
        subset: v2
      weight: 100
~~~
모든 트래픽 유입이 v2버전으로 전환됨을 확인한다.
이걸로 canary deploy 가능




