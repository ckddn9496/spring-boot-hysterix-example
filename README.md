## Spring boot - Circuit Breaker 예제
Spring 가이드 Circuit Breaker를 공부하며 만든 프로젝트입니다.

### Circuit Breaker Pattern
MSA 패턴은 많은 서비스에 의존적이다. 예를들어, 100개의 api콜이 존재한다면 1개의 서비스는 쓰레드의 실패나 timeout과 같은 문제로 다른 프로세스를 위해 할당되지 못한다.
또한 이러한 문제는 brust하여 연쇄적으로 다른 쓰레드의 실패를 야기할 수 있다.
즉, 이러한 구조는 잠재적인 실패의 가능성을 가지고 있다. Netflix의 Hystrix는 MSA환경에서
실패에 대해 내성을 갖게하는 방법을 제공한다. CommandPattern을 이용하여 Command의 실패시 Client가
이용할 데이터를 미리 정의하여 이를 던져주는 방법을 취한다.

HystrixCommand를 상속하여 직접 Command를 구현하는 방법도 존재한다. 하지만, 이 프로젝트에서는 `@HystrixCommand`
어노테이션을 사용하여 서비스를 구성했다.

- 참고자료
    * [예제 프로젝트](https://spring.io/guides/gs/circuit-breaker/#initial)
    * [히스트릭스 깃허브](https://github.com/Netflix/Hystrix)
    

***

### <h1>프로젝트 구조
두개의 모듈로 구성되어 있으며 서비스를 제공해주는 `bookstore`, 서비스를 이용하는 클라이언트 `reading`으로 구성된다. 

- bookstore: Service
- reading: Client

***

### Service 구성
- `bookstore`는 `/recommended`에서 `String`을 리턴한다.
```java
@RestController
@SpringBootApplication
public class CircuitBreakerBookstoreApplication {

  @RequestMapping(value = "/recommended")
  public String readingList(){
  return "Spring in Action (Manning), Cloud Native Java (O'Reilly), Learning Spring Boot (Packt)";
  }

  public static void main(String[] args) {
  SpringApplication.run(CircuitBreakerBookstoreApplication.class, args);
  }
}
```

- 포트번호는 `8090`으로 설정
application.properties
> server.port=8090

***

### Client 구성
- `reading`은 `/to-read`에서 `bookService.readingList()`를 통해 Service인 `bookstore`를 사용하여 값을 가져온다.

```java
@EnableCircuitBreaker
@RestController
@SpringBootApplication
public class CircuitBreakerReadingApplication {

  @Autowired
  private BookService bookService;

  @Bean
  public RestTemplate rest(RestTemplateBuilder builder) {
  return builder.build();
  }

  @RequestMapping("/to-read")
  public String toRead() {
  return bookService.readingList();
  }

  public static void main(String[] args) {
  SpringApplication.run(CircuitBreakerReadingApplication.class, args);
  }
}
```

- 포트번호는 `8080`으로 설정
application.properties
> server.port=8080

***

### Client: Circuit Breaker Pattern

`@HystrixCommand`를 어노테이션을 적용하여 실패시 실행된 메서드를 지정해준다 (`fallbackMethod`).
```java
@Service
public class BookService {

  private final RestTemplate restTemplate;

  public BookService(RestTemplate rest) {
  this.restTemplate = rest;
  }

  @HystrixCommand(fallbackMethod = "reliable")
  public String readingList() {
  URI uri = URI.create("http://localhost:8090/recommended");

  return this.restTemplate.getForObject(uri, String.class);
  }

  public String reliable() {
  return "Cloud Native Java (O'Reilly)";
  }

}
```

***

### Test
`bookstore`와 `reading`, 두개의 모듈을 함께 실행한다.

브라우저를 통해 http://localhost:8080/to-read 에 접속해보자 
> Spring in Action (Manning), Cloud Native Java (O'Reilly), Learning Spring Boot (Packt)

이라는 값이 잘 나오는 것을 확인할 수 있다.

이제 `bookstore`를 종료하고 다시 접속해보자.
> Cloud Native Java (O'Reilly)

이라는 값이 나온다. `bookstore`서비스가 동작하지 않기 때문에 `@HystrixCommand(fallbackMethod="reliable)`에 따라 BookService의 `reliable`함수가 동작한것을 알 수 있다.
