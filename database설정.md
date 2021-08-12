
order 서비스의 Database 설정을 아래와 같이 변경하므로써, 외부의 데이터베이스에 접근 가능하게 된다:

application.yml (or application-prod.yml)
~~~
spring:
  jpa:
    hibernate:
      naming:
        physical-strategy: org.hibernate.boot.model.naming.PhysicalNamingStrategyStandardImpl
      ddl-auto: update
    properties:
      hibernate:
        show_sql: true
        format_sql: true
        dialect: org.hibernate.dialect.MySQL57Dialect
  datasource:
    url: jdbc:mysql://${_DATASOURCE_ADDRESS:[ip]:[port]}/${_DATASOURCE_TABLESPACE:my-database}
    username: ${_DATASOURCE_USERNAME:root1}
    password: ${_DATASOURCE_PASSWORD:secretpassword}
    driverClassName: com.mysql.cj.jdbc.Driver
~~~

변경한 정보를 환경변수에서 얻어오도록 설정하였고, Deployment 에서 위의 값이 전달되도록 주입할 수 있다:
~~~
apiVersion: "apps/v1"
kind: "Deployment"
metadata: 
  name: "order"
  labels: 
    app: "order"
spec: 
  selector: 
    matchLabels: 
      app: "order"
  replicas: 1
  template: 
    metadata: 
      labels: 
        app: "order"
    spec: 
      containers: 
        - 
          name: "order"
          image: "[image명]"
          ports: 
            - 
              containerPort: 80
          env:
            - name: superuser.userId
              value: some_value					
            - name: _DATASOURCE_ADDRESS
              value: mysql
            - name: _DATASOURCE_TABLESPACE
              value: orderdb
            - name: _DATASOURCE_USERNAME
              value: root
            - name: _DATASOURCE_PASSWORD
              value: admin
~~~

~~~
kubectl get po # pod 명 확인
kubectl logs <pod 명>
~~~


값을 위와 같이 Deployment 설정에 직접 입력하는것은 개발자와 운영자사이에 역할이 혼재되므로, 운영자가 해당 설정 부분만을 관리할 수 있도록 별도의 Configuration 을 위한 쿠버네티스 객체인 ConfigMap (혹은 Secret)을 선언하여 연결할 수 있다. 여기서는 패스워드가 노출되면 안되므로 PASSWORD 에 대해서만 Secret 을 이용하여 분리해준다:
mysql-secret.yml
~~~
apiVersion: v1
kind: Secret
metadata:
  name: mysql-pass
type: Opaque
data:
  password: YWRtaW4=     
~~~

Secret 객체를 생성한다:
~~~
$ kubectl create -f mysql-secret.yaml
secret/mysql-pass created
# 생성된 secret 확인
$ kubectl get secrets
~~~


해당 Secret 을 Deployment 에 설정:
~~~
          env:
            - name: superuser.userId
              value: userId
            - name: _DATASOURCE_ADDRESS
              value: mysql
            - name: _DATASOURCE_TABLESPACE
              value: orderdb
            - name: _DATASOURCE_USERNAME
              value: root
            - name: _DATASOURCE_PASSWORD
              valueFrom:
                secretKeyRef: # 이부분!
                  name: mysql-pass
                  key: password
~~~




MySQL 을 위한 Pod 설치
mysql.yaml
~~~
apiVersion: v1
kind: Pod
metadata:
  name: mysql
  labels:
    name: lbl-k8s-mysql
spec:
  containers:
  - name: mysql
    image: mysql:latest
    env:
    - name: MYSQL_ROOT_PASSWORD
      valueFrom:
        secretKeyRef:
          name: mysql-pass
          key: password
    ports:
    - name: mysql
      containerPort: 3306
      protocol: TCP
    volumeMounts:
    - name: k8s-mysql-storage
      mountPath: /var/lib/mysql
  volumes:
  - name: k8s-mysql-storage
    emptyDir: {}
~~~

Pod 에 접속하여 orderdb 데이터베이스 공간을 만들어주고 데이터베이스가 잘 동작하는지 확인한다:
~~~
$ kubectl exec mysql -it -- bash

# echo $MYSQL_ROOT_PASSWORD
admin

# mysql --user=root --password=$MYSQL_ROOT_PASSWORD

mysql> create database orderdb;
    -> ;
Query OK, 1 row affected (0.01 sec)

mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| orderdb            |
| performance_schema |
| sys                |
+--------------------+
5 rows in set (0.01 sec)

mysql> exit
~~~

주문 마이크로 서비스를 쿠버네티스 DNS 체계내에서 접근가능하게 하기 위해 ClusterIP 로 서비스를 생성해준다. 주문 서비스에서 mysql 접근을 위하여 "mysql"이라는 도메인명으로 접근하고 있으므로, 같은 이름으로 서비스를 만들어준다:
~~~
kubectl expose pod mysql --port=3306
~~~

port-forward 하고 아래 명령어 날리면 데이터 들어가야 하는데
order:8080이 아닌 localhost:8080으로 해야 들어간다. 뭔가 잘못한듯 ~
~~~
http order:8080/orders productId=1 customerId="jjy"
~~~


