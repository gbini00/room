# Saga
	https://github.com/event storming/products/ 의 saga rollback 브랜치 확인
	https://github.com/event storming/orders/ 의 saga rollback 브랜치 확인
# CQRS
	https://github.com/eventstorming/mypage/blob/master/src/main/java/com/example/template/event/EventListener.java
# Correlation
	https://github.com/sea-boy/correlation-id-in-msa
	이벤트와 폴리시 Correlation key 연결
# Req/Resp
	동기호출
# Gateway

# Config Map/ Persistence Volume
	configmap.yml
	deployment.yml
	describe pod
![deployment yml](https://user-images.githubusercontent.com/38099203/119283834-269fbe80-bc79-11eb-9624-d824f450ff3d.PNG)
![describe pod](https://user-images.githubusercontent.com/38099203/119283835-27385500-bc79-11eb-84e8-e3caf50295c3.PNG)
![configmap](https://user-images.githubusercontent.com/38099203/119283836-27d0eb80-bc79-11eb-8232-25f5e2170956.PNG)


# 운영

## CI/CD 설정

각 구현체들은 각자의 source repository 에 구성되었고, 사용한 CI/CD는 buildspec.yml을 이용한 AWS codebuild를 사용하였습니다.

CodeBuild 프로젝트를 생성하고 AWS_ACCOUNT_ID, KUBE_URL, KUBE_TOKEN 환경 변수 세팅을 한다
```
SA 생성
kubectl apply -f eks-admin-service-account.yml
```
![codebuild(sa)](https://user-images.githubusercontent.com/38099203/119293259-ff52ec80-bc8c-11eb-8671-b9a226811762.PNG)
```
Role 생성
kubectl apply -f eks-admin-cluster-role-binding.yml
```
![codebuild(role)](https://user-images.githubusercontent.com/38099203/119293300-1abdf780-bc8d-11eb-9b07-ad173237efb1.PNG)
```
Token 확인
kubectl -n kube-system get secret
kubectl -n kube-system describe secret eks-admin-token-rjpmq
```
![codebuild(token)](https://user-images.githubusercontent.com/38099203/119293511-84d69c80-bc8d-11eb-99c7-e8929e6a41e4.PNG)
```
buildspec.yml 파일 
마이크로 서비스 room의 yml 파일 이용하도록 세팅
```
![codebuild(buildspec)](https://user-images.githubusercontent.com/38099203/119283849-30292680-bc79-11eb-9f86-cbb715e74846.PNG)
```
codebuild 프로젝트 및 빌드 이력
```
![codebuild(프로젝트)](https://user-images.githubusercontent.com/38099203/119283851-315a5380-bc79-11eb-9b2a-b4522d22d009.PNG)
![codebuild(로그)](https://user-images.githubusercontent.com/38099203/119283850-30c1bd00-bc79-11eb-9547-1ff1f62e48a4.PNG)



## 동기식 호출 / 서킷 브레이킹 / 장애격리

* 서킷 브레이킹 프레임워크의 선택: istio 사용하여 구현함

시나리오는 예약(reservation)--> 룸(room) 시의 연결을 RESTful Request/Response 로 연동하여 구현이 되어있고, 예약 요청이 과도할 경우 CB 를 통하여 장애격리.

- DestinationRule 를 생성하여 circuit break 가 발생할 수 있도록 설정
최소 connection pool 설정
```
# destination-rule.yml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: dr-room
  namespace: airbnb
spec:
  host: room
  trafficPolicy:
    connectionPool:
      http:
        http1MaxPendingRequests: 1
        maxRequestsPerConnection: 1
#    outlierDetection:
#      interval: 1s
#      consecutiveErrors: 1
#      baseEjectionTime: 10s
#      maxEjectionPercent: 100
```

* istio-injection 활성화 및 room pod container 확인

```
kubectl get ns -L istio-injection
kubectl label namespace airbnb istio-injection=enabled 

![Circuit Breaker(istio-enjection)](https://user-images.githubusercontent.com/38099203/119295450-d6812600-bc91-11eb-8aad-46eeac968a41.PNG)

![Circuit Breaker(pod)](https://user-images.githubusercontent.com/38099203/119295568-0cbea580-bc92-11eb-9d2b-8580f3576b47.PNG)

```

* 부하테스터 siege 툴을 통한 서킷 브레이커 동작 확인:

siege 실행

```
kubectl run siege --image=apexacme/siege-nginx -n airbnb
kubectl exec -it siege -c siege -n airbnb -- /bin/bash
```


- 동시사용자 1로 부하
```
siege -c1 -t10S -v --content-type "application/json" 'http://room:8080/rooms POST {"desc": "Beautiful House3"}'

** SIEGE 4.0.4
** Preparing 1 concurrent users for battle.
The server is now under siege...
HTTP/1.1 201     0.49 secs:     254 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.05 secs:     254 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.02 secs:     254 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.03 secs:     254 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.02 secs:     254 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.02 secs:     254 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.03 secs:     254 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.03 secs:     254 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.03 secs:     254 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.03 secs:     256 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.03 secs:     256 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.02 secs:     256 bytes ==> POST http://room:8080/rooms
```

- 동시사용자 2로 부하 
503 에러 발생
```
siege -c2 -t10S -v --content-type "application/json" 'http://room:8080/rooms POST {"desc": "Beautiful House3"}'

** SIEGE 4.0.4
** Preparing 2 concurrent users for battle.
The server is now under siege...
HTTP/1.1 201     0.02 secs:     258 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.02 secs:     258 bytes ==> POST http://room:8080/rooms
HTTP/1.1 503     0.10 secs:      81 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.02 secs:     258 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.04 secs:     258 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.05 secs:     258 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.22 secs:     258 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.08 secs:     258 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.07 secs:     258 bytes ==> POST http://room:8080/rooms
HTTP/1.1 503     0.01 secs:      81 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.01 secs:     258 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.03 secs:     258 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.02 secs:     258 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.01 secs:     258 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.02 secs:     258 bytes ==> POST http://room:8080/rooms
HTTP/1.1 503     0.01 secs:      81 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.01 secs:     258 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.02 secs:     258 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.02 secs:     258 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.02 secs:     258 bytes ==> POST http://room:8080/rooms
HTTP/1.1 503     0.00 secs:      81 bytes ==> POST http://room:8080/rooms
```

- kiali 화면에 서킷 브레이크 확인
```

![Circuit Breaker(kiali)](https://user-images.githubusercontent.com/38099203/119283822-1d165680-bc79-11eb-9b33-a984c4c6c70e.PNG)

```

- 다시 최소 Connection pool로 부하 다시 정상 확인

```
** SIEGE 4.0.4
** Preparing 1 concurrent users for battle.
The server is now under siege...
HTTP/1.1 201     0.01 secs:     260 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.01 secs:     260 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.01 secs:     260 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.03 secs:     260 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.00 secs:     260 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.02 secs:     260 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.01 secs:     260 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.01 secs:     260 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.01 secs:     260 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.00 secs:     260 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.01 secs:     260 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.01 secs:     260 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.01 secs:     260 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.00 secs:     260 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.01 secs:     260 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.01 secs:     260 bytes ==> POST http://room:8080/rooms

:
:

Lifting the server siege...
Transactions:                   1139 hits
Availability:                 100.00 %
Elapsed time:                   9.19 secs
Data transferred:               0.28 MB
Response time:                  0.01 secs
Transaction rate:             123.94 trans/sec
Throughput:                     0.03 MB/sec
Concurrency:                    0.98
Successful transactions:        1139
Failed transactions:               0
Longest transaction:            0.04
Shortest transaction:           0.00

```

- 운영시스템은 죽지 않고 지속적으로 CB 에 의하여 적절히 회로가 열림과 닫힘이 벌어지면서 자원을 보호하고 있음을 보여줌.
  virtualhost 설정과 동적 Scale out (replica의 자동적 추가,HPA) 을 통하여 시스템을 확장 해주는 후속처리가 필요.


# Autoscale (HPA)
    kubectl autoscale deployment room -n airbnb --cpu-percent=50 --min=1 --max=10
    kubectl get hpa -n airbnb
	siege -c100 -t60S -r10 -v --content-type "application/json" 'http://room:8080/rooms POST {"desc": "Beautiful House3"}'

![Autoscale (HPA)](https://user-images.githubusercontent.com/38099203/119283787-0a038680-bc79-11eb-8d9b-d8aed8847fef.PNG)
![Autoscale (HPA)(siege)](https://user-images.githubusercontent.com/38099203/119283780-08d25980-bc79-11eb-8c34-0f1c8885f208.PNG)
![Autoscale (HPA)(kubectl autoscale)](https://user-images.githubusercontent.com/38099203/119283789-0a038680-bc79-11eb-9d2e-e6821ca101b9.PNG)
![Autoscale (HPA)(결과)](https://user-images.githubusercontent.com/38099203/119283785-096af000-bc79-11eb-8227-6133c31aed87.PNG)
	
# Zero-downtime deploy (Readiness Probe)

	siege -c100 -t60S -r10 -v --content-type "application/json" 'http://room:8080/rooms POST {"desc": "Beautiful House3"}'
![Zero-downtime deploy (Readiness Probe)(siege)](https://user-images.githubusercontent.com/38099203/119283891-4931d780-bc79-11eb-9efe-ca34b0fd30e2.PNG)
	
# Self-healing (Liveness Probe)
    ## 방법1) 
	  - room deployment.yml 파일 수정 
	    콘테이너 실행후 /tmp/healthy 파일을 만들고 30초 후 삭제하도록 함
		livenessProbe에 'cat /tmp/healthy'으로 검증하도록 함
	    이미지 경로 : deployment yml(1)(빨).PNG
	  - kubectl describe pod room -n airbnb 실행으로 확인
	    컨테이너 실행 후 30초 동인은 정상이나 30초 이후 /tmp/healthy 파일이 삭제되어 livenessProbe에서 실패를 리턴하게 됨
	    이미지 경로 : Self-healing (Liveness Probe)(1-1)(빨).PNG
    ## 방법2) 
	  - room deployment.yml 파일 수정
	    정상 기동까지 약 20초 정도 걸리나 initialDelaySeconds를 2초 periodSeconds를 1초로 함 
		20초 > initialDelaySeconds (2초) + (failureThreshold (디폴트 3초) * periodSeconds (1초))가 되어 컨테이터 다시 시작
		이미지 경로 : log(2).PNG, deployment yml(2)(빨).PNG
      - kubectl describe pod room -n airbnb 실행으로 확인
	    정상 기동까지 약 5초 걸리나, 초기 딜레이 2초 후 livenessProbe 실행하나 살패를 리턴하고 1초 후 다시 livenessProbe 실행하나 실패하여 재기동을 함
		이미지 경로 : Self-healing (Liveness Probe)(2-1)(빨).PNG

![Self-healing (Liveness Probe)(2-1)](https://user-images.githubusercontent.com/38099203/119283861-3b7c5200-bc79-11eb-85f2-3e3a5dedb352.PNG)
![deployment yml(1)(빨)](https://user-images.githubusercontent.com/38099203/119283865-3c14e880-bc79-11eb-906b-3764af10d8ad.PNG)
![deployment yml(1)](https://user-images.githubusercontent.com/38099203/119283866-3cad7f00-bc79-11eb-8f2c-41dfddf79267.PNG)
![deployment yml(2)(빨)](https://user-images.githubusercontent.com/38099203/119283867-3cad7f00-bc79-11eb-879d-c8071f544aa5.PNG)
![deployment yml(2)](https://user-images.githubusercontent.com/38099203/119283868-3d461580-bc79-11eb-9636-26158af94101.PNG)
![log(2)(빨)](https://user-images.githubusercontent.com/38099203/119283869-3d461580-bc79-11eb-9b78-229fd9206c88.PNG)
![log(2)](https://user-images.githubusercontent.com/38099203/119283871-3ddeac00-bc79-11eb-8ba1-ea479bcfbe93.PNG)
![Self-healing (Liveness Probe)(1)](https://user-images.githubusercontent.com/38099203/119283872-3e774280-bc79-11eb-8bb0-37bce6f2d642.PNG)
![Self-healing (Liveness Probe)(1-1)(빨)](https://user-images.githubusercontent.com/38099203/119283874-3e774280-bc79-11eb-9d79-733b1c465739.PNG)
![Self-healing (Liveness Probe)(1-1)](https://user-images.githubusercontent.com/38099203/119283875-3f0fd900-bc79-11eb-9c2e-aca5885d84f5.PNG)
![Self-healing (Liveness Probe)(2-1)(빨)](https://user-images.githubusercontent.com/38099203/119283876-3f0fd900-bc79-11eb-8d9e-9d30520f2354.PNG)


# Polyglot
