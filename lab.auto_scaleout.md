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
# deployment.yml 파일 수정

          ports:
            - containerPort: 8080
          resources: # 이부분 !! 라인 잘 맞춰서 추가해야 한다.
            requests:
              cpu: "200m"
~~~

변경된 yaml 파일을 사용하여 쿠버네티스에 배포한다.
~~~
kubectl apply -f deployment.yml
~~~

배포 완료 후 kubectl get deploy order -o yaml 명령을 쳐서 image 와 resources의 값이 정상적으로 설정되어있는지 확인
kubectl get po 실행하여 STATUS가 정상적으로 Running 상태 확인

- 테스트
siege 사용해서 부하 줌
~~~
# terminal1

kubectl exec -it siege -- /bin/bash
siege -c20 -t40S -v http://order:8080/orders
exit
~~~

auto scaling 하는 것 확인 할 수 있음 (siege 전에 실행시켜 놓는게 보기 편함)
~~~
root@labs--1534752307:/home/project/ops-autoscale/shopmall/order/kubernetes# kubectl get po -w
NAME                     READY   STATUS    RESTARTS   AGE
order-7b66547f76-ljfj4   1/1     Running   0          78s
siege                    1/1     Running   0          139m
order-7b66547f76-dplnp   0/1     Pending   0          0s
order-7b66547f76-dplnp   0/1     Pending   0          0s
order-7b66547f76-j94g7   0/1     Pending   0          0s
order-7b66547f76-j94g7   0/1     Pending   0          0s
order-7b66547f76-dplnp   0/1     ContainerCreating   0          0s
order-7b66547f76-j94g7   0/1     ContainerCreating   0          0s
order-7b66547f76-j94g7   0/1     Running             0          8s
order-7b66547f76-dplnp   0/1     Running             0          10s
~~~

CPU 값이 늘어난 것을 확인
~~~
root@labs--1534752307:/home/project/ops-autoscale/shopmall/order/kubernetes# kubectl get hpa
NAME    REFERENCE          TARGETS    MINPODS   MAXPODS   REPLICAS   AGE
order   Deployment/order   374%/20%   1         3         3          18m
~~~
