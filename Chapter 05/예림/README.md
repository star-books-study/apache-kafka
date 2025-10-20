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
private void addHdfsFileBuffer(ConsumerRecord<String, String> record) { // 레코드를 받아서 메시지값을 버퍼에 넣는 코드. 
  List<String> buffer = bufferString.getOrDefault(record.partition(), new ArrayList<>());
  buffer.add(record.value());
  bufferString.put(record.partition(), buffer);

  if(buffer.size() == 1) // 만약 버퍼 크기가 1이라면 버퍼의 가장 처음 오프셋이라는 뜻이므로 currentFileOffset 변수에 넣는다. 이렇게 변수에 넣으면 추후파일을 저장할 때 파티션 이름과 오프셋 번호를 붙여서 저장할 수 있기 때문에 이슈 발생 시 파티션과 오프셋에 대한 정보를 알 수 있다.
    concurrentFileOffset.put(record.partition(), record.offset());
}

private void saveBufferToHdfsFile(Set<TopicParition> partition) {
  partition.forEach(p -> checkFlushCount(p.partition));
}

private void checkFlushCount(int partitionNo) {
  if(bufferString.get(partitionNo) != null) {
    if(bufferString.get(partitionNo).size() > FLUSH_RECORD_COUNT - 1) {
      save(partitionNo);
    }
  }
}

