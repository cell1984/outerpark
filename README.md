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
![image](https://user-images.githubusercontent.com/84000848/122335324-0dd3b180-cf76-11eb-967a-6ddd4c7aaeaa.png)

- deployment.yml을 사용하여 배포 (reservation의 deployment.yml 추가)
![image](https://user-images.githubusercontent.com/84000848/122332320-2d1c1000-cf71-11eb-8766-b494f157f247.png)
- deployment.yml로 서비스 배포
```
kubectl apply -f kubernetes/deployment.yml
```


