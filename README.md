# Outline
![image](https://github.com/2Jo-Logistics-Management/main/assets/111555414/6cf0cc9c-14c2-4752-ae6d-41f2d3f0e05a)
Spring AOP를 사용하여 위의 이미지와 유사한 구조로 DB 쿼리 성능에 대한 동작 시간 단위 별 테스트를 진행하고자 한다.

* **주 목적은 아래과 같음**

> 1. 시간이 오래 걸릴 가능성이 있는 SELECT문의 쿼리 성능을 테스트 후 개선
> 2. ++


# Main

먼저 Spring AOP를 사용하기 위해서는 의존성 주입이 필요하다.
```pom.xml
<dependencies>
	<!-- AOP 사용을 위한 의존성 -->
	<dependency>
		<groupId>org.springframework</groupId>
		<artifactId>spring-aop</artifactId>
		<version>5.3.28</version>
	</dependency>
	<!-- AspectJ 라이브러리 의존성 -->
	<dependency>
		<groupId>org.aspectj</groupId>
		<artifactId>aspectjrt</artifactId>
		<version>1.9.7</version>
	</dependency>
	<!-- AspectJ 라이브러리 의존성, 실행 시 클래스 로딩 역할 -->
	<dependency>
		<groupId>org.aspectj</groupId>
		<artifactId>aspectjweaver</artifactId>
		<version>1.9.7</version>
	</dependency>
</dependencies>
```
* aop 디렉토리 구조

![image](https://github.com/2Jo-Logistics-Management/main/assets/111555414/b744769f-43e2-42d4-b628-03d8930cded5)


* TimeTraceAspect

```java
@Slf4j
@Aspect
@Component
public class TimeTraceAspect {
    @Pointcut("@annotation(com.douzon.smartlogistics.global.common.aop.annotation.TimeTrace)")
    private void timer(){};

    @Around("timer()")
    public Object logTimeData(ProceedingJoinPoint joinPoint) throws Throwable {

        long startTime = System.currentTimeMillis();

        Object returnValue = joinPoint.proceed();

        MethodSignature signature = (MethodSignature) joinPoint.getSignature();
        String methodName = signature.getMethod().getName();

        long totalTimeMillis = System.currentTimeMillis() - startTime;

        log.info("실행 메서드: {}, 실행시간 = {}ms", methodName, totalTimeMillis);

        return returnValue;
    }
}
```


* TimeTrace

```java
@Target({ElementType.METHOD, ElementType.ANNOTATION_TYPE})
@Retention(RetentionPolicy.RUNTIME)
public @interface TimeTrace {
}
```

* 테스트할 위치 지정
```java
@Mapper
public interface ReceiveMapper {
    @TimeTrace
    List<ReceiveList> findReceive(
            @Param("receiveCode") String receiveCode,
            @Param("manager") String manager,
            @Param("itemCode") Integer itemCode,
            @Param("itemName") String itemName,
            @Param("accountNo") Integer accountNo,
            @Param("accountName") String accountName,
            @Param("startDate") String startDate,
            @Param("endDate") String endDate
    );
    @TimeTrace
    List<CmpPOrder> waitingReceive(
            @Param("porderCode") String porderCode,
            @Param("itemCode") Integer itemCode,
            @Param("itemName") String itemName,
            @Param("manager") String manager,
            @Param("accountNo") Integer accountNo,
            @Param("accountName") String accountName,
            @Param("startDate") String startDate,
            @Param("endDate") String endDate
    );
}
```

# Conclusion
## 1차
* 호출
```url
http://localhost:8888/api/receive/list
```
* 결과
```java
2023-08-01 13:53:50.776  INFO 22677 --- [nio-8888-exec-4] c.d.s.g.c.aop.advisor.TimeTraceAspect    : 실행 메서드: findReceive, 실행시간 = 21ms
```
## 2차
* 입고 테이블 기준 20만개 가량 추가한 후 테스트 재진행
* 20만개 추가 #64
* 30만개 LIMIT 으로 조회 테스트해본 결과
콘솔에서 쿼리 부분 체크 시간은 5초 가량 걸리고 리스트 다 불러오고 로딩 끝나는 시간 까지는 약 1분가량 걸림
&nbsp; * 쿼리 부분 콘솔 체크
&nbsp;![image](https://github.com/2Jo-Logistics-Management/main/assets/111555414/af2ecffe-7113-46ea-b0ea-e3b13a8e11ee)
&nbsp; * 페이지 로딩 완료 시간 체크 
&nbsp;<img width="150" alt="image" src="https://github.com/2Jo-Logistics-Management/main/assets/111555414/54d5e1de-5552-4387-9194-47ee60874e19">
* 56만개 이상으로 조회할 시 IOException pipe broken 발생됨
* 55만개 이하로 조회할 시 IOException pipe broken 발생하지 않음
![image](https://github.com/2Jo-Logistics-Management/main/assets/111555414/8294b760-f990-4f69-acb8-aea4f156b0ff)

### 쿼리 성능 개선방법
* 인덱스 사용하기
* 적절한 JOIN 사용하기
* 서브쿼리 최소화하기
* 쿼리 실행 계획 확인하기

### 개선 방법 중 인덱스 사용으로 쿼리 성능 테스트를 진행
#### 인덱스 사용한 쿼리 튜닝 시 참고할 점
* 가급적 WHERE 조건에서는 인덱스 컬럼을 모두 사용한다.
* 인덱스 컬럼에 사용하는 연산자는 가급적 동등 연산자를 사용한다. (** LIKE, IS NULL, IS NOT NULL, NOT IN 등이 사용될 경우 인덱스 효율이 떨어짐)
* 인덱스 컬럼은 변형하여 사용하지 않도록 한다.
* OR 보다는 AND를 사용한다.
* 그룹핑 쿼리를 사용할 경우 HAVING 보다는 WHERE 절에서 데이터를 필터링한다. (** HAVING 절은 이미 WHERE 절에서 처리된 로우들을 대상으로 조건을 조회하기 떄문에 좋은 성는 발휘가 힘듦)
* DISTINCT는 가급적 사용하지 않는다. (내부적으로 정렬 작업이 일어나기 때문)
* IN, NOT IN 대신 EXIST 와 NOT EXIST를 사용한다.

#### 인덱스 생성 전 쿼리 시간 (LIMIT 30만 기준)
![image](https://github.com/2Jo-Logistics-Management/main/assets/111555414/8e68f2db-be70-42ad-ae97-17bd30b8dcac)

#### 인덱스 생성 후 쿼리 시간
![DBF599B6-D577-4D06-969A-EEDACE35ED2F](https://github.com/2Jo-Logistics-Management/main/assets/111555414/4e9360cc-9611-4023-9288-77915a5a1e44)
위 그림과 같이 인덱스를 적용한 컬럼들을 조건 걸어서 약 21만개 가량의 데이터를 조회했으나 인덱스 적용 전과 큰 차이가 없었다. 다른 방법을 조사해야할 것으로 보임.
<img width="319" alt="image" src="https://github.com/2Jo-Logistics-Management/main/assets/111555414/50185d7e-c1c6-4b68-86f0-569cb93bbfb8">



### 추가 확인 및 개선할 부분

1. AOP에 대한 추가 학습 필요
2. _~~AOP 적용 시 결과가
&nbsp;&nbsp;{
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"success": true,
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"data": null,
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;"error": null
&nbsp;&nbsp;}
&nbsp;&nbsp;이렇게 반환되는 상황인데 aop 적용과 리스트 결과 반환이 서로 잘되도록 수정 필요~~_
&nbsp;&nbsp;-> 메서드 타입 Object로 변경. 'joinPoint.proceed()' 호출한 후 원래의 메서드를 실행하고 반환값을 'returnValue' 변수에 저장하도록 함.
이 후 해당 값을 리턴하도록 수정하여 해결.
3. _~~데이터가 그렇게 많지 않아 쿼리 시간 체크가 애매해서 데이터 추가 및 쿼리 수정 등을 통한 추가 테스트가 필요해보임~~_
&nbsp;-> shell script 사용해서 입고 테이블 기준 20만개 가량 추가하고 테스트 진행 중
4. _~~쿼리 동작 시간 개선 방법 조사~~_
5. 확인 기준 55만개까지는 괜찮았으나 56만개 부터는 'java.io.IOException: Broken pipe' 이 발생하여 처리 방법에 대한 조사도 필요해보임
6. 인덱스 사용했으나 크게 변화가 없어서 다른 방법 조사

# References
| [Reference] | [Link] |
| - | - |
| Outline Image | https://webcoding-start.tistory.com/3 |
| Settings | https://atoz-develop.tistory.com/entry/AspectJ-Weaver%EB%A5%BC-%EC%82%AC%EC%9A%A9%ED%95%9C-%EC%95%A0%EB%85%B8%ED%85%8C%EC%9D%B4%EC%85%98-%EA%B8%B0%EB%B0%98%EC%9D%98-%EC%8A%A4%ED%94%84%EB%A7%81-AOP-%EA%B5%AC%ED%98%84-%EB%B0%A9%EB%B2%95 |
| 간단한 AOP 적용예제 | https://velog.io/@dhk22?tag=aop |
| AOP를 적용하여 메서드 호출 속도 측정 | https://velog.io/@woply/Spring-AOP-%EA%B4%80%EC%A0%90%EC%9C%BC%EB%A1%9C-%EC%8B%A4%ED%96%89-%EC%86%8D%EB%8F%84-%EC%B8%A1%EC%A0%95%ED%95%98%EA%B8%B0 |
| Spring AOP 정리, 개념, 프록시 기반 AOP, @AOP | https://engkimbs.tistory.com/746 |
| Spring Boot AOP 적용해보기 | https://devjaewoo.tistory.com/87 |
| 특정상황에 스프링 AOP 적용 | https://hstory0208.tistory.com/entry/Spring-%ED%8A%B9%EC%A0%95%EC%83%81%ED%99%A9%EC%97%90-%EC%8A%A4%ED%94%84%EB%A7%81-AOP-%EC%A0%81%EC%9A%A9%ED%95%98%EA%B8%B0 |
| SQL 쿼리 튜닝하는 여러가지 방법 | https://chung-develop.tistory.com/145 |
| SQL 튜닝 (일반적인 튜닝방법) | https://cornswrold.tistory.com/87 |
| SQL 튜닝 방법 정리 | https://velog.io/@gillog/SQL-%ED%8A%9C%EB%8B%9D |
