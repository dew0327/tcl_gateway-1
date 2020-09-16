# Restaurant 

음식을 주문하고 요리하여 배달하는 현황을 확인 할 수 있는 CNA의 개발

# Table of contents

- [Restaurant](#---)
  - [서비스 시나리오](#서비스-시나리오)
  - [분석/설계](#분석설계)
  - [구현:](#구현-)
    - [DDD 의 적용](#ddd-의-적용)
    - [폴리글랏 퍼시스턴스](#폴리글랏-퍼시스턴스)
    - [폴리글랏 프로그래밍](#폴리글랏-프로그래밍)
    - [동기식 호출 과 Fallback 처리](#동기식-호출-과-Fallback-처리)
    - [비동기식 호출 과 Eventual Consistency](#비동기식-호출-과-Eventual-Consistency)
  - [운영](#운영)
    - [CI/CD 설정](#cicd설정)
    - [동기식 호출 / 서킷 브레이킹 / 장애격리](#동기식-호출-서킷-브레이킹-장애격리)
    - [오토스케일 아웃](#오토스케일-아웃)
    - [무정지 재배포](#무정지-재배포)
  - [신규 개발 조직의 추가](#신규-개발-조직의-추가)


# 서비스 시나리오

음식을 주문하고, 요리현황 및 배달현황을 조회

## 기능적 요구사항

1. 고객이 주문을 하면 주문정보를 바탕으로 요리가 시작된다.
1. 요리가 완료되면 배달이 시작된다. 
1. 고객이 주문취소를 하게 되면 요리가 취소된다.
1. 고객 주문 시 제고가 없을 경우 주문이 취소된다. 
1. 고객은 Mypage를 통해, 주문과 요리, 배달의 전체 상황을 조회할수 있다.

## 비기능적 요구사항
1. 트랜잭션
    1. 주문 취소시 요리취소가 함께 처리되도록 한다. - Sync 호출 
    1. 제고 부족시 주문이 함께 취소 되도록 해야 한다.
1. 장애격리
    1. 주문시스템이 과중되면 사용자를 잠시동안 받지 않고 잠시후에 주문하도록 유도한다 - Circuit breaker, fallback
1. 성능
    1. 고객이 mypage를 통해 주문 - 요리 - 배달의 전체 상황을 조회할수 있다. - CQRS


# 분석/설계

## Event Storming 결과
* MSAEz 로 모델링한 이벤트스토밍 결과 : http://www.msaez.io/#/storming/t5Z5EXdDP0UOZDvGzeNH61hF8qG3/mine/52e31337a76ddeacc1d288ea11e24158/-MH4jm58lJNE_9tgT82F
![EventStorming_Restaurant](https://user-images.githubusercontent.com/54210936/93179266-66201d80-f770-11ea-9530-aae4dfef7e4d.png)

### 이벤트 도출
1. 주문됨
1. 주문취소됨
1. 요리재고체크됨
1. 요리완료
1. 배달


### 어그리게잇으로 묶기

  * 고객의 주문(Order), 식당의 요리(Cook), 배달(Delivery)은 그와 연결된 command와 event 들에 의하여 트랙잭션이 유지되어야 하는 단위로 그들끼리 묶어 줌 

### Policy 부착 

### Policy와 컨텍스트 매핑 (점선은 Pub/Sub, 실선은 Req/Res)

### 기능적 요구사항 검증
 * 고객이 메뉴를 주문한다.(ok)
 * 주문된 주문정보를 레스토랑으로 전달한다.(ok)
 * 주문정보를 바탕으로 요리가 시작된다.(ok)
 * 요리가 완료되면 배달이 시작된다.(ok)
 * 고객은 본인의 주문을 취소할 수 있다.(ok)
 * 주문이 취소되면 요리를 취소한다.(ok)
 * 주문이 취소되면, 요리취소 내용을 고객에게 전달한다.(ok)
 * 고객이 주문 시 재고량을 체크한다.(ok)
 * 재고가 없을 경우 주문이 취소된다.(ok)
 * 고객은 Mypage를 통해, 주문과 요리, 배달의 전체 상황을 조회할수 있다.(ok)



# 구현:

분석/설계 단계에서 도출된 아키텍처에 따라, 각 BC별로 마이크로서비스들을 스프링부트 + JAVA로 구현하였다. 각 마이크로서비스들은 Kafka와 RestApi로 연동되며 스프링부트의 내부 H2 DB를 사용한다.


## DDD 의 적용

- 각 서비스내에 도출된 핵심 Aggregate Root 객체를 Entity 로 선언하였다: (예시는 주문- Order 마이크로서비스).

```
package myProject_LSP;

import javax.persistence.*;
import org.springframework.beans.BeanUtils;
import java.util.List;

@Entity
@Table(name="Order_table")
public class Order {

    @Id
    @GeneratedValue(strategy=GenerationType.AUTO)
    private Long id;
    private Integer restaurantId;
    private Integer restaurantMenuId;
    private Integer customerId;
    private Integer qty;
    private Long modifiedDate;
    private String status;
    
    ....
}
```
- JPA를 활용한 Repository Pattern을 적용하여 이후 데이터소스 유형이 변경되어도 별도의 처리 없이 사용 가능한 Spring Data REST 의 RestRepository 를 적용하였다
```
package myProject_LSP;
import org.springframework.data.repository.PagingAndSortingRepository;
public interface OrderRepository extends PagingAndSortingRepository<Order, Long>{

}
```


## 동기식 호출 과 Fallback 처리

분석단계에서의 조건 중 하나로 주문->취소 간의 호출은 트랜잭션으로 처리. 호출 프로토콜은 Rest Repository의 REST 서비스를 FeignClient 를 이용하여 호출(Req/Res 사용).
- 요리(cook) 서비스를 호출하기 위하여 Stub과 (FeignClient) 를 이용하여 Service 대행 인터페이스 (Proxy) 를 구현 

```
@FeignClient(name="cook", url="${api.url.cook}")
public interface CancellationService {

  @RequestMapping(method= RequestMethod.GET, path="/cancellations")
  public void cancel(@RequestBody Cancellation cancellation);

}
```

- 주문이 취소 될 경우 Cancellation 현황에 취소 내역을 접수한다.
```
@PrePersist
public void onPrePersist(){
   CookCancelled cookCancelled = new CookCancelled();
   BeanUtils.copyProperties(this, cookCancelled);
   cookCancelled.setStatus("COOK : ORDER CANCELED");
   cookCancelled.publishAfterCommit();
```



## 비동기식 호출 / 장애격리 

주문 접수 및 배달 접수, 재고부족으로 인한 주문 취소는 비동기식으로 처리하여 시스템 상황에 따라 접수 및 취소가 블로킹 되지 않도록 처리 한다. 
요리 단계 접수시에는 재고를 체크하고 재고가 부족할 경우 주문단계로 비동기식 요리 불가 발행(publish). SAGA Pattern 사용됨.
 
```
@Entity
@Table(name="Cook_table")
public class Cook {
    private boolean flowchk = true;
    ....
    @PostPersist
    public void onPostPersist(){
        if(flowchk) {   // 요리를 할 수 있는 재고가 있을 때 요리를 시작한다
            Cooked cooked = new Cooked();
            BeanUtils.copyProperties(this, cooked);
            this.setStatus("COOK : ORDER RECEIPT");
            this.qty--;
            cooked.publishAfterCommit();
        }else{
            CookQtyChecked cookQtyChecked = new CookQtyChecked();
            BeanUtils.copyProperties(this, cookQtyChecked);
            cookQtyChecked.publishAfterCommit();
        }
    }

    @PrePersist
    public void onPrePersist(){
        // 요리를 할 수 있는 재고가 없을 때 요리를 시작한다
        if(this.getQty() <= 0) {
            this.setStatus("COOK : QTY OVER");
            flowchk = false;
        }
    }
}
```

# 운영

## CI/CD 설정

  * 각 구현체들은 github의 각각의 source repository 에 구성
  * AWS codebuild를 설정하여 github이 업데이트 되면 자동으로 빌드 및 배포 작업이 이루어짐
  
## 서킷 브레이킹 / 오토스케일(HPA)

* 서킷 브레이킹 :
주문이 과도할 경우 CB 를 통하여 장애격리. 500 에러가 5번 발생하면 10분간 CB 처리하여 100% 접속 차단
```
# AWS codebuild에 설정(https://github.com/dew0327/final-cna-order/blob/master/buildspec.yml)
 http:
   http1MaxPendingRequests: 1   # 연결을 기다리는 request 수를 1개로 제한 (Default 
   maxRequestsPerConnection: 1  # keep alive 기능 disable
 outlierDetection:
  consecutiveErrors: 1          # 5xx 에러가 5번 발생하면
  interval: 1s                  # 1초마다 스캔 하여
  baseEjectionTime: 10m         # 10분 동안 circuit breaking 처리   
  maxEjectionPercent: 100       # 100% 로 차단
```

* 오토스케일(HPA) :
CPU사용률 10% 초과 시 replica를 5개까지 확장해준다. 상용에서는 70%로 세팅하지만 여기에서는 기능적용 확인을 위해 수치를 조절.
```
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: skcchpa-order
  namespace: teamc
  spec:
    scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: $_PROJECT_NAME                # order (주문) 서비스 HPA 설정
    minReplicas: 3                      # 최소 3개
    maxReplicas: 5                      # 최대 5개
    targetCPUUtilizationPercentage: 10  # cpu사용율 10프로 초과 시 
```    
* 부하테스트(Siege)를 활용한 부하 적용 후 서킷브레이킹 / 오토스케일 내역을 확인한다.
![HPA, Circuit Breaker  SEIGE_STATUS](https://user-images.githubusercontent.com/54210936/93168766-9ced3800-f75e-11ea-9d6b-fdf37591b97a.jpg)
![HPA  TOBE_STATUS](https://user-images.githubusercontent.com/54210936/93167897-95c52a80-f75c-11ea-8f0e-51a94332141b.jpg)


## 무정지 재배포(ZeroDowntime Deploy, Readiness Probe)

* 무정지 배포를 위해 ECR 이미지를 업데이트 하고 이미지 체인지를 시도 함. Github에 소스가 업데이트 되면 자동으로 AWS CodeBuild에서 컴파일 하여 이미지를 ECR에 올리고 EKS에 반영.
  이후 아래 옵션에 따라 무정지 배포 적용 된다.
  

```
# AWS codebuild에 설정(https://github.com/dew0327/final-cna-cook/blob/master/buildspec.yml)
  spec:
    replicas: 5
    minReadySeconds: 10   # 최소 대기 시간 10초
    strategy:
      type: RollingUpdate
      rollingUpdate:
      maxSurge: 1         # 1개씩 업데이트 진행
      maxUnavailable: 0   # 업데이트 프로세스 중에 사용할 수 없는 최대 파드의 수

```

- 새버전으로의 배포 시작(V3로 배포)
![ZeroDownTime  console - pod change status](https://user-images.githubusercontent.com/54210936/93277970-4c2d1c00-f7fe-11ea-87ce-82cdd77e84ac.jpg)

- siege를 이용한 부하 적용. Availability가 100% 미만으로 떨어짐. 쿠버네티스가 새로 올려진 서비스를 Ready 상태로 인식하여 서비스 유입을 진행 하였음. Readiness Probe 설정하여 조치 필요.
![ZeroDownTime  SEIGE_STATUS](https://user-images.githubusercontent.com/54210936/93277995-6109af80-f7fe-11ea-9ebf-5de918c150cc.jpg)

- 새버전 배포 확인(V3 적용)
![ZeroDownTime  console - pod describe](https://user-images.githubusercontent.com/54210936/93278015-6d8e0800-f7fe-11ea-82d1-dc80b96b601c.jpg)


- Readiness Probe 설정을 통한 ZeroDownTime 설정.
```
  readinessProbe:
    tcpSocket:
      port: 8080
      initialDelaySeconds: 15      # 서비스 어플 기동 후 15초 뒤 시작
      periodSeconds: 20            # 20초 주기로 readinessProbe 실행 
```
![ZeroDownTime  SEIGE_STATUS_read](https://user-images.githubusercontent.com/54210936/93278989-1473a380-f801-11ea-8140-f7edbc2c9b6f.jpg)




