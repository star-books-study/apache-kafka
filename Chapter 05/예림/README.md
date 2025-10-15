# Chapter 5. 카프카 실전 프로젝트
## 5.1 웹 페이지 이벤트 적재 파이프라인 생성
<img width="360" height="203" alt="image" src="https://github.com/user-attachments/assets/76be6856-43b9-460a-bb08-9e642c0b5073" />
- 이벤트 수집은 서비스에 영향을 미치지 않아야 한다.
- 여기서는 웹 페이지에서 생성되는 이벤트들을 분석하기 위해 HDFS와 엘라스틱서치에 적재하는 파이프라인을 만드는 프로젝트를 진행한다.


### 5.1.1 요구사항
- 이름을 입력하고 좋아하는 색상 고르면 유저 에이전트 정보를 카프카 토픽으로 전달하고 하둡과 엘라스틱서치에 적재
- `하둡`은 대용량 데이터를 분석 처리 / `HDFS`는 대용량 파일을 하둡에 안정적으로 저장할 수 있게 하는 파일 시스템

### 5.1.2 정책 및 기능 정의
#### 적재 정책
- 적재 파이프라인을 만들 때 가장 처음 결정해야 하는 게 적재 정책
- 파이프라인의 운영 난이도는 정책에 따라 달라진다.
- 0.11.0.0 이후 버전부터는 멱등성 프로듀서를 통해 **정확히 한번 전달**을 지원한다.
  
- 전달 : 프로듀서부터 브로커까지 전달되는 것
- 적재 : 프로듀서부터 컨슈머를 넘어 최종적으로 하둡이나 엘라스틱서치까지 데이터가 저장되는 것
- 컨슈머의 커밋 시점과 데이터 적재가 동일 트랜잭션에서 처리되어야 정확히 한 번 적재될 수 있긴 함
  - HDFS 적재, S3 적재 컨슈머 애플리케이션에서는 컨슈머의 커밋과 저장이 동일 트랜잭션으로 처리하는 것이 불가능하다
  - 그렇기 때문에 HDFS 적재, S3 적재 컨슈머 애플리케이션은 컨슈머의 장애가 발생하면 정확히 한번 적재하지 못한다.
- 정확히 한번 적재가 필요한 경우 멱등성 프로듀서 사용하고 고유한 키를 지원하는 DBMS 사용하는 게 가장 확실

- 웹 페이지 이벤트 수집은 유실 발생 가능 & 중복 발생 가능
- 따라서 파이프라인은 다음과 같이 정리된다.
  - 일부 데이터의 유실 또는 중복 허용
  - 안정적으로 끊임없는 적재
  - 갑작스럽게 발생하는 많은 데이터양을 허용

#### 데이터 포맷
- 데이터 파이프라인에서 데이터를 담는 용도로 사용되는 데이터 포맷은 매우 다양한 선택지가 있다.
-  VO(Value Object) 형태로 객체를 선언하여 직렬화하여 전송하는 방법
  - 보편적이지만 프로듀서와 컨슈머에서 동일한 버전의 VO 객체 선언 필요
  - 스키마가 변경될 경우 프로듀서와 컨슈머 둘 다 소스코드 업데이트가 필요하므로 비용이 크다.
- 데이터 포맷 선택할 때 생각해볼 수 있는 것
  - 스키마의 변화의 유연성
  - 명령어를 통한 디버깅의 편리성
- JSON 선택
#### 웹페이지
- html 사용

#### 프로듀서
- 웹 페이지에서 생성된 이벤트를 받는 REST API 클라이언트를 만들고 전달받은 이벤트를 가공하여 토픽으로 전달하는 역할을 한다.
- REST API는 스프링 부트 + RestController 사용해 개발
- RestController로 받은 데이터를 토픽으로 전달할 때는 스프링 카프카 라이브러리 사용
➡️ RestController로 REST API를 받는 메서드를 만들고 스프링 카프카의 KafkaTemplate로 프로듀서를 구현하여 데이터를 전송한다.

- **acks는 어떤 값으로 설정할 것인가?**
  - all : 클러스터 또는 네트워크에 이상이 생겼을 경우 복구 확률이 가장 높지만 데이터 저장하는데 오래 걸림
  - 1 또는 0 : 속도는 빠르지만 데이터 유실 발생 가능
-> 우리는 유실이 발생하더라도 안정적이고 빠른 파이프라인이 우선이므로 1로 설정