private void save(int partitionNo) { // 실질적인 HDFS 적재를 수행하는 메서드
  if (bufferString.get(partitionNo).size() > 0)
    try {
      String fileName = "/data/color-" + partitionNo + "-" + currentFileOffset.get(partitionNO) + ".log";
      Configuration configuration = new Configuration();
      configuration.set("fs.defaultFS", "hdfs://localhost:9000");
      FileSystem hdfsFileSystem = FileSystem.get(configuration);
      FSDataOutputStream fileOutputStream = hdfsFileSystem.create(new Path(fileName));
      fileOutputStream.writeBytes(StringUtils.join(bufferString.get(partitionNO), "\n"));
      fileOutputStream.close();

      bufferString.put(partitionNo, new ArrayList<>());
  } catch (Exception e) {
    logger.error(e.getMessage(), e);
  }
}     
```
- 마지막으로 안전한 컨슈머 스레드 종료를 위한 로직이다.
  ```java
  private void saveRemainBufferToHdfsFile() { // 버퍼에 남아있는 모든 데이터를 저장하기 위한 메서드이다. 컨슈머 스레드 종료 시에 호출된다.
    bufferString.forEach((partitionNo, v) -> this.save(partitionNo));
  }

  public void stopAndWakeup() { // 셧다운 훅이 발생했을 때 안전한 종료를 위해 consumer에 wakeup() 메서드를 호출한다. 남은 버퍼의 데이터를 모두 저장
    logger.info("stopAndWakeup");
    consumer.wakeup();
    saveRemainBufferToHdfsFile();
  }
  ```
<img width="555" height="242" alt="image" src="https://github.com/user-attachments/assets/034dadb2-fe71-4954-9ea4-b1b7de310291" />


#### 엘라스틱서치 싱크 커넥터 개발
- 토픽의 데이터를 엘라스틱서치에 적재하기 위해 커넥터 개발
```gradle
dependencies {
  compile 'com.google.code.gsongson:2.8.6'
  compile 'org.apache.kafkatconnect-api:2.5.0'
  compile 'org.slf4j:slf4j-simple:1.7.38'
  compile 'org.elasticsearch.client:elasticsearch-rest-high-level-client:7.9.2'
}
```
- `ElasticSearchSinkConnectorConfig` : 엘라스틱서치에 저장할 때 필요한 설정
  
  <img width="420" height="410" alt="image" src="https://github.com/user-attachments/assets/8ba8182d-ee01-41ed-9637-3d9dabb4f3d1" />

- `ElasticSearchSinkConnecor` : 커넥터를 생성했을 때 최초로 실행됨. SinkConnector 추상 클래스를 상속받은 는 총 6개의 메서드를 구현해야 함
  - 직접적으로 데이터 적재하는 로직이 포함되지는 않음
  - 태스크 클래스를 지정하는 역할 수행
    - version()
    - start()
    - taskClass()
    - taskConfigs()
    - config()
    - stop()
- ElasticSearchSinkTask : 실질적인 엘라스틱 서치 적재 로직이 들어감
<img width="533" height="217" alt="image" src="https://github.com/user-attachments/assets/38334ff8-508b-4e4e-9635-a9bed3ca0bf3" />

### 5.1.4 기능 테스트
- 개발한 코드들을 로컬 개발환경에서 실행하기 위해 총 5가지 단계 실행
  - REST 프로듀서 실행
  - 하둡 적재 컨슈머 실행
  - 분산 모드 카프카 커넥트 실행
  - 엘라스틱서치 싱크 커넥터 실행
  - 웹 페이지 실행 및 웹 이벤트 발생
    <img width="255" height="225" alt="image" src="https://github.com/user-attachments/assets/25efcf3c-330e-4bf0-83c1-ef0b276c42c7" />


### 5.1.5 상용 인프라 아키텍처
- 최소한으로 웹 이벤트 수집 파이프라인 인프라를 구축하여 안전하게 운영하고 싶다면 아래와 같이 구성 추천
  • L4 로드밸런서: 웹 이벤트를 받아서 프로듀서로 분배 역할
  • 프로듀서: 2개 이상의 서버, 각 서버당 1개 프로듀서
  • 카프카 클러스터: 3개 이상의 브로커로 구성
  • 컨슈머: 2개 이상의 서버, 각 서버당 1개 컨슈머
  • 커넥트: 2개 이상의 서버, 분산 모드 커넥트로 구성

    <img width="465" height="261" alt="스크린샷 2025-10-19 오후 10 08 23" src="https://github.com/user-attachments/assets/a4a7d430-dcef-4d6e-ad35-8db4cec2e0c1" />

## 5.2 서버 지표 수집 파이프라인 생성과 카프카 스트림즈 활용

- 서버 지표들(서버의 CPU, 메모리, 네트워크, 디스크 지표 등)을 카프카로 수집하는 데이터 파이프라인을 만들고 적재된 데이터를 실시간 처리하는 로직을 개발한다.
<img width="446" height="211" alt="스크린샷 2025-10-19 오후 10 09 29" src="https://github.com/user-attachments/assets/bfcdb996-c97d-4021-ab0b-f9a27cd4a3aa" />

### 5.2.1 요구 사항
- 컴퓨터의 서버 지표 중 CPU와 메모리 데이터를 수집해서 토픽으로 전송한다.
- 토픽에 전송된 전체 지표 데이터는 분기 처리하여 CPU와 메모리 토픽에 각각 저장한다.
- 로컬 컴퓨터 CPU 사용량이 50%가 넘을 경우에는 hostname과 timestamp 정보를 비정상 CPU 토픽으로 전송한다.
- 서버 지표 수집에는 메트릭비트(metricbeat) 사용
  - 서버 지표 수집에 특화된 경량 에이전트
  - 서버 지표 수집과 프로듀서 역할을 동시에 함
- 수집된 서버 지표 데이터를 실시간 처리하는 데는 카프카 스트림즈 사용
  - 별개의 클러스터가 필요하지 않고 독립된 자바 애플리케이션으로 동작
  - 
### 5.2.2 정책 및 기능 정의
#### 적재 정책
- 서버 지표 수집할 때 중요한 건 24시간 365일 끊임없이 수집이 가능해야 한다는 것
- 데이터를 적재할 때 지속적으로 데이털르 처리할 수 있으면서도 데이터 유실되거나 중복되는 것을 감안해서 파이프라인을 구성하자

#### 토픽
- 필요한 토픽
  - 전체 서버의 지표들을 저장하는 토픽
  - CPU 지표만 저장하는 토픽
  - 메모리 지표만 저장하는 토픽
  - 비정상 CPU 지표 정보를 저장하는 토픽
- 서버 지표를 처리하는 데는 데이터 처리 순서보다는 유연하고 처리량을 늘리는 게 더 중요하기 때문에 메시지키는 사용 X
- 파티션 크기는 3
  - 서버들 개수가 많아지면 파티션 크기 늘리면 됨
- 복제 개수는 2

#### 데이터 포맷
- JSON

#### 메트릭 비트
- 지표 수집 간격은 10초로 설정해 과도하게 수집되지 않도록 설정

#### 카프카 스트림즈
- 처리해야 할 부분 두 가지
**1. 전체 지표를 가진 토픽의 데이터를 CPU와 메모리 토픽으로 분기하는 로직**
  <img width="277" height="243" alt="image" src="https://github.com/user-attachments/assets/4b115daa-37d9-4046-bf8c-fa35bb1995fb" />

  - 지표 토픽 소스를 KStream으로 선언하고 `branch()` 메서드로 KStream 배열을 리턴받아서 데 이터를 분기처리할 수 있다. 메트릭비트가 보낸 JSON 데이터의 metricset,name값으로 보내 는 네이터의 종류를 구분할 수 있기 때문에 CPU인 경우와 memory인 경우를 `Predicate` 인터 페이스로 구현하여 분기를 선언할 수 있다.
**2. CPU 지표 중 전체 사용량이 50%가 넘는 경우에 대해 필터링하고 hostname과 timestamp 값으로 생성하는 로직**
    
<img width="359" height="109" alt="스크린샷 2025-10-19 오후 10 18 21" src="https://github.com/user-attachments/assets/ed42e042-70d8-458f-8bf6-17c84dd79703" />

- 분기처리로 받은 CPU토픽 KStream객체를 필터링하는 네는 `flter()` 메서드를 사용하고 메시지 값을 변환하는 데에는 `mapValues()` 메서드를 사용하면 된다.
- `filter()` 메서드의 파라미터로 Predicate 인터페이스를 받을 수 있는데, JSON 데이터의 전체 CPU 사용량의 50%가 넘는 경우 반환하도록 조건 처리를 추가한다.
- 비정상 CPU 토픽에 hostname과 timestamp 네이터 를 보내기 위해 JSON 데이터의 변환이 필요한데 메시지 값만 변환하면 되므로 `mapValues()` 메서드를 활용하면 CPU 지표 토픽의 데이터를 변환하여 다음 싱크로 보낼 수 있다.

**최종 토폴로지**
<img width="478" height="210" alt="스크린샷 2025-10-19 오후 10 19 05" src="https://github.com/user-attachments/assets/b6c52ae9-7e62-45e3-92ce-cf7ede601334" />

### 5.2.3 기능 구현
- 아키텍처는 다음과 같다
  <img width="506" height="205" alt="스크린샷 2025-10-19 오후 10 19 35" src="https://github.com/user-attachments/assets/e1260dfc-c1c8-49ba-8880-7916315883b5" />
#### 토픽 생성
- 생성할 토픽은 총 4개
- 토픽의 파티션은 3, 복제 개수는 2로 설정하되 나머지 토픽 설정들은 기본값으로 생성한다.
```bash
$ bin/kafka-topics.sh --create \
  --bootstrap-server my-kafka:9092 \
  --replicatioin-factor 2\
  --partiition 3 \
  --topic metric.all
