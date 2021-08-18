# yanolza-team



구현:
분석/설계 단계에서 도출된 헥사고날 아키텍처에 따라, 각 BC별로 대변되는 마이크로 서비스들을 스프링부트와 파이선으로 구현하였다. 구현한 각 서비스를 로컬에서 실행하는 방법은 아래와 같다 (각자의 포트넘버는 8081 ~ 808n 이다)

cd customer
mvn spring-boot:run

cd order
mvn spring-boot:run 

cd payment
mvn spring-boot:run  

cd reservation
mvn spring-boot:run  


DDD 의 적용
각 서비스내에 도출된 핵심 Aggregate Root 객체를 Entity 로 선언하였다: (예시는 order 마이크로 서비스).


order.java

package yanolza;

import javax.persistence.*;
import org.springframework.beans.BeanUtils;
import java.util.List;
import java.util.Date;

@Entity
@Table(name="Order_table")
public class Order {

    @Id
    @GeneratedValue(strategy=GenerationType.AUTO)
    private Long id;
    private String name;
    private Long cardNo;
    private String status;

    @PostPersist
    public void onPostPersist(){
        Ordered ordered = new Ordered();
        BeanUtils.copyProperties(this, ordered);
        ordered.publishAfterCommit();

        //Following code causes dependency to external APIs
        // it is NOT A GOOD PRACTICE. instead, Event-Policy mapping is recommended.

        yanolza.external.PaymentHistory paymentHistory = new yanolza.external.PaymentHistory();
        // mappings goes here
        
        paymentHistory.setOrderId(this.id);
        
        OrderApplication.applicationContext.getBean(yanolza.external.PaymentHistoryService.class)
            .pay(paymentHistory);

    }
    @PostUpdate
    public void onPostUpdate(){
        OrderCanceled orderCanceled = new OrderCanceled();
        BeanUtils.copyProperties(this, orderCanceled);
        
        orderCanceled.setStatus("Cancled");
        
        orderCanceled.publishAfterCommit();

    }
    @PrePersist
    public void onPrePersist(){
    }

    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }
    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }
    public Long getCardNo() {
        return cardNo;
    }

    public void setCardNo(Long cardNo) {
        this.cardNo = cardNo;
    }
    public String getStatus() {
        return status;
    }

    public void setStatus(String status) {
        this.status = status;
    }




}



Entity Pattern 과 Repository Pattern 을 적용하여 JPA 를 통하여 다양한 데이터소스 유형 (RDB or NoSQL) 에 대한 별도의 처리가 없도록 
데이터 접근 어댑터를 자동 생성하기 위하여 Spring Data REST 의 RestRepository 를 적용하였다

yanolza\order\src\main\java\yanolza\OrderRepository.java

package yanolza;

import org.springframework.data.repository.PagingAndSortingRepository;
import org.springframework.data.rest.core.annotation.RepositoryRestResource;

@RepositoryRestResource(collectionResourceRel="orders", path="orders")
public interface OrderRepository extends PagingAndSortingRepository<Order, Long>{


}

적용 후 REST API 의 테스트

# 주문 상태 확인
http localhost:8081/orders/1

# 주문 요청
http POST http://localhost:8088/orders name="me" cardNo=123 status="Order Start"


폴리글랏 퍼시스턴스
예약(Reservation)은 서비스 특성상 많은 사용자의 유입과 상품 정보의 다양한 콘텐츠를 저장해야 하는 특징으로 인해
RDB 보다는 Document DB / NoSQL 계열의 데이터베이스인 Mongo DB 를 사용하기로 하였다. 
이를 위해 order 의 선언에는 @Entity 가 아닌 @Document 로 마킹되었으며,
별다른 작업없이 기존의 Entity Pattern 과 Repository Pattern 적용과 데이터베이스 제품의 설정 (application.yml) 만으로 MongoDB 에 부착시켰다


# 주문Repository.java
package fooddelivery;

public interface 주문Repository extends JpaRepository<Order, UUID>{
}

# yanolza\reservation\pom.xml 
    

    <dependency>
			<groupId>org.hsqldb</groupId>
			<artifactId>hsqldb</artifactId>
			<version>2.4.0</version>
			<scope>runtime</scope>
		</dependency>
    
    
    
    << 캡쳐 >> 
    
    
    
    
    동기식 호출 과 Fallback 처리
분석단계에서의 조건 중 하나로 주문(order)->결제(payment) 간의 호출은 동기식 일관성을 유지하는 트랜잭션으로 처리하기로 하였다.
호출 프로토콜은 이미 앞서 Rest Repository 에 의해 노출되어있는 REST 서비스를 FeignClient 를 이용하여 호출하도록 한다.

yanolza\order\src\main\java\yanolza\external\PaymentHistoryService.java

package yanolza.external;

@FeignClient(name="payment", url="http://localhost:8082")
public interface PaymentHistoryService {
    @RequestMapping(method= RequestMethod.POST, path="/paymentHistories")
    public void pay(@RequestBody PaymentHistory paymentHistory);

}






Fallback 처리
order와 payment의 Request/Response 구조에 Spring Hystrix를 사용하여 FallBack 기능 구현

[order > src > main > java > order > external > DeliveryService.java]에 configuration, fallback 옵션 추가
configuration 클래스 및 fallback 클래스 추가
[order > src > main > resources > application.yml]에 hystrix
[order > src > main > java > twdproject > external > DeliveryService.java]

