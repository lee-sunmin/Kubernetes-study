1.4 오토 스케일링 설정명령어 호출
~~~
kubectl autoscale deployment order --cpu-percent=20 --min=1 --max=3
~~~

kubectl get hpa 명령어로 설정값을 확인 한다.
~~~
NAME    REFERENCE          TARGETS         MINPODS   MAXPODS   REPLICAS   AGE
order   Deployment/order   <unknown>/20%   1         3         0          7s
~~~
auto scale이 20%로 설정


~~~
          ports:
            - containerPort: 8080
          resources: # 이부분
            requests:
              cpu: "200m"
~~~
