## OuterPark(뮤지컬 예약)

# 서비스 시나리오

기능적 요구사항
1. 공연관리자가 예약가능한 뮤지컬과 좌석수를 등록한다.
1. 고객이 뮤지컬 좌석을 예약한다.
1. 뮤지컬이 예약되면 예약된 좌석수 만큼 예약가능한 좌석수에서 차감된다.
1. 예약이 되면 결제정보를 승인한다.
1. 결제정보가 승인되면 알림메시지를 발송한다. 
1. 고객이 뮤지컬 예약을 취소할 수 있다. 
1. 예약이 취소되면 결제가 취소된다.
1. 결제가 최소되면 알림메시지를 발송한다.
1. 고객이 모든 진행내역을 볼 수 있어야 한다.

비기능적 요구사항
1. 트랜잭션
    1. 예약 가능한 좌석이 부족하면 예약이 되지 않는다.> Sync 호출
    1. 예약이 취소되면 결제가 취소되고 예약가능한 좌석수가 증가한다.> SAGA, 보상 트랜젝션
1. 장애격리
    1. 결제가 완료되지 않아도 예약은 365일 24시간 받을 수 있어야 한다.> Async (event-driven), Eventual Consistency
    1. 결제시스템이 과중되면 주문을 잠시동안 받지 않고 결제를 잠시후에 하도록 유도한다> Circuit breaker, fallback
1. 성능
    1. 고객이 모든 진행내역을 조회 할 수 있도록 성능을 고려하여 별도의 view로 구성한다.> CQRS


# 체크포인트

1. Saga
1. CQRS
1. Correlation
1. Req/Resp
1. Gateway
1. Deploy/ Pipeline
1. Circuit Breaker
1. Autoscale (HPA)
1. Zero-downtime deploy (Readiness Probe)
1. Config Map/ Persistence Volume
1. Polyglot
1. Self-healing (Liveness Probe)


