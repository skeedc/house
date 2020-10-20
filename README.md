
- 체크포인트 : https://workflowy.com/s/assessment-check-po/T5YrzcMewfo4J6LW


# Table of contents

- [숙소예약](#---)
  - [서비스 시나리오](#서비스-시나리오)
  - [체크포인트](#체크포인트)
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



기능적 요구사항
1.	관리자는 숙소를 등록할 수 있다.
2.	고객이 숙소를 선택해 예약하면 결제가 진행된다.
3.	예약이 결제되면 숙소의 예약 가능 여부가 변경된다.
4.	숙소가 예약 불가 상태로 변경되면 예약이 확정된다.
5.	고객은 예약을 취소할 수 있다.
6.	예약이 취소되면 결제가 취소되고 숙소의 예약 가능 여부가 변경된다.
7.	고객은 숙소 예약가능여부를 확인할 수 있다.




비기능적 요구사항
1. 트랜잭션
    1. 결제가 완료 되지 않은 예약 건은 예약이 성립되지 않는다.  Sync 호출
    1. 예약과 결제는 동시에 진행된다.  Sync 호출
    1. 예약 취소와 결제 취소는 동시에 진행된다.  Sync 호출
1. 장애격리
    1. 숙소 시스템이 수행되지 않더라도 예약 / 결제는365일 24시간 받을 수 있어야 한다.  Async 호출 (event-driven)
    1. 숙소 시스템이 과중 되면 예약 / 결제를 받지 않고 결제 취소를 잠시 후에 하도록 유도한다.  Circuit breaker, fallback
    1. 제가 취소되면 숙소의 예약 취소가 확정되고, 숙소의 예약 가능 여부가 변경된다.  Circuit breaker, fallback
1. 성능
    1. 고객은 숙소 예약 가능 여부를 확인할 수 있다.  CQRS
    1. 예약/결제 취소 정보가 변경 될 때마다 숙소 예약 가능 여부가 변경 될 수 있어야 한다.  Event Driven





# 체크포인트

- 분석 설계


  - 이벤트스토밍: 
    - 스티커 색상별 객체의 의미를 제대로 이해하여 헥사고날 아키텍처와의 연계 설계에 적절히 반영하고 있는가?
    - 각 도메인 이벤트가 의미있는 수준으로 정의되었는가?
    - 어그리게잇: Command와 Event 들을 ACID 트랜잭션 단위의 Aggregate 로 제대로 묶었는가?
    - 기능적 요구사항과 비기능적 요구사항을 누락 없이 반영하였는가?    

  - 서브 도메인, 바운디드 컨텍스트 분리
    - 팀별 KPI 와 관심사, 상이한 배포주기 등에 따른  Sub-domain 이나 Bounded Context 를 적절히 분리하였고 그 분리 기준의 합리성이 충분히 설명되는가?
      - 적어도 3개 이상 서비스 분리
    - 폴리글랏 설계: 각 마이크로 서비스들의 구현 목표와 기능 특성에 따른 각자의 기술 Stack 과 저장소 구조를 다양하게 채택하여 설계하였는가?
    - 서비스 시나리오 중 ACID 트랜잭션이 크리티컬한 Use 케이스에 대하여 무리하게 서비스가 과다하게 조밀히 분리되지 않았는가?
  - 컨텍스트 매핑 / 이벤트 드리븐 아키텍처 
    - 업무 중요성과  도메인간 서열을 구분할 수 있는가? (Core, Supporting, General Domain)
    - Request-Response 방식과 이벤트 드리븐 방식을 구분하여 설계할 수 있는가?
    - 장애격리: 서포팅 서비스를 제거 하여도 기존 서비스에 영향이 없도록 설계하였는가?
    - 신규 서비스를 추가 하였을때 기존 서비스의 데이터베이스에 영향이 없도록 설계(열려있는 아키택처)할 수 있는가?
    - 이벤트와 폴리시를 연결하기 위한 Correlation-key 연결을 제대로 설계하였는가?

  - 헥사고날 아키텍처
    - 설계 결과에 따른 헥사고날 아키텍처 다이어그램을 제대로 그렸는가?
    
- 구현
  - [DDD] 분석단계에서의 스티커별 색상과 헥사고날 아키텍처에 따라 구현체가 매핑되게 개발되었는가?
    - Entity Pattern 과 Repository Pattern 을 적용하여 JPA 를 통하여 데이터 접근 어댑터를 개발하였는가
    - [헥사고날 아키텍처] REST Inbound adaptor 이외에 gRPC 등의 Inbound Adaptor 를 추가함에 있어서 도메인 모델의 손상을 주지 않고 새로운 프로토콜에 기존 구현체를 적응시킬 수 있는가?
    - 분석단계에서의 유비쿼터스 랭귀지 (업무현장에서 쓰는 용어) 를 사용하여 소스코드가 서술되었는가?
  - Request-Response 방식의 서비스 중심 아키텍처 구현
    - 마이크로 서비스간 Request-Response 호출에 있어 대상 서비스를 어떠한 방식으로 찾아서 호출 하였는가? (Service Discovery, REST, FeignClient)
    - 서킷브레이커를 통하여  장애를 격리시킬 수 있는가?
  - 이벤트 드리븐 아키텍처의 구현
    - 카프카를 이용하여 PubSub 으로 하나 이상의 서비스가 연동되었는가?
    - Correlation-key:  각 이벤트 건 (메시지)가 어떠한 폴리시를 처리할때 어떤 건에 연결된 처리건인지를 구별하기 위한 Correlation-key 연결을 제대로 구현 하였는가?
    - Message Consumer 마이크로서비스가 장애상황에서 수신받지 못했던 기존 이벤트들을 다시 수신받아 처리하는가?
    - Scaling-out: Message Consumer 마이크로서비스의 Replica 를 추가했을때 중복없이 이벤트를 수신할 수 있는가
    - CQRS: Materialized View 를 구현하여, 타 마이크로서비스의 데이터 원본에 접근없이(Composite 서비스나 조인SQL 등 없이) 도 내 서비스의 화면 구성과 잦은 조회가 가능한가?

  - 폴리글랏 플로그래밍
    - 각 마이크로 서비스들이 하나이상의 각자의 기술 Stack 으로 구성되었는가?
    - 각 마이크로 서비스들이 각자의 저장소 구조를 자율적으로 채택하고 각자의 저장소 유형 (RDB, NoSQL, File System 등)을 선택하여 구현하였는가?
  - API 게이트웨이
    - API GW를 통하여 마이크로 서비스들의 집입점을 통일할 수 있는가?
    - 게이트웨이와 인증서버(OAuth), JWT 토큰 인증을 통하여 마이크로서비스들을 보호할 수 있는가?
- 운영
  - SLA 준수
    - 셀프힐링: Liveness Probe 를 통하여 어떠한 서비스의 health 상태가 지속적으로 저하됨에 따라 어떠한 임계치에서 pod 가 재생되는 것을 증명할 수 있는가?
    - 서킷브레이커, 레이트리밋 등을 통한 장애격리와 성능효율을 높힐 수 있는가?
    - 오토스케일러 (HPA) 를 설정하여 확장적 운영이 가능한가?
    - 모니터링, 앨럿팅: 
  - 무정지 운영 CI/CD (10)
    - Readiness Probe 의 설정과 Rolling update을 통하여 신규 버전이 완전히 서비스를 받을 수 있는 상태일때 신규버전의 서비스로 전환됨을 siege 등으로 증명 
    - Contract Test :  자동화된 경계 테스트를 통하여 구현 오류나 API 계약위반를 미리 차단 가능한가?


# 분석/설계


## AS-IS 조직 (Horizontally-Aligned)
  ![image](https://user-images.githubusercontent.com/487999/79684144-2a893200-826a-11ea-9a01-79927d3a0107.png)

## TO-BE 조직 (Vertically-Aligned)
  ![image](https://user-images.githubusercontent.com/487999/79684159-3543c700-826a-11ea-8d5f-a3fc0c4cad87.png)


## Event Storming 결과
* MSAEz 로 모델링한 이벤트스토밍 결과:  http://msaez.io/#/storming/nZJ2QhwVc4NlVJPbtTkZ8x9jclF2/every/a77281d704710b0c2e6a823b6e6d973a/-M5AV2z--su_i4BfQfeF


### 이벤트 도출
![image](https://user-images.githubusercontent.com/70302894/96394182-54f97f00-11fc-11eb-9100-41032900dbcf.JPG)



### 어그리게잇으로 묶기 / 액터, 커맨드 부착하여 읽기 좋게
![image](https://user-images.githubusercontent.com/70302894/96394196-59259c80-11fc-11eb-9127-461cd5522b87.JPG)

    - 숙소 예약, 결제, 숙소 관리 등은 그와 연결된 command 와 event 들에 의하여 트랜잭션이 유지되어야 하는 단위로 묶어준다.

### 바운디드 컨텍스트로 묶기

![image](https://user-images.githubusercontent.com/70302894/96394197-59be3300-11fc-11eb-9342-1106528fcb0c.JPG)

    - 도메인 서열 분리 
        - 예약 : 고객 예약 오류를 최소화 한다. (Core)
        - 결제 : 결제 오류를 최소화 한다. (Supporting)
        - 숙소 : 숙소 예약 상태 오류를 최소화 한다. (Supporting)

### 폴리시 부착 (괄호는 수행주체, 폴리시 부착을 둘째단계에서 해놔도 상관 없음. 전체 연계가 초기에 드러남)

![image](https://user-images.githubusercontent.com/70302894/96394198-5a56c980-11fc-11eb-912d-df12c1954bed.JPG)

### 폴리시의 이동과 컨텍스트 매핑 (점선은 Pub/Sub, 실선은 Req/Resp)

![image](https://user-images.githubusercontent.com/70302894/96394199-5a56c980-11fc-11eb-8f6f-9246f8ded1e2.JPG)

### 완성된 1차 모형

![image](https://user-images.githubusercontent.com/70302894/96394200-5aef6000-11fc-11eb-807d-86c87e834f44.jpg)



### 1차 완성본에 대한 기능적/비기능적 요구사항을 커버하는지 검증

![image](https://user-images.githubusercontent.com/70302894/96394204-5c208d00-11fc-11eb-9ac7-f15fb95d5fac.jpg)

    - 고객이 숙소 예약 가능 여부를 확인한다.(?)
    - 고객이 숙소를 선택해 예약을 진행한다. (OK)
    - 예약 시 자동으로 결제가 진행된다. (OK)
    - 결제가 성공하면 숙소가 예약불가 상태가 된다. (OK)
    - 숙소 상태 변경 시 예약이 확정상태가 된다. (OK)    
    






![image](https://user-images.githubusercontent.com/70302894/96394205-5cb92380-11fc-11eb-8801-afefc12aa9b1.jpg)


    - 고객이 예약/결제를 취소한다. (OK)
    - 예약/결제 취소 시 자동 예약/결제 취소된다. (OK)
    - 결제가 취소되면 숙소가 예약가능 상태가 된다. (OK)






### 모델 수정

![image](https://user-images.githubusercontent.com/70302894/96394207-5d51ba00-11fc-11eb-80d9-1d5bb4356b1a.JPG)
    
    - View Model 추가
    - 수정된 모델은 모든 요구사항을 커버함.


### 비기능 요구사항에 대한 검증

![image](https://user-images.githubusercontent.com/70302894/96396179-8aed3200-1201-11eb-9ef9-ff872125f76e.jpg)


    - 1. 숙소 등록 서비스를 예약/결제 서비스와 격리하여 숙소 등록 서비스 장애 시에도 예약이 가능
    - 2. 숙소가 예약 불가 상태일 경우 예약 확정이 불가함
    - 3. 먼저 결제가 이루어진 숙소에 대해서는 예약을 불가 하도록 함.    









## 헥사고날 아키텍처 다이어그램 도출
    
![image](https://user-images.githubusercontent.com/70302894/96400680-5763d500-120c-11eb-85c4-bf826e22cecf.JPG)


    - Chris Richardson, MSA Patterns 참고하여 Inbound adaptor와 Outbound adaptor를 구분함
    - 호출관계에서 PubSub 과 Req/Resp 를 구분함
    - 서브 도메인과 바운디드 컨텍스트의 분리:  각 팀의 KPI 별로 아래와 같이 관심 구현 스토리를 나눠가짐


# 구현:

분석/설계 단계에서 도출된 헥사고날 아키텍처에 따라, 각 BC별로 대변되는 마이크로 서비스들을 스프링부트로 구현하였다. 구현한 각 서비스를 로컬에서 실행하는 방법은 아래와 같다 (각자의 포트넘버는 8081 ~ 808n 이다)

```
cd book
mvn spring-boot:run

cd payment
mvn spring-boot:run 

cd house
mvn spring-boot:run  

cd mypage
mvn spring-boot:run  
```

## DDD 의 적용

- 각 서비스 내에 도출된 핵심 Aggregate Root 객체를 Entity 로 선언
```
package housebook;

import org.springframework.beans.BeanUtils;

import javax.persistence.*;

@Entity
@Table(name="Payment_table")
public class Payment {

    @Id
    @GeneratedValue(strategy=GenerationType.AUTO)
    private Long id;
    private Long houseId;
    .../... 중략  .../...
    private Double housePrice;

    public Long getId() {
        return id;
    }
    public void setId(Long id) {
        this.id = id;
    }
    
    public Long gethouseId() {
        return houseId;
    }
    public void sethouseId(Long houseId) {
        this.houseId = houseId;
    }
    .../... 중략  .../...

}
```



- Entity Pattern 과 Repository Pattern 을 적용하여 JPA 를 통하여 다양한 데이터소스 유형 (RDB or NoSQL) 에 대한 별도의 처리가 없도록    
데이터 접근 어댑터를 자동 생성하기 위하여 Spring Data REST 의 RestRepository 를 적용
```
package housebook;
import org.springframework.data.repository.PagingAndSortingRepository;
    public interface PaymentRepository extends PagingAndSortingRepository<Payment, Long>{

}
```
   
   
---
#### 적용 후 REST API 의 테스트

1. 숙소1 등록
``` http http://localhost:8083/houses id=1 status=WAITING houseName=신라호텔 housePrice=200000 ```

<img width="457" alt="숙소등록1" src="https://user-images.githubusercontent.com/54618778/96413666-f0074e80-1226-11eb-88ca-1278f0077fc9.png">


2. 숙소2 등록
``` http http://localhost:8083/houses id=2 status=WAITING houseName=SK펜션 housePrice=500000 ```

<img width="463" alt="숙소등록2" src="https://user-images.githubusercontent.com/54618778/96413673-f269a880-1226-11eb-9b1e-62ad3f98cd30.png">


3. 숙소1 예약 
``` http POST http://localhost:8081/books id=1 status=BOOKED houseId=1 bookDate=20201016 housePrice=200000 ```

<img width="448" alt="숙소예약1" src="https://user-images.githubusercontent.com/54618778/96413678-f4336c00-1226-11eb-8665-1ed312adbed1.png">


4. 숙소2 예약
``` http POST http://localhost:8081/books id=2 status=BOOKED houseId=2 bookDate=20201016 housePrice=500000 ```

<img width="450" alt="숙소예약2" src="https://user-images.githubusercontent.com/54618778/96413681-f4cc0280-1226-11eb-8f6c-f3d0e03c0456.png">


5. 숙소2 예약 취소
``` http http://localhost:8081/books id=2 status=BOOK_CANCELED houseId=2 ```

<img width="451" alt="숙소취소" src="https://user-images.githubusercontent.com/54618778/96413687-f5fd2f80-1226-11eb-87fd-2f8c7ea695c5.png">


6. 예약 보기
```http localhost:8081/books ```

<img width="573" alt="예약상태보기" src="https://user-images.githubusercontent.com/54618778/96413688-f695c600-1226-11eb-9659-11ba9322f19d.png">


7. 숙소 보기 
``` http localhost:8083/houses ```

<img width="591" alt="숙소상태보기" src="https://user-images.githubusercontent.com/54618778/96413674-f3023f00-1226-11eb-830e-d6ab51cb745b.png">


8. 숙소 예약된 상태 (MyPage)
``` http localhost:8084/mypages/7 ```

<img width="569" alt="숙소예약된상태" src="https://user-images.githubusercontent.com/54618778/96413683-f5649900-1226-11eb-8ec6-a384afb76ead.png">


9. 숙소 예약취소된 상태 (MyPage)
``` http localhost:8084/mypages/9 ```

<img width="545" alt="MyPage_예약취소" src="https://user-images.githubusercontent.com/54618778/96413690-f72e5c80-1226-11eb-9a1e-72df208097fc.png">


---


## 폴리글랏 퍼시스턴스

각 마이크로서비스는 별도의 H2 DB를 가지고 있으며 CQRS를 위한 Mypage에서는 H2가 아닌 HSQLDB를 적용하였다.

```
# Mypage의 pom.xml에 dependency 추가
<!-- 
		<dependency>
			<groupId>com.h2database</groupId>
			<artifactId>h2</artifactId>
			<scope>runtime</scope>
		</dependency>
 -->
		<dependency>
		    <groupId>org.hsqldb</groupId>
		    <artifactId>hsqldb</artifactId>
		    <version>2.4.0</version>
		    <scope>runtime</scope>
		</dependency>


```





## 동기식 호출과 Fallback 처리
Book → Payment 간 호출은 동기식 일관성 유지하는 트랜잭션으로 처리.     
호출 프로토콜은 이미 앞서 Rest Repository 에 의해 노출되어있는 REST 서비스를 FeignClient 를 이용하여 호출.     

```
BookApplication.java.
import org.springframework.cloud.openfeign.EnableFeignClients;

@SpringBootApplication
@EnableBinding(KafkaProcessor.class)
@EnableFeignClients
public class BookApplication {
    protected static ApplicationContext applicationContext;
    public static void main(String[] args) {
        applicationContext = SpringApplication.run(BookApplication.class, args);
    }
}
```

FeignClient 방식을 통해서 Request-Response 처리.     
Feign 방식은 넷플릭스에서 만든 Http Client로 Http call을 할 때, 도메인의 변화를 최소화 하기 위하여 interface 로 구현체를 추상화.    
→ 실제 Request/Response 에러 시 Fegin Error 나는 것 확인   




- 예약 받은 직후(@PostPersist) 결제 요청함
```
-- Book.java
    @PostPersist
    public void onPostPersist(){
        Booked booked = new Booked();
        BeanUtils.copyProperties(this, booked);
        booked.publishAfterCommit();

        //Following code causes dependency to external APIs
        // it is NOT A GOOD PRACTICE. instead, Event-Policy mapping is recommended.

        housebook.external.Payment payment = new housebook.external.Payment();
        // mappings goes here
        
        payment.setBookId(booked.getId());
        payment.setHouseId(booked.getHouseId());
        ...// 중략 //...

        BookApplication.applicationContext.getBean(housebook.external.PaymentService.class)
            .paymentRequest(payment);

    }
```



- 동기식 호출에서는 호출 시간에 따른 타임 커플링이 발생하며, 결제 시스템이 장애가 나면 주문도 못받는다는 것을 확인함.   
```
Book -- (http request/response) --> Payment

# Payment 서비스 종료

# Book 등록
http http://localhost:8081/books id=1 status=BOOKED houseId=1 bookDate=20201016 housePrice=200000    #Fail!!!!
```
Payment를 종료한 시점에서 상기 Book 등록 Script 실행 시, 500 Error 발생.
("Could not commit JPA transaction; nested exception is javax.persistence.RollbackException: Error while committing the transaction")   
![](images/결제서비스_중지_시_예약시도.png)   


---
## 비동기식 호출 / 시간적 디커플링 / 장애격리 / 최종 (Eventual) 일관성 테스트

Payment가 이루어진 후에(PAID) House시스템으로 이를 알려주는 행위는 동기식이 아니라 비 동기식으로 처리.   
House 시스템의 처리를 위하여 결제주문이 블로킹 되지 않아도록 처리.   
이를 위하여 결제이력에 기록을 남긴 후에 곧바로 결제승인이 되었다는 도메인 이벤트를 카프카로 송출한다(Publish).   

- House 서비스에서는 PAID 이벤트에 대해서 이를 수신하여 자신의 정책을 처리하도록 PolicyHandler 를 구현한다:   
```
@Service
public class PolicyHandler{

    @Autowired
    HouseRepository houseRepository;
    
    @StreamListener(KafkaProcessor.INPUT)
    public void onStringEventListener(@Payload String eventString){

    }

    @StreamListener(KafkaProcessor.INPUT)
    public void wheneverPaid_Rent(@Payload Paid paid){
        if(paid.isMe()){
            System.out.println("##### listener Rent : " + paid.toJson());

            Optional<House> optional = houseRepository.findById(paid.getHouseId());
            House house = optional.get();
            house.setBookId(paid.getBookId());
            house.setStatus("RENTED");

            houseRepository.save(house);
        }
    }
```

- House 시스템은 주문/결제와 완전히 분리되어있으며, 이벤트 수신에 따라 처리되기 때문에, House 시스템이 유지보수로 인해 잠시 내려간 상태라도 주문을 받는데 문제가 없다:
```
# House Service 를 잠시 내려놓음 (ctrl+c)

#PAID 처리
http http://localhost:8082/payments id=1 status=PAID bookId=1 houseId=1 paymentDate=20201016 housePrice=200000 #Success!!

#결제상태 확인
http http://localhost:8082/payments  #제대로 Data 들어옴   

#House 서비스 기동
cd house
mvn spring-boot:run

#House 상태 확인
http http://localhost:8083/houses     # 제대로 kafka로 부터 data 수신 함을 확인
```


---



# 운영

## CI/CD 설정


각 구현체들은 각자의 source repository 에 구성되었고, 사용한 CI/CD 플랫폼은 AWS CodeBuild를 사용하였으며, pipeline build script 는 각 프로젝트 폴더 이하에 buildspec.yml 에 포함되었다.

![image](https://user-images.githubusercontent.com/70302894/96394196-59259c80-11fc-11eb-9127-461cd5522b87.JPG)

Webhook으로 연결되어 github에서 수정 시 혹은 codebuild에서 곧바로 빌드가 가능하다.

![image](https://user-images.githubusercontent.com/70302894/96580321-15718680-1313-11eb-9cac-e2407f7579d1.JPG)


## 동기식 호출 / 서킷 브레이킹 / 장애격리

* 서킷 브레이킹 프레임워크의 선택: Spring FeignClient + Hystrix 옵션을 사용하여 구현함

시나리오는 book -> payment 시의 연결을 RESTful Request/Response 로 연동하여 구현이 되어있고, 결제 요청이 과도할 경우 CB 를 통하여 장애격리.

- Hystrix 를 설정:  요청처리 쓰레드에서 처리시간이 610 밀리가 넘어서기 시작하여 어느정도 유지되면 CB 회로가 닫히도록 (요청을 빠르게 실패처리, 차단) 설정

![image](https://user-images.githubusercontent.com/70302894/96580900-f6bfbf80-1313-11eb-8210-4a4d96039f69.JPG)


- 피호출 서비스(결제:payment) 의 임의 부하 처리 - 400 밀리에서 증감 220 밀리 정도 왔다갔다 하게
```
# Payment.java (Entity)
    @PostPersist
    public void onPostPersist(){
        System.out.println("##### onPostPersist status = " + this.getStatus());
        
        try {
            Thread.currentThread().sleep((long) (400 + Math.random() * 220));
            System.out.println("##### SLEEP");

        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        
        if (this.getStatus().equals("BOOKED") || this.getStatus().equals("PAID")) {
            Paid paid = new Paid();
            BeanUtils.copyProperties(this, paid);
            paid.setStatus("PAID");
            paid.publishAfterCommit();
        }
    }
```

* 부하테스터 siege 툴을 통한 서킷 브레이커 동작 확인:
- 동시사용자 100명
- 60초 동안 실시

```
siege -c30 -t10S -v  --content-type "application/json" 'http://skccuser04-payment:8080/payments POST {"houseId":"1"}'

```
미적용 시, 모든 요청이 
![image](https://user-images.githubusercontent.com/70302894/96579636-1e158d00-1312-11eb-9a17-277b3caf3876.JPG)


데스티네이션 룰 적용
```
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: dr-payment
  namespace: istio-cb-ns
spec:
  host: payment
  trafficPolicy:
    connectionPool:
      http:
        http1MaxPendingRequests: 5
        maxRequestsPerConnection: 5
    outlierDetection:
      interval: 1s
      consecutiveErrors: 1
      baseEjectionTime: 10m
      maxEjectionPercent: 100
EOF
```

적용 후 부하테스트
![image](https://user-images.githubusercontent.com/70302894/96580900-f6bfbf80-1313-11eb-8210-4a4d96039f69.JPG)

서킷브레이커의 동작으로 미연결된 결과가 보임
![image](https://user-images.githubusercontent.com/70302894/96579639-1f46ba00-1312-11eb-8b13-1c552b108711.JPG)

78%정도의 요청 성공률 확인
![image](https://user-images.githubusercontent.com/70302894/96579632-1ce46000-1312-11eb-8c7a-8aff5c351056.JPG)

키알리 화면 캡쳐
![image](https://user-images.githubusercontent.com/70302894/96579638-1eae2380-1312-11eb-8b9e-c3e21aec7d75.JPG)

- 운영시스템은 죽지 않고 지속적으로 CB 에 의하여 적절히 회로가 열림과 닫힘이 벌어지면서 자원을 보호하고 있음을 보여줌. 하지만, 63.55% 가 성공하였고, 46%가 실패했다는 것은 고객 사용성에 있어 좋지 않기 때문에 Retry 설정과 동적 Scale out (replica의 자동적 추가,HPA) 을 통하여 시스템을 확장 해주는 후속처리가 필요.


- Retry 의 설정 (istio)

```
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: vs-order-network-rule
  namespace: istio-cb-ns
spec:
  hosts:
  - book
  http:
  - route:
    - destination:
        host: payment
    timeout: 3s
    retries:
      attempts: 3
      perTryTimeout: 2s
      retryOn: 5xx,retriable-4xx,gateway-error,connect-failure,refused-stream
EOF
```


- Availability 가 높아진 것을 확인 (siege)





### 오토스케일 아웃
앞서 CB 는 시스템을 안정되게 운영할 수 있게 해줬지만 사용자의 요청을 100% 받아들여주지 못했기 때문에 이에 대한 보완책으로 자동화된 확장 기능을 적용하고자 한다. 


- 결제서비스에 대한 replica 를 동적으로 늘려주도록 HPA 를 설정한다. 설정은 CPU 사용량이 15프로를 넘어서면 replica 를 10개까지 늘려준다:
```
kubectl autoscale deploy pay --min=1 --max=10 --cpu-percent=15
```
- CB 에서 했던 방식대로 워크로드를 2분 동안 걸어준다.
```
siege -c100 -t120S -r10 --content-type "application/json" 'http://localhost:8081/orders POST {"item": "chicken"}'
```
- 오토스케일이 어떻게 되고 있는지 모니터링을 걸어둔다:
```
kubectl get deploy pay -w
```
- 어느정도 시간이 흐른 후 (약 30초) 스케일 아웃이 벌어지는 것을 확인할 수 있다:
```
NAME    DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
pay     1         1         1            1           17s
pay     1         2         1            1           45s
pay     1         4         1            1           1m
:
```
- siege 의 로그를 보아도 전체적인 성공률이 높아진 것을 확인 할 수 있다. 
```
Transactions:		        5078 hits
Availability:		       92.45 %
Elapsed time:		       120 secs
Data transferred:	        0.34 MB
Response time:		        5.60 secs
Transaction rate:	       17.15 trans/sec
Throughput:		        0.01 MB/sec
Concurrency:		       96.02
```


## 무정지 재배포

* 먼저 무정지 재배포가 100% 되는 것인지 확인하기 위해서 Autoscaler 이나 Readiness Probe 미설정 시 무정지 재배포 가능여부 확인을 위해 buildspec.yml의 Readiness Probe 설정을 제거함

- seige 로 배포작업 직전에 워크로드를 모니터링 함.
```
siege -c100 -t120S -r10 --content-type "application/json" 'http://localhost:8081/orders POST {"item": "chicken"}'

** SIEGE 4.0.5
** Preparing 100 concurrent users for battle.
The server is now under siege...

HTTP/1.1 201     0.68 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     0.68 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     0.70 secs:     207 bytes ==> POST http://localhost:8081/orders
HTTP/1.1 201     0.70 secs:     207 bytes ==> POST http://localhost:8081/orders
:

```

- 새버전으로의 배포 시작
```
kubectl set image ...
```

- seige 의 화면으로 넘어가서 Availability 가 100% 미만으로 떨어졌는지 확인
```
Transactions:		        3078 hits
Availability:		       70.45 %
Elapsed time:		       120 secs
Data transferred:	        0.34 MB
Response time:		        5.60 secs
Transaction rate:	       17.15 trans/sec
Throughput:		        0.01 MB/sec
Concurrency:		       96.02

```
배포기간중 Availability 가 평소 100%에서 70% 대로 떨어지는 것을 확인. 원인은 쿠버네티스가 성급하게 새로 올려진 서비스를 READY 상태로 인식하여 서비스 유입을 진행한 것이기 때문. 이를 막기위해 Readiness Probe 를 설정함:

```
# deployment.yaml 의 readiness probe 의 설정:


kubectl apply -f kubernetes/deployment.yaml
```

- 동일한 시나리오로 재배포 한 후 Availability 확인:
```
Transactions:		        3078 hits
Availability:		       100 %
Elapsed time:		       120 secs
Data transferred:	        0.34 MB
Response time:		        5.60 secs
Transaction rate:	       17.15 trans/sec
Throughput:		        0.01 MB/sec
Concurrency:		       96.02

```

배포기간 동안 Availability 가 변화없기 때문에 무정지 재배포가 성공한 것으로 확인됨.