- **최소 동기화 리플리카는 어떻게 설정할 것인가?**
  - acks 1로 설정 시 min.insync.replicas 설정을 무시하고 리더 파티션에 지속 적재하므로 따로 설정할 필요가 없다.
 
- 재시도 설정
  - 토픽의 데이터 순서를 지키지 않고 데이터의 중복을 허용하기 때문에 retries 옵션은 따로 설정하지 않고 기본값을 사용한다.

- 프로듀서의 압축 옵션
  - gzip, snappy, lz4, astd
- 나머지 프로듀서 옵션들은 기본 옵션값으로 사용한다.

#### 토픽
- 데이터를 순서대로 적재하지 않더라도 하둡과 엘라스틱서치에서 데이터를 조회할 때 시간순 조회 가능
- **이벤트가 발생할 때 이벤트 발생 시간을 데이터에 같이 조합해서 보내면 됨**
- 여기서는 데이터 처리 순서를 지키지 않아도 되므로 파티션 개수를 엄격히 정해서 가져가지 않아도 됨 -> 파티션 개수 2개

#### 컨슈머
- 토픽에 저장되어 있는 웹 이벤트를 하둡과 엘라스틱서치에 저장하는 로직을 만드는 방법은 2가지

  
  1. 컨슈머API를 사용하여 직접 애플리케이션을 개발
  3. 커넥트 사용
 
### 5.1.3 기능 구현
    
- 아키텍처
  <img width="453" height="139" alt="스크린샷 2025-09-30 오후 9 44 07" src="https://github.com/user-attachments/assets/1d479523-5567-44db-bc2d-328e97e2a41b" />

- 필요한 작업 리스트
  - 로컬 하둡, 엘라스틱서치, 키바나 설치
  - 토픽 생성
  - 이벤트 수집 웹 페이지 개발
  - REST API 프로듀서 애플리케이션 개발
  - 하둡 적재 컨슈머 애플리케이션 개발
  - 엘라스틱서치 싱크 커넥터 개발

#### 로컬 하둡, 엘라스틱서치, 키바나 설치
```
brew install hadoop elasticsearch kibana
```
- fs.defaultFS 값은 하둡 경로의 core-site.xml에 다음과 같이 옵션을 넣으면 된다.
  ```
  <configuration>
    «property>
      <name›fs.defaultFS</name>
      <value>hdfs://localhost: 9000</value>
    </property>
  </configuration>
  ```

#### 토픽 생성
- 웹 페이지 이벤트 수집 기능 정의에 따라 kafka-topics 명령으로 토픽 옵션을 조절하여 생성한다.
- 복제 개수 2, 파티션 개수 3, 기타 옵션은 기본값
```
§ bin/kafka-topics.sh --create
-bootstrap-server my-kafka:9092 \
--replication-factor 2 \
--partitions 3 \
--topic select-color
```


#### 웹페이지 개발
- html과 jQuery를 이용해 아래와 같은 화면 개발 (생략)
<img width="321" height="200" alt="스크린샷 2025-10-02 오후 11 51 02" src="https://github.com/user-attachments/assets/207d6cef-4d95-47e7-a0c3-8ecd0fba157f" />

#### REST API 프로듀서 개발
- 스프리부트와 스프링 카프카 기반으로 개발
- 필요한 라이브러리
  ```gradle
  dependencies {
    compile 'org.springframework.kafka:spring-kafka:2.5.10.RELEASE'
    compile 'org.springframework.boot:spring-boot-starter-web:2.3.4.RELEASE
    compile 'com. google. code. gson:gson:2.8.6'
  }
  ```
- application.yml
  ```yml
  spring:
    kafka:
      bootstrap-servers: my-kafka:9092
      producer:
        acks: 1
        key-serialier: org.apache.kafka.common.serialization.StringSerializer
        value-serializer: org.kafka.common.serialization.StringSerialier
  ```
