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


# Deploy/ Pipeline
각 구현체들은 각자의 source repository 에 구성되었고, 사용한 CI/CD는 buildspec.yml을 이용한 AWS codebuild를 사용하였습니다.

CodeBuild 프로젝트를 생성하고 AWS_ACCOUNT_ID, KUBE_URL, KUBE_TOKEN 환경 변수 세팅을 한다
		kubectl apply -f eks-admin-service-account.yml
![codebuild(sa)](https://user-images.githubusercontent.com/38099203/119293259-ff52ec80-bc8c-11eb-8671-b9a226811762.PNG)
		kubectl apply -f eks-admin-cluster-role-binding.yml
![codebuild(role)](https://user-images.githubusercontent.com/38099203/119293300-1abdf780-bc8d-11eb-9b07-ad173237efb1.PNG)
		kubectl -n kube-system get secret
		kubectl -n kube-system describe secret eks-admin-token-rjpmq
![codebuild(token)](https://user-images.githubusercontent.com/38099203/119293511-84d69c80-bc8d-11eb-99c7-e8929e6a41e4.PNG)
		buildspec.yml 파일 
			마이크로 서비스 room의 yml 파일 이용하도록 세팅
![codebuild(buildspec)](https://user-images.githubusercontent.com/38099203/119283849-30292680-bc79-11eb-9f86-cbb715e74846.PNG)
		codebuild 프로젝트 및 빌드 이력
![codebuild(프로젝트)](https://user-images.githubusercontent.com/38099203/119283851-315a5380-bc79-11eb-9b2a-b4522d22d009.PNG)
[codebuild(로그)](https://user-images.githubusercontent.com/38099203/119283850-30c1bd00-bc79-11eb-9547-1ff1f62e48a4.PNG)


# Circuit Breaker
	kubectl run siege --image=apexacme/siege-nginx -n airbnb
	kubectl exec -it siege -c siege -n airbnb -- /bin/bash
	kubectl get ns -L istio-injection
	kubectl label namespace airbnb istio-injection=enabled ==> deploy 재 실행하면 pod 내 container가 2개 생김 
	kubectl label namespace airbnb istio-injection- ==> deploy 재 실행하면 pod 내 container가 1개만 생김
	siege -c100 -t60S -r10 -v --content-type "application/json" 'http://room:8080/rooms POST {"desc": "Beautiful House3"}'
	siege -c1 -t10S -v --content-type "application/json" 'http://room:8080/rooms POST {"desc": "Beautiful House3"}'
    siege -c2 -t10S -v --content-type "application/json" 'http://room:8080/rooms POST {"desc": "Beautiful House3"}'

![Circuit Breaker(yml)](https://user-images.githubusercontent.com/38099203/119283820-1c7dc000-bc79-11eb-9040-352939537d83.PNG)
![Circuit Breaker(siege)](https://user-images.githubusercontent.com/38099203/119283819-1be52980-bc79-11eb-8642-5361f65dee3e.PNG)
![Circuit Breaker(kiali)](https://user-images.githubusercontent.com/38099203/119283822-1d165680-bc79-11eb-9b33-a984c4c6c70e.PNG)
	
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