# Gateway 적용
- gateway > applitcation.yml 설정
![image](https://user-images.githubusercontent.com/84000848/122344337-a6236380-cf81-11eb-83d9-98f2311b4f6a.png)
- gateway 테스트
```
http POST http://gateway:8080/musicals musicalId=1003 name=HOT reservableSeat=100000 
```
![image](https://user-images.githubusercontent.com/84000848/122344967-4b3e3c00-cf82-11eb-8bb1-9cd21999a6d3.png)
```
http GET http://gateway:8080/musicals/2 
```
![image](https://user-images.githubusercontent.com/84000848/122345044-601acf80-cf82-11eb-8b79-14a11fdd838e.png)

# 운영

# Deploy

- 1)네임스페이스 만들기
```
kubectl create ns outerpark
kubectl get ns
```
![image](https://user-images.githubusercontent.com/84000848/122322035-c4786780-cf5f-11eb-904f-48d96217d2a1.png)
- 소스가져오기
```
git clone https://github.com/hyucksookwon/outerpark.git
```
![image](https://user-images.githubusercontent.com/84000848/122329826-0a87f800-cf6d-11eb-927a-688f208fab5a.png)

- 2)빌드하기
```
cd outerpark/reservation
mvn package
```
![image](https://user-images.githubusercontent.com/84000848/122330314-eb3d9a80-cf6d-11eb-82cd-8faf7b0c1de7.png)

- 3)도커라이징: Azure 레지스트리에 도커 이미지 빌드 후 푸시하기
```
az acr build --registry outerparkskacr --image outerparkskacr.azurecr.io/reservation:latest .
```
![image](https://user-images.githubusercontent.com/84000848/122330874-e3cac100-cf6e-11eb-89bf-771e533c66ef.png)
![image](https://user-images.githubusercontent.com/84000848/122330924-f513cd80-cf6e-11eb-9c72-0562a27eabcd.png)
![image](https://user-images.githubusercontent.com/84000848/122331422-c2b6a000-cf6f-11eb-8c6d-88820b5c0e20.png)

- 4)컨테이너라이징: 디플로이 생성 확인
```
kubectl create deploy reservation --image=outerparkskacr.azurecr.io/reservation:latest -n outerpark
kubectl get all -n outerpark
```
![image](https://user-images.githubusercontent.com/84000848/122331554-fb567980-cf6f-11eb-83ac-9578bd657c1c.png)

- 5)컨테이너라이징: 서비스 생성 확인
```
kubectl expose deploy reservation --type="ClusterIP" --port=8080 -n outerpark
kubectl get all -n outerpark
```
![image](https://user-images.githubusercontent.com/84000848/122331656-2771fa80-cf70-11eb-8479-aa6cfe567981.png)
- payment, musical, notice, customercenter, gateway에도 동일한 작업 반복
*최종 결과
![image](https://user-images.githubusercontent.com/84000848/122349147-eafdc900-cf86-11eb-96bb-a50afe56ad58.png)
- deployment.yml을 사용하여 배포 (reservation의 deployment.yml 추가)
![image](https://user-images.githubusercontent.com/84000848/122332320-2d1c1000-cf71-11eb-8766-b494f157f247.png)
- deployment.yml로 서비스 배포
```
kubectl apply -f kubernetes/deployment.yml
```

# 동기식 호출 / 서킷 브레이킹 / 장애격리
- 시나리오는 예약(reservation)-->공연(musical) 시의 연결을 RESTful Request/Response 로 연동하여 구현이 되어있고, 예약이 과도할 경우 CB 를 통하여 장애격리.
- Hystrix 설정: 요청처리 쓰레드에서 처리시간이 250 밀리가 넘어서기 시작하여 어느정도 유지되면 CB 회로가 닫히도록 (요청을 빠르게 실패처리, 차단) 설정
```
# circuit breaker 설정 start
feign:
  hystrix:
    enabled: true

hystrix:
  command:
    # 전역설정
    default:
      execution.isolation.thread.timeoutInMilliseconds: 250
# circuit breaker 설정 end
```
- 부하테스터 siege 툴을 통한 서킷 브레이커 동작 확인: 동시사용자 100명, 60초 동안 실시
시즈 접속
```
kubectl exec -it pod/siege-d484db9c-9dkgd -c siege -n outerpark -- /bin/bash
```
- 부하테스트 동시사용자 100명 60초 동안 공연예약 수행
```
siege -c100 -t60S -r10 -v --content-type "application/json" 'http://reservation:8080/reservations POST {"musicalId": "1003", "seats":1}'
```
- 부하 발생하여 CB가 발동하여 요청 실패처리하였고, 밀린 부하가 musical에서 처리되면서 다시 reservation 받기 시작
![image](https://user-images.githubusercontent.com/84000848/122355980-52b71280-cf8d-11eb-9d48-d9848d7189bc.png)

- 레포트결과
![image](https://user-images.githubusercontent.com/84000848/122356067-68c4d300-cf8d-11eb-9186-2dc33ebc806d.png)
서킷브레이킹 동작확인완료


# Autoscale(HPA)
- 오토스케일 테스트를 위해 리소스 제한설정 함
- reservation/kubernetes/deployment.yml 설정

```
resources:
	limits:
		cpu : 500m
	requests: 
		cpu : 200m
```
- 예약 시스템에 대한 replica 를 동적으로 늘려주도록 HPA 를 설정한다. 설정은 CPU 사용량이 15프로를 넘어서면 replica 를 10개까지 늘려준다
```
kubectl autoscale deploy reservation --min=1 --max=10 --cpu-percent=15 -n outerpark
```
![image](https://user-images.githubusercontent.com/84000848/122361127-edb1eb80-cf91-11eb-93ff-2c386af48961.png)

- 부하테스트 동시사용자 200명 120초 동안 공연예약 수행
```
siege -c200 -t120S -r10 -v --content-type "application/json" 'http://reservation:8080/reservations POST {"musicalId": "1003", "seats":1}'
```

- 최초수행 결과
![image](https://user-images.githubusercontent.com/84000848/122360142-21d8dc80-cf91-11eb-9868-85dffcc21309.png)

- 오토스케일 모니터링 수행
```
kubectl get deploy reservation -w -n outerpark
```
![image](https://user-images.githubusercontent.com/84000848/122361571-55683680-cf92-11eb-802b-28f47fdada7b.png)

- 부하테스트 재수행 시 Availability가 높아진 것을 확인
![image](https://user-images.githubusercontent.com/84000848/122361773-86e10200-cf92-11eb-9ab7-c8f62b519174.png)

-  replica 를 10개 까지 늘어났다가 부하가 적어져서 다시 줄어드는걸 확인 가능 함
![image](https://user-images.githubusercontent.com/84000848/122361938-ad06a200-cf92-11eb-9a55-35f9b6ceefe0.png)

# Self-healing (Liveness Probe)
- musical 서비스 정상 확인
![image](https://user-images.githubusercontent.com/84000848/122398259-adb03000-cfb4-11eb-9f49-5cf7018b81d4.png)

- musical의 deployment.yml 에 Liveness Probe 옵션 변경하여 계속 실패하여 재기동 되도록 yml 수정
```
#          livenessProbe:
#            httpGet:
#              path: '/actuator/health'
#              port: 8080
#            initialDelaySeconds: 120
#            timeoutSeconds: 2
#            periodSeconds: 5
#            failureThreshold: 5
          livenessProbe:
            tcpSocket:
              port: 8081
            initialDelaySeconds: 5
            periodSeconds: 5	
```
![image](https://user-images.githubusercontent.com/84000848/122398788-2dd69580-cfb5-11eb-91ce-bc82d7cf66a1.png)

-musical pod에 liveness가 적용된 부분 확인

![image](https://user-images.githubusercontent.com/84000848/122400529-c4578680-cfb6-11eb-8d06-a54f37ced872.png)

-musical 서비스의 liveness가 발동되어 7번 retry 시도 한 부분 확인
![image](https://user-images.githubusercontent.com/84000848/122401681-c66e1500-cfb7-11eb-9417-4ff189919f62.png)

# Zero-downtime deploy(Readiness Probe)
- Zero-downtime deploy를 위해 Autoscale 및 CB 설정 제거 
- readiness 옵션이 없는 reservation 배포
- seige로 부하 준비 후 실행 
- seige로 부하 실행 중 reservation 새로운 버전의 이미지로 교체
- readiness 옵션이 없는 경우 배포 중 서비스 요청처리 실패

![image](https://user-images.githubusercontent.com/84000848/122414855-69c42780-cfc2-11eb-8955-30e623e721c6.png)

- deployment.yml에 readiness 옵션을 추가

![image](https://user-images.githubusercontent.com/84000848/122416039-5d8c9a00-cfc3-11eb-84b1-9eb4ce1b6e9d.png)

-readiness적용된 deployment.yml 적용
```
kubectl apply -f kubernetes/deployment.yml
```
-새로운 버전의 이미지로 교체
```
az acr build --registry outerparkskacr --image outerparkskacr.azurecr.io/reservation:v3 .
kubectl set image deploy reservation reservation=outerparkskacr.azurecr.io/reservation:v3 -n outerpark
```
-기존 버전과 새 버전의 store pod 공존 중

![image](https://user-images.githubusercontent.com/84000848/122417105-36829800-cfc4-11eb-9849-054cf58119f2.png)

-Availability: 100.00 % 확인

![image](https://user-images.githubusercontent.com/84000848/122417302-5a45de00-cfc4-11eb-87d1-cc7482113a33.png)

# Config Map
apllication.yml 설정

-default부분

![image](https://user-images.githubusercontent.com/84000848/122422699-51570b80-cfc8-11eb-9cb9-f0fe332fb26a.png)

-docker 부분

![image](https://user-images.githubusercontent.com/84000848/122422842-70559d80-cfc8-11eb-8a0f-8bf10957140f.png)

Deployment.yml 설정

![image](https://user-images.githubusercontent.com/84000848/122423101-a3982c80-cfc8-11eb-821f-8c3aad8be16f.png)

config map 생성 후 조회
```
kubectl create configmap apiurl --from-literal=url=http://musical:8080 -n outerpark
```

![image](https://user-images.githubusercontent.com/84000848/122423850-346f0800-cfc9-11eb-90d8-9cb6c55bec21.png)

- 설정한 url로 주문 호출
```
http POST http://reservation:8080/reservations musicalId=1001 seats=10
```
![image](https://user-images.githubusercontent.com/84000848/122424027-5a94a800-cfc9-11eb-8fa9-363b80e6b899.png)

-configmap 삭제 후 app 서비스 재시작

```
kubectl delete configmap apiurl -n outerpark
kubectl get pod/reservation-57d8f8c4fd-74csz -n outerpark -o yaml | kubectl replace --force -f-
```

![image](https://user-images.githubusercontent.com/84000848/122424266-89ab1980-cfc9-11eb-8683-ac313e971ed6.png)


-configmap 삭제된 상태에서 주문 호출

![image](https://user-images.githubusercontent.com/84000848/122423447-e3f7aa80-cfc8-11eb-8760-6df5eb08f039.png)

![image](https://user-images.githubusercontent.com/84000848/122423364-d3dfcb00-cfc8-11eb-8b35-9145c00659b9.png)