- SpringApiProducer 클래스 (@SpringBootApplication 어노테이션이 포함된 실행 클래스
  ```java
  @SpringBootApplication
  public class RestApiProducer {
    public static void main(String[] args) {
      SpringApplication.run(RestApiProducer.class, args);
    }
  }
  ```
- VO 형태로 클래스 작성 (UserEventVO)
  ```java
  public class UserEventVO {
    public UserEventVO(String timestamp, String userAgent, String colorName, String userName) {
      this.timestamp = timestamp;
      this.userAgent = userAgent;
      this.colorName = colorName;
      this.userName = userName;
    }
    private String timestamp;
    private String userAgent;
    private String colorName;
    private String userName;
  }
  ```
- 컨트롤러 클래스 (ProductController)
  ```java
  @RestController
  @CrossOrigin(origins = "*", allowHeaders = "*")
  public class ProduceController {
    private final Logger logger = LoggerFactory.getLogger(ProduceController.class);

    private final KafkaTemplate<String, String> kafkaTemplate);

    public ProduceController(KafkaTemplate<String, String> kafkaTemplate) {
      this.kakaTemplate = kafkaTemplate;
    }

    @GetMapping("/api/select")
    public void selectColor{
      @RequestHeader("user-agent") String userAgentName,
      @RequestParam(value = "color") String colorName,
      @RequestParam(value = "user") String userName) {
      SimpleDateFormat sdfDate = new SimpleDateFormat("yyyy-MM-dd'T'HH:mm:ss.SSSZZ");
      Date now = new Date();
      Gson gson = new Gson();
      UserEventVO userEventVO = new UserEventVO(sdfDate.format(now), userAgentName, colorName, userName);
      String jsonColorLog = gson.toJson(userEventVO);
      kafkaTemplate.send("select-color", jsonColorLog).addCallback
   (new ListenableFutureCallback<SendResult<String, String>>() { // select-color 토픽에 데이터를 전송한다. 메시지 키를 지정하지 않으므로 send() 메서드에는 토픽과 메시지 값만 넣으면 된다.

        // 전송 성공, 실패 내역을 로그로 남긴다
        @Override
        public void onSuccess(SendResult<String, String> result) {
          logger.info(result.toString());
        }

        @Override
        public void onFalilure(Throwable ex) {
          logger.error(ex.getMessage(), ex);
        }
      });
    }
  }
  ```
#### 하둡 적재 컨슈머 애플리케이션 개발
- 컨슈머 애프리케이션 개발 전 어떤 스레드 전략으로 데이터를 적재할지 정하고 시작해야 함
- 3개 파티션으로 이루어진 select-color 토픽에 할당되는 컨슈머의 최대 개수는 3개임
- 스레드 개수는 변수로 선언 (토픽의 데이터가 급속도로 늘어나 파티션을 늘리더라도 컨슈머의 스레드 개수를 효과적으로 늘릴 수 있도록)
  <img width="378" height="149" alt="스크린샷 2025-10-07 오후 9 55 23" src="https://github.com/user-attachments/assets/682d2425-d813-4222-a752-73fa147c4bd1" />
- 하둡과 연동하기 위해 필요한 라이브러리
  ```gradle
  dependencies {
    compile 'org.slf4j:slf4j-simple:1.7.30
    compile 'org.apache. kafka:kafka-clients:2.5.0 compile 'org.apache.hadoop: hadoop-client:3.3.0'
  }
  ```
- 메인 스레드 (컨슈머 스레드 실행)
  ```java
  public class HdfsSinkApplication {

    private final static String TOPIC_NAME = "select-color"; // 생성할 스레드 개수를 변수로 선언
  
    // 생략
  
    public static void main(String[] args) {
      Runtime.getRuntime().addShutdownHook (new ShutdownThread()); // 안전한 컨슈머의 종료를 위해 셧다운 훅 선언
      Properties configs = new Properties(); // 컨슈머의 설정 선언
      configs.put (ConsumerCon fig. BOOTSTRAP_SERVERS_CONFIG, BOOTSTRAP_SERVERS); // 
      configs.put(ConsumerCon fig.GROUP_ID_CONFIG, GROUP_ID);
      configs.put (ConsumerCon fig.KEY_DESERIALIZER_CLASS_CONFIG,
      StringDeserializer.class.getName ());
      configs.put (ConsumerConfig. VALUE_DESERIALIZER_CLASS_CONFIG,
      StringDeserializer.class.getName ());

      ExecutorService executorService = Executors. newCachedThreadPool(); // 컨슈머 스레드를 스레드 풀로 관리하기 위해 newCachedThreadPool() 선언
      for (int i = 0; 1 < CONSUMER_COUNT; i++) {
        workers.add(new ConsumerWorker (configs, TOPIC_NAME, i)); // 컨슈머 스레드 CONSUMER_COUNT 변수 값만큼(여기서는 3) 생성한다. 생성된 컨슈머 스레드 인스턴스들을 묶음으로 관리하기 위해 List<ConsumerWorker>로 선언된 workers 변수에 추가한다
        workers. forEach (executor Service: execute); // 컨슈머 스레드 인스턴스들을 스레드 풀에 포함시켜 실행
      }
    }
    
    static class ShutdownThread extends Thread {
      public void run() {
        Logger. info("Shutdown hook");
        workers.forEach (ConsumerWorker: : stopAndWakeup); 셧다운 훅이 발생했을 경우 각 컨슈머 스레드에 종료를 알리도록 명시적으로 stopAndWakeup() 메서드 호출
      }
    }
  }
  ```
- ConsumerWorker는 HdfSinkApplication에서 전달받은 Properties 인스턴스로 컨슈머를 생성, 실행하는 스레드 클래스
- 토픽에서 데이터를 받아 HDFS에 메시지 값들을 저장
- 데이터를 저장하는 방식 2가지
  - append - 파일이 없으면 파일 생성하고 poll()로 받은 데이터를 파일에 추가한다.
    - 이 방식은 파일 개수를 늘리지 않고 계속해서 데이터를 추가할 수 있다는 장점이 있다
  - flush - 버퍼에 일정 기간 데이터를 쌓고 일정 수준(시간 또는 개수)이 되면 데이터를 저장하는 방식
- 어떤 방식으로 데이터를 적재하느냐는 데이털르 최종 적재하는 타깃 애플리케이션의 기능 지원 여부에 따라 달라진다.
- 컨슈머 멀티 스레드 환경은 동일 데이터의 동시 접근에 유의해야 한다. 여러 개의 컨슈머가 동일한 HDFS 파일에 접근을 시도한다면 교착 상태에 빠질 수 있는 위험이 있기 때문이다.
- 이를 해결하기 위한 방법
  - 가장 간단하고 명확한 방법 - 파티션 번호에 따라 HDFS 파일 따로 저장
    <img width="588" height="218" alt="image" src="https://github.com/user-attachments/assets/b4aa4188-cafc-4feb-8024-2989269a1a9c" />
  - 각 스레드가 각 파티션 이름의 파일에 적재
    <img width="658" height="218" alt="image" src="https://github.com/user-attachments/assets/313f2606-6ea3-4480-b4cb-6c3c2b65eed6" />
- 지금까지 정리한 로직을 바탕으로 멀티 스레드로 동작하는 HDFS 적재 컨슈머 코드를 작성한다.
- 크게 세 부분으로 나뉨
  - poll() 메서드 호출
  - HDFS 적재 로직
  - 셧다운 로직
```java
public class ConsumerWorker implements Runnable {
  private final Logger logger = LoggerFactory.getLogger(ConsumerWorker.class);
  private static Map<Integer, List<String>> bufferString = new ConcurrentHashMap<>();
  private static Map<Integer, Long> currentFileOffset = new ConcurrentHashMap<>();
  ConcurrentHashMap<>();

  private final static int FLUSH_REcORD_COUNT = 10;
  private Properteis prop;
  private String topic;
  private String threadName;
  private KafkaConsumer<String, String> consumer;

  public ConsumerWorker(Properties prop, String topic, int number) {
    logger.info("Generate ConsumerWorker");
    this.prop = prop;
    this.topic = topic;
    this.threadName = "consumer-thread-" + number;
  }

  @Override
  public void run() {
    Thread.currentThread().setName(threadName);
    consumer = new KafkaConsumer<>(prop);
    consumer.subscribe(Arrays.asList(topic));
    try {
      while (true) {
        ConsumerRecords<String, String> records = consumer.poll(Duration.ofSeconds(1));

        for (ConsuerRecord<String, String> record : records) {
          addHdfsFileBuffer(record);
        }
        saveBufferToHdfsFile(consumer.assignment());
    }
  } catch (WakeupException e) {
      logger.warn("Wakeup consumer");
  } catch (Exception e) {
      logger.error(e.getMessage(), e);
  } finally {
    consumer.close();
  }
  ...
}
```
- 다음은 HDFS 적재를 위한 로직이다.
```java
private void addHdfsFileBuffer(ConsumerRecord<String, String> record) {
  List<String> buffer = bufferString.getOrDefault(record.partition(), new ArrayList<>());
  buffer.add(record.value());
  bufferString.put(record.partition(), buffer);

  if(buffer.size() == 1)
    concurrentFileOffset.put(record.partition(), record.offset());
}

private void saveBufferToHdfsFile(Set<TopicParition> partition) {
  partition.forEach(p -> checkFlushCount(p.partition));
}

privagte void checkFlushCount(int partitionNo) {
  if(bufferString.get(partitionNo) != null) {
    if(bufferString.get(partitionNo).size() > FLUSH_RECORD_COUNT - 1) {
      save(partitionNo);
    }
  }
}
```