```
#### CPU 지표를 저장하는 토픽 생성
```bash
§ bin/kafka-topics.sh --create \
--bootstrap-server my-kafka:9092 \
--replication-factor 2 \
--partitions 3 \
--topic metric.cpu
```

#### 메모리 지표를 저장하는 토픽 생성
```bash
§ bin/kafka-topics.sh --create \
--bootstrap-server my-kafka:9092 \
--replication-factor 2 \
--partitions 3 \
--topic metric.memory
```

#### 비정상 CPU 지표 정보를 저장하는 토픽 생성
```bash
§ bin/kafka-topics.sh --create \
--bootstrap-server my-kafka:9092 \
--replication-factor 2 \
--partitions 3 \
--topic metric.cpu.alert
```


#### 메트릭비트 설치 및 설정
- metricbeat 바이너리 파일과 동일한 디렉토리에 metricbeat.yml 파일을 생성하고 설정값 입력
#### 카프카 스트림즈 개발
```gradle
dependencies {
  compile group: 'org.apache.kafka', name: 'kafka-streams', version: '2.5.0'
  compile 'com.google.code.gsongson:2.8.0'
}
```

- MetricJsonUtils 클래스와 각 메서드를 작성해보자
  ```java
  public class MetricJsonUtils {
    public static double getTotalCpuPercent(String value) {
      return new JsonParser().parse(value).getAsJsonObject().get("system").getAsJsonObject().get("cpu").getAsJsonObject().get("total").getAsJsonObject().get("norm").getAsJsonObject().get("pct").getAsDouble();
  }

    public static String getMatricName(String value) {
      return new JsonParser().parse(value).getAsJsonObject9).get("matricset").getAsJsonObject().get("name").getAsString();
    }
  
    publc static String getHostTimeStamp(String value) {
      JsonObject objectiveValue = new JsonParser().parse(value).getAsJsonObject();
      JsonObject result = objectValue.getAsJsonObject("host");
      result.add("timestamp", objectiveValue.get("@timestamp"));
      return result.toString();
    }
  }
  ```
- 실질적인 토픽의 데이터 스트림 처리를 하는 MetricStream
  
  <img width="490" height="243" alt="스크린샷 2025-10-19 오후 10 29 01" src="https://github.com/user-attachments/assets/e193594e-8835-4564-aa66-556a52c78953" />
  ```java
  public class MetricStreams {
    private static KafkaStreams streams;

    public static void main(final String[] args) {
      Runtime.getRuntime().addShutdownHook(new ShutdownThread());
  
      ...
  
      KStream<String, String> metrics = builder.stream("metric.all");
      KStream<String, String>[] metricBranch = metric.branch((key, value) -> MetricJsonUtils.getMatricName(value).equals("cpu"), (key, value) -> MetricJsonUtils.getMatricName(value).equals("memory"));
  
      metricBranch[0].to("metric.cpu");
      metricBranch[1].to("metric.memory");
  
      KStrema<String, String> filteredCpuMetric = metricBranch[0].filter((key, value) -> MatricJsonUtils.getTotalCpuPercent(value) > 0.5);
  
      filteredCpuMetric.mapValue(value -> MetricJsonUtils.getHostTimestamp(value)).to("metric.cpu.alert");
  
      streams = new KafkaStream(builder.builder(), props);
      streams.start();
    }

    static class ShutdownThread extends Thread {
      public void run() {
        streams.close();
      }
    }
  }
  ```

  ### 5.2.4 기능 테스트
  - 개발된 코드들을 로컬 개발환경에서 실행하기 위해 2가지 단계를 실행한다.
    - 메트릭비트 실행
    - 스트림즈 애플리케이션 실행
      <img width="175" height="168" alt="스크린샷 2025-10-20 오후 6 17 56" src="https://github.com/user-attachments/assets/6e2cf7d0-7dad-4ed6-88b7-6ea0a2a13c2a" />
  - 로컬 환경에서 테스트를 진행하는 경우 애플리케이션들이 실행되는 모습은 다음과 같다.

#### 메트릭비트 실행
```bash
$ ./metricbeat -c metricbeat.yml
```
- 메트릭비트가 정상적으로 실행되면 metric.all 토픽에 10초마다 모든 지표 데이터가 들어오는 것을 확인할 수 있다.

#### 스트림즈 애플리케이션 실행

