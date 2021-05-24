Saga
	https://github.com/event storming/products/ 의 saga rollback 브랜치 확인
	https://github.com/event storming/orders/ 의 saga rollback 브랜치 확인
CQRS
	https://github.com/eventstorming/mypage/blob/master/src/main/java/com/example/template/event/EventListener.java
Correlation
	https://github.com/sea-boy/correlation-id-in-msa
	이벤트와 폴리시 Correlation key 연결
Req/Resp
	동기호출
Gateway

# Config Map/ Persistence Volume
	configmap.yml
	deployment.yml
	describe pod



# Deploy/ Pipeline
    CodeBuild 프로젝트 생성
	AWS_ACCOUNT_ID 환경 변수 세팅
	ECR 관련 정책을 추가
		SPC03-codebuild-policy 정책 생성
	KUBE_URL 환경 변수 세팅
		https://40ED69158C1B9643987B31F49DC2F915.gr7.ap-northeast-2.eks.amazonaws.com
	KUBE_TOKEN 환경 변수 세팅
		kubectl apply -f eks-admin-service-account.yml
		kubectl apply -f eks-admin-cluster-role-binding.yml
		kubectl -n kube-system get secret
			eks-admin-token-rjpmq                            kubernetes.io/service-account-token   3      2m5s
		kubectl -n kube-system describe secret eks-admin-token-rjpmq
			Name:         eks-admin-token-rjpmq
			Namespace:    kube-system
			Labels:       <none>
			Annotations:  kubernetes.io/service-account.name: eks-admin
						  kubernetes.io/service-account.uid: 135e355d-c80f-41ea-85ca-67ad7a9bcd0b

			Type:  kubernetes.io/service-account-token

			Data
			====
			namespace:  11 bytes
			token:      eyJhbGciOiJSUzI1NiIsImtpZCI6IjNmdEpnM1JzZEk1bm05a2N1SDAwWV9kWUliNDVfRFVIbHpwUG9BZERrOGsifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJla3MtYWRtaW4tdG9rZW4tcmpwbXEiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoiZWtzLWFkbWluIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQudWlkIjoiMTM1ZTM1NWQtYzgwZi00MWVhLTg1Y2EtNjdhZDdhOWJjZDBiIiwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50Omt1YmUtc3lzdGVtOmVrcy1hZG1pbiJ9.Qz4pL4IXkI423an0hxQhIIEpGB-6o48JPdPzgEp5_nfNDihjf97g8zZ2CWUv480mAcxcDMg3bt6RyyopVL8N2tGkN5TMA6ii6VuuGbSrt5gvSGyzroBJ6bkJNFVIhb-XY6J1YPVVQE-tmgHG0p2OuvZDYxfNVnvyly4IGDu_OkLS4TIVjkF9W0EQSlk9ARpxXFvWCwB0TQGFdvtHsTzJ-Cmfqv-GN63Hykx_dWIow-vVRtBtIBEG1AvDawA8rypqaHAC4AM1Psy7FAI276qt9RR8O1UYGTm_uH1HNPWQiT1vczyJVxmXcd_UuzQIfh7Sat8IJxM-xA9cPat2rCC5Iw
			ca.crt:     1025 bytes


# Circuit Breaker
	kubectl run siege --image=apexacme/siege-nginx -n airbnb
	kubectl exec -it siege -c siege -n airbnb -- /bin/bash
	kubectl get ns -L istio-injection
	kubectl label namespace airbnb istio-injection=enabled ==> deploy 재 실행하면 pod 내 container가 2개 생김 
	kubectl label namespace airbnb istio-injection- ==> deploy 재 실행하면 pod 내 container가 1개만 생김
	siege -c100 -t60S -r10 -v --content-type "application/json" 'http://room:8080/rooms POST {"desc": "Beautiful House3"}'
	siege -c1 -t10S -v --content-type "application/json" 'http://room:8080/rooms POST {"desc": "Beautiful House3"}'
    siege -c2 -t10S -v --content-type "application/json" 'http://room:8080/rooms POST {"desc": "Beautiful House3"}'
	
# Autoscale (HPA)

    kubectl autoscale deployment room -n airbnb --cpu-percent=50 --min=1 --max=10
    kubectl get hpa -n airbnb
	siege -c100 -t60S -r10 -v --content-type "application/json" 'http://room:8080/rooms POST {"desc": "Beautiful House3"}'

# Zero-downtime deploy (Readiness Probe)

	siege -c100 -t60S -r10 -v --content-type "application/json" 'http://room:8080/rooms POST {"desc": "Beautiful House3"}'
	
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


Polyglot
